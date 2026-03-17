# IronClaw 架构决策记录 (ADR)

> 关键设计决策及其理由

---

## ADR-001: 双数据库后端架构

**状态**: 已接受
**日期**: 2024-11
**决策者**: NEAR AI Team

### 背景

个人 AI 助手需要支持两种使用场景：
1. **云端/服务器部署** - 需要 PostgreSQL 的完整功能
2. **本地/边缘部署** - 需要嵌入式数据库的轻量特性

### 决策

采用 **PostgreSQL + libSQL/Turso** 双后端架构，通过 trait 统一抽象。

### 理由

| 方案 | 优点 | 缺点 |
|------|------|------|
| 仅 PostgreSQL | 功能完整 | 边缘部署困难 |
| 仅 SQLite | 轻量 | 无向量搜索 |
| **双后端** | 兼顾两者 | 实现复杂度高 |

- PostgreSQL 提供 pgvector 扩展用于向量搜索
- libSQL 提供 Turso 同步支持边缘场景
- 通过 `Database` trait 隐藏后端差异

### 后果

- 所有新持久化功能必须支持双后端
- 迁移脚本需维护两个版本
- 查询需使用通用 SQL 子集

### 代码位置

- `src/db/mod.rs` - Database trait
- `src/db/postgres/` - PostgreSQL 实现
- `src/db/libsql/` - libSQL 实现

---

## ADR-002: WASM 沙箱执行

**状态**: 已接受
**日期**: 2024-10
**决策者**: NEAR AI Team

### 背景

第三方工具需要安全执行环境，防止恶意代码危害主机。

### 决策

使用 **Wasmtime** 作为 WASM 运行时，实现能力安全 (Capability-based security)。

### 理由

- **启动快**: ms 级 vs Docker 的秒级
- **资源可控**: Fuel 计量、内存限制原生支持
- **能力模型**: 显式授权 HTTP、密钥、工具调用
- **组件模型**: 支持 WIT 接口定义

### 拒绝的方案

| 方案 | 拒绝理由 |
|------|---------|
| V8 Isolate | 内存占用大，资源控制复杂 |
| Docker 单独 | 启动慢，不适合高频调用 |
| 原生插件 | 无隔离，安全风险高 |

### 后果

- 工具开发者需用 Rust/C 编写 WASM
- 需维护 WIT 接口定义
- Host 函数提供受限 API

### 代码位置

- `src/tools/wasm/` - WASM 运行时
- `src/tools/wasm/limits.rs` - 资源限制
- `src/tools/wasm/allowlist.rs` - 网络白名单

---

## ADR-003: 共享 Agentic 循环引擎

**状态**: 已接受
**日期**: 2024-12
**决策者**: NEAR AI Team

### 背景

系统有 3 种执行模式：
- 交互式聊天 (Chat)
- 后台任务 (Job)
- 容器内执行 (Container)

每种模式最初有独立循环，导致代码重复。

### 决策

提取 **通用 Agentic 循环引擎** (`run_agentic_loop`)，通过 `LoopDelegate` trait 适配不同模式。

### 理由

**Before**: 3 个独立循环 → 每处处理信号检查、超时、重试、日志 → ~450 行 × 3

**After**: 1 个通用引擎 + 3 个轻量 delegate → ~150 行 + ~50 行 × 3

```rust
pub async fn run_agentic_loop<D: LoopDelegate>(
    delegate: &mut D,
    reasoning: ReasoningConfig,
    reason_ctx: ReasonContext,
    config: &AgentConfig,
) -> Result<LoopOutcome, AgentError>
```

### 后果

- 核心逻辑修改只需改一处
- 新增执行模式只需实现 Delegate
- Delegate 接口需保持稳定

### 代码位置

- `src/agent/agentic_loop.rs` - 通用引擎
- `src/agent/chat.rs` - ChatDelegate
- `src/agent/job.rs` - JobDelegate
- `src/agent/container.rs` - ContainerDelegate

---

## ADR-004: 纵深防御安全模型

**状态**: 已接受
**日期**: 2024-10
**决策者**: NEAR AI Team

### 背景

AI 助手执行用户提供的工具调用，面临提示注入、密钥泄露、恶意代码等风险。

### 决策

实现 **4 层防御体系**，每层独立可替换。

```
WASM ──► Allowlist ──► Leak Scan ──► Credential ──► Execute ──► Leak Scan ──► WASM
         Validator     (request)     Injector       Request     (response)
```

### 各层职责

| 层级 | 组件 | 防御目标 |
|------|------|---------|
| Layer 1 | WASM 沙箱 | 恶意代码隔离 |
| Layer 2 | 网络白名单 | 未授权外联 |
| Layer 3 | 泄露检测 | 密钥外泄 |
| Layer 4 | Safety Layer | 提示注入 |

### 理由

- 单点失败不会导致安全崩溃
- 每层可独立升级/替换
- 符合纵深防御原则

### 代码位置

- `src/safety/` - Safety Layer
- `src/safety/leak_detector.rs` - 泄露检测
- `src/tools/wasm/allowlist.rs` - 网络白名单
- `src/sandbox/proxy/` - 网络代理

---

## ADR-005: 数据库 Trait 子分解

**状态**: 已接受
**日期**: 2024-11
**决策者**: NEAR AI Team

### 背景

`Database` trait 最初包含所有存储操作，消费者传递 `Arc<dyn Database>` 过于宽泛。

### 决策

将 `Database` 拆分为 **7 个专注子 trait**。

```rust
#[async_trait]
pub trait Database:
    ConversationStore
    + JobStore
    + SandboxStore
    + RoutineStore
    + ToolFailureStore
    + SettingsStore
    + WorkspaceStore
    + Send
    + Sync
```

### 子 Trait 列表

| Trait | 职责 | 典型消费者 |
|-------|------|----------|
| `ConversationStore` | 对话存储 | Agent |
| `JobStore` | 任务存储 | Scheduler |
| `SandboxStore` | 沙箱存储 | SandboxManager |
| `RoutineStore` | 例行任务 | Heartbeat |
| `ToolFailureStore` | 失败记录 | ToolRegistry |
| `SettingsStore` | 用户设置 | Config |
| `WorkspaceStore` | 工作空间 | MemoryStore |

### 理由

- 消费者可按需依赖最窄接口
- 避免传递 `Arc<dyn Database>` 到只需要读 settings 的函数
- 便于单独测试/mock

### 代码位置

- `src/db/mod.rs` - Trait 定义
- `src/db/stores/` - 各 trait 实现

---

## ADR-006: MCP 协议集成

**状态**: 已接受
**日期**: 2024-12
**决策者**: NEAR AI Team

### 背景

需要与外部工具服务（如 GitHub、Slack API）集成。

### 决策

实现 **Model Context Protocol (MCP)** 客户端，支持 HTTP/SSE/Unix 传输。

### 理由

- 标准化工具调用协议
- 社区生态支持
- 与 Anthropic 生态兼容
- 支持动态工具发现

### 限制

- 暂不支持流式响应
- stdio/HTTP/Unix 均使用 request-response

### 代码位置

- `src/tools/mcp/` - MCP 实现
- `src/tools/mcp/client.rs` - MCP 客户端
- `src/tools/mcp/protocol.rs` - JSON-RPC 协议

---

## ADR-007: LLM Provider 装饰器链

**状态**: 已接受
**日期**: 2024-11
**决策者**: NEAR AI Team

### 背景

LLM 调用需要通用能力：重试、降级、缓存。

### 决策

使用 **装饰器模式** 包装 `LlmProvider` trait。

```
CachingLlmProvider ──► RetryLlmProvider ──► FallbackLlmProvider ──► RealProvider
```

### 装饰器列表

| 装饰器 | 功能 |
|--------|------|
| `RetryLlmProvider` | 指数退避重试 |
| `FallbackLlmProvider` | 主提供商失败时切换 |
| `CachingLlmProvider` | 相同请求缓存响应 |

### 理由

- 职责分离
- 可自由组合
- 不影响具体提供商实现

### 代码位置

- `src/llm/decorators/` - 装饰器实现
- `src/llm/provider.rs` - LlmProvider trait

---

## ADR-008: 持久化混合搜索

**状态**: 已接受
**日期**: 2024-11
**决策者**: NEAR AI Team

### 背景

记忆系统需要同时支持：
- 关键词搜索 (精确匹配)
- 语义搜索 (向量相似)

### 决策

实现 **RRF (Reciprocal Rank Fusion)** 混合搜索。

```rust
// FTS 结果 + Vector 结果 → RRF 融合 → 排序结果
pub fn hybrid_search(
    fts_results: Vec<FtsResult>,
    vector_results: Vec<VectorResult>,
    k: usize, // RRF 常数，默认 60
) -> Vec<HybridResult>
```

### 理由

- FTS 擅长精确关键词
- 向量擅长语义相似
- RRF 简单有效，无需训练

### 后端实现

| 后端 | FTS | Vector |
|------|-----|--------|
| PostgreSQL | native + trigram | pgvector |
| libSQL | native FTS5 | native 向量 |

### 代码位置

- `src/workspace/search.rs` - 混合搜索
- `src/db/stores/workspace.rs` - 存储实现

---

## ADR-009: Docker 沙箱编排

**状态**: 已接受
**日期**: 2024-10
**决策者**: NEAR AI Team

### 背景

WASM 适合轻量工具，但某些任务需要完整 Linux 环境。

### 决策

实现 **Docker 沙箱** 作为 WASM 的补充。

### 架构

```
Orchestrator (主进程)
    ├── Worker Container (长期运行)
    │     └── Claude Code Bridge
    └── Job Containers (短期任务)
            └── 具体工具执行
```

### 理由

- WASM 无法运行编译型语言工具
- 某些工具需要 apt 安装依赖
- Docker 提供完整隔离

### 安全

- 网络代理强制域名白名单
- 凭证在主机边界注入
- 只读/可写/完全访问三级策略

### 代码位置

- `src/sandbox/` - Docker 沙箱
- `src/orchestrator/` - 编排器 API
- `src/worker/` - 容器内执行

---

## ADR-010: 技能系统 (SKILL.md)

**状态**: 已接受
**日期**: 2024-12
**决策者**: NEAR AI Team

### 背景

需要允许用户自定义 Agent 行为，无需修改代码。

### 决策

实现 **SKILL.md** 提示扩展系统。

### 设计

```markdown
---
name: code-review
tags: [rust, review]
triggers: [*.rs]
---

# 代码审查技能

审查 Rust 代码时...
```

### 选择管道

1. **Gating**: 检查 bin/env/config 要求
2. **Scoring**: 关键词/模式/标签匹配
3. **Budget**: 适配 `SKILLS_MAX_TOKENS`
4. **Attenuation**: 信任级别决定工具上限

### 信任模型

| 来源 | 信任级别 | 能力 |
|------|---------|------|
| 用户放置 | Trusted | 全工具访问 |
| 注册表安装 | Installed | 只读工具 |

### 代码位置

- `.claude/rules/skills.md` - 技能规范
- `src/skills/` - 技能系统实现

---

## ADR-011: 多通道架构

**状态**: 已接受
**日期**: 2024-10
**决策者**: NEAR AI Team

### 背景

用户希望从不同入口与 AI 交互：终端、Web、即时通讯。

### 决策

实现 **统一 Channel trait**，所有通道实现相同接口。

```rust
trait Channel: Send + Sync {
    async fn start(&self) -> Result<(), ChannelError>;
    async fn send(&self, response: OutgoingResponse) -> Result<(), ChannelError>;
    fn incoming(&self) -> Receiver<IncomingMessage>;
}
```

### 通道类型

| 通道 | 实现 | 用途 |
|------|------|------|
| REPL | `cli/` | 本地 TUI |
| HTTP | `http.rs` | Webhook |
| Web Gateway | `web/` | 浏览器 UI |
| WASM | `wasm/` | Telegram, Discord 等 |

### 理由

- 新增通道不影响核心逻辑
- `ChannelManager` 统一合并流
- 通道间正交独立

### 代码位置

- `src/channels/channel.rs` - Channel trait
- `src/channels/manager.rs` - 通道管理器
- `src/channels/wasm/` - WASM 通道运行时

---

## ADR-012: 密钥主机边界注入

**状态**: 已接受
**日期**: 2024-10
**决策者**: NEAR AI Team

### 背景

WASM 工具需要 API 密钥，但密钥不能存在于 WASM 模块中。

### 决策

**主机边界注入**: 在请求离开主机前注入凭证。

```
WASM Tool ──► 请求 ──► [主机边界] ──► 注入凭证 ──► 外部 API
```

### 理由

- WASM 不接触明文密钥
- 密钥作用域可控
- 泄露检测在注入前扫描

### 存储

- AES-256-GCM 加密
- 主密钥存储在系统密钥链
- `secrecy` crate 防止意外打印

### 代码位置

- `src/secrets/` - 密钥管理
- `src/tools/wasm/credential_injector.rs` - 注入实现
- `src/sandbox/proxy/credential.rs` - 代理层注入

---

## ADR-013: 心跳与自主执行

**状态**: 已接受
**日期**: 2024-12
**决策者**: NEAR AI Team

### 背景

传统 AI 助手被动响应，需要实现主动执行能力。

### 决策

实现 **Heartbeat 系统**：定期读取 `HEARTBEAT.md` 并执行检查。

### 流程

```
定时触发 (默认 30min)
    ↓
读取 ~/.ironclaw/workspace/HEARTBEAT.md
    ↓
执行心跳 Agent (自主决策)
    ↓
如有发现 → 通过 Channel 通知用户
```

### 理由

- 无需修改代码即可定义周期性任务
- 与 Agent 核心共享同一套工具
- 保持用户控制（通过 markdown 配置）

### 代码位置

- `src/agent/heartbeat.rs` - 心跳实现
- `HEARTBEAT.md` - 用户配置模板

---

## ADR-014: 零 unwrap/expect 政策

**状态**: 已接受
**日期**: 2024-10
**决策者**: NEAR AI Team

### 背景

Rust 的 `unwrap()`/`expect()` 会导致 panic，生产环境不可接受。

### 决策

**强制零 panic 政策**：生产代码禁用 `unwrap`/`expect`。

### 执行机制

```bash
# pre-commit 检查
grep -rnE '\.unwrap\(|\.expect\(' src/ || exit 0
```

### 正确模式

```rust
// ❌ 禁止
let db = LibSqlBackend::new_local(path).await.unwrap();

// ✅ 允许
let db = LibSqlBackend::new_local(path)
    .await
    .map_err(|e| DatabaseError::Pool(e.to_string()))?;
```

### 例外

- 单元测试 (`#[cfg(test)]`)
- 编译时已知安全的场景（如 `include_str!`）

### 理由

- 提升可靠性
- 强制错误处理思考
- 符合 Rust 最佳实践

### 代码位置

- `.claude/rules/review-discipline.md` - 规范文档
- `scripts/pre-commit-safety.sh` - 自动化检查

---

## ADR-015: 强类型错误 (thiserror)

**状态**: 已接受
**日期**: 2024-10
**决策者**: NEAR AI Team

### 背景

需要结构化错误信息，支持针对性处理。

### 决策

使用 `thiserror` 派生强类型错误。

```rust
#[derive(Error, Debug)]
pub enum AgentError {
    #[error("LLM provider error: {0}")]
    Llm(#[from] LlmError),

    #[error("Tool execution failed: {name}")]
    Tool { name: String, source: ToolError },

    #[error("Safety violation: {policy}")]
    Safety { policy: String, severity: Severity },
}
```

### 理由

- 错误变体携带结构化数据
- 自动 `From` 实现
- `match` 可精确处理

### 对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| `anyhow` | 简单 | 无结构化信息 |
| **thiserror** | 强类型 | 稍冗长 |
| 自定义 | 完全控制 | 维护成本高 |

### 代码位置

- `src/error.rs` - 全局错误类型
- 各模块 `error.rs` - 模块级错误

---

## ADR-016: Transaction Safety

**状态**: 已接受
**日期**: 2024-11
**决策者**: NEAR AI Team

### 背景

多步数据库操作（INSERT+INSERT, UPDATE+DELETE）可能部分失败，导致不一致。

### 决策

**强制事务包装**：多步操作必须原子化。

### 规则

```rust
// ❌ 错误：两个独立事务
store.create_job(&ctx).await?;
store.create_subtasks(&subtasks).await?;

// ✅ 正确：原子事务
store.transaction(|tx| {
    tx.create_job(&ctx).await?;
    tx.create_subtasks(&subtasks).await?;
    Ok(())
}).await?;
```

### 理由

- 避免 TOCTOU (Time-of-check to time-of-use) 竞态
- 保证数据一致性
- 双后端统一行为

### 代码位置

- `src/db/mod.rs` - Transaction trait 方法
- `.claude/rules/review-discipline.md` - 规范

---

## ADR-017: 配置优先级体系

**状态**: 已接受
**日期**: 2024-10
**决策者**: NEAR AI Team

### 背景

配置来自多个来源：默认值、TOML 文件、数据库、环境变量。

### 决策

确立明确的 **配置优先级**：

```
环境变量 > TOML 文件 > 数据库 > 默认值
```

### 实现

```rust
impl Config {
    /// 从 DB + env + TOML 加载
    pub async fn from_db(store: &dyn SettingsStore, user_id: &str) -> Result<Self, ConfigError>;

    /// 仅 env 加载 (启动前)
    pub async fn from_env() -> Result<Self, ConfigError>;
}
```

### 理由

- 环境变量最高优先级（临时覆盖）
- TOML 适合版本控制
- 数据库适合运行时修改
- 默认值保证可用

### 代码位置

- `src/config/mod.rs` - Config 加载
- `src/config/builder.rs` - 配置构建

---

## ADR-018: UTF-8 String Safety

**状态**: 已接受
**日期**: 2024-11
**决策者**: NEAR AI Team

### 背景

Rust 字符串是 UTF-8，字节切片可能导致 panic：

```rust
let s = "你好"; // 6 bytes
&s[..3]; // panic! 不是字符边界
```

### 决策

**禁止裸字节切片**，使用字符安全方法。

### 规则

```rust
// ❌ 禁止
&s[..n]

// ✅ 检查边界
if s.is_char_boundary(n) {
    &s[..n]
}

// ✅ 使用字符迭代
s.char_indices().take(n).collect::<String>()
```

### 理由

- 多语言内容安全
- 避免运行时 panic
- 符合 Rust 字符串语义

### 代码位置

- `.claude/rules/review-discipline.md` - 规范
- `scripts/pre-commit-safety.sh` - 自动化检查

---

## ADR-019: 模块自包含初始化

**状态**: 已接受
**日期**: 2024-12
**决策者**: NEAR AI Team

### 背景

模块初始化逻辑散落在 `main.rs`/`app.rs`，导致跨模块依赖复杂。

### 决策

**模块自包含**：每个模块提供公共工厂函数。

### 原则

1. 模块特定初始化在模块内完成
2. `main.rs`/`app.rs` 仅负责编排调用
3. Feature flag 分支限于所属模块

### 示例

```rust
// src/db/mod.rs - 模块拥有初始化
pub async fn create_database(config: &DatabaseConfig) -> Result<Arc<dyn Database>, Error> {
    #[cfg(feature = "postgres")]
    if config.backend == "postgres" {
        return Ok(Arc::new(PostgresBackend::new(config).await?));
    }
    // ...
}

// src/app.rs - 仅编排
let db = db::create_database(&config.database).await?;
```

### 代码位置

- `CLAUDE.md` - 架构规范
- 各模块 `mod.rs` - 工厂函数

---

## ADR-020: 状态机约束

**状态**: 已接受
**日期**: 2024-11
**决策者**: NEAR AI Team

### 背景

Job 生命周期需要严格状态管理，防止非法转换。

### 决策

使用 **Enum 状态机** + 受控转换方法。

```rust
enum JobState {
    Pending,
    InProgress,
    Completed,
    Failed,
    Stuck,
}

impl JobContext {
    fn mark_in_progress(&mut self) -> Result<(), Error> {
        match self.state {
            JobState::Pending | JobState::Stuck => {
                self.state = JobState::InProgress;
                Ok(())
            }
            _ => Err(Error::InvalidStateTransition),
        }
    }
}
```

### 状态图

```
Pending -> InProgress -> Completed -> Submitted -> Accepted
                     \-> Failed
                     \-> Stuck -> InProgress (recovery)
                              \-> Failed
```

### 理由

- 非法状态转换在编译期不可表示
- 运行时明确校验
- 自文档化业务逻辑

### 代码位置

- `src/context/mod.rs` - JobState 定义
- `src/context/job.rs` - 状态转换方法

---

## ADR 索引

| ADR | 主题 | 影响范围 | 状态 |
|-----|------|---------|------|
| 001 | 双数据库后端 | `src/db/` | 已接受 |
| 002 | WASM 沙箱 | `src/tools/wasm/` | 已接受 |
| 003 | 共享 Agentic 循环 | `src/agent/` | 已接受 |
| 004 | 纵深防御 | `src/safety/`, `src/tools/wasm/` | 已接受 |
| 005 | 子 Trait 分解 | `src/db/mod.rs` | 已接受 |
| 006 | MCP 集成 | `src/tools/mcp/` | 已接受 |
| 007 | LLM 装饰器 | `src/llm/decorators/` | 已接受 |
| 008 | 混合搜索 | `src/workspace/` | 已接受 |
| 009 | Docker 沙箱 | `src/sandbox/`, `src/worker/` | 已接受 |
| 010 | 技能系统 | `src/skills/`, `.claude/rules/skills.md` | 已接受 |
| 011 | 多通道 | `src/channels/` | 已接受 |
| 012 | 密钥注入 | `src/secrets/`, `src/tools/wasm/` | 已接受 |
| 013 | 心跳系统 | `src/agent/heartbeat.rs` | 已接受 |
| 014 | 零 panic | 全局 | 已接受 |
| 015 | 强类型错误 | `src/error.rs` | 已接受 |
| 016 | 事务安全 | `src/db/` | 已接受 |
| 017 | 配置优先级 | `src/config/` | 已接受 |
| 018 | UTF-8 安全 | 全局 | 已接受 |
| 019 | 模块自包含 | 架构 | 已接受 |
| 020 | 状态机 | `src/context/` | 已接受 |

---

*最后更新: 2026-03-17*
