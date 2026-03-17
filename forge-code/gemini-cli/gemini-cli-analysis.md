# Gemini CLI 架构分析

> 基于《A Philosophy of Software Design》设计哲学的深度分析

## 1. 整体架构概览

### 1.1 模块层级图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户接口层                                       │
├─────────────────┬─────────────────┬─────────────────┬───────────────────────┤
│   CLI (Ink)     │   A2A Server    │   SDK API       │   VSCode Extension    │
│   (packages/    │   (packages/    │   (packages/    │   (packages/          │
│    cli)         │    a2a-server)  │    sdk)         │    vscode-ide)        │
└────────┬────────┴────────┬────────┴────────┬────────┴───────────┬───────────┘
         │                 │                 │                    │
         └─────────────────┴─────────────────┴────────────────────┘
                                   │
         ┌─────────────────────────▼─────────────────────────┐
         │                  CORE 核心层                       │
         │           (packages/core)                          │
         │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
         │  │   Tools     │ │  Scheduler  │ │   Policy    │  │
         │  │   Registry  │ │             │ │   Engine    │  │
         │  └─────────────┘ └─────────────┘ └─────────────┘  │
         │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
         │  │   Gemini    │ │   Message   │ │    Hook     │  │
         │  │   Client    │ │    Bus      │ │   System    │  │
         │  └─────────────┘ └─────────────┘ └─────────────┘  │
         │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
         │  │    Skill    │ │   Context   │ │    Chat     │  │
         │  │   Manager   │ │   Manager   │ │ Compression │  │
         │  └─────────────┘ └─────────────┘ └─────────────┘  │
         └────────────────────────────────────────────────────┘
                                   │
         ┌─────────────────────────▼─────────────────────────┐
         │                  基础设施层                        │
         │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
         │  │   Git       │ │    MCP      │ │   Browser   │  │
         │  │  Service    │ │   Client    │ │   Manager   │  │
         │  └─────────────┘ └─────────────┘ └─────────────┘  │
         │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
         │  │   Agent     │ │  Telemetry  │ │   Safety    │  │
         │  │   Registry  │ │             │ │   Checker   │  │
         │  └─────────────┘ └─────────────┘ └─────────────┘  │
         └────────────────────────────────────────────────────┘
```

### 1.2 包依赖关系

```
cli              a2a-server       sdk
  │                │               │
  ├────────────────┼───────────────┤
  │                │               │
  ▼                ▼               ▼
  └────────────────┴───────────────┘
                  │
                  ▼
              core
                  │
                  ▼
            test-utils (dev)
```

---

## 2. 核心模块深度分析

### 2.1 Scheduler - 调度器 (Deep Module ✓)

**文件**: `packages/core/src/scheduler/scheduler.ts` (~783 行)

#### 设计评估

| 维度 | 评估 | 说明 |
|------|------|------|
| **接口复杂度** | 低 | `schedule()` 方法简单，内部状态机复杂 |
| **功能深度** | 高 | 完整的工具生命周期管理 |
| **信息隐藏** | 良好 | 内部状态转换对调用者透明 |

#### 状态机设计

```
Validating → Scheduled → Executing → Success
                                ↘
                                  → Error
                                ↙
                              Cancelled
                                ↓
                          AwaitingApproval
```

#### 关键代码结构

```typescript
export class Scheduler {
  private readonly state: SchedulerStateManager;      // 状态管理
  private readonly executor: ToolExecutor;            // 执行器
  private readonly modifier: ToolModificationHandler; // 修改处理器

  async schedule(request: ToolCallRequestInfo | ToolCallRequestInfo[], signal: AbortSignal): Promise<CompletedToolCall[]>
  private async _processQueue(signal: AbortSignal): Promise<void>
  private async _processToolCall(toolCall: ValidatingToolCall, signal: AbortSignal): Promise<void>
}
```

#### 设计哲学分析

**✓ 优点**:
- **Deep Module**: `schedule()` 接口简单，内部包含完整的并发控制、状态转换、错误处理
- **Pull Complexity Downward**: 调用者只需调用 `schedule()`，复杂的状态管理在内部处理
- **Information Hiding**: `SchedulerStateManager` 封装状态转换逻辑

**⚠ 注意点**:
- 工具调用类型定义较复杂（ValidatingToolCall, ScheduledToolCall, etc.），但这是状态机固有的复杂性，已尽可能封装

---

### 2.2 MessageBus - 消息总线 (Deep Module ✓)

**文件**: `packages/core/src/confirmation-bus/message-bus.ts` (~199 行)

#### 架构角色

MessageBus 是连接调度器与 UI 的**解耦层**，实现策略检查与确认流程的分离。

#### 关键设计

```typescript
export class MessageBus extends EventEmitter {
  constructor(
    private readonly policyEngine: PolicyEngine,
    private readonly debug = false,
  ) {}

  async publish(message: Message): Promise<void>
  derive(subagentName: string): MessageBus  // 子代理派生
  request<TRequest, TResponse>(): Promise<TResponse>  // 请求-响应模式
}
```

#### 消息类型 (MessageBusType)

| 消息类型 | 用途 |
|----------|------|
| `TOOL_CONFIRMATION_REQUEST` | 请求工具执行确认 |
| `TOOL_CONFIRMATION_RESPONSE` | 确认响应 |
| `TOOL_POLICY_REJECTION` | 策略拒绝 |
| `TOOL_EXECUTION_SUCCESS/FAILURE` | 执行结果 |
| `UPDATE_POLICY` | 策略更新 |
| `ASK_USER_REQUEST/RESPONSE` | 用户交互 |

#### 设计哲学分析

**✓ 优点**:
- **Deep Module**: 简单的 `publish()`/`subscribe()` 接口，内部处理策略检查、超时、派生
- **Define Errors Out of Existence**: 无效消息在 `isValidMessage` 中处理，不向调用者抛出
- **Request-Response Pattern**: 在异步事件总线上实现同步风格的调用

---

### 2.3 ToolRegistry - 工具注册表 (Deep Module ✓)

**文件**: `packages/core/src/tools/tool-registry.ts` (~742 行)

#### 职责

管理三类工具的注册、发现与优先级排序：
1. **Built-in Tools**: 内置工具（read-file, edit, shell 等）
2. **Discovered Tools**: 通过命令发现的工具
3. **MCP Tools**: Model Context Protocol 服务器提供的工具

#### 工具优先级

```typescript
// 工具排序逻辑
return tools.sort((a, b) => {
  // 1. 内置工具优先
  const aIsBuiltin = !(a instanceof DiscoveredTool || a instanceof DiscoveredMCPTool);
  const bIsBuiltin = !(b instanceof DiscoveredTool || b instanceof DiscoveredMCPTool);
  if (aIsBuiltin !== bIsBuiltin) return aIsBuiltin ? -1 : 1;

  // 2. 非 MCP 发现工具次之
  const aIsDiscovered = a instanceof DiscoveredTool;
  const bIsDiscovered = b instanceof DiscoveredTool;
  if (aIsDiscovered !== bIsDiscovered) return aIsDiscovered ? -1 : 1;

  // 3. 按名称排序
  return a.name.localeCompare(b.name);
});
```

#### 设计哲学分析

**✓ 优点**:
- **General-Purpose Interface**: `registerTool()`, `getTool()`, `getAllTools()` 通用接口支持多种工具类型
- **Information Hiding**: 工具发现过程（子进程执行、JSON 解析）完全封装
- **Deep Module**: 简单的注册接口，内部处理优先级排序、别名解析、排除逻辑

---

### 2.4 PolicyEngine - 策略引擎 (Deep Module ✓)

**文件**: `packages/core/src/policy/policy-engine.ts`

#### 策略决策类型

```typescript
export enum PolicyDecision {
  ALLOW = 'allow',      // 自动允许
  DENY = 'deny',        // 自动拒绝
  ASK_USER = 'ask_user', // 询问用户
}
```

#### 策略规则匹配

支持复杂的规则匹配：
- 工具名通配符（`*`, `mcp_serverName_*`）
- 参数模式匹配（`argsPattern`）
- 子代理上下文
- 注解匹配

#### 设计哲学分析

**✓ 优点**:
- **Pull Complexity Downward**: 调用者只需调用 `check()`，复杂的匹配逻辑在内部处理
- **Define Errors Out of Existence**: 无匹配规则时默认 ASK_USER，而非抛出异常

---

### 2.5 GeminiClient - AI 客户端 (Deep Module ✓)

**文件**: `packages/core/src/core/client.ts` (~1258 行)

#### 核心职责

- 流式消息处理（async generators）
- 工具调用循环管理
- 聊天历史压缩
- 循环检测
- IDE 上下文同步

#### 关键方法

```typescript
private async *processTurn(
  request: PartListUnion,
  signal: AbortSignal,
  prompt_id: string,
  boundedTurns: number,
  isInvalidStreamRetry: boolean,
  displayContent?: PartListUnion,
): AsyncGenerator<ServerGeminiStreamEvent, Turn>
```

#### 设计哲学分析

**✓ 优点**:
- **Deep Module**: 简单的流式接口，内部处理重试、压缩、检测、挂钩
- **Information Hiding**: 令牌限制、压缩策略对调用者透明
- **Pull Complexity Downward**: 聊天压缩、循环检测等复杂性内部处理

**⚠ 注意点**:
- 类较大（1258 行），可考虑将 ChatCompressionService、LoopDetectionService 进一步解耦

---

### 2.6 HookSystem - 钩子系统 (Deep Module ✓)

**文件**: `packages/core/src/hooks/hookSystem.ts`

#### 事件类型

| 类别 | 事件 |
|------|------|
| Session | SessionStart, SessionEnd |
| Agent | BeforeAgent, AfterAgent |
| Model | BeforeModel, AfterModel |
| Tool | BeforeTool, AfterTool |

#### 设计特点

```typescript
export class HookSystem {
  private readonly hookRegistry: HookRegistry;
  private readonly hookRunner: HookRunner;
  private readonly hookAggregator: HookAggregator;
  private readonly hookPlanner: HookPlanner;
  private readonly hookEventHandler: HookEventHandler;
}
```

**✓ 优点**:
- **Layered Abstraction**: 注册、运行、聚合、计划分层清晰
- **Deep Module**: 简单的 `fireBeforeModelEvent()` 接口，内部处理并行执行、错误聚合

---

## 3. SDK 层分析

### 3.1 SdkTool - 工具封装

**文件**: `packages/sdk/src/tool.ts`

#### Zod 模式验证

```typescript
export interface ToolDefinition<T extends z.ZodTypeAny> {
  name: string;
  description: string;
  inputSchema: T;
  sendErrorsToModel?: boolean;
}

export function tool<T extends z.ZodTypeAny>(
  definition: ToolDefinition<T>,
  action: (params: z.infer<T>, context?: SessionContext) => Promise<unknown>,
): Tool<T>
```

#### 设计哲学分析

**✓ 优点**:
- **Deep Module**: `tool()` 函数式 API 简单，内部封装 Zod 验证、错误处理
- **Type Safety**: 泛型 `T` 确保输入类型安全
- **ModelVisibleError**: 区分应向模型展示的错误和内部错误

---

### 3.2 GeminiCliAgent - Agent 管理

**文件**: `packages/sdk/src/agent.ts`

```typescript
export class GeminiCliAgent {
  session(options?: { sessionId?: string }): GeminiCliSession
  async resumeSession(sessionId: string): Promise<GeminiCliSession>
}
```

**✓ 优点**:
- **Simple Interface**: 会话创建和恢复接口清晰
- **Information Hiding**: 存储初始化、会话查找逻辑封装

---

## 4. A2A Server 分析

### 4.1 Task 管理

**文件**: `packages/a2a-server/src/agent/task.ts` (~1240 行)

#### A2A Task 生命周期

```
submitted → working → (input-required → working) → completed
                         ↓
                    cancelled / failed
```

#### 与 Scheduler 集成

```typescript
// Task 内部使用 Scheduler 执行工具
const scheduler = this.sessionContext.scheduler;
const completedCalls = await scheduler.schedule(validToolCalls, signal);
```

#### 设计哲学分析

**✓ 优点**:
- **Layered Abstraction**: A2A Task 层与内部 Scheduler 层分离
- **Event-Driven**: 通过 MessageBus 实现流式事件输出
- **Streaming Support**: 完整的 SSE 流式响应支持

---

## 5. 设计模式总结

### 5.1 应用的设计模式

| 模式 | 应用位置 | 效果 |
|------|----------|------|
| **事件总线** | MessageBus | 解耦调度器与 UI |
| **状态机** | Scheduler | 清晰的工具生命周期管理 |
| **策略模式** | PolicyEngine | 灵活的权限控制 |
| **注册表** | ToolRegistry, AgentRegistry | 动态发现与管理 |
| **钩子** | HookSystem | 扩展点机制 |
| **请求-响应** | MessageBus.request() | 同步风格的异步调用 |
| **派生/克隆** | MessageBus.derive() | 子代理隔离 |

### 5.2 复杂度管理

#### 控制良好的复杂度

- **Scheduler**: 状态机复杂但封装良好
- **GeminiClient**: 功能多但职责清晰
- **ToolRegistry**: 工具类型多但接口统一

#### 可改进点

- **GeminiClient** 行数较多（1258 行），可考虑将以下功能进一步剥离：
  - ChatCompressionService → 已独立，但 client 仍直接调用
  - LoopDetectionService → 已独立，但 client 仍直接调用
  - IdeContextSync → 可考虑独立服务

### 5.3 信息隐藏评估

| 模块 | 信息隐藏 | 说明 |
|------|----------|------|
| Scheduler | ✓ 良好 | 状态转换内部处理 |
| MessageBus | ✓ 良好 | 策略检查、超时内部处理 |
| ToolRegistry | ✓ 良好 | 发现过程封装 |
| GeminiClient | △ 中等 | 部分内部逻辑可进一步分离 |
| HookSystem | ✓ 良好 | 执行细节封装 |

---

## 6. 关键实现细节

### 6.1 流式处理架构

```
Gemini API → GeminiClient.processTurn() → AsyncGenerator
                      ↓
              MessageBus.publish()
                      ↓
              UI (Ink/React) 渲染
```

### 6.2 工具执行流程

```
1. LLM 返回 functionCall
2. GeminiClient 提取 tool calls
3. 创建 checkpoint (git snapshot)
4. MessageBus 发送确认请求
5. PolicyEngine 检查策略
   - ALLOW → 直接执行
   - DENY → 返回拒绝
   - ASK_USER → UI 确认
6. Scheduler 执行工具
7. 返回结果给 LLM
```

### 6.3 聊天压缩策略

```typescript
// Reverse Token Budget - 优先保留最近的消息
const tokenBudget = {
  systemInstruction: 0.1,
  recentMessages: 0.5,
  olderMessages: 0.3,
  functionResponses: 0.1,
};
```

### 6.4 循环检测

多层检测策略：
1. **启发式检测**: 工具调用重复、内容分块
2. **LLM 验证**: 置信度评分确认

---

## 7. 与软件设计哲学的对照

### 7.1 应用良好的原则

| 原则 | 应用情况 |
|------|----------|
| **Deep Modules** | Scheduler、MessageBus、ToolRegistry 都是典型的深模块 |
| **Information Hiding** | 状态管理、策略检查、工具发现等细节良好封装 |
| **Pull Complexity Downward** | 复杂性下沉到模块内部，调用者接口简单 |
| **Define Errors Out of Existence** | 无效消息处理、策略默认行为 |
| **General-Purpose Modules** | ToolRegistry 支持多种工具类型，MessageBus 支持多种消息类型 |
| **Different Layers, Different Abstractions** | CLI/TUI、Core、SDK 层抽象层次清晰 |

### 7.2 设计权衡

| 权衡点 | 选择 | 理由 |
|--------|------|------|
| EventBus vs Direct Call | EventBus | 解耦 UI 与核心，支持 A2A/Headless |
| Zod vs JSON Schema | Zod | 类型安全 + 运行时验证 |
| Monorepo vs Multi-repo | Monorepo | 包间依赖紧密，统一定义核心类型 |
| Class vs Function | Class | 需要状态管理、生命周期 |

### 7.3 潜在改进点

1. **GeminiClient 瘦身**
   - 将 chat compression、loop detection 完全委托给独立服务
   - 当前 client 直接调用，可考虑纯委托模式

2. **ToolRegistry 工具发现**
   - 发现命令使用子进程 spawn，可考虑异步迭代器优化大输出处理

3. **ConfirmationBus 类型**
   - 遗留类型 `ToolCallConfirmationDetails` 与新的 `SerializableConfirmationDetails` 并存
   - 建议完成迁移后清理

---

## 8. 核心文件清单

| 文件 | 职责 | 设计质量 |
|------|------|----------|
| `core/src/scheduler/scheduler.ts` | 工具调度 | ★★★★★ |
| `core/src/confirmation-bus/message-bus.ts` | 消息总线 | ★★★★★ |
| `core/src/tools/tool-registry.ts` | 工具管理 | ★★★★★ |
| `core/src/core/client.ts` | AI 客户端 | ★★★★☆ |
| `core/src/policy/policy-engine.ts` | 策略引擎 | ★★★★★ |
| `core/src/hooks/hookSystem.ts` | 钩子系统 | ★★★★★ |
| `sdk/src/tool.ts` | SDK 工具 | ★★★★★ |
| `a2a-server/src/agent/task.ts` | A2A 任务 | ★★★★★ |

---

*分析基于代码版本: 0.35.0-nightly.20260313*
*分析日期: 2026-03-17*
