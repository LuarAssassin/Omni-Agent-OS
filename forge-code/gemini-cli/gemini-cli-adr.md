# Gemini CLI 架构决策记录 (ADR)

> 记录关键设计决策及其权衡

## 目录

- [ADR-001: Monorepo 结构](#adr-001-monorepo-结构)
- [ADR-002: 事件驱动架构](#adr-002-事件驱动架构)
- [ADR-003: 状态机调度器](#adr-003-状态机调度器)
- [ADR-004: Zod vs JSON Schema](#adr-004-zod-vs-json-schema)
- [ADR-005: Deep Modules 设计](#adr-005-deep-modules-设计)
- [ADR-006: Policy Engine 规则引擎](#adr-006-policy-engine-规则引擎)
- [ADR-007: Ink + React TUI](#adr-007-ink--react-tui)
- [ADR-008: MCP 协议集成](#adr-008-mcp-协议集成)
- [ADR-009: A2A 协议支持](#adr-009-a2a-协议支持)
- [ADR-010: OpenTelemetry 可观测性](#adr-010-opentelemetry-可观测性)

---

## ADR-001: Monorepo 结构

### 状态

已接受 (2024-Q1)

### 背景

需要组织多个相关包：CLI、核心逻辑、SDK、A2A 服务器等。

### 决策

使用 npm workspaces 管理 monorepo，包含 6 个包：
- `@google/gemini-cli` - CLI 入口
- `@google/gemini-cli-core` - 核心逻辑
- `@google/gemini-cli-sdk` - 程序化 API
- `@google/gemini-cli-a2a-server` - A2A 服务器
- `@google/gemini-cli-devtools` - 开发工具
- `@google/gemini-cli-test-utils` - 测试工具

### 权衡

| 替代方案 | 评估 |
|---------|------|
| 多仓库 | 包间依赖紧密，类型共享困难 |
| Monorepo | 统一版本管理，代码共享方便，构建复杂 |

### 后果

**正面**:
- 核心类型在 core 包统一定义
- 跨包重构容易
- 统一构建流程

**负面**:
- 构建时间增加
- 需要处理循环依赖风险
- 发布流程复杂

---

## ADR-002: 事件驱动架构

### 状态

已接受 (2024-Q1)

### 背景

调度器需要与 UI 解耦，支持多种前端（CLI TUI、A2A HTTP、Headless）。

### 决策

使用 EventEmitter-based MessageBus 作为核心通信机制。

```
Scheduler → MessageBus → UI (CLI/A2A/Headless)
                ↓
         PolicyEngine
```

### 设计哲学应用

**Deep Module**: MessageBus 提供简单的 `publish()`/`subscribe()` 接口，内部处理：
- 策略检查
- 超时管理
- 子代理派生
- 请求-响应模式

**Information Hiding**: UI 层不知道 PolicyEngine 的存在。

### 替代方案

| 方案 | 问题 |
|------|------|
| 直接调用 | UI 与核心紧耦合，无法支持 A2A |
| Redux | 过重，不适合服务端 |
| RxJS | 学习曲线陡，团队熟悉 EventEmitter |

---

## ADR-003: 状态机调度器

### 状态

已接受 (2024-Q1)

### 背景

工具调用有复杂生命周期：验证 → 调度 → 执行 → (确认) → 完成。

### 决策

使用显式状态机管理工具调用生命周期。

```
Validating → Scheduled → Executing → Success
                              ↘
                                → Error
                              ↙
                            Cancelled
                              ↓
                        AwaitingApproval
```

### 设计哲学应用

**Pull Complexity Downward**: 调用者只需调用 `schedule()`，复杂的状态转换内部处理。

**Define Errors Out of Existence**: 无效状态转换自动处理，不抛出异常。

### 实现

```typescript
class Scheduler {
  async schedule(request: ToolCallRequestInfo[]): Promise<CompletedToolCall[]>
  // 内部处理所有状态转换
}
```

---

## ADR-004: Zod vs JSON Schema

### 状态

已接受 (2024-Q2)

### 背景

SDK 需要运行时类型验证 + JSON Schema 生成（用于 LLM）。

### 决策

使用 Zod 作为主要 schema 定义工具。

```typescript
// Zod 定义
const schema = z.object({
  path: z.string(),
  content: z.string(),
});

// 自动转 JSON Schema
const jsonSchema = zodToJsonSchema(schema);
```

### 权衡

| 方案 | 类型安全 | 运行时验证 | JSON Schema | 学习曲线 |
|------|---------|-----------|-------------|---------|
| JSON Schema | ✗ | ✓ | 原生 | 中 |
| Zod | ✓ | ✓ | 需转换 | 低 |
| TypeBox | ✓ | ✓ | 需转换 | 低 |

### 设计哲学应用

**General-Purpose Interface**: Zod schema 同时满足：
- TypeScript 类型推导
- 运行时验证
- JSON Schema 生成

---

## ADR-005: Deep Modules 设计

### 状态

已接受 (持续原则)

### 背景

根据《A Philosophy of Software Design》，优先设计 Deep Modules。

### 决策

核心模块遵循 Deep Module 原则：

| 模块 | 接口复杂度 | 实现复杂度 | 深度 |
|------|-----------|-----------|------|
| Scheduler | 低 | 高 | ✓ Deep |
| MessageBus | 低 | 高 | ✓ Deep |
| ToolRegistry | 低 | 中 | ✓ Deep |
| PolicyEngine | 低 | 中 | ✓ Deep |

### 应用示例

```typescript
// Scheduler: 简单接口
scheduler.schedule([tool1, tool2]);

// 内部：并发控制、状态机、确认流程、错误处理
```

---

## ADR-006: Policy Engine 规则引擎

### 状态

已接受 (2024-Q2)

### 背景

需要灵活的工具执行权限控制，支持多种场景。

### 决策

实现基于规则的 PolicyEngine，支持三种决策：
- `ALLOW` - 自动允许
- `DENY` - 自动拒绝
- `ASK_USER` - 询问用户

### 规则匹配

```toml
[[rules]]
pattern = "shell_*"
decision = "ask_user"

[[rules]]
pattern = "read_file"
argsPattern = { path = "/safe/**" }
decision = "allow"
```

### 设计哲学应用

**Define Errors Out of Existence**: 无匹配规则时默认 `ASK_USER`，不抛异常。

**Pull Complexity Downward**: 调用者只需调用 `check()`，复杂的通配符匹配内部处理。

---

## ADR-007: Ink + React TUI

### 状态

已接受 (2024-Q1)

### 背景

CLI 需要丰富的终端交互界面。

### 决策

使用 Ink (React for Terminal) 构建 TUI。

### 权衡

| 方案 | 优点 | 缺点 |
|------|------|------|
| Native Node.js | 无依赖 | 代码复杂 |
| Oclif | 成熟 | 不够灵活 |
| Ink | React 生态、组件化 | 学习曲线 |

### 实现细节

- 使用 fork `@jrichman/ink` 包含自定义修复
- React 19 支持并发特性
- 组件化架构：`App`, `Chat`, `ConfirmationPrompt`

---

## ADR-008: MCP 协议集成

### 状态

已接受 (2024-Q3)

### 背景

需要支持外部工具扩展，标准化工具发现机制。

### 决策

集成 Model Context Protocol (MCP)。

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────┐
│   Gemini CLI    │────→│  MCP Client  │────→│ MCP Server  │
│                 │     │              │     │ (External)  │
└─────────────────┘     └──────────────┘     └─────────────┘
```

### 设计哲学应用

**Layered Abstraction**: MCP 层与 CLI 核心层分离。

**General-Purpose**: ToolRegistry 同时支持内置工具、发现工具、MCP 工具。

---

## ADR-009: A2A 协议支持

### 状态

已接受 (2024-Q3)

### 背景

需要支持 Agent-to-Agent 通信，标准化跨代理交互。

### 决策

实现 A2A 协议服务器，作为独立包 `@google/gemini-cli-a2a-server`。

### 架构

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   A2A Client │────→│  A2A Server  │────→│   Gemini     │
│              │     │   (Express)  │     │   CLI Core   │
└──────────────┘     └──────────────┘     └──────────────┘
```

### 关键决策

A2A 服务器复用 Core 包的 Scheduler 和 MessageBus，确保行为一致。

---

## ADR-010: OpenTelemetry 可观测性

### 状态

已接受 (2024-Q2)

### 背景

需要生产级的可观测性：分布式追踪、指标收集。

### 决策

采用完整的 OpenTelemetry 栈，导出到 Google Cloud。

```
┌─────────────────┐
│   Gemini CLI    │
│  ┌───────────┐  │
│  │  OpenTel  │  │────→ Cloud Trace
│  │  SDK      │  │────→ Cloud Monitoring
│  └───────────┘  │
└─────────────────┘
```

### 实现

- `@opentelemetry/sdk-node`
- `@google-cloud/opentelemetry-cloud-trace-exporter`
- `@google-cloud/opentelemetry-cloud-monitoring-exporter`

### 权衡

**正面**: 标准化，与 Google Cloud 原生集成
**负面**: 依赖较多，增加包体积

---

## ADR-011: 聊天压缩策略

### 状态

已接受 (2024-Q2)

### 背景

上下文长度限制要求智能的聊天历史管理。

### 决策

实现 Reverse Token Budget 策略：

```typescript
const tokenBudget = {
  systemInstruction: 0.1,  // 10% 系统提示
  recentMessages: 0.5,     // 50% 最近消息
  olderMessages: 0.3,      // 30% 历史消息
  functionResponses: 0.1,  // 10% 工具响应
};
```

### 设计哲学应用

**Information Hiding**: 压缩策略对 GeminiClient 调用者透明。

**Pull Complexity Downward**: 调用者无需关心 token 限制。

---

## ADR-012: 钩子系统 (Hook System)

### 状态

已接受 (2024-Q2)

### 背景

需要提供扩展点，允许外部代码介入关键生命周期。

### 决策

实现事件驱动的 HookSystem，支持 8 种事件类型：

| 类别 | 事件 |
|------|------|
| Session | SessionStart, SessionEnd |
| Agent | BeforeAgent, AfterAgent |
| Model | BeforeModel, AfterModel |
| Tool | BeforeTool, AfterTool |

### 架构

```typescript
class HookSystem {
  private readonly hookRegistry: HookRegistry;
  private readonly hookRunner: HookRunner;
  private readonly hookAggregator: HookAggregator;
}
```

### 设计哲学应用

**Layered Abstraction**: 注册、运行、聚合分层清晰。

---

## 决策变更记录

| 日期 | ADR | 变更 |
|------|-----|------|
| 2024-Q3 | ADR-008 | 新增 MCP 支持 |
| 2024-Q3 | ADR-009 | 新增 A2A 支持 |
| 2024-Q2 | ADR-004 | 确定使用 Zod |
| 2024-Q2 | ADR-006 | 新增 PolicyEngine |

---

*文档版本: 0.35.0-nightly.20260313*
*最后更新: 2026-03-17*
