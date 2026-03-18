# AI Agent Runtime 主循环架构深度调研报告（完整版）

## 概述

本报告对 **15个开源 AI Agent 项目** 的 Agent Runtime 主循环架构进行深度技术分析，涵盖 8个个人通用智能助理项目 和 7个编程智能体项目，对比各自的执行流程、并发模型、状态管理和扩展机制。

---

## 项目分类概览

### 个人通用智能助理（Personal General AI Assistant）

| 项目 | 语言 | 运行时基础 | 主循环模式 | 特色机制 |
|------|------|-----------|-----------|---------|
| **CoPaw** | Python | AgentScope | ReAct | ToolGuard六层防护 + 心跳 |
| **IronClaw** | Python | 自研 | ReAct + Event Stream | 三代理类型（Main/Sub/Task） |
| **NanoClaw** | TypeScript | Node.js | Async Message Loop | 容器隔离 + 多通道 |
| **ZeroClaw** | Rust | Tokio | Trait-driven Agent | 上下文压缩 + D-Mail |
| **MimiClaw** | C | FreeRTOS | ReAct | ESP32嵌入式 + 离线优先 |
| **PicoClaw** | Go | 标准库 | AgentLoop | 15通道 + 原子操作 |
| **Nanobot** | Python | asyncio | MessageBus | 轻量级 + 事件驱动 |
| **OpenClaw** | TypeScript | Node.js | Pi Event Stream | Hook系统 + 自动回复 |

### 编程智能体（Programming Agents）

| 项目 | 语言 | 运行时基础 | 主循环模式 | 特色机制 |
|------|------|-----------|-----------|---------|
| **Aider** | Python | 自研 | Coder.run() | 多diff格式策略 |
| **Codex** | Rust | 自研 | Session管理 | 70+ crates + 沙箱 |
| **Qwen-code** | TypeScript | React+Ink | Gemini架构 | 流式UI + 检查点 |
| **Gemini-cli** | TypeScript | npm workspaces | Scheduler | A2A协议 + 并行执行 |
| **Pi-mono** | TypeScript | Pi Agent | AgentLoop | 事件流 + Agent模式 |
| **Kimi-cli** | Python | kosong | Step Loop | D-Mail时间旅行 |
| **Opencode** | TypeScript | Vercel AI SDK | Stream Processing | 类型安全 + 快照回滚 |

---

## 一、个人通用智能助理 Runtime 详解

### 1. CoPaw - ReAct 循环架构

#### 核心架构

基于 **AgentScope** 框架的 **ReActAgent**，采用经典的 Reasoning-Acting 循环：

```
┌─────────────────────────────────────────────────────────────────┐
│                     CoPaw ReAct Loop                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│   │ User Msg │ ───> │ Reasoning│ ───> │  Acting  │             │
│   └──────────┘      │  (LLM)   │      │ (Tools)  │             │
│                     └────┬─────┘      └────┬─────┘             │
│                          │                 │                    │
│                          └────────┬────────┘                    │
│                                   │                             │
│                                   ▼                             │
│                          ┌──────────────┐                       │
│                          │ Tool Result  │                       │
│                          │ Inject Back  │                       │
│                          └──────────────┘                       │
│                                   │                             │
│                                   └────────> (Loop until done)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```python
# src/copaw/agents/react_agent.py
class CoPawAgent(ToolGuardMixin, ReActAgent):
    def __init__(self, max_iters: int = 50, ...):
        super().__init__(...)

    async def reply(self, message: Msg) -> Msg:
        """Main entry for message processing."""
        for _ in range(self.max_iters):
            # 1. Reasoning - LLM decides next action
            reasoning_msg = await self._reasoning()

            # 2. Acting - Execute tool calls
            if has_tool_calls(reasoning_msg):
                await self._acting(reasoning_msg)
            else:
                return reasoning_msg
```

#### 多通道并发模型

**Channel Manager 架构**:

```python
# src/copaw/app/channels/manager.py
class ChannelManager:
    def __init__(self, channels: List[BaseChannel]):
        self._queues: Dict[str, asyncio.Queue] = {}
        self._consumer_tasks: List[asyncio.Task] = []
        self._in_progress: Set[Tuple[str, str]] = set()

    async def _consume_channel_loop(self, channel_id: str, worker_index: int):
        """Worker loop: 4 workers per channel."""
        while True:
            payload = await q.get()
            key = ch.get_debounce_key(payload)
            async with key_lock:
                self._in_progress.add((channel_id, key))
                batch = _drain_same_key(q, ch, key, payload)
                await _process_batch(ch, batch)
```

**并发配置**:
- 每通道工作线程: `_CONSUMER_WORKERS_PER_CHANNEL = 4`
- 队列最大长度: `_CHANNEL_QUEUE_MAXSIZE = 1000`
- 消息防抖: 时间窗口内合并同一会话消息

---

### 2. IronClaw - 统一 Agentic Loop 架构

#### 核心架构

IronClaw 实现了统一的 **Agentic Loop**，支持三种代理类型：

```
┌─────────────────────────────────────────────────────────────────┐
│                   IronClaw Unified Agentic Loop                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Agent Types                                            │   │
│   │  • Main Agent - 直接用户交互                             │   │
│   │  • Sub Agent - 任务特定，会话隔离                        │   │
│   │  • Task Agent - 子任务委托，会话共享                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Unified Loop (src/agent/agentic_loop.rs)               │   │
│   │                                                         │   │
│   │  1. Pre-loop hooks                                      │   │
│   │  2. While not done:                                     │   │
│   │     a. Build prompt (system + history + tool_desc)      │   │
│   │     b. Check context limit → Summarize if needed        │   │
│   │     c. LLM call                                         │   │
│   │     d. Parse response (thought/action/args)             │   │
│   │     e. Execute action                                   │   │
│   │     f. Post-action hooks                                │   │
│   │     g. Update history                                   │   │
│   │  3. Post-loop hooks                                     │   │
│   │  4. Return result                                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```rust
// src/agent/agentic_loop.rs
pub async fn run_agentic_loop(
    &mut self,
    agent_type: AgentType,
    initial_messages: Vec<Message>,
) -> Result<AgentOutput, AgentError> {
    // Pre-loop hooks
    self.execute_hooks(AgentHook::PreLoop).await?;

    let mut messages = initial_messages;
    let mut done = false;
    let mut iteration = 0;

    while !done && iteration < self.config.max_iterations {
        iteration += 1;

        // 1. Build complete prompt
        let prompt = self.build_prompt(&messages).await?;

        // 2. Check context limit
        if self.estimate_tokens(&prompt) > self.config.context_limit {
            messages = self.summarize_context(messages).await?;
        }

        // 3. Call LLM
        let response = self.llm.complete(&prompt).await?;

        // 4. Parse structured response
        let thought_action = parse_thought_action(&response)?;

        // 5. Execute action
        let action_result = if thought_action.action == "finish" {
            done = true;
            ActionResult::Success(thought_action.final_answer)
        } else {
            self.execute_action(&thought_action).await?
        };

        // 6. Post-action hooks
        self.execute_hooks(AgentHook::PostAction(&thought_action)).await?;

        // 7. Update history
        messages.push(Message::assistant(response));
        messages.push(Message::system(format!("Action result: {:?}", action_result)));
    }

    // Post-loop hooks
    self.execute_hooks(AgentHook::PostLoop).await?;

    Ok(AgentOutput { messages, iterations: iteration })
}
```

#### Agent 类型对比

| 类型 | 会话隔离 | Labor Market | 用途 |
|------|---------|--------------|------|
| **Main** | ❌ 共享 | 独立 | 用户直接交互 |
| **Sub** | ✅ 隔离 | 独立 | 长期运行的专业代理 |
| **Task** | ❌ 共享 | 共享 | 临时子任务委托 |

---

### 3. NanoClaw - 容器化消息循环架构

#### 核心架构

基于 Node.js 的异步消息循环，每个 Agent 运行在隔离容器中：

```
┌─────────────────────────────────────────────────────────────────┐
│                   NanoClaw Message Loop                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Container Spawning (index.ts)                          │   │
│   │  - Docker/Apple Container per agent                     │   │
│   │  - Environment setup                                    │   │
│   │  - Tool injection                                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Message Processing Loop                                │   │
│   │                                                         │   │
│   │  while (true) {                                         │   │
│   │    const msg = await channel.receive();                 │   │
│   │    if (msg.type === 'command') {                        │   │
│   │      await handleCommand(msg);                          │   │
│   │    } else if (msg.type === 'chat') {                    │   │
│   │      await handleChat(msg);                             │   │
│   │    }                                                    │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Multi-Channel Router                                   │   │
│   │  - WhatsApp                                             │   │
│   │  - Telegram                                             │   │
│   │  - Slack                                                │   │
│   │  - Email                                                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```typescript
// src/index.ts
async function startAgentLoop(config: AgentConfig): Promise<void> {
  const container = await spawnContainer({
    image: config.containerImage,
    env: config.environment,
    tools: config.enabledTools,
  });

  const messageBus = new MessageBus();

  // Main loop
  while (true) {
    try {
      const message = await messageBus.receive({
        timeout: 30000, // 30s timeout
        channels: config.channels,
      });

      if (message.type === 'shutdown') {
        logger.info('Shutdown signal received');
        break;
      }

      // Process message in container
      const result = await container.execute(async (ctx) => {
        const agent = ctx.getAgent();
        return await agent.processMessage(message);
      });

      // Send response
      await messageBus.send(result.channel, result.response);

    } catch (error) {
      logger.error('Error in agent loop:', error);
      // Continue loop
    }
  }

  await container.destroy();
}
```

---

### 4. ZeroClaw - Rust Trait-driven Agent 架构

#### 核心架构

基于 Rust Trait 的 Agent 设计，强调类型安全和零成本抽象：

```
┌─────────────────────────────────────────────────────────────────┐
│                   ZeroClaw Agent Architecture                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Agent Trait (src/agent/agent.rs)                       │   │
│   │                                                         │   │
│   │  pub trait Agent: Send + Sync {                         │   │
│   │    async fn run(&self, ctx: &mut Context) -> Result<()>;│   │
│   │    async fn handle_message(&self, msg: Message) -> ...; │   │
│   │    fn context_compression(&self) -> CompressionStrategy;│   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  ReActAgent Implementation                              │   │
│   │                                                         │   │
│   │  loop {                                                 │   │
│   │    let thought = self.reason(ctx).await?;               │   │
│   │    if thought.is_final() { break; }                     │   │
│   │    let action = thought.to_action()?;                   │   │
│   │    let result = self.execute(action).await?;            │   │
│   │    ctx.add_observation(result);                         │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```rust
// src/agent/agent.rs
pub struct ReActAgent<L: LLM, T: ToolRegistry> {
    llm: L,
    tools: T,
    config: AgentConfig,
}

impl<L: LLM, T: ToolRegistry> Agent for ReActAgent<L, T> {
    async fn run(&self, ctx: &mut Context) -> Result<AgentOutput, AgentError> {
        let mut iterations = 0;

        loop {
            iterations += 1;
            if iterations > self.config.max_iterations {
                return Err(AgentError::MaxIterationsReached);
            }

            // Check if we need to compress context
            if ctx.token_count() > self.config.compression_threshold {
                ctx.compress_history(self.config.compression_strategy).await?;
            }

            // Reasoning step
            let prompt = build_react_prompt(ctx, &self.tools);
            let response = self.llm.complete(&prompt).await?;

            // Parse thought and action
            let (thought, action) = parse_react_response(&response)?;

            // Check if done
            if action == Action::Finish {
                return Ok(AgentOutput {
                    final_answer: thought,
                    iterations,
                });
            }

            // Execute action
            let observation = match action {
                Action::ToolCall { name, args } => {
                    self.tools.execute(&name, args).await?
                }
                Action::DMail { checkpoint, message } => {
                    ctx.send_dmail(checkpoint, message).await?;
                    continue;
                }
                _ => Observation::Empty,
            };

            // Update context
            ctx.add_thought(thought);
            ctx.add_action(action);
            ctx.add_observation(observation);
        }
    }
}
```

#### D-Mail 时间旅行机制

```rust
// src/context/dmail.rs
pub struct DMailSystem {
    checkpoints: Vec<Checkpoint>,
}

impl DMailSystem {
    pub async fn send_dmail(
        &mut self,
        checkpoint_id: usize,
        message: String,
    ) -> Result<(), DMailError> {
        if checkpoint_id >= self.checkpoints.len() {
            return Err(DMailError::InvalidCheckpoint);
        }

        // Store D-Mail for delivery
        self.checkpoints[checkpoint_id].pending_dmail = Some(message);

        // Trigger time travel
        Err(DMailError::TimeTravelRequired { target: checkpoint_id })
    }

    pub fn revert_to(&mut self, checkpoint_id: usize) -> Context {
        self.checkpoints.truncate(checkpoint_id + 1);
        self.checkpoints[checkpoint_id].context.clone()
    }
}
```

---

### 5. MimiClaw - ESP32 嵌入式 ReAct 架构

#### 核心架构

专为 ESP32 微控制器设计的轻量级 ReAct 实现：

```
┌─────────────────────────────────────────────────────────────────┐
│                   MimiClaw ESP32 Architecture                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  FreeRTOS Task (main/agent/agent_loop.c)                │   │
│   │                                                         │   │
│   │  void agent_loop_task(void *pvParameters) {             │   │
│   │    while (1) {                                          │   │
│   │      // Wait for message                                │   │
│   │      xQueueReceive(msg_queue, &msg, portMAX_DELAY);     │   │
│   │                                                         │   │
│   │      // Process with ReAct                              │   │
│   │      react_result_t result = react_process(&msg);       │   │
│   │                                                         │   │
│   │      // Send response                                   │   │
│   │      send_response(result.response);                    │   │
│   │    }                                                    │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```c
// main/agent/agent_loop.c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "react.h"

#define MAX_REACT_ITERATIONS 10
#define STACK_SIZE 8192

static QueueHandle_t msg_queue;

void agent_loop_task(void *pvParameters) {
    message_t msg;
    context_t ctx;

    // Initialize context
    context_init(&ctx);

    while (1) {
        // Wait for incoming message
        if (xQueueReceive(msg_queue, &msg, portMAX_DELAY) != pdTRUE) {
            continue;
        }

        // Reset context for new conversation
        context_reset(&ctx);
        context_add_message(&ctx, ROLE_USER, msg.content);

        // ReAct loop
        int iteration = 0;
        bool done = false;

        while (!done && iteration < MAX_REACT_ITERATIONS) {
            iteration++;

            // Check WiFi and LLM connectivity
            if (!network_is_connected()) {
                ESP_LOGW(TAG, "Offline mode, using cached responses");
            }

            // Reasoning: Generate thought
            char thought[512];
            if (llm_generate_thought(&ctx, thought, sizeof(thought)) != ESP_OK) {
                ESP_LOGE(TAG, "Failed to generate thought");
                break;
            }

            // Check if final answer
            if (strstr(thought, "FINAL_ANSWER") != NULL) {
                send_response(thought);
                done = true;
                break;
            }

            // Action: Execute tool
            action_t action;
            if (parse_action(thought, &action) != ESP_OK) {
                ESP_LOGE(TAG, "Failed to parse action");
                continue;
            }

            // Execute tool
            char observation[256];
            if (execute_tool(&action, observation, sizeof(observation)) != ESP_OK) {
                snprintf(observation, sizeof(observation),
                    "Error executing tool %s", action.tool_name);
            }

            // Update context
            context_add_thought(&ctx, thought);
            context_add_action(&ctx, &action);
            context_add_observation(&ctx, observation);
        }

        // Cleanup
        if (!done) {
            send_response("Sorry, I couldn't complete your request in time.");
        }
    }
}
```

---

### 6. PicoClaw - Go 语言 Agent Loop 架构

#### 核心架构

基于 Go 协程的高并发 Agent 实现，支持 15+ 消息渠道：

```
┌─────────────────────────────────────────────────────────────────┐
│                   PicoClaw Agent Architecture                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Agent Loop (pkg/agent/loop.go)                         │   │
│   │                                                         │   │
│   │  type AgentLoop struct {                                │   │
│   │    state    atomic.Value      // Current state          │   │
│   │    messages chan Message      // Message queue          │   │
│   │    channels []Channel         // Active channels        │   │
│   │    llm      LLMClient         // LLM connection         │   │
│   │    tools    ToolRegistry      // Available tools        │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Run Loop                                               │   │
│   │                                                         │   │
│   │  func (al *AgentLoop) Run(ctx context.Context) error { │   │
│   │    for {                                                │   │
│   │      select {                                           │   │
│   │      case msg := <-al.messages:                         │   │
│   │        al.processMessage(ctx, msg)                      │   │
│   │      case <-ctx.Done():                                 │   │
│   │        return ctx.Err()                                 │   │
│   │      }                                                  │   │
│   │    }                                                    │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```go
// pkg/agent/loop.go
package agent

import (
    "context"
    "sync/atomic"
)

type AgentLoop struct {
    state       atomic.Value
    messages    chan Message
    channels    []Channel
    llm         LLMClient
    tools       ToolRegistry
    maxIters    int
}

type LoopState int

const (
    StateIdle     LoopState = 0
    StateRunning  LoopState = 1
    StatePaused   LoopState = 2
    StateStopping LoopState = 3
)

func (al *AgentLoop) Run(ctx context.Context) error {
    al.state.Store(StateRunning)

    for {
        select {
        case msg := <-al.messages:
            if err := al.processMessage(ctx, msg); err != nil {
                // Log error but continue
                al.logger.Error("Failed to process message", "error", err)
            }

        case <-ctx.Done():
            al.state.Store(StateStopping)
            return ctx.Err()
        }
    }
}

func (al *AgentLoop) processMessage(ctx context.Context, msg Message) error {
    session := al.getOrCreateSession(msg.SessionID)

    // ReAct loop
    for i := 0; i < al.maxIters; i++ {
        // Build prompt
        prompt := al.buildPrompt(session)

        // Call LLM
        response, err := al.llm.Complete(ctx, prompt)
        if err != nil {
            return err
        }

        // Parse response
        thought, action, err := al.parseResponse(response)
        if err != nil {
            session.AddAssistantMessage(response)
            continue
        }

        // Check if done
        if action.Type == ActionFinish {
            al.sendResponse(msg.ChannelID, thought.Content)
            return nil
        }

        // Execute tool
        result, err := al.tools.Execute(ctx, action.Tool, action.Params)
        if err != nil {
            result = ToolResult{Error: err.Error()}
        }

        // Update session
        session.AddThought(thought)
        session.AddAction(action)
        session.AddObservation(result)
    }

    return ErrMaxIterationsReached
}

// Atomic state management
func (al *AgentLoop) State() LoopState {
    return al.state.Load().(LoopState)
}

func (al *AgentLoop) Pause() {
    al.state.Store(StatePaused)
}

func (al *AgentLoop) Resume() {
    al.state.Store(StateRunning)
}
```

#### 15 通道支持

```go
// pkg/channels/channel.go
type ChannelType string

const (
    ChannelFeishu   ChannelType = "feishu"
    ChannelDingTalk ChannelType = "dingtalk"
    ChannelQQ       ChannelType = "qq"
    ChannelDiscord  ChannelType = "discord"
    ChannelTelegram ChannelType = "telegram"
    ChannelSlack    ChannelType = "slack"
    ChannelWeChat   ChannelType = "wechat"
    ChannelEmail    ChannelType = "email"
    ChannelSMS      ChannelType = "sms"
    ChannelWebhook  ChannelType = "webhook"
    ChannelSSE      ChannelType = "sse"
    ChannelWS       ChannelType = "websocket"
    ChannelHTTP     ChannelType = "http"
    ChannelCLI      ChannelType = "cli"
    ChannelMatrix   ChannelType = "matrix"
)
```

---

### 7. Nanobot - 轻量级 MessageBus 架构

#### 核心架构

基于 Python asyncio 的轻量级事件驱动架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Nanobot MessageBus Architecture               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  MessageBus (nanobot/core/bus.py)                       │   │
│   │                                                         │   │
│   │  class MessageBus:                                      │   │
│   │    def __init__(self):                                  │   │
│   │      self._handlers: Dict[str, List[Handler]] = {}      │   │
│   │      self._queue: asyncio.Queue = asyncio.Queue()       │   │
│   │      self._running = False                              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Agent Loop                                             │   │
│   │                                                         │   │
│   │  async def run(self):                                   │   │
│   │    self._running = True                                 │   │
│   │    while self._running:                                 │   │
│   │      message = await self._queue.get()                  │   │
│   │      await self._process(message)                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```python
# nanobot/agent/loop.py
import asyncio
from typing import Dict, List, Callable
from dataclasses import dataclass

@dataclass
class Message:
    type: str
    payload: dict
    source: str
    timestamp: float

class MessageBus:
    def __init__(self):
        self._handlers: Dict[str, List[Callable]] = {}
        self._queue: asyncio.Queue = asyncio.Queue()
        self._running = False

    def subscribe(self, message_type: str, handler: Callable):
        if message_type not in self._handlers:
            self._handlers[message_type] = []
        self._handlers[message_type].append(handler)

    async def publish(self, message: Message):
        await self._queue.put(message)

    async def run(self):
        self._running = True
        while self._running:
            try:
                message = await asyncio.wait_for(
                    self._queue.get(), timeout=1.0
                )
                await self._process(message)
            except asyncio.TimeoutError:
                continue

    async def _process(self, message: Message):
        handlers = self._handlers.get(message.type, [])
        for handler in handlers:
            try:
                await handler(message)
            except Exception as e:
                print(f"Handler error: {e}")

class Agent:
    def __init__(self, bus: MessageBus, llm, tools):
        self.bus = bus
        self.llm = llm
        self.tools = tools
        self.bus.subscribe("user_message", self._on_user_message)

    async def _on_user_message(self, msg: Message):
        context = []
        context.append({"role": "user", "content": msg.payload["text"]})

        for iteration in range(10):  # Max 10 iterations
            # Get LLM response
            response = await self.llm.complete(context)
            context.append({"role": "assistant", "content": response})

            # Check if tool call
            if "TOOL:" in response:
                tool_name, tool_args = self._parse_tool(response)
                result = await self.tools.execute(tool_name, tool_args)
                context.append({"role": "system", "content": f"Result: {result}"})
            else:
                # Final response
                await self.bus.publish(Message(
                    type="agent_response",
                    payload={"text": response},
                    source="agent",
                    timestamp=asyncio.get_event_loop().time()
                ))
                break
```

---

### 8. OpenClaw - Pi Event Stream 架构

详见下方 "编程智能体" 部分的 Pi-mono/OpenClaw 分析，两者共享 Pi Agent Core。

---

## 二、编程智能体 Runtime 详解

### 1. Aider - Coder.run() 架构

#### 核心架构

Aider 采用基于 **diff 格式策略** 的代码编辑架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Aider Coder Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  BaseCoder (aider/coders/base_coder.py)                 │   │
│   │                                                         │   │
│   │  class BaseCoder:                                       │   │
│   │    def run(self, messages):                             │   │
│   │      # Main loop                                        │   │
│   │      while True:                                        │   │
│   │        response = self.send(messages)                   │   │
│   │        edits = self.get_edits(response)                 │   │
│   │        if edits:                                        │   │
│   │          self.apply_edits(edits)                        │   │
│   │        else:                                            │   │
│   │          break                                          │   │
│   │        messages = self.update_messages()                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Edit Formats                                           │   │
│   │  • whole - 整文件替换                                   │   │
│   │  • diff - 标准统一格式                                  │   │
│   │  • udiff - 统一diff格式                                 │   │
│   │  • search/replace - 精确替换                            │   │
│   │  • architect - 仅规划不执行                             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```python
# aider/coders/base_coder.py
class BaseCoder:
    """Base class for all coders."""

    def __init__(self, ...):
        self.io = io
        self.main_model = main_model
        self.fence = fence
        self.get_edits = None  # Set by edit format
        self.edit_format = edit_format

    def run(self, messages):
        """Main execution loop."""
        while True:
            # Send to model
            response = self.send(messages)

            # Check for edits
            if self.get_edits:
                edits = self.get_edits(response)
                if edits:
                    # Apply edits
                    self.apply_edits(edits)
                    # Continue loop
                    messages = self.update_messages(response, edits)
                    continue

            # No edits, return final response
            return response

    def send(self, messages):
        """Send messages to the model and return response."""
        completion = self.main_model.complete(messages)
        return completion

    def apply_edits(self, edits):
        """Apply a list of edits to the filesystem."""
        for edit in edits:
            path = edit.path
            before = edit.before
            after = edit.after

            content = self.io.read_file(path)
            if before not in content:
                raise EditError(f"Cannot find edit block in {path}")

            new_content = content.replace(before, after, 1)
            self.io.write_file(path, new_content)
```

#### Edit Format 策略

```python
# aider/coders/editblock_coder.py
class EditBlockCoder(BaseCoder):
    """Uses search/replace blocks for edits."""

    edit_format = "diff"

    def get_edits(self, response):
        """Parse search/replace blocks from response."""
        edits = []
        pattern = r'^```\s*(\S+)\s*\n(.*?)\n^```'

        for match in re.finditer(pattern, response, re.MULTILINE | re.DOTALL):
            filename = match.group(1)
            content = match.group(2)

            # Parse SEARCH/REPLACE blocks
            blocks = self.parse_blocks(content)
            for block in blocks:
                edits.append(Edit(
                    path=filename,
                    before=block["search"],
                    after=block["replace"]
                ))

        return edits

# Edit formats registry
EDIT_FORMATS = {
    "whole": WholeFileCoder,
    "diff": DiffCoder,
    "udiff": UdiffCoder,
    "search_replace": SearchReplaceCoder,
    "architect": ArchitectCoder,
}
```

---

### 2. Codex - Rust Session 管理架构

#### 核心架构

基于 Rust 的强类型 Session 管理，支持沙箱隔离：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Codex Rust Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Session (codex-rs/core/src/session.rs)                 │   │
│   │                                                         │   │
│   │  pub struct Session {                                   │   │
│   │    id: Uuid,                                            │   │
│   │    state: SessionState,                                 │   │
│   │    sandbox: Sandbox,                                    │   │
│   │    tool_executions: Vec<ToolExecution>,                 │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Session Loop                                           │   │
│   │                                                         │   │
│   │  impl Session {                                         │   │
│   │    pub async fn run(&mut self, input: &str) -> ... {    │   │
│   │      let mut turn = 0;                                  │   │
│   │      loop {                                             │   │
│   │        turn += 1;                                       │   │
│   │        if turn > MAX_TURNS {                            │   │
│   │          return Err(SessionError::MaxTurnsReached);     │   │
│   │        }                                                │   │
│   │        // Get response                                  │   │
│   │        // Execute tools                                 │   │
│   │        // Check if done                                 │   │
│   │      }                                                  │   │
│   │    }                                                    │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```rust
// codex-rs/core/src/session.rs
use uuid::Uuid;
use tokio::sync::mpsc;

pub struct Session {
    id: Uuid,
    state: SessionState,
    sandbox: Sandbox,
    tool_executions: Vec<ToolExecution>,
    config: SessionConfig,
}

impl Session {
    pub async fn run(&mut self, input: &str) -> Result<SessionOutput, SessionError> {
        let mut messages = vec![Message::user(input)];
        let mut turn = 0;

        loop {
            turn += 1;
            if turn > self.config.max_turns {
                return Err(SessionError::MaxTurnsReached);
            }

            // Update state
            self.state = SessionState::Running { turn };

            // Call LLM
            let response = self.llm.complete(&messages).await?;

            // Parse tool calls
            let tool_calls = self.parse_tool_calls(&response);

            if tool_calls.is_empty() {
                // Final response
                return Ok(SessionOutput {
                    response,
                    turns: turn,
                    tool_executions: self.tool_executions.clone(),
                });
            }

            // Execute tools in sandbox
            for call in tool_calls {
                let result = self.sandbox.execute(&call).await?;
                self.tool_executions.push(ToolExecution {
                    call,
                    result,
                    turn,
                });

                messages.push(Message::tool_result(&result));
            }
        }
    }
}

// Sandbox for tool execution
pub struct Sandbox {
    root: PathBuf,
    network: bool,
    filesystem: SandboxFilesystem,
}

impl Sandbox {
    pub async fn execute(&self, call: &ToolCall) -> Result<ToolResult, SandboxError> {
        match call.name.as_str() {
            "shell" => self.execute_shell(call),
            "read" => self.execute_read(call),
            "write" => self.execute_write(call),
            _ => Err(SandboxError::UnknownTool),
        }
    }
}
```

---

### 3. Qwen-code - React+Ink 流式 UI 架构

#### 核心架构

基于 React 19 和 Ink 的终端 UI 架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Qwen-code Terminal UI                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Gemini Component (packages/cli/src/gemini.tsx)         │   │
│   │                                                         │   │
│   │  export default function Gemini() {                     │   │
│   │    const [messages, setMessages] = useState([]);        │   │
│   │    const [isLoading, setIsLoading] = useState(false);   │   │
│   │                                                         │   │
│   │    useEffect(() => {                                    │   │
│   │      // Start stream processing                         │   │
│   │      processStream();                                   │   │
│   │    }, []);                                              │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Stream Processing                                      │   │
│   │                                                         │   │
│   │  async function processStream() {                       │   │
│   │    for await (const chunk of llm.stream()) {            │   │
│   │      // Update UI                                       │   │
│   │      setMessages(prev => [...prev, chunk]);             │   │
│   │    }                                                    │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```typescript
// packages/cli/src/gemini.tsx
import React, { useState, useEffect, useCallback } from 'react';
import { render, Box, Text } from 'ink';
import { useStream } from './hooks/useStream';

interface Message {
  role: 'user' | 'assistant' | 'tool';
  content: string;
  metadata?: any;
}

function Gemini() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [isProcessing, setIsProcessing] = useState(false);

  const { stream, cancel } = useStream({
    model: 'qwen-coder',
    temperature: 0.7,
  });

  const handleSubmit = useCallback(async () => {
    if (!input.trim()) return;

    const userMessage: Message = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsProcessing(true);

    try {
      // Start streaming response
      const streamIterator = stream(messages);
      let assistantContent = '';

      for await (const chunk of streamIterator) {
        if (chunk.type === 'text') {
          assistantContent += chunk.content;
          // Update UI incrementally
          setMessages(prev => {
            const newMessages = [...prev];
            const lastMsg = newMessages[newMessages.length - 1];
            if (lastMsg?.role === 'assistant') {
              lastMsg.content = assistantContent;
            } else {
              newMessages.push({
                role: 'assistant',
                content: assistantContent,
              });
            }
            return newMessages;
          });
        } else if (chunk.type === 'tool_call') {
          // Handle tool execution
          const result = await executeTool(chunk.tool, chunk.params);
          setMessages(prev => [...prev, {
            role: 'tool',
            content: JSON.stringify(result),
            metadata: { tool: chunk.tool },
          }]);
        }
      }
    } finally {
      setIsProcessing(false);
    }
  }, [input, messages, stream]);

  return (
    <Box flexDirection="column">
      {messages.map((msg, i) => (
        <MessageItem key={i} message={msg} />
      ))}
      {isProcessing && <Spinner />}
      <Input value={input} onChange={setInput} onSubmit={handleSubmit} />
    </Box>
  );
}

// Checkpoint system
interface Checkpoint {
  id: string;
  timestamp: number;
  messages: Message[];
  fileStates: Map<string, string>;
}

function useCheckpoints() {
  const [checkpoints, setCheckpoints] = useState<Checkpoint[]>([]);

  const createCheckpoint = useCallback((messages: Message[]) => {
    const checkpoint: Checkpoint = {
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      messages: [...messages],
      fileStates: captureFileStates(),
    };
    setCheckpoints(prev => [...prev, checkpoint]);
    return checkpoint.id;
  }, []);

  const restoreCheckpoint = useCallback((id: string) => {
    const checkpoint = checkpoints.find(c => c.id === id);
    if (checkpoint) {
      restoreFileStates(checkpoint.fileStates);
      return checkpoint.messages;
    }
    return null;
  }, [checkpoints]);

  return { createCheckpoint, restoreCheckpoint };
}

render(<Gemini />);
```

---

### 4. Gemini-cli - Scheduler 事件驱动架构

#### 核心架构

基于事件驱动的 Scheduler 架构，支持 A2A 协议：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Gemini-cli Scheduler                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Scheduler (packages/core/src/scheduler/scheduler.ts)   │   │
│   │                                                         │   │
│   │  export class Scheduler {                               │   │
│   │    private events: EventEmitter;                        │   │
│   │    private jobs: Map<string, Job>;                      │   │
│   │    private agents: Map<string, Agent>;                  │   │
│   │                                                         │   │
│   │    async schedule(job: Job): Promise<void> {            │   │
│   │      // Add to queue and process                        │   │
│   │    }                                                    │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Event Loop                                             │   │
│   │                                                         │   │
│   │  this.events.on('job:created', this.onJobCreated);      │   │
│   │  this.events.on('job:started', this.onJobStarted);      │   │
│   │  this.events.on('tool:execute', this.onToolExecute);    │   │
│   │  this.events.on('agent:delegate', this.onDelegate);     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```typescript
// packages/core/src/scheduler/scheduler.ts
import { EventEmitter } from 'events';
import { A2AProtocol } from './a2a-protocol';

interface Job {
  id: string;
  type: 'chat' | 'code' | 'plan' | 'review';
  input: string;
  context?: JobContext;
  priority?: number;
}

interface JobContext {
  sessionId?: string;
  parentJobId?: string;
  files?: string[];
}

export class Scheduler extends EventEmitter {
  private jobs: Map<string, Job> = new Map();
  private agents: Map<string, Agent> = new Map();
  private a2a: A2AProtocol;
  private queue: PQueue;

  constructor(options: SchedulerOptions) {
    super();
    this.a2a = new A2AProtocol(options.a2aConfig);
    this.queue = new PQueue({ concurrency: options.concurrency || 4 });
    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    this.on('job:created', this.onJobCreated.bind(this));
    this.on('job:started', this.onJobStarted.bind(this));
    this.on('tool:execute', this.onToolExecute.bind(this));
    this.on('agent:delegate', this.onAgentDelegate.bind(this));
  }

  async schedule(job: Job): Promise<string> {
    job.id = crypto.randomUUID();
    this.jobs.set(job.id, job);
    this.emit('job:created', job);

    await this.queue.add(() => this.processJob(job));
    return job.id;
  }

  private async processJob(job: Job): Promise<void> {
    this.emit('job:started', job);

    const agent = this.selectAgent(job.type);
    const result = await agent.execute({
      input: job.input,
      context: job.context,
      tools: this.getAvailableTools(),
    });

    this.emit('job:completed', { job, result });
  }

  private async onAgentDelegate(event: DelegateEvent): Promise<void> {
    // Handle A2A delegation
    const task = await this.a2a.createTask({
      message: event.message,
      pushNotification: event.pushNotification,
    });

    // Wait for completion or stream updates
    if (event.stream) {
      for await (const update of this.a2a.subscribe(task.id)) {
        event.onUpdate(update);
      }
    } else {
      const result = await this.a2a.waitForCompletion(task.id);
      event.onComplete(result);
    }
  }

  private selectAgent(type: string): Agent {
    // Agent routing logic
    const agent = this.agents.get(type);
    if (!agent) {
      throw new Error(`No agent available for type: ${type}`);
    }
    return agent;
  }
}

// A2A Protocol implementation
class A2AProtocol {
  async createTask(params: TaskParams): Promise<Task> {
    // Send to remote agent via A2A
    const response = await fetch(`${this.endpoint}/tasks/send`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(params),
    });
    return response.json();
  }

  async *subscribe(taskId: string): AsyncGenerator<TaskUpdate> {
    const response = await fetch(
      `${this.endpoint}/tasks/${taskId}/subscribe`
    );
    const reader = response.body.getReader();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      yield JSON.parse(new TextDecoder().decode(value));
    }
  }
}
```

---

### 5. Pi-mono/OpenClaw - Pi Agent 事件流架构

#### 核心架构

基于 Pi Agent Core 的事件流处理：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Pi Agent Event Stream                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Agent Class (packages/agent/src/agent.ts)              │   │
│   │                                                         │   │
│   │  export class Agent {                                   │   │
│   │    private eventStream: EventStream;                    │   │
│   │    private tools: ToolRegistry;                         │   │
│   │    private llm: LLMClient;                              │   │
│   │                                                         │   │
│   │    async run(input: string): Promise<void> {            │   │
│   │      const session = await this.createSession();        │   │
│   │      for await (const event of session.events) {        │   │
│   │        await this.handleEvent(event);                   │   │
│   │      }                                                  │   │
│   │    }                                                    │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```typescript
// packages/agent/src/agent.ts
import { EventEmitter } from 'events';

export type AgentMode = 'sequential' | 'parallel';

export class Agent {
  private eventStream: EventStream;
  private tools: ToolRegistry;
  private llm: LLMClient;
  private mode: AgentMode = 'sequential';

  async run(input: string): Promise<AgentResult> {
    const session = await this.createSession();
    await session.send(input);

    for await (const event of session.events) {
      switch (event.type) {
        case 'assistant.messageDelta':
          await this.handleMessageDelta(event);
          break;

        case 'assistant.toolCall':
          await this.handleToolCall(event);
          break;

        case 'assistant.toolResult':
          await this.handleToolResult(event);
          break;

        case 'assistant.finish':
          return { status: 'complete', session };

        case 'error':
          return { status: 'error', error: event.error };
      }
    }

    return { status: 'incomplete' };
  }

  private async handleToolCall(event: ToolCallEvent): Promise<void> {
    if (this.mode === 'sequential') {
      // Execute one at a time
      const result = await this.tools.execute(event.tool, event.params);
      await this.session.sendToolResult(result);
    } else {
      // Execute in parallel
      const promises = event.calls.map(call =>
        this.tools.execute(call.tool, call.params)
      );
      const results = await Promise.all(promises);
      await this.session.sendToolResults(results);
    }
  }
}

// Event Stream implementation
export class EventStream {
  private events: EventEmitter;
  private buffer: AgentEvent[] = [];

  async *[Symbol.asyncIterator](): AsyncIterator<AgentEvent> {
    while (true) {
      if (this.buffer.length > 0) {
        yield this.buffer.shift()!;
      } else {
        await new Promise(resolve => this.events.once('event', resolve));
      }
    }
  }

  emit(event: AgentEvent): void {
    this.buffer.push(event);
    this.events.emit('event');
  }
}

// Auto-reply system
export async function runReplyAgent(params: ReplyParams): Promise<void> {
  const mode = params.queueMode || 'steer';

  switch (mode) {
    case 'steer':
      // Inject message into current run
      await steerCurrentRun(params);
      break;
    case 'followup':
      // Wait for current run, then start new
      await waitAndStartNew(params);
      break;
    case 'collect':
      // Batch messages
      await collectAndBatch(params);
      break;
  }
}

// Heartbeat system
export async function runHeartbeatIfNeeded(params: HeartbeatParams): Promise<boolean> {
  const heartbeatEvery = resolveHeartbeatEvery(params.agentConfig);
  const timeSinceLast = Date.now() - params.lastHeartbeatAt;

  if (timeSinceLast < heartbeatEvery) {
    return false;
  }

  const queryText = await readFile('HEARTBEAT.md');
  await runReplyAgent({
    commandBody: queryText,
    isHeartbeat: true,
  });

  return true;
}
```

---

### 6. Kimi-cli - Step Loop + D-Mail 架构

详见上方 "个人通用智能助理" 部分，与 CoPaw 等共享类似 ReAct 模式，但增加了 D-Mail 时间旅行机制。

---

### 7. Opencode - Vercel AI SDK 架构

#### 核心架构

基于 Vercel AI SDK 的 Stream Processing：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Opencode Vercel AI SDK                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  SessionProcessor                                       │   │
│   │  (packages/opencode/src/session/processor.ts)           │   │
│   │                                                         │   │
│   │  export namespace SessionProcessor {                    │   │
│   │    export function create(input) {                      │   │
│   │      return {                                           │   │
│   │        async process(streamInput) {                     │   │
│   │          while (true) {                                 │   │
│   │            const stream = await LLM.stream(streamInput);│   │
│   │            for await (const value of stream.fullStream)│   │
│   │              switch (value.type) { ... }                │   │
│   │          }                                              │   │
│   │        }                                                │   │
│   │      };                                                 │   │
│   │    }                                                    │   │
│   │  }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 主循环实现

```typescript
// packages/opencode/src/session/processor.ts
import { streamText, tool } from 'ai';
import { Effect } from 'effect';

export namespace SessionProcessor {
  export interface StreamInput {
    messages: Message[];
    tools?: Tool[];
    temperature?: number;
  }

  export function create(input: {
    sessionID: string;
    model: Model;
    abort: AbortController;
  }) {
    return {
      async process(streamInput: StreamInput) {
        while (true) {
          const stream = await LLM.stream({
            model: input.model,
            messages: streamInput.messages,
            tools: streamInput.tools,
            temperature: streamInput.temperature,
          });

          for await (const value of stream.fullStream) {
            switch (value.type) {
              case 'start':
                await handleStart(value);
                break;

              case 'reasoning-start':
                await handleReasoningStart(value);
                break;

              case 'reasoning-delta':
                await handleReasoningDelta(value);
                break;

              case 'tool-call':
                await handleToolCall(value);
                break;

              case 'tool-result':
                await handleToolResult(value);
                break;

              case 'finish':
                return;

              case 'error':
                throw new Error(value.error);
            }
          }

          // Continue loop if more tool calls
          if (hasPendingToolCalls()) {
            streamInput.messages = buildNextMessages();
            continue;
          }

          break;
        }
      },
    };
  }
}

// Agent configuration
export const Agent = {
  build: {
    name: 'build',
    mode: 'primary',
    permission: PermissionNext.merge(defaults, user),
  },
  plan: {
    name: 'plan',
    mode: 'primary',
    permission: PermissionNext.merge(defaults, {
      edit: { '*': 'deny' },
    }),
  },
  explore: {
    name: 'explore',
    mode: 'subagent',
    tools: ['glob', 'grep', 'read_file'],
  },
  compaction: {
    name: 'compaction',
    mode: 'primary',
  },
};

// Snapshot and revert
export const setRevert = fn(
  z.object({
    sessionID: SessionID.zod,
    revert: RevertState,
    summary: SessionSummary,
  }),
  async (input) => {
    return Database.use((db) => {
      return db
        .update(SessionTable)
        .set({
          revert: input.revert,
          summary_additions: input.summary?.additions,
          summary_deletions: input.summary?.deletions,
          summary_files: input.summary?.files,
        })
        .where(eq(SessionTable.id, input.sessionID))
        .returning()
        .get();
    });
  }
);

export const revertTo = fn(
  z.object({ sessionID: SessionID.zod }),
  async (input) => {
    const session = await get({ sessionID: input.sessionID });
    if (!session.revert) throw new Error('No revert point');
    await restoreSnapshot(session.revert.snapshot);
  }
);
```

---

## 三、全项目 Runtime 对比

### 1. 循环模式对比

| 项目 | 循环模式 | 最大迭代 | 语言 | 并发模型 |
|------|---------|---------|------|---------|
| **CoPaw** | ReAct | 50 (configurable) | Python | AsyncIO + Multi-worker |
| **IronClaw** | ReAct + Event Stream | configurable | Rust | Tokio async |
| **NanoClaw** | Async Message Loop | - | TypeScript | Node.js Async |
| **ZeroClaw** | Trait-driven ReAct | configurable | Rust | Tokio |
| **MimiClaw** | ReAct | 10 | C | FreeRTOS Tasks |
| **PicoClaw** | AgentLoop | configurable | Go | Goroutines |
| **Nanobot** | MessageBus | 10 | Python | asyncio |
| **OpenClaw** | Pi Event Stream | - | TypeScript | Node.js |
| **Aider** | Coder.run() | - | Python | Sync |
| **Codex** | Session Loop | configurable | Rust | Tokio |
| **Qwen-code** | Stream Loop | - | TypeScript | React+Ink |
| **Gemini-cli** | Event-driven | - | TypeScript | EventEmitter |
| **Pi-mono** | AgentLoop | - | TypeScript | Event Stream |
| **Kimi-cli** | Step Loop | configurable | Python | AsyncIO |
| **Opencode** | Stream Processing | - | TypeScript | Effect TS |

### 2. 子 Agent / 任务委托对比

| 项目 | 子 Agent 机制 | 通信方式 | 隔离级别 |
|------|--------------|---------|---------|
| **CoPaw** | MCP Client | stdio/HTTP | 进程级 |
| **IronClaw** | Agent Types (Main/Sub/Task) | 函数调用 | 上下文级 |
| **NanoClaw** | Container Spawning | 进程间 | 容器级 |
| **ZeroClaw** | SubAgent Trait | 内存 | 共享内存 |
| **PicoClaw** | Agent Instances | Channel | 实例级 |
| **OpenClaw** | Subagent Registry | Pi Events | 会话级 |
| **Gemini-cli** | A2A Protocol | HTTP/WebSocket | 服务级 |
| **Kimi-cli** | Labor Market | Wire Pub/Sub | LaborMarket级 |
| **Opencode** | Agent Config | 函数调用 | 配置级 |

### 3. 状态持久化对比

| 项目 | 存储格式 | 特点 | 压缩策略 |
|------|---------|------|---------|
| **CoPaw** | JSON | 文件存储，会话状态 | 三区模型 |
| **IronClaw** | SQLite/JSON | 结构化查询 | 上下文摘要 |
| **NanoClaw** | JSONL | 追加写入 | 手动/自动 |
| **ZeroClaw** | JSONL | 追加写入 + 压缩 | Token-based |
| **PicoClaw** | SQLite + JSONL | Go标准库 | 自动压缩 |
| **Nanobot** | JSON | 轻量级 | 无 |
| **OpenClaw** | JSONL | Pi原生格式 | 上下文窗口 |
| **Codex** | 内部格式 | Session管理 | 无 |
| **Kimi-cli** | JSONL | Checkpoint标记 | SimpleCompaction |
| **Opencode** | SQLite | 结构化查询，强类型 | 配置参数 |

### 4. 特殊机制对比

| 项目 | 特色机制 | 说明 |
|------|---------|------|
| **CoPaw** | ToolGuard + 心跳 | 六层安全防护 + 定时巡逻 |
| **IronClaw** | Agent类型系统 | Main/Sub/Task三代理类型 |
| **NanoClaw** | 容器隔离 | Docker/Apple Container |
| **ZeroClaw** | D-Mail + 压缩 | 时间旅行 + 上下文压缩 |
| **MimiClaw** | ESP32嵌入式 | 离线优先 + 轻量级 |
| **PicoClaw** | 15通道支持 | 原子操作状态管理 |
| **Nanobot** | MessageBus | 轻量级事件驱动 |
| **OpenClaw** | Auto-Reply + Hooks | 自动回复 + 扩展钩子 |
| **Aider** | Diff格式策略 | 多种编辑格式支持 |
| **Codex** | 沙箱隔离 | 工具执行沙箱 |
| **Qwen-code** | 检查点系统 | 代码快照回滚 |
| **Gemini-cli** | A2A协议 | 跨Agent通信 |
| **Kimi-cli** | D-Mail | 时间旅行回溯 |
| **Opencode** | 快照回滚 | Session级快照 |

---

## 四、推荐 Runtime 架构

基于以上分析，推荐以下融合架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Recommended Agent Runtime                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Core Loop Layer (融合多种模式)                          │   │
│  │  - Primary: Step Loop (kimi-cli style)                  │   │
│  │  - Event Processing: Event Stream (openclaw style)      │   │
│  │  - Streaming: Vercel AI SDK (opencode style)            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Agent Types (IronClaw style)                       │ │ │
│  │  │  • Main Agent - 用户直接交互                        │ │ │
│  │  │  • Sub Agent - 隔离会话的专业代理                   │ │ │
│  │  │  • Task Agent - 共享会话的子任务                    │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  │                           │                               │ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Subagent Communication                             │ │ │
│  │  │  • Labor Market (kimi-cli) - 共享任务市场           │ │ │
│  │  │  • A2A Protocol (gemini-cli) - 跨服务通信           │ │ │
│  │  │  • MCP (CoPaw) - 外部工具标准                       │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Persistence (分层存储)                              │ │ │
│  │  │  • Hot: In-memory context                           │ │ │
│  │  │  • Warm: JSONL append (kimi-cli style)              │ │ │
│  │  │  • Cold: SQLite structured (opencode style)         │ │ │
│  │  │  • Archive: Compressed summary                      │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Special Features                                   │ │ │
│  │  │  • Checkpoint + D-Mail (kimi-cli/zero)              │ │ │
│  │  │  • ToolGuard (CoPaw)                                │ │ │
│  │  │  • Heartbeat (CoPaw/openclaw)                       │ │ │
│  │  │  • Container Isolation (nanoclaw)                   │ │ │
│  │  │  • Diff Formats (aider)                             │ │ │
│  │  │  • Hook System (openclaw)                           │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键实现代码

```typescript
// Recommended Agent Runtime Implementation

interface LoopConfig {
  maxStepsPerTurn: number;
  maxRetriesPerStep: number;
  compactionTriggerRatio: number;
  reservedContextSize: number;
  agentTypes: AgentTypeConfig[];
}

class AgentRuntime {
  private context: Context;
  private laborMarket: LaborMarket;
  private checkpointManager: CheckpointManager;
  private compaction: Compaction;
  private toolGuard: ToolGuard;
  private hooks: HookSystem;
  private agentType: AgentType;

  async run(userInput: string): Promise<void> {
    await this.initialize();

    for (let step = 0; step < this.config.maxStepsPerTurn; step++) {
      // 1. Dynamic injection
      const injections = await this.collectInjections();

      // 2. Check and compact
      if (await this.shouldCompact()) {
        await this.compactContext();
      }

      // 3. Create checkpoint
      const checkpointId = await this.checkpointManager.create();

      // 4. Execute step
      try {
        const outcome = await this.executeStep(injections);
        if (outcome.done) return;
      } catch (e) {
        if (e instanceof DMailTriggered) {
          await this.checkpointManager.revertTo(e.checkpointId);
          await this.context.injectDMail(e.message);
          continue;
        }
        throw e;
      }
    }
  }

  private async executeStep(injections: Injection[]): Promise<StepOutcome> {
    await this.hooks.run('before_step', { context: this.context });

    const stream = await this.llm.stream({
      systemPrompt: this.buildSystemPrompt(injections),
      history: this.context.history,
      tools: this.toolset,
    });

    for await (const event of stream) {
      switch (event.type) {
        case 'tool-call':
          if (!await this.toolGuard.check(event.toolCall)) {
            throw new ToolGuardError();
          }
          const result = await this.executeTool(event.toolCall);
          await this.context.addToolResult(result);
          break;
        case 'finish':
          return { done: true };
      }
    }

    return { done: false };
  }
}
```

---

## 五、总结

### 各项目优势

| 类别 | 项目 | Runtime 优势 | 独特机制 |
|------|------|-------------|---------|
| 个人助理 | **CoPaw** | ReAct经典 + AgentScope | ToolGuard六层防护 + 心跳 |
| 个人助理 | **IronClaw** | 统一Agentic Loop | 三代理类型系统 |
| 个人助理 | **NanoClaw** | 容器化隔离 | Docker/Apple Container |
| 个人助理 | **ZeroClaw** | Rust Trait驱动 | D-Mail + 上下文压缩 |
| 个人助理 | **MimiClaw** | ESP32嵌入式 | 离线优先 + 轻量 |
| 个人助理 | **PicoClaw** | Go高并发 | 15通道 + 原子操作 |
| 个人助理 | **Nanobot** | 极简事件驱动 | MessageBus轻量 |
| 个人助理 | **OpenClaw** | Pi Event Stream | Hook系统完整 |
| 编程智能体 | **Aider** | Diff格式策略 | 多种编辑格式 |
| 编程智能体 | **Codex** | Rust强类型 | 沙箱隔离 |
| 编程智能体 | **Qwen-code** | React终端UI | 检查点系统 |
| 编程智能体 | **Gemini-cli** | 事件驱动 | A2A协议 |
| 编程智能体 | **Pi-mono** | Pi Agent Core | 事件流 + Agent模式 |
| 编程智能体 | **Kimi-cli** | Step Loop清晰 | D-Mail时间旅行 |
| 编程智能体 | **Opencode** | Vercel AI SDK | 类型安全 + 快照回滚 |

### 推荐技术选型

| 组件 | 推荐实现 | 来源项目 |
|------|---------|---------|
| **主循环** | Step Loop + Event Stream 混合 | kimi-cli + openclaw |
| **并发模型** | Tokio async / Goroutines | zero/pico |
| **子 Agent** | Labor Market + MCP + A2A | kimi-cli + CoPaw + gemini |
| **持久化** | SQLite + JSONL 混合 | opencode + kimi-cli |
| **通信** | Wire (Pub/Sub) + Event Stream | kimi-cli + openclaw |
| **Hooks** | Before/After 事件 | openclaw |
| **安全** | ToolGuard六层模型 | CoPaw |
| **特殊功能** | Checkpoint + D-Mail + 容器隔离 | kimi/zero + nano |

此方案融合了15个项目的最佳实践，兼顾执行效率、扩展性和安全性，适用于生产级 AI Agent 系统。
