# AI Agent Runtime Architecture Analysis

## Executive Summary

This document provides a comprehensive analysis of **agent runtime architectures** across **9 production AI agent projects**, spanning **4 technology ecosystems** (Rust, Go, Python, TypeScript). The analysis reveals distinct patterns in how agents execute their core reasoning loops, manage state, handle concurrency, and isolate execution contexts.

### Runtime Maturity Classification

| Tier | Projects | Characteristics |
|------|----------|-----------------|
| **Production-Grade** | IronClaw, ZeroClaw | Trait-driven extensibility, sophisticated state management, enterprise features |
| **Production-Ready** | NanoClaw, Nanobot, PicoClaw | Clean separation of concerns, container isolation, worker pools |
| **Specialized** | Edict, Agent-Browser, PinchTab | Domain-specific architectures, unique execution models |
| **Framework-Based** | CoPaw | Built on existing frameworks (AgentScope), higher-level abstractions |

---

## 1. IronClaw: The LoopDelegate Pattern

IronClaw represents the most sophisticated agent runtime architecture, featuring a **trait-based shared agentic loop** that powers three distinct execution modes through a unified abstraction.

### 1.1 Core LoopDelegate Trait

**Source**: `ironclaw/src/agent/agentic_loop.rs:1-80`

```rust
#[async_trait]
pub trait LoopDelegate: Send + Sync {
    /// Check for external signals (pause, stop, shutdown)
    async fn check_signals(&self) -> LoopSignal;

    /// Hook before LLM call - allows modification of reasoning context
    async fn before_llm_call(
        &self,
        reason_ctx: &mut ReasoningContext,
        iteration: usize,
    ) -> Option<LoopOutcome>;

    /// Execute the actual LLM call
    async fn call_llm(
        &self,
        reasoning: &Reasoning,
        reason_ctx: &mut ReasoningContext,
        iteration: usize,
    ) -> Result<RespondOutput, Error>;

    /// Handle text response (vs tool calls)
    async fn handle_text_response(
        &self,
        text: &str,
        reason_ctx: &mut ReasoningContext,
    ) -> TextAction;

    /// Execute tool calls and return outcome
    async fn execute_tool_calls(
        &self,
        tool_calls: Vec<ToolCall>,
        content: Option<String>,
        reason_ctx: &mut ReasoningContext,
    ) -> Result<Option<LoopOutcome>, Error>;

    /// Post-iteration hook for cleanup or continuation decisions
    async fn after_iteration(
        &self,
        reason_ctx: &mut ReasoningContext,
        iteration: usize,
    ) -> Option<LoopOutcome>;
}
```

### 1.2 The Unified Agentic Loop

**Source**: `ironclaw/src/agent/agentic_loop.rs:80-150`

```rust
pub async fn run_agentic_loop<D: LoopDelegate>(
    delegate: &D,
    reasoning: &Reasoning,
    reason_ctx: &mut ReasoningContext,
    config: &AgenticLoopConfig,
) -> Result<LoopOutcome, Error> {
    for iteration in 0..config.max_iterations {
        // 1. Check for control signals
        if let LoopSignal::Stop = delegate.check_signals().await {
            return Ok(LoopOutcome::Stopped);
        }

        // 2. Pre-LLM hook
        if let Some(outcome) = delegate.before_llm_call(reason_ctx, iteration).await {
            return Ok(outcome);
        }

        // 3. LLM invocation
        let respond_output = delegate.call_llm(reasoning, reason_ctx, iteration).await?;

        // 4. Handle response (text vs tool calls)
        let outcome = match respond_output {
            RespondOutput::Text(text) => {
                match delegate.handle_text_response(&text, reason_ctx).await {
                    TextAction::Return(outcome) => return Ok(outcome),
                    TextAction::Continue => None,
                }
            }
            RespondOutput::ToolCalls(tool_calls, content) => {
                delegate.execute_tool_calls(tool_calls, content, reason_ctx).await?
            }
        };

        if let Some(outcome) = outcome {
            return Ok(outcome);
        }

        // 5. Post-iteration hook
        if let Some(outcome) = delegate.after_iteration(reason_ctx, iteration).await {
            return Ok(outcome);
        }
    }

    Ok(LoopOutcome::MaxIterationsReached)
}
```

### 1.3 Three Delegates, One Loop

IronClaw leverages the LoopDelegate pattern to share the core agentic logic across three execution contexts:

| Delegate | Purpose | Location |
|----------|---------|----------|
| `ChatDelegate` | Interactive chat sessions | `src/agent/chat.rs` |
| `JobDelegate` | Background job execution | `src/worker/job.rs` |
| `ContainerDelegate` | Sandboxed container workers | `src/worker/container.rs` |

**ChatDelegate Implementation Pattern**:
```rust
pub struct ChatDelegate {
    session: Arc<Session>,
    channel: Box<dyn Channel>,
    tool_registry: Arc<ToolRegistry>,
}

#[async_trait]
impl LoopDelegate for ChatDelegate {
    async fn check_signals(&self) -> LoopSignal {
        self.session.check_control_signals().await
    }

    async fn call_llm(&self, reasoning: &Reasoning, ctx: &mut ReasoningContext, iter: usize) -> Result<RespondOutput, Error> {
        // Interactive chat-specific LLM handling with streaming
    }

    async fn execute_tool_calls(&self, calls: Vec<ToolCall>, content: Option<String>, ctx: &mut ReasoningContext) -> Result<Option<LoopOutcome>, Error> {
        // Tool execution with per-tool rate limiting
    }
}
```

### 1.4 Key Architectural Insights

**Advantages**:
1. **Code Reuse**: Single tested implementation of complex ReAct logic
2. **Consistency**: Identical behavior across chat, jobs, and containers
3. **Testability**: Mock delegates for unit testing the loop itself
4. **Extensibility**: New execution modes just implement the trait

**Pattern Applicability**: Ideal for systems requiring multiple execution contexts with shared core logic.

---

## 2. ZeroClaw: Trait-Driven Query Classification

ZeroClaw demonstrates a **classification-first architecture** where queries are categorized before execution, enabling optimized routing and tool filtering.

### 2.1 Query Classification System

**Source**: `zeroclaw/src/agent/classifier.rs:50-120`

```rust
pub enum QueryCategory {
    /// Direct tool execution (no reasoning needed)
    ToolOnly(Vec<ToolCall>),
    /// Simple lookup from memory/knowledge
    MemoryLookup(String),
    /// Requires full reasoning chain
    Reasoning(ReasoningRequirements),
    /// Requires multi-step planning
    Planning(PlanningRequirements),
}

pub struct ReasoningRequirements {
    pub max_iterations: u32,
    pub required_tools: Vec<ToolId>,
    pub context_depth: ContextDepth,
    pub safety_level: SafetyLevel,
}
```

### 2.2 Classifier-Driven Execution

**Source**: `zeroclaw/src/agent/mod.rs:200-280`

```rust
pub async fn process_query(&self, query: UserQuery) -> AgentResult {
    // 1. Classify the query
    let category = self.classifier.classify(&query).await?;

    // 2. Route based on classification
    match category {
        QueryCategory::ToolOnly(calls) => {
            // Fast path: execute tools directly
            self.execute_tool_calls(calls).await
        }
        QueryCategory::MemoryLookup(key) => {
            // Memory-only path
            self.memory.retrieve(&key).await
        }
        QueryCategory::Reasoning(reqs) => {
            // Full reasoning loop with configured parameters
            self.reasoning_loop(query, reqs).await
        }
        QueryCategory::Planning(reqs) => {
            // Multi-step planning with checkpointing
            self.planning_loop(query, reqs).await
        }
    }
}
```

### 2.3 Context Compaction Strategy

ZeroClaw implements **D-Mail** (Digital Mail) context compaction:

**Source**: `zeroclaw/src/agent/context.rs:150-220`

```rust
pub async fn compact_context(&mut self) -> Result<CompactedContext, Error> {
    // 1. Extract key facts from conversation history
    let facts = self.extract_facts().await?;

    // 2. Summarize tool execution results
    let summaries = self.summarize_results().await?;

    // 3. Create compact representation
    let compacted = CompactedContext {
        summary: self.llm.summarize(&self.messages).await?,
        key_facts: facts,
        tool_summaries: summaries,
        original_turns: self.messages.len(),
        compacted_tokens: self.estimate_tokens(&summaries)?,
    };

    // 4. Replace history with compact form
    self.messages = vec![Message::system(compacted.summary)];

    Ok(compacted)
}
```

---

## 3. NanoClaw: Container Isolation Architecture

NanoClaw demonstrates a **container-first approach** where each agent execution runs in an isolated Docker container with carefully managed volume mounts.

### 3.1 Container Runner Architecture

**Source**: `nanoclaw/src/container-runner.ts:1-120`

```typescript
export interface ContainerInput {
  prompt: string;
  groupFolder: string;
  timeoutMs?: number;
  envVars?: Record<string, string>;
}

export interface ContainerResult {
  exitCode: number;
  stdout: string;
  stderr: string;
  durationMs: number;
  containerId: string;
}

export async function runContainer(input: ContainerInput): Promise<ContainerResult> {
  const containerName = `nanoclaw-${input.groupFolder}`;

  // Volume mounts for isolation with controlled access
  const volumeMounts = [
    `${groupPath}:/workspace/groups/${input.groupFolder}:rw`,
    `${skillsPath}:/workspace/skills:ro`,
    `${memoryPath}:/workspace/memory:rw`,
  ];

  const args = [
    'run', '--rm',
    '--name', containerName,
    '--network', 'nanoclaw-network',
    '--memory', '2g',
    '--cpus', '2',
    ...volumeMounts.flatMap(m => ['-v', m]),
    '-e', `CLAUDE_API_KEY=${process.env.CLAUDE_API_KEY}`,
    'nanoclaw-agent:latest',
    'claude', input.prompt
  ];

  return spawnDocker(args, input.timeoutMs || 300000);
}
```

### 3.2 Container Lifecycle Management

**Source**: `nanoclaw/src/container-runner.ts:120-200`

```typescript
async function spawnDocker(args: string[], timeoutMs: number): Promise<ContainerResult> {
  return new Promise((resolve, reject) => {
    const startTime = Date.now();
    const child = spawn('docker', args, {
      stdio: ['pipe', 'pipe', 'pipe']
    });

    let stdout = '';
    let stderr = '';

    child.stdout?.on('data', (data) => { stdout += data; });
    child.stderr?.on('data', (data) => { stderr += data; });

    // Timeout handling with graceful termination
    const timeout = setTimeout(() => {
      child.kill('SIGTERM');
      setTimeout(() => child.kill('SIGKILL'), 5000);
    }, timeoutMs);

    child.on('close', (exitCode) => {
      clearTimeout(timeout);
      resolve({
        exitCode: exitCode || 0,
        stdout,
        stderr,
        durationMs: Date.now() - startTime,
        containerId: extractContainerId(args)
      });
    });

    child.on('error', (error) => {
      clearTimeout(timeout);
      reject(new ContainerError(`Docker spawn failed: ${error.message}`));
    });
  });
}
```

### 3.3 Security-First Design

NanoClaw's container architecture provides:

| Security Feature | Implementation |
|-----------------|----------------|
| **Filesystem Isolation** | Read-only skills, RW only to group folder |
| **Network Isolation** | Dedicated Docker network per deployment |
| **Resource Limits** | Memory (2GB) and CPU (2 cores) caps |
| **Ephemeral Execution** | `--rm` flag ensures cleanup |
| **Secret Injection** | API keys via environment variables |

---

## 4. PicoClaw: Go Worker Pool Architecture

PicoClaw demonstrates a **minimalist Go approach** using goroutines and atomic operations for lightweight concurrency.

### 4.1 AgentLoop Structure

**Source**: `picoclaw/pkg/agent/loop.go:20-100`

```go
type AgentLoop struct {
    bus           *bus.MessageBus
    running       atomic.Bool
    shutdown      chan struct{}
    workers       int
    wg            sync.WaitGroup
    fallback      *providers.FallbackChain
    memory        *memory.Store
}

func NewAgentLoop(bus *bus.MessageBus, workers int, fallback *providers.FallbackChain) *AgentLoop {
    return &AgentLoop{
        bus:      bus,
        workers:  workers,
        shutdown: make(chan struct{}),
        fallback: fallback,
        memory:   memory.NewStore(),
    }
}
```

### 4.2 Worker Pool Pattern

**Source**: `picoclaw/pkg/agent/loop.go:100-180`

```go
func (a *AgentLoop) Start(ctx context.Context) error {
    if !a.running.CompareAndSwap(false, true) {
        return fmt.Errorf("agent loop already running")
    }

    // Start worker goroutines
    for i := 0; i < a.workers; i++ {
        a.wg.Add(1)
        go a.worker(ctx, i)
    }

    return nil
}

func (a *AgentLoop) worker(ctx context.Context, id int) {
    defer a.wg.Done()

    for {
        select {
        case msg := <-a.bus.Messages():
            if err := a.processMessage(ctx, msg); err != nil {
                log.Printf("Worker %d: error processing message: %v", id, err)
            }
        case <-a.shutdown:
            log.Printf("Worker %d: shutting down", id)
            return
        case <-ctx.Done():
            log.Printf("Worker %d: context cancelled", id)
            return
        }
    }
}
```

### 4.3 Message Processing Pipeline

**Source**: `picoclaw/pkg/agent/loop.go:180-250`

```go
func (a *AgentLoop) processMessage(ctx context.Context, msg bus.Message) error {
    // 1. Context enrichment from memory
    contextMsg := a.enrichWithMemory(msg)

    // 2. Provider selection via fallback chain
    provider := a.fallback.SelectProvider(contextMsg)

    // 3. Stream-based LLM interaction
    stream, err := provider.StreamComplete(ctx, contextMsg)
    if err != nil {
        return fmt.Errorf("stream error: %w", err)
    }

    // 4. Handle streaming response
    var response strings.Builder
    for chunk := range stream {
        if chunk.Error != nil {
            return chunk.Error
        }
        response.WriteString(chunk.Content)

        // Handle tool calls in stream
        if chunk.ToolCall != nil {
            if err := a.executeToolCall(ctx, chunk.ToolCall); err != nil {
                return fmt.Errorf("tool execution failed: %w", err)
            }
        }
    }

    // 5. Store interaction in memory
    a.memory.Store(msg, response.String())

    return nil
}
```

### 4.4 Atomic State Management

PicoClaw uses Go's `atomic` package for lock-free state tracking:

```go
// Thread-safe state checks
if a.running.Load() {
    // Process message
}

// Graceful shutdown
a.running.Store(false)
close(a.shutdown)
a.wg.Wait()  // Wait for all workers to finish
```

---

## 5. Edict: Multi-Agent Orchestration with State Machine

Edict implements a **Chinese government-inspired hierarchical orchestration** system with Redis Streams for event-driven multi-agent coordination.

### 5.1 Department Hierarchy

**Source**: `edict/edict/backend/app/workers/orchestrator_worker.py:20-80`

```python
# Ancient Chinese government hierarchy mapping
WATCHED_TOPICS = [
    "task.new",           # 太子 (Crown Prince) - Initial triage
    "task.planning",      # 中书省 (Zhongshu) - Planning/Policy
    "task.review",        # 门下省 (Menxia) - Review/Veto
    "task.dispatch",      # 尚书省 (Shangshu) - Dispatch
    "task.execute.*",     # 六部 (Six Ministries) - Execution
    "task.report",        # 回奏 (Report back) - Final report
]

# Six Ministries breakdown
SIX_MINISTRIES = {
    "task.execute.personnel": "吏部 - Personnel/Appointments",
    "task.execute.revenue":   "户部 - Revenue/Census",
    "task.execute.rites":     "礼部 - Rites/Ceremonies",
    "task.execute.war":       "兵部 - Military/Defense",
    "task.execute.justice":   "刑部 - Justice/Law",
    "task.execute.works":     "工部 - Public Works/Engineering",
}
```

### 5.2 Redis Streams Consumer Groups

**Source**: `edict/edict/backend/app/workers/orchestrator_worker.py:100-180`

```python
class OrchestratorWorker:
    def __init__(self, redis_client: Redis, worker_id: str):
        self.redis = redis_client
        self.worker_id = worker_id
        self.consumer_group = "edict-orchestrators"
        self.running = False

    async def start(self):
        """Start consuming from Redis Streams with consumer groups"""
        self.running = True

        # Ensure consumer group exists
        for topic in WATCHED_TOPICS:
            try:
                await self.redis.xgroup_create(
                    topic,
                    self.consumer_group,
                    id='0',
                    mkstream=True
                )
            except ResponseError:
                pass  # Group already exists

        # Start consumer loop
        while self.running:
            try:
                # Read from multiple streams simultaneously
                messages = await self.redis.xreadgroup(
                    groupname=self.consumer_group,
                    consumername=self.worker_id,
                    streams={topic: '>' for topic in WATCHED_TOPICS},
                    count=10,
                    block=5000
                )

                for stream, msgs in messages:
                    for msg_id, fields in msgs:
                        await self.process_message(stream, msg_id, fields)

            except Exception as e:
                logger.error(f"Consumer error: {e}")
                await asyncio.sleep(1)
```

### 5.3 Hierarchical Task Routing

**Source**: `edict/edict/backend/app/workers/orchestrator_worker.py:180-280`

```python
async def process_message(self, stream: str, msg_id: str, fields: dict):
    """Route messages based on department hierarchy"""
    task = Task.from_dict(fields)

    routing_map = {
        "task.new": self.triage_task,           # Crown Prince
        "task.planning": self.plan_task,        # Zhongshu
        "task.review": self.review_task,        # Menxia
        "task.dispatch": self.dispatch_task,    # Shangshu
        "task.report": self.finalize_task,      # Report back
    }

    # Handle Six Ministries with wildcard matching
    handler = routing_map.get(stream)
    if not handler and stream.startswith("task.execute."):
        handler = self.execute_task

    if handler:
        result = await handler(task)
        await self.acknowledge_message(stream, msg_id, result)
    else:
        logger.warning(f"No handler for stream: {stream}")

async def triage_task(self, task: Task) -> TaskResult:
    """太子 (Crown Prince) - Initial classification and routing"""
    classification = await self.llm.classify(task.content)

    # Route to appropriate department
    if classification.requires_planning:
        await self.redis.xadd("task.planning", task.to_dict())
    else:
        await self.redis.xadd("task.dispatch", task.to_dict())

    return TaskResult(routed_to="task.planning" if classification.requires_planning else "task.dispatch")

async def plan_task(self, task: Task) -> TaskResult:
    """中书省 (Zhongshu) - Create execution plan"""
    plan = await self.llm.plan(task.content)

    # Send to review
    task.plan = plan
    await self.redis.xadd("task.review", task.to_dict())

    return TaskResult(created_plan=True, steps=len(plan.steps))
```

---

## 6. Nanobot: Python Asyncio MessageBus

Nanobot demonstrates a **lightweight Python approach** using asyncio and a custom MessageBus for event-driven agent execution.

### 6.1 MessageBus Implementation

**Source**: `nanobot/src/bus.py:1-80`

```python
import asyncio
from typing import Callable, Dict, List, Type, Any
from dataclasses import dataclass

@dataclass
class Message:
    topic: str
    payload: Any
    sender: str
    timestamp: float

class MessageBus:
    """Async pub/sub message bus for decoupled component communication"""

    def __init__(self):
        self._subscribers: Dict[str, List[Callable]] = {}
        self._queue: asyncio.Queue[Message] = asyncio.Queue()
        self._running = False

    def subscribe(self, topic: str, handler: Callable[[Message], None]):
        """Subscribe to a topic with a handler function"""
        if topic not in self._subscribers:
            self._subscribers[topic] = []
        self._subscribers[topic].append(handler)

    def publish(self, message: Message):
        """Publish a message to the bus"""
        self._queue.put_nowait(message)

    async def start(self):
        """Start the message processing loop"""
        self._running = True
        while self._running:
            try:
                message = await asyncio.wait_for(
                    self._queue.get(),
                    timeout=1.0
                )
                await self._dispatch(message)
            except asyncio.TimeoutError:
                continue

    async def _dispatch(self, message: Message):
        """Dispatch message to all subscribers"""
        handlers = self._subscribers.get(message.topic, [])
        for handler in handlers:
            try:
                if asyncio.iscoroutinefunction(handler):
                    await handler(message)
                else:
                    handler(message)
            except Exception as e:
                logger.error(f"Handler error for {message.topic}: {e}")
```

### 6.2 MCP Integration

**Source**: `nanobot/src/mcp_client.py:1-100`

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

class MCPClient:
    """Model Context Protocol client for tool integration"""

    def __init__(self, server_params: StdioServerParameters):
        self.server_params = server_params
        self.session: ClientSession | None = None
        self.tools: List[Tool] = []

    async def connect(self):
        """Connect to MCP server and discover tools"""
        self._streams = await stdio_client(self.server_params).__aenter__()
        self.session = await ClientSession(*self._streams).__aenter__()
        await self.session.initialize()

        # Discover available tools
        tools_result = await self.session.list_tools()
        self.tools = tools_result.tools

    async def call_tool(self, name: str, arguments: dict) -> ToolResult:
        """Call a tool through MCP"""
        if not self.session:
            raise RuntimeError("Not connected to MCP server")

        result = await self.session.call_tool(name, arguments)
        return ToolResult(
            content=result.content,
            is_error=result.isError
        )
```

### 6.3 Agent Loop Integration

**Source**: `nanobot/src/agent.py:50-130`

```python
class NanobotAgent:
    def __init__(self, bus: MessageBus, mcp_client: MCPClient, llm: LLMProvider):
        self.bus = bus
        self.mcp = mcp_client
        self.llm = llm
        self.memory: List[Message] = []

        # Subscribe to incoming messages
        self.bus.subscribe("incoming.message", self._on_message)
        self.bus.subscribe("tool.result", self._on_tool_result)

    async def _on_message(self, msg: Message):
        """Handle incoming user message"""
        # Add to memory
        self.memory.append(msg)

        # Get available tools from MCP
        tools = await self._format_tools()

        # Call LLM with tools
        response = await self.llm.complete(
            messages=self.memory,
            tools=tools
        )

        # Handle tool calls
        if response.tool_calls:
            for tool_call in response.tool_calls:
                result = await self.mcp.call_tool(
                    tool_call.name,
                    tool_call.arguments
                )
                self.bus.publish(Message(
                    topic="tool.result",
                    payload=result,
                    sender="agent",
                    timestamp=time.time()
                ))
        else:
            # Direct response
            await self._send_response(response.content)
```

---

## 7. CoPaw: Framework-Based ReAct

CoPaw is built on the **AgentScope** framework, demonstrating how higher-level abstractions can simplify agent development.

### 7.1 AgentScope ReActAgent

**Source**: `copaw/copaw/agents/react_agent.py:1-100`

```python
import agentscope
from agentscope.agents import ReActAgent
from agentscope.message import Msg

class CoPawAgent(ReActAgent):
    """
    CoPaw agent built on AgentScope's ReActAgent.
    Inherits the standard ReAct loop: Thought -> Action -> Observation
    """

    def __init__(
        self,
        name: str,
        model_config_name: str,
        tools: List[Callable],
        sys_prompt: str,
        max_iters: int = 10,
        verbose: bool = True,
    ):
        super().__init__(
            name=name,
            model_config_name=model_config_name,
            tools=tools,
            sys_prompt=sys_prompt,
            max_iters=max_iters,
            verbose=verbose,
        )
        self.tool_guard = ToolGuard()  # Custom safety layer

    def reply(self, x: dict = None) -> Msg:
        """
        Override reply to add ToolGuard protection.
        AgentScope calls this for each interaction.
        """
        # Pre-process with safety checks
        if x:
            x = self.tool_guard.sanitize_input(x)

        # Call parent's ReAct implementation
        msg = super().reply(x)

        # Post-process output
        msg = self.tool_guard.sanitize_output(msg)

        return msg
```

### 7.2 ToolGuard Safety Layer

**Source**: `copaw/copaw/safety/tool_guard.py:1-120`

```python
class ToolGuard:
    """
    Six-layer protection system for tool execution
    """

    def __init__(self):
        self.layers = [
            InputValidator(),      # Layer 1: Input validation
            PromptInjectionDetector(),  # Layer 2: Prompt injection detection
            ToolAllowlist(),       # Layer 3: Tool allowlisting
            RateLimiter(),         # Layer 4: Rate limiting
            OutputSanitizer(),     # Layer 5: Output sanitization
            AuditLogger(),         # Layer 6: Audit logging
        ]

    def sanitize_input(self, msg: Msg) -> Msg:
        """Apply layers 1-3 before tool execution"""
        for layer in self.layers[:3]:
            msg = layer.process(msg)
            if msg.is_blocked:
                raise SafetyError(f"Blocked by {layer.name}")
        return msg

    def sanitize_output(self, msg: Msg) -> Msg:
        """Apply layers 4-6 after tool execution"""
        for layer in self.layers[3:]:
            msg = layer.process(msg)
        return msg
```

### 7.3 Memory Compaction

**Source**: `copaw/copaw/memory/compaction.py:1-80`

```python
class MemoryCompactor:
    """
    Automatic context compaction to prevent context rot
    """

    def __init__(self, max_turns: int = 20, max_tokens: int = 4000):
        self.max_turns = max_turns
        self.max_tokens = max_tokens

    def should_compact(self, memory: List[Msg]) -> bool:
        """Check if compaction is needed"""
        return len(memory) > self.max_turns or self._count_tokens(memory) > self.max_tokens

    async def compact(self, memory: List[Msg], llm: LLM) -> List[Msg]:
        """
        Compact memory by summarizing old turns
        """
        # Keep recent messages
        recent = memory[-5:]  # Keep last 5 turns

        # Summarize older messages
        older = memory[:-5]
        summary = await llm.summarize(older)

        # Create compacted memory
        compacted = [
            Msg(role="system", content=f"Previous conversation summary: {summary}"),
            *recent
        ]

        return compacted
```

---

## 8. Agent-Browser: Dual Runtime Architecture

Agent-Browser features a **hybrid TypeScript/Rust architecture** where a Node.js/Playwright daemon handles browser automation while a Rust CLI provides native performance.

### 8.1 TypeScript Daemon

**Source**: `agent-browser/src/daemon.ts:1-100`

```typescript
interface CommandRequest {
  id: string;
  command: string;
  params: Record<string, unknown>;
}

interface CommandResponse {
  id: string;
  success: boolean;
  data?: unknown;
  error?: string;
}

class BrowserDaemon {
  private server: net.Server;
  private manager: BrowserManager;
  private commandQueue: CommandRequest[] = [];
  private processing = false;

  async start(socketPath: string): Promise<void> {
    this.server = net.createServer((socket) => {
      socket.on('data', (data) => {
        this.handleData(data, socket);
      });
    });

    await new Promise<void>((resolve) => {
      this.server.listen(socketPath, resolve);
    });

    console.log(`Daemon listening on ${socketPath}`);
  }

  private async handleData(data: Buffer, socket: net.Socket): Promise<void> {
    const lines = data.toString().split('\n');

    for (const line of lines) {
      if (!line.trim()) continue;

      try {
        const request: CommandRequest = JSON.parse(line);
        this.commandQueue.push(request);
        this.processQueue(socket);
      } catch (e) {
        await this.writeResponse(socket, {
          id: 'error',
          success: false,
          error: `Parse error: ${e}`
        });
      }
    }
  }

  private async processQueue(socket: net.Socket): Promise<void> {
    if (this.processing) return;
    this.processing = true;

    while (this.commandQueue.length > 0) {
      const request = this.commandQueue.shift()!;
      const response = await this.executeCommand(request);
      await this.writeResponse(socket, response);
    }

    this.processing = false;
  }

  private async executeCommand(request: CommandRequest): Promise<CommandResponse> {
    try {
      const result = await executeCommand(request.command, this.manager, request.params);
      return { id: request.id, success: true, data: result };
    } catch (error) {
      return { id: request.id, success: false, error: String(error) };
    }
  }
}
```

### 8.2 Rust Native CLI

**Source**: `agent-browser/cli/src/native/daemon.rs:1-120`

```rust
use tokio::net::{UnixListener, UnixStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

pub struct NativeDaemon {
    browser: Arc<Mutex<Browser>>,
}

impl NativeDaemon {
    pub async fn start(socket_path: &str) -> Result<(), DaemonError> {
        // Remove existing socket
        let _ = std::fs::remove_file(socket_path);

        let listener = UnixListener::bind(socket_path)?;
        let state = Arc::new(DaemonState::new());

        loop {
            tokio::select! {
                accept_result = listener.accept() => {
                    match accept_result {
                        Ok((stream, _)) => {
                            let state = state.clone();
                            tokio::spawn(async move {
                                handle_connection(stream, state).await;
                            });
                        }
                        Err(e) => {
                            eprintln!("Accept error: {}", e);
                        }
                    }
                }
                _ = tokio::signal::ctrl_c() => {
                    break;
                }
            }
        }

        Ok(())
    }
}

async fn handle_connection(mut stream: UnixStream, state: Arc<DaemonState>) {
    let mut buf = vec![0u8; 8192];

    loop {
        match stream.read(&mut buf).await {
            Ok(0) => break, // Connection closed
            Ok(n) => {
                let data = String::from_utf8_lossy(&buf[..n]);

                for line in data.lines() {
                    if line.is_empty() { continue; }

                    match process_request(line, &state).await {
                        Ok(response) => {
                            let _ = stream.write_all(response.as_bytes()).await;
                        }
                        Err(e) => {
                            let error = format!("{{\"error\": \"{}\"}}\n", e);
                            let _ = stream.write_all(error.as_bytes()).await;
                        }
                    }
                }
            }
            Err(e) => {
                eprintln!("Read error: {}", e);
                break;
            }
        }
    }
}
```

### 8.3 Dual Runtime Command Pattern

Both runtimes implement the same command interface:

| Command | TypeScript Implementation | Rust Implementation |
|---------|--------------------------|---------------------|
| `navigate` | Playwright `page.goto()` | Chromium DevTools Protocol |
| `click` | Playwright `locator.click()` | CDP `Input.dispatchMouseEvent` |
| `type` | Playwright `locator.fill()` | CDP `Input.dispatchKeyEvent` |
| `screenshot` | Playwright `page.screenshot()` | CDP `Page.captureScreenshot` |

---

## 9. PinchTab: HTTP API Runtime

PinchTab demonstrates a **stateless HTTP API approach** for browser automation, designed for easy integration by AI agents.

### 9.1 Go HTTP Server

**Source**: `pinchtab/cmd/pinchtab/main.go:1-100`

```go
package main

import (
    "context"
    "log"
    "net/http"
    "time"

    "github.com/chromedp/chromedp"
)

type Server struct {
    browsers map[string]*BrowserInstance
    mux      *http.ServeMux
}

type BrowserInstance struct {
    ID      string
    Context context.Context
    Cancel  context.CancelFunc
    Tabs    map[string]*Tab
}

func main() {
    s := &Server{
        browsers: make(map[string]*BrowserInstance),
        mux:      http.NewServeMux(),
    }

    // Routes
    s.mux.HandleFunc("/browsers", s.handleBrowsers)
    s.mux.HandleFunc("/browsers/", s.handleBrowser)
    s.mux.HandleFunc("/tabs", s.handleTabs)
    s.mux.HandleFunc("/tabs/", s.handleTab)
    s.mux.HandleFunc("/actions/", s.handleAction)

    log.Println("PinchTab server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", s.mux))
}
```

### 9.2 Browser Lifecycle Management

**Source**: `pinchtab/cmd/pinchtab/main.go:100-180`

```go
func (s *Server) handleBrowsers(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodPost:
        // Create new browser instance
        id := generateID()
        ctx, cancel := chromedp.NewContext(context.Background())

        browser := &BrowserInstance{
            ID:      id,
            Context: ctx,
            Cancel:  cancel,
            Tabs:    make(map[string]*Tab),
        }

        s.browsers[id] = browser

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{
            "browser_id": id,
            "status":     "created",
        })

    case http.MethodGet:
        // List all browsers
        var ids []string
        for id := range s.browsers {
            ids = append(ids, id)
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]interface{}{
            "browsers": ids,
        })
    }
}

func (s *Server) handleAction(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }

    var req ActionRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    browser, ok := s.browsers[req.BrowserID]
    if !ok {
        http.Error(w, "Browser not found", http.StatusNotFound)
        return
    }

    // Execute action in browser context
    result, err := s.executeAction(browser, &req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}
```

---

## 10. Comparative Analysis

### 10.1 Runtime Pattern Matrix

| Project | Loop Pattern | Concurrency | State Management | Isolation |
|---------|-------------|-------------|------------------|-----------|
| **IronClaw** | LoopDelegate trait | Tokio async | ReasoningContext | Docker (optional) |
| **ZeroClaw** | Classification-driven | Tokio async | CompactedContext | Process |
| **NanoClaw** | Container-per-task | Node.js async | Volume mounts | Docker (mandatory) |
| **PicoClaw** | Worker pool | Goroutines | In-memory + SQLite | None |
| **Nanobot** | MessageBus | asyncio | In-memory list | None |
| **Edict** | State machine | asyncio | Redis Streams | Process |
| **CoPaw** | ReAct (framework) | ThreadPool | AgentScope memory | None |
| **Agent-Browser** | Dual runtime | Both | Socket-based | Browser instances |
| **PinchTab** | HTTP API | Goroutines | chromedp contexts | Browser instances |

### 10.2 Design Philosophy Comparison

| Project | Philosophy | Best For |
|---------|-----------|----------|
| **IronClaw** | Maximum reusability through traits | Multiple execution modes, enterprise |
| **ZeroClaw** | Optimization through classification | Performance-sensitive, varied workloads |
| **NanoClaw** | Security through isolation | Multi-tenant, untrusted code |
| **PicoClaw** | Simplicity through minimalism | Resource-constrained, embedded |
| **Nanobot** | Flexibility through events | Rapid prototyping, experimentation |
| **Edict** | Scale through hierarchy | Complex multi-agent workflows |
| **CoPaw** | Productivity through frameworks | Quick development, standard patterns |
| **Agent-Browser** | Compatibility through duality | Cross-platform browser automation |
| **PinchTab** | Integration through APIs | External tool integration |

### 10.3 Technology Choice Drivers

```
Choose IronClaw when:
- You need multiple execution modes (chat, background jobs, containers)
- Code reuse and maintainability are priorities
- You're building enterprise-grade software

Choose ZeroClaw when:
- Query patterns vary significantly
- You need automatic optimization
- Context management is a bottleneck

Choose NanoClaw when:
- Security isolation is mandatory
- Running untrusted agent code
- Multi-tenant deployment

Choose PicoClaw when:
- Deploying to resource-constrained environments
- Simplicity is valued over features
- Go ecosystem is preferred

Choose Edict when:
- Coordinating multiple specialized agents
- Workflow requires approval chains
- Event sourcing is desired

Choose Agent-Browser when:
- Browser automation is primary use case
- Need both reliability (Playwright) and speed (Rust)
- Cross-platform compatibility required
```

---

## 11. Recommended Unified Architecture

Based on the analysis of these 9 projects, we propose a **layered agent runtime architecture** that combines the best patterns:

### 11.1 Proposed Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Execution Modes                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Interactive  │  │ Background   │  │ Container    │           │
│  │ Chat         │  │ Jobs         │  │ Sandbox      │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
└─────────┼─────────────────┼─────────────────┼───────────────────┘
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Delegate Interface                          │
│              (IronClaw LoopDelegate Pattern)                     │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  check_signals() → before_llm_call() → call_llm() →         ││
│  │  handle_response() → execute_tools() → after_iteration()    ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Classification Layer                          │
│              (ZeroClaw Query Classification)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Tool Only    │  │ Memory       │  │ Full         │           │
│  │ Fast Path    │  │ Lookup       │  │ Reasoning    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Core ReAct Loop                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  while not done:                                            ││
│  │    signal = check_signals()                                 ││
│  │    if signal == STOP: break                                 ││
│  │    response = llm.complete(context)                         ││
│  │    if response.has_tool_calls:                              ││
│  │      results = execute_tools(response.tool_calls)           ││
│  │      context.add_results(results)                           ││
│  │    else:                                                    ││
│  │      return response.content                                ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Infrastructure Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ MessageBus   │  │ State Store  │  │ Observability│           │
│  │ (Nanobot)    │  │ (Edict/Redis)│  │ (IronClaw)   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **LoopDelegate trait** | Enables code reuse across execution modes (IronClaw lesson) |
| **Classification layer** | Optimizes for different query types (ZeroClaw lesson) |
| **Pluggable isolation** | Docker optional, not mandatory (NanoClaw lesson) |
| **MessageBus integration** | Decouples components (Nanobot lesson) |
| **Worker pool support** | Scales with goroutines when needed (PicoClaw lesson) |
| **State machine hooks** | Supports complex workflows (Edict lesson) |
| **Safety layering** | Defense in depth (CoPaw lesson) |
| **Dual runtime option** | When platform compatibility matters (Agent-Browser lesson) |

### 11.3 Trait Definition (Rust)

```rust
/// Core trait for agent runtime implementations
#[async_trait]
pub trait AgentRuntime: Send + Sync {
    /// Execute a single query and return the result
    async fn execute(&self, query: Query) -> Result<Response, Error>;

    /// Start continuous operation (for server modes)
    async fn run(&self) -> Result<(), Error>;

    /// Graceful shutdown
    async fn shutdown(&self) -> Result<(), Error>;
}

/// Trait for pluggable isolation strategies
#[async_trait]
pub trait IsolationStrategy: Send + Sync {
    async fn execute(&self, task: Task) -> Result<TaskResult, Error>;
}

/// Trait for query classification
#[async_trait]
pub trait QueryClassifier: Send + Sync {
    async fn classify(&self, query: &Query) -> QueryCategory;
}

/// Trait for context management
#[async_trait]
pub trait ContextManager: Send + Sync {
    async fn load(&self, session_id: &str) -> Result<Context, Error>;
    async fn save(&self, session_id: &str, context: &Context) -> Result<(), Error>;
    async fn compact(&self, context: &mut Context) -> Result<(), Error>;
}
```

---

## 12. Conclusion

The analysis of 9 production AI agent projects reveals a spectrum of runtime architectures, each optimized for different constraints:

1. **IronClaw's LoopDelegate** demonstrates how traits can maximize code reuse across execution modes
2. **ZeroClaw's classification** shows how preprocessing can optimize for varied workloads
3. **NanoClaw's containers** prove that security through isolation is viable for multi-tenant scenarios
4. **PicoClaw's minimalism** reminds us that simple solutions often suffice
5. **Edict's hierarchy** illustrates how ancient wisdom (Chinese bureaucracy) can inform modern distributed systems
6. **CoPaw's framework approach** shows the productivity benefits of building on existing abstractions
7. **Agent-Browser's duality** proves that hybrid architectures can bridge ecosystem gaps

The recommended unified architecture draws from all these lessons, proposing a layered design that prioritizes:
- **Reusability** through the LoopDelegate pattern
- **Performance** through classification-driven optimization
- **Security** through pluggable isolation strategies
- **Scalability** through event-driven MessageBus integration
- **Maintainability** through clear trait boundaries

This architecture can serve as a blueprint for building the next generation of AI agent runtimes that are simultaneously powerful, secure, and maintainable.

---

## Appendix: Source Code Index

| Project | Key Files |
|---------|-----------|
| IronClaw | `src/agent/agentic_loop.rs`, `src/agent/chat.rs`, `src/worker/job.rs`, `src/worker/container.rs` |
| ZeroClaw | `src/agent/mod.rs`, `src/agent/classifier.rs`, `src/agent/context.rs` |
| NanoClaw | `src/container-runner.ts`, `src/index.ts` |
| PicoClaw | `pkg/agent/loop.go`, `pkg/bus/bus.go` |
| Nanobot | `src/bus.py`, `src/agent.py`, `src/mcp_client.py` |
| Edict | `edict/backend/app/workers/orchestrator_worker.py` |
| CoPaw | `copaw/agents/react_agent.py`, `copaw/safety/tool_guard.py` |
| Agent-Browser | `src/daemon.ts`, `cli/src/native/daemon.rs` |
| PinchTab | `cmd/pinchtab/main.go` |

---

*Document Version: 2.0*
*Last Updated: 2025-01-18*
*Analysis Scope: 9 production AI agent projects across 4 technology ecosystems*
