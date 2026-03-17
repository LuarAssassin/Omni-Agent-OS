# Codex 架构决策记录 (ADR)

> 记录项目中的关键架构决策及其理由
> 遵循：《A Philosophy of Software Design》设计哲学
> 格式：基于 NIST ADR 模板

---

## 已接受决策

### ADR-001: SQ/EQ 协议解耦 UI 与引擎

**状态**: 已接受 ✅
**日期**: 2024-Q3
**影响范围**: protocol, core, tui, app-server

#### 背景

需要支持多种 UI 前端（TUI、IDE 插件、桌面应用），引擎逻辑应与展示层解耦，同时支持异步、可中断的执行流程。

#### 决策

采用 **Submission Queue (SQ)** 与 **Event Queue (EQ)** 双队列协议：

```
UI ──Op──► SQ ──► Codex ──► EventMsg ──► EQ ──► UI
```

- **SQ**: UI → Codex，发送 Op（如 UserTurn、Interrupt、ExecApproval）
- **EQ**: Codex → UI，发送 EventMsg（如 AgentMessage、TurnComplete）

#### 理由

| 优势 | 说明 |
|------|------|
| **解耦** | 任意 UI 实现只需实现 SQ/EQ 接口 |
| **可测试** | 引擎可独立于 UI 进行单元测试 |
| **扩展性** | 新 UI 无需修改 core |
| **异步** | 队列天然支持异步消息流 |
| **跨进程** | 支持 UI 和 Core 在不同进程（如 App Server 模式）|

#### 实现

- `protocol/src/protocol.rs` - Op、EventMsg 定义
- `codex-rs/docs/protocol_v1.md` - 协议文档
- `#[non_exhaustive]` 属性确保向后兼容

#### 权衡

- **复杂度**: 引入了队列管理的复杂度
- **延迟**: 相比直接调用有轻微序列化开销
- **调试**: 异步流程调试难度增加

---

### ADR-002: Rust 作为主实现语言

**状态**: 已接受 ✅
**日期**: 2024-Q1
**影响范围**: 全项目

#### 背景

需要高性能、内存安全、跨平台的本地代理实现，支持沙箱集成和单二进制分发。

#### 决策

使用 **Rust 2024 Edition** 实现 Codex CLI 核心。

#### 理由

| 优势 | 说明 |
|------|------|
| **性能** | 零成本抽象，适合流式处理 |
| **安全** | 内存安全、并发安全，沙箱集成（Landlock）|
| **跨平台** | 单二进制，支持 macOS/Linux/Windows |
| **生态** | tokio、reqwest、tree-sitter 等成熟库 |
| **部署** | 静态链接，无运行时依赖 |

#### 权衡

- **学习曲线**: 较陡，团队成员需熟悉 Rust 所有权模型
- **编译时间**: 72 crates，增量编译仍需时间
- **开发速度**: 相比 Python/TypeScript 开发周期更长

#### 相关决策

- ADR-009: 多 Crate Workspace（缓解编译时间）

---

### ADR-003: 分层沙箱策略

**状态**: 已接受 ✅
**日期**: 2024-Q2
**影响范围**: sandboxing, linux-sandbox, windows-sandbox-rs, core

#### 背景

AI 代理执行命令存在安全风险，需限制文件系统与网络访问，同时保持可用性。

#### 决策

实现分层沙箱模式：

| 模式 | 文件系统 | 网络 | 使用场景 |
|------|----------|------|----------|
| **ReadOnly** | 只读 | 禁止 | 安全审查 |
| **WorkspaceWrite** | 工作区可写 | 禁止 | 日常开发 |
| **DangerFullAccess** | 无限制 | 无限制 | 紧急情况（需确认）|
| **Linux Sandbox** | Landlock | 代理控制 | Linux 完整隔离 |
| **Windows Sandbox** | Windows API | 代理控制 | Windows 隔离 |
| **macOS Seatbelt** | sandbox-exec | 代理控制 | macOS 隔离 |

#### 理由

- **默认安全**: 新用户默认受限，需明确放宽
- **渐进式**: 用户可按需提升权限
- **平台原生**: 利用 OS 原生安全能力
- **多层防护**: 策略层 + 沙箱层 + AI 审核层

#### 实现

```
codex-rs/
├── linux-sandbox/       # Landlock + bubblewrap
├── windows-sandbox-rs/  # Windows Sandbox API
├── core/src/sandboxing/ # 统一抽象层
│   ├── mod.rs          # SandboxPolicy 定义
│   └── seatbelt.rs     # macOS 实现
```

#### 安全架构

```
用户命令
    │
    ▼
┌─────────────┐    ExecPolicy (Starlark)
│ 策略审核    │    用户自定义规则
└──────┬──────┘
       │
       ▼
┌─────────────┐    平台沙箱
│ 沙箱执行    │    Landlock/Windows/Seatbelt
└──────┬──────┘
       │
       ▼
┌─────────────┐    AI 风险评估
│ Guardian    │    异常行为检测
└──────┬──────┘
       │
       ▼
    执行
```

---

### ADR-004: Starlark 执行策略

**状态**: 已接受 ✅
**日期**: 2024-Q3
**影响范围**: execpolicy, core

#### 背景

需要灵活配置何时批准/拒绝命令执行，硬编码规则无法满足多样化需求。

#### 决策

使用 **Starlark** 脚本定义执行策略（execpolicy）。

#### 理由

- **可配置**: 用户可自定义规则
- **沙箱**: Starlark 本身受限，不可访问系统
- **表达力**: 支持条件、模式匹配、函数
- **可读性**: Python-like 语法，易于编写

#### 示例策略

```starlark
def check_exec(cmd, cwd):
    # 危险命令需审批
    if is_dangerous(cmd):
        return "ask"
    # 网络命令需审批
    if has_network_access(cmd):
        return "ask"
    # 其他自动执行
    return "auto"
```

#### 实现

- `execpolicy/` - 策略解析与执行
- `core/src/exec_policy.rs` - 集成点

#### 权衡

- **性能**: 解释执行，有轻微开销
- **依赖**: 引入 starlark 库依赖

---

### ADR-005: SQLite 状态存储

**状态**: 已接受 ✅
**日期**: 2024-Q2
**影响范围**: state, core

#### 背景

需要持久化会话、线程、配置状态，支持跨会话恢复。

#### 决策

使用 **SQLite** 存储状态，路径由 `CODEX_SQLITE_HOME` 或 `sqlite_home` 配置。

#### 理由

- **轻量**: 单文件，无服务依赖
- **可靠**: ACID、成熟、广泛使用
- **可移植**: 跨平台，文件可迁移
- **查询**: SQL 支持复杂查询

#### 实现

- `state/` - sqlx 驱动
- `core/src/state_db.rs` - 业务层封装
- `state/migrations/` - Schema 版本管理

#### 数据模型

```sql
-- 核心表
sessions      -- 会话配置
threads       -- 对话线程
events        -- 事件日志
approvals     -- 审批记录
```

---

### ADR-006: MCP 原生支持

**状态**: 已接受 ✅
**日期**: 2024-Q4
**影响范围**: rmcp-client, mcp-server, core

#### 背景

Model Context Protocol 成为 AI 工具生态标准，需支持外部工具生态。

#### 决策

**双模式 MCP 支持：**
1. **MCP Client**: Codex 作为客户端连接外部 MCP 服务器
2. **MCP Server**: Codex 作为服务器供 IDE 等外部调用

#### 理由

- **生态**: 与 MCP 工具兼容（文件系统、数据库、API 等）
- **扩展**: 用户可添加自定义工具
- **标准化**: 遵循行业协议
- **双向**: 既是消费者也是提供者

#### 实现

```
codex-rs/
├── rmcp-client/      # MCP 客户端
│   └── 连接管理、工具发现、调用
├── mcp-server/       # MCP 服务器
│   └── 工具暴露、请求处理
└── core/src/mcp*.rs  # 核心集成
```

---

### ADR-007: 技能系统基于 Markdown

**状态**: 已接受 ✅
**日期**: 2024-Q3
**影响范围**: skills, core

#### 背景

需要可扩展的领域知识注入，支持项目特定上下文。

#### 决策

使用 **SKILL.md** 文件定义技能，支持 BM25 搜索与层级加载。

#### 理由

- **可读**: Markdown 易编写、易审查
- **版本控制**: Git 友好，可追溯
- **检索**: BM25 按相关性加载
- **层级**: 支持项目级、用户级、系统级技能

#### 技能格式

```markdown
---
title: "React 开发规范"
---

# React 组件规范

## 函数组件
- 使用 TypeScript
- Props 接口命名：${ComponentName}Props
...
```

#### 实现

- `skills/` - 技能加载、索引、搜索
- `core/src/skills/` - 运行时集成

---

### ADR-008: 配置 TOML 格式

**状态**: 已接受 ✅
**日期**: 2024-Q1
**影响范围**: core

#### 背景

需要人类可读的配置文件，支持注释和多行字符串。

#### 决策

使用 **TOML** 作为 `config.toml` 格式，支持 JSON Schema 校验。

#### 理由

- **可读**: 比 JSON 更易编辑，支持注释
- **类型**: 支持多行字符串、数组、表格
- **Schema**: schemars 生成校验
- **生态**: Rust 原生支持良好

#### 配置结构

```toml
[model]
model = "gpt-4o"
reasoning_effort = "high"

[sandbox]
sandbox_policy = "workspace_write"
approval_policy = "ask"

[[mcp.servers]]
name = "filesystem"
command = "npx -y @modelcontextprotocol/server-filesystem"
```

---

### ADR-009: 多 Crate Workspace

**状态**: 已接受 ✅
**日期**: 2024-Q1
**影响范围**: 全项目结构

#### 背景

代码量达 65 万+ 行，单 crate 编译时间过长，模块边界不清晰。

#### 决策

拆分为 **72 个 Cargo crates**，按职责垂直划分。

#### 理由

- **编译**: 增量编译、并行编译
- **边界**: 强制模块边界，依赖关系显式化
- **复用**: 独立 crate 可单独发布
- **测试**: 各 crate 可独立测试

#### Workspace 结构

```
codex-rs/
├── cli/                    # 主入口
├── core/                   # 核心引擎
├── protocol/              # 协议定义
├── tui/                   # TUI 界面
├── tui_app_server/        # App Server 模式 TUI
├── exec/                  # 非交互执行
├── app-server/            # JSON-RPC 服务器
├── linux-sandbox/         # Linux 沙箱
├── windows-sandbox-rs/    # Windows 沙箱
├── mcp-server/            # MCP 服务器
├── rmcp-client/           # MCP 客户端
├── skills/                # 技能系统
├── state/                 # 状态存储
├── connectors/            # ChatGPT 连接器
├── codex-api/             # OpenAI API 客户端
├── backend-client/        # 底层 API 通信
├── file-search/           # 文件搜索
├── package-manager/       # 包管理集成
├── execpolicy/            # 执行策略
├── hooks/                 # 钩子系统
└── utils/                 # 15+ 工具 crates
```

#### 权衡

- **复杂度**: 依赖图复杂，跨 crate 重构成本高
- **工具**: 需管理多 crate 版本协调
- **IDE**: 大型 workspace 可能降低 IDE 性能

---

### ADR-010: 禁止 stdout/stderr 直写

**状态**: 已接受 ✅
**日期**: 2024-Q2
**影响范围**: core, 所有库代码

#### 背景

库代码直接写 stdout/stderr 会破坏 TUI 渲染和日志重定向。

#### 决策

在 core 中启用 `#![deny(clippy::print_stdout, clippy::print_stderr)]`。

#### 理由

- **抽象**: 输出必须经 TUI 或 tracing 系统
- **可测试**: 无副作用输出，可捕获验证
- **一致性**: 所有输出可控、可重定向

#### 实现

```rust
// core/src/lib.rs
#![deny(clippy::print_stdout, clippy::print_stderr)]
```

#### 输出路径

```
库代码
  │
  ├──► tracing (日志)
  │
  ├──► EQ (Event Queue) → UI 显示
  │
  └──► 结构化输出（如 App Server JSON-RPC）
```

---

### ADR-011: 严格 Clippy 规则

**状态**: 已接受 ✅
**日期**: 2024-Q1
**影响范围**: 全项目

#### 背景

需要统一代码风格与质量，减少常见错误。

#### 决策

在 workspace 中 deny 多项 Clippy lint。

#### 规则示例

```toml
[workspace.lints.clippy]
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
print_stdout = "deny"
print_stderr = "deny"
needless_borrow = "deny"
collapsible_if = "deny"
uninlined_format_args = "deny"
redundant_closure_for_method_calls = "deny"
```

#### 理由

- **质量**: 减少常见错误（unwrap、panic）
- **一致性**: 团队统一风格
- **可维护**: 显式错误处理

#### 例外处理

```rust
// 允许特定位置的 unwrap，需注释说明
let val = map.get(key).unwrap(); // SAFETY: key 之前已插入
```

---

### ADR-012: App Server JSON-RPC 协议

**状态**: 已接受 ✅
**日期**: 2024-Q3
**影响范围**: app-server, app-server-protocol, tui_app_server

#### 背景

IDE 插件等外部客户端需要标准化协议驱动 Codex。

#### 决策

实现 **App Server**，通过 **JSON-RPC over stdio** 与 Codex 通信。

#### 理由

- **标准化**: JSON-RPC 通用协议
- **跨语言**: TypeScript/Python SDK 易于实现
- **解耦**: 客户端与 Codex 进程分离
- **版本**: 支持 v1/v2 API 版本

#### 协议设计

```rust
// 请求
{
    "jsonrpc": "2.0",
    "method": "thread/read",
    "params": { "thread_id": "..." },
    "id": 1
}

// 响应
{
    "jsonrpc": "2.0",
    "result": { ... },
    "id": 1
}
```

#### 实现

- `app-server/` - 服务端实现
- `app-server-protocol/` - 协议类型定义
- `app-server-client/` - 客户端库
- `sdk/typescript/`, `sdk/python/` - 多语言 SDK

---

### ADR-013: 模块化 TUI 架构

**状态**: 已接受 ✅
**日期**: 2024-Q4
**影响范围**: tui

#### 背景

TUI 代码量庞大（app.rs 312KB），需按功能域拆分。

#### 决策

按功能域拆分 TUI 为多个子模块：

```
tui/src/
├── app.rs              # 主应用状态（精简）
├── app_event.rs        # 事件处理
├── bottom_pane/        # 底部面板
│   ├── chat_composer.rs
│   ├── footer.rs
│   └── mod.rs
├── chatwidget/         # 聊天界面
│   ├── chatwidget.rs   # 从 app.rs 拆分
│   └── ...
├── exec_cell/          # 执行单元
├── notifications/      # 通知系统
├── onboarding/         # 首次使用引导
├── public_widgets/     # 共享组件
├── render/             # 渲染工具
├── status/             # 状态指示器
└── streaming/          # 流式内容处理
```

#### 理由

- **维护**: 单文件 <500 LoC（目标）
- **并行**: 多人开发减少冲突
- **测试**: 各模块可独立测试
- **认知**: 降低单文件认知负载

---

### ADR-014: 记忆系统两阶段处理

**状态**: 已接受 ✅
**日期**: 2024-Q4
**影响范围**: core/src/memories

#### 背景

需要跨会话持久化关键信息，支持上下文恢复。

#### 决策

实现**两阶段记忆处理**：

1. **Phase 1**: 实时提取关键信息
2. **Phase 2**: 异步整理、去重、聚类

#### 理由

- **实时性**: Phase 1 不阻塞主流程
- **质量**: Phase 2 深度处理，提高记忆质量
- **可扩展**: 各阶段可独立优化

#### 实现

```
core/src/memories/
├── mod.rs              # 记忆管理器接口
├── phase1.rs           # 原始提取
├── phase2.rs           # 整理聚合
├── types.rs            # 数据结构
└── persistence.rs      # 持久化
```

---

## 待定决策

### ADR-PENDING-001: Bazel 与 Cargo 双构建系统

**问题**: Bazel 用于远程构建，Cargo 用于本地开发，两者如何长期共存？
**状态**: 评估中
**选项**:
- A: 保持双系统，维护两套配置
- B: 放弃 Bazel，优化 Cargo 支持远程构建
- C: 放弃 Cargo，仅使用 Bazel

### ADR-PENDING-002: 多模型提供商统一抽象

**问题**: Ollama、LMStudio、Responses API 等接口差异如何统一？
**状态**: 持续演进
**当前**: `model_provider_info.rs` 提供基础抽象

### ADR-PENDING-003: Core Crate 拆分

**问题**: core crate 过大（288KB lib.rs），是否拆分为子 crate？
**状态**: 讨论中
**建议方案**:
```
core/ → codex-agent/ + codex-tools/ + codex-mcp/ + codex-config/
```

---

## 拒绝的决策

### ADR-REJECTED-001: 使用 Protocol Buffers

**状态**: 已拒绝 ❌
**理由**:
- 项目早期已使用 JSON，迁移成本高
- 协议主要内部使用，JSON 可读性更好
- 性能不是瓶颈

### ADR-REJECTED-002: 嵌入式数据库 (如 sled)

**状态**: 已拒绝 ❌
**理由**:
- SQLite 更成熟、生态更好
- SQL 支持复杂查询
- 运维工具丰富

---

## 附录：ADR 索引

| ADR | 标题 | 状态 | 影响范围 |
|-----|------|------|----------|
| 001 | SQ/EQ 协议 | ✅ | protocol, core, tui |
| 002 | Rust 主语言 | ✅ | 全项目 |
| 003 | 分层沙箱 | ✅ | sandboxing |
| 004 | Starlark 策略 | ✅ | execpolicy |
| 005 | SQLite 存储 | ✅ | state |
| 006 | MCP 原生支持 | ✅ | mcp-* |
| 007 | Markdown 技能 | ✅ | skills |
| 008 | TOML 配置 | ✅ | core |
| 009 | 多 Crate | ✅ | 全项目结构 |
| 010 | 禁止 stdout | ✅ | core |
| 011 | 严格 Clippy | ✅ | 全项目 |
| 012 | JSON-RPC | ✅ | app-server |
| 013 | 模块化 TUI | ✅ | tui |
| 014 | 两阶段记忆 | ✅ | memories |

---

*ADR 持续更新中。新决策需经过讨论、文档化、实施三步流程。*
