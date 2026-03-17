# ZeroClaw 架构决策记录 (ADRs)

> 关键设计决策及其理由

---

## ADR-001: Rust 作为主要开发语言

### 状态
已接受

### 上下文
AI Agent 运行时通常使用 Python (LangChain, AutoGPT) 或 Node.js 构建。ZeroClaw 需要高性能、低资源占用和高可靠性。

### 决策
选择 Rust 作为主要开发语言。

### 理由

1. **性能** - 零成本抽象，接近 C/C++ 的性能
2. **内存安全** - 编译期保证，无 GC 停顿
3. **并发** - 一等公民的 async/await 支持
4. **二进制大小** - 可优化到 <10MB
5. **可靠性** - 类型系统捕获大量运行时错误

### 后果

**正面**:
- 高性能运行时
- 低内存占用
- 强类型保证

**负面**:
- 开发速度较慢
- 学习曲线陡峭
- 异步生态复杂度

---

## ADR-002: Trait 驱动架构

### 状态
已接受

### 上下文
需要支持多种 LLM Provider、消息通道、记忆后端和沙盒实现，且这些组件应可独立替换。

### 决策
使用 Rust Trait 定义组件接口，通过 Trait 对象 (`Box<dyn Trait>`) 实现运行时多态。

### 理由

1. **接口隔离** - 每个组件有清晰的契约
2. **可测试性** - 易于 mock 和测试
3. **扩展性** - 新实现只需实现 Trait
4. **解耦** - 组件间无直接依赖

### 核心 Trait 定义

```rust
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn schema(&self) -> ToolSchema;
    async fn execute(&self, params: Value) -> Result<ToolOutput>;
}
```

### 后果

**正面**:
- 清晰的模块边界
- 易于扩展新实现
- 统一的测试模式

**负面**:
- Trait 对象有动态分发开销
- 复杂的生命周期管理
- 需要 `async-trait` 宏

---

## ADR-003: 工厂模式 + 特性门控

### 状态
已接受

### 上下文
需要支持多种后端 (SQLite/Postgres/Qdrant)，但用户可能只需要其中一部分。

### 决策
使用工厂函数创建组件，通过 Cargo 特性标志控制编译。

### 理由

1. **编译期选择** - 未使用的后端不编译
2. **运行时选择** - 配置决定使用哪个后端
3. **依赖隔离** - 可选依赖不会污染基础构建

### 实现模式

```rust
pub async fn create_memory(config: &MemoryConfig) -> Result<Arc<dyn Memory>> {
    match config.backend.as_str() {
        "sqlite" => Ok(Arc::new(SqliteMemory::new(config)?)),
        #[cfg(feature = "memory-postgres")]
        "postgres" => Ok(Arc::new(PostgresMemory::new(config).await?)),
        _ => Err(anyhow!("Unknown backend: {}", config.backend)),
    }
}
```

### 后果

**正面**:
- 最小二进制大小
- 灵活的功能组合
- 清晰的依赖边界

**负面**:
- 复杂的特性管理
- 可能的编译错误 (缺少特性)
- 测试矩阵膨胀

---

## ADR-004: SecurityPolicy 作为横切关注点

### 状态
已接受

### 上下文
需要在整个系统中执行安全策略 (自主级别、沙盒类型、访问控制)。

### 决策
创建 `SecurityPolicy` 结构，通过 `Arc<SecurityPolicy>` 注入到所有工具和安全敏感组件。

### 理由

1. **单一职责** - 策略定义与执行分离
2. **可审计** - 策略集中管理
3. **可测试** - 易于构造不同策略场景

### 设计

```rust
pub struct SecurityPolicy {
    pub autonomy: AutonomyLevel,
    pub sandbox_kind: String,
    pub allow_shell: bool,
    pub allow_file_write: bool,
    // ...
}
```

### 后果

**正面**:
- 一致的安全策略执行
- 易于调整安全级别
- 清晰的策略传播路径

**负面**:
- 需要传递 `Arc<>` 到各处
- 可能形成隐式依赖

---

## ADR-005: anyhow + thiserror 错误处理

### 状态
已接受

### 上下文
需要处理来自多个 crate 的错误，同时保持内部错误类型的精确性。

### 决策
使用 `anyhow` 进行错误传播，`thiserror` 定义自定义错误类型。

### 理由

1. **anyhow** - 简化错误传播，自动添加上下文
2. **thiserror** - 减少错误类型的样板代码

### 模式

```rust
// 内部错误定义
#[derive(Error, Debug)]
#[error("sandbox execution failed: {0}")]
pub struct SandboxError(String);

// 函数返回
pub fn execute() -> anyhow::Result<Output> {
    risky_operation().context("operation failed")?;
    Ok(output)
}
```

### 后果

**正面**:
- 简洁的错误处理代码
- 丰富的错误上下文

**负面**:
- anyhow 会擦除具体错误类型
- 需要权衡何时使用具体错误类型

---

## ADR-006: 单一 crate + 模块组织

### 状态
已接受

### 上下文
考虑使用 workspace 将组件拆分为多个 crate。

### 决策
使用单一 crate，通过模块 (`mod`) 组织代码。

### 理由

1. **编译优化** - LTO 可以跨模块优化
2. **简单性** - 单一 Cargo.toml 管理
3. **内聚性** - 代码在同一 crate 内共享实现细节

### 目录结构

```
src/
├── lib.rs        # 模块导出
├── main.rs       # CLI 入口
├── agent/        # Agent 模块
├── channels/     # 通道模块
├── tools/        # 工具模块
└── ...
```

### 后果

**正面**:
- 简单的工作流
- 完全的优化能力
- 快速的编译 (单一 artifact)

**负面**:
- 代码量大时编译变慢
- 不能独立发布子模块
- 边界可能模糊

---

## ADR-007: Tokio 作为异步运行时

### 状态
已接受

### 上下文
需要选择异步运行时 (Tokio, async-std, smol)。

### 决策
使用 Tokio 作为异步运行时。

### 理由

1. **生态成熟度** - 最广泛的库支持
2. **性能** - 高度优化的调度器
3. **功能丰富** - 通道、信号、进程等
4. **Axum 依赖** - Web 框架依赖 Tokio

### 后果

**正面**:
- 丰富的生态系统
- 优秀的文档和社区
- 与 Axum 无缝集成

**负面**n- 较重的依赖
- 特定的运行时绑定

---

## ADR-008: Axum 作为 Web 框架

### 状态
已接受

### 上下文
需要选择 Web 框架实现 Gateway。

### 决策
使用 Axum 作为 Web 框架。

### 理由

1. **Tokio 原生** - 与运行时深度集成
2. **Tower 生态** - 可组合的中间件
3. **类型安全** - 强类型的路由和处理函数
4. **性能** - 基于 Hyper，性能优秀

### 后果

**正面**:
- 与 Tokio 生态一致
- 强大的中间件系统
- 编译时路由检查

**负面**:
- 相对较新，生态不如 Actix-web 成熟
- 学习曲线

---

## ADR-009: 特性门控 vs 运行时配置

### 状态
已接受

### 上下文
对于可选功能 (如 Postgres、Prometheus)，应该使用编译期特性还是运行时配置？

### 决策
使用编译期特性门控控制可选依赖，运行时配置选择具体实现。

### 理由

1. **编译期** - 未使用代码不编译，减小二进制
2. **运行时** - 同一二进制支持多种配置
3. **安全性** - 编译期保证代码可用性

### 混合模式

```rust
// 编译期: 特性门控控制代码是否编译
#[cfg(feature = "memory-postgres")]
mod postgres;

// 运行时: 配置选择具体实现
match config.backend {
    "sqlite" => SqliteMemory::new(),
    #[cfg(feature = "memory-postgres")]
    "postgres" => PostgresMemory::new(),
    _ => fallback(),
}
```

### 后果

**正面**:
- 最优的二进制大小
- 灵活的运行时配置
- 编译期错误检查

**负面**:
- 复杂的测试矩阵
- 需要构建多个变体
- 用户可能困惑于特性选择

---

## ADR-010: MCP (Model Context Protocol) 原生支持

### 状态
已接受

### 上下文
MCP 正在成为 LLM 工具调用的标准协议。

### 决策
原生实现 MCP 协议支持，而非依赖外部库。

### 理由

1. **控制** - 完全控制实现细节
2. **集成** - 与现有工具系统深度集成
3. **性能** - 避免序列化开销

### 实现

- `src/tools/mcp_protocol.rs` - 协议定义
- `src/tools/mcp_client.rs` - 客户端实现
- `src/tools/mcp_tool.rs` - Tool Trait 适配

### 后果

**正面**:
- 深度定制能力
- 最优性能
- 与现有架构一致

**负面**:
- 维护成本
- 需要跟进协议更新

---

## ADR-011: RAG 用于硬件数据手册

### 状态
已接受

### 上下文
硬件开发需要查询数据手册，传统方式效率低。

### 决策
实现专用的 RAG (Retrieval-Augmented Generation) 系统用于硬件数据手册检索。

### 理由

1. **专用性** - 针对硬件领域优化
2. **Pin 别名** - 支持硬件特定的别名解析
3. **上下文** - 自动注入相关硬件信息

### 实现

- `src/rag/mod.rs` - RAG 核心
- 支持 Markdown、PDF (可选特性)
- Pin 别名解析

### 后果

**正面**:
- 硬件开发的独特优势
- 本地运行，无需外部服务

**负面**:
- 增加代码复杂度
- 需要维护数据手册

---

## ADR-012: 内联测试 (#[cfg(test)])

### 状态
已接受

### 上下文
测试应该放在单独文件还是实现文件中？

### 决策
使用内联测试模块 (`#[cfg(test)] mod tests`)，放在实现文件底部。

### 理由

1. **内聚性** - 测试与实现在一起
2. **可见性** - 可访问 `pub(crate)` 成员
3. **发现性** - 易于找到相关测试

### 模式

```rust
pub fn factory() -> impl Trait {
    // 实现
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn factory_creates_correct_type() {
        // 测试
    }
}
```

### 后果

**正面**:
- 高内聚
- 访问私有成员
- 易于维护

**负面**:
- 文件变长
- 编译测试时增加编译时间

---

## 决策时间线

| 日期 | ADR | 描述 |
|------|-----|------|
| 初始 | ADR-001 | 选择 Rust 语言 |
| 初始 | ADR-002 | Trait 驱动架构 |
| 初始 | ADR-003 | 工厂模式 + 特性门控 |
| 初始 | ADR-004 | SecurityPolicy 注入 |
| 初始 | ADR-005 | anyhow + thiserror |
| 初始 | ADR-006 | 单一 crate 组织 |
| 初始 | ADR-007 | Tokio 运行时 |
| 初始 | ADR-008 | Axum Web 框架 |
| 迭代 | ADR-009 | 特性门控策略 |
| 迭代 | ADR-010 | MCP 原生支持 |
| 迭代 | ADR-011 | RAG 硬件支持 |
| 迭代 | ADR-012 | 内联测试模式 |

---

*最后更新: 2026-03-17*
