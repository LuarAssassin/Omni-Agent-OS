# Workflow/Task Engine 架构分析

## 目录
1. [执行摘要](#执行摘要)
2. [Edict - 三省六部制状态机](#edict---三省六部制状态机)
3. [IronClaw - 并行作业调度器](#ironclaw---并行作业调度器)
4. [NanoClaw - 容器队列管理](#nanoclaw---容器队列管理)
5. [Codex - CSV批处理工作池](#codex---csv批处理工作池)
6. [Gemini CLI - 事件驱动调度器](#gemini-cli---事件驱动调度器)
7. [Qwen Code - 策略感知调度器](#qwen-code---策略感知调度器)
8. [Kimi CLI - 后台任务与心跳](#kimi-cli---后台任务与心跳)
9. [Pi Mono - 转向/跟进队列](#pi-mono---转向跟进队列)
10. [ZeroClaw - 流式调度器](#zeroclaw---流式调度器)
11. [NanoBot - 异步Cron服务](#nanobot---异步cron服务)
12. [PicoClaw - ReAct工具循环](#picoclaw---react工具循环)
13. [Mimiclaw - 嵌入式Cron](#mimiclaw---嵌入式cron)
14. [CoPaw - APScheduler + ReAct](#copaw---apscheduler--react)
15. [OpenClaw - 技能工具库](#openclaw---技能工具库)
16. [OpenCode - 简单间隔调度器](#opencode---简单间隔调度器)
17. [Agent-Browser - 浏览器任务队列](#agent-browser---浏览器任务队列)
18. [架构对比矩阵](#架构对比矩阵)
19. [推荐统一工作流架构](#推荐统一工作流架构)

---

## 执行摘要

### 工作流成熟度分级

| 等级 | 项目 | 特征 |
|------|------|------|
| **Level 4** | Edict, IronClaw, Codex | 完整状态机、分布式队列、并发控制、持久化 |
| **Level 3** | NanoClaw, Gemini CLI, Qwen Code | 高级调度器、策略检查、批处理、错误恢复 |
| **Level 2** | Kimi CLI, Pi Mono, ZeroClaw | 后台任务管理、队列系统、并发执行 |
| **Level 1** | CoPaw, NanoBot, PicoClaw | Cron调度、ReAct循环、基础并发 |
| **Level 0** | OpenClaw, OpenCode, Mimiclaw, Agent-Browser | 简单调度、单线程、无持久化 |

### 核心模式分类

```
┌─────────────────────────────────────────────────────────────────┐
│                    Workflow Engine Patterns                     │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  State Machine  │  Queue-Based    │  Event-Driven               │
├─────────────────┼─────────────────┼─────────────────────────────┤
│  Edict          │  NanoClaw       │  Edict (Redis Streams)      │
│  IronClaw       │  Codex          │  Gemini CLI                 │
│  Kimi CLI       │  Agent-Browser  │  Pi Mono                    │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

---

## Edict - 三省六部制状态机

### 架构概览

Edict实现了中国古代政府体系-inspired的工作流：**太子 → 中书省 → 门下省 → 尚书省 → 六部 → 回奏**

```
用户请求
    │
    ▼
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  太子    │───→│  中书省   │───→│  门下省   │───→│  尚书省   │
│ (分拣)   │    │ (起草)   │    │ (审议)   │    │ (派发)   │
└─────────┘    └──────────┘    └──────────┘    └──────────┘
                                                      │
                         ┌────────────┬──────────────┼────────────┐
                         ▼            ▼              ▼            ▼
                     ┌────────┐  ┌────────┐     ┌────────┐  ┌────────┐
                     │  吏部   │  │  户部   │     │  礼部   │  │  兵部   │
                     └────────┘  └────────┘     └────────┘  └────────┘
                     ┌────────┐  ┌────────┐
                     │  刑部   │  │  工部   │
                     └────────┘  └────────┘
                              │
                              ▼
                         ┌──────────┐
                         │   回奏    │
                         │ (审查汇总) │
                         └──────────┘
```

### 状态机定义

**File:** `edict/edict/backend/app/models/task.py`

```python
class TaskState(str, enum.Enum):
    Taizi = "Taizi"           # 太子分拣
    Zhongshu = "Zhongshu"     # 中书省起草
    Menxia = "Menxia"         # 门下省审议
    Assigned = "Assigned"     # 尚书省已将任务派发
    Next = "Next"             # 待执行
    Doing = "Doing"           # 六部执行中
    Review = "Review"         # 审查汇总
    Done = "Done"             # 完成
    Blocked = "Blocked"       # 阻塞
    Cancelled = "Cancelled"   # 取消

# 严格的状态转换规则
STATE_TRANSITIONS = {
    TaskState.Taizi: {TaskState.Zhongshu, TaskState.Cancelled},
    TaskState.Zhongshu: {TaskState.Menxia, TaskState.Cancelled, TaskState.Blocked},
    TaskState.Menxia: {TaskState.Assigned, TaskState.Zhongshu, TaskState.Cancelled},
    TaskState.Assigned: {TaskState.Next, TaskState.Cancelled},
    TaskState.Next: {TaskState.Doing, TaskState.Cancelled},
    TaskState.Doing: {TaskState.Review, TaskState.Cancelled},
    TaskState.Review: {TaskState.Done, TaskState.Zhongshu, TaskState.Cancelled},
}
```

### Redis Streams 事件总线

**File:** `edict/edict/backend/app/services/event_bus.py`

```python
class EventBus:
    async def publish(self, topic: str, trace_id: str,
                      event_type: str, payload: dict) -> str:
        event = {
            "trace_id": trace_id,
            "type": event_type,
            "payload": json.dumps(payload),
            "timestamp": datetime.utcnow().isoformat(),
        }
        # Stream for persistence
        entry_id = await self.redis.xadd(
            f"edict:stream:{topic}", event, maxlen=10000
        )
        # Pub/sub for real-time notifications
        await self.redis.publish(
            f"edict:pubsub:{topic}", json.dumps(event)
        )
        return entry_id

    async def consume(self, topic: str, group: str,
                      consumer: str, block_ms: int = 5000):
        """Consumer group pattern for load balancing"""
        stream_key = f"edict:stream:{topic}"
        results = await self.redis.xreadgroup(
            groupname=group,
            consumername=consumer,
            streams={stream_key: ">"},
            count=10,
            block=block_ms,
        )
        return [(entry_id, json.loads(data)) for entry_id, data in results]

    async def claim_stale(self, topic: str, group: str,
                          consumer: str, idle_ms: int = 30000):
        """Crash recovery for unacknowledged events"""
        stream_key = f"edict:stream:{topic}"
        results = await self.redis.xautoclaim(
            stream_key, group, consumer,
            min_idle_time=idle_ms, count=10
        )
        return results
```

### 分发Worker与并发控制

**File:** `edict/edict/backend/app/workers/dispatch_worker.py`

```python
class DispatchWorker:
    def __init__(self, bus: EventBus, semaphore: asyncio.Semaphore):
        self._bus = bus
        self._semaphore = semaphore  # 并发控制

    async def run(self):
        while self._running:
            try:
                messages = await self._bus.consume(
                    TOPIC_TASK_DISPATCH, GROUP, self._consumer_id
                )
                for entry_id, event in messages:
                    await self._dispatch(entry_id, event)
            except Exception as e:
                logger.error(f"Dispatch error: {e}")
                await asyncio.sleep(5)

    async def _dispatch(self, entry_id: str, event: dict):
        async with self._semaphore:  # 限制并发数
            task_id = event["task_id"]
            agent = event["agent"]
            message = event["message"]

            # 调用OpenClaw CLI执行
            result = await self._call_openclaw(
                agent, message, task_id, trace_id
            )

            # ACK确认
            await self._bus.ack(TOPIC_TASK_DISPATCH, GROUP, entry_id)
```

---

## IronClaw - 并行作业调度器

### 架构概览

IronClaw实现了完整的Job/Subtask层级调度系统，支持并行批处理。

```
┌─────────────────────────────────────────────────────────────┐
│                     Scheduler                               │
├─────────────────────────────────────────────────────────────┤
│  Jobs: HashMap<Uuid, ScheduledJob>                         │
│  Subtasks: HashMap<Uuid, ScheduledSubtask>                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Job       │  │  Subtask    │  │  Background Task    │ │
│  │ (User Task) │  │ (Tool Exec) │  │ (Async Handler)     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────┬───────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ Pending  │    │ Running  │    │ Completed│
        └──────────┘    └──────────┘    └──────────┘
```

### 核心实现

**File:** `ironclaw/src/agent/scheduler.rs`

```rust
pub struct Scheduler {
    config: AgentConfig,
    context_manager: Arc<ContextManager>,
    llm: Arc<dyn LlmProvider>,
    tools: Arc<ToolRegistry>,
    jobs: Arc<RwLock<HashMap<Uuid, ScheduledJob>>>,
    subtasks: Arc<RwLock<HashMap<Uuid, ScheduledSubtask>>>,
}

pub enum Task {
    Job {
        id: Uuid,
        title: String,
        description: String,
    },
    ToolExec {
        parent_id: Uuid,
        tool_name: String,
        params: serde_json::Value,
    },
    Background {
        id: Uuid,
        handler: Arc<dyn TaskHandler>,
    },
}

impl Scheduler {
    pub async fn dispatch_job(
        &self,
        user_id: &str,
        title: &str,
        description: &str,
        metadata: Option<serde_json::Value>,
    ) -> Result<Uuid, JobError> {
        let job_id = Uuid::new_v4();

        let job = ScheduledJob {
            id: job_id,
            user_id: user_id.to_string(),
            title: title.to_string(),
            description: description.to_string(),
            status: JobStatus::Pending,
            created_at: Utc::now(),
            metadata,
        };

        self.jobs.write().await.insert(job_id, job);

        // Spawn background task
        tokio::spawn(async move {
            if let Err(e) = self.execute_job(job_id).await {
                error!("Job {} failed: {}", job_id, e);
            }
        });

        Ok(job_id)
    }

    pub async fn spawn_batch(
        &self,
        parent_id: Uuid,
        tasks: Vec<Task>,
    ) -> Vec<Result<TaskOutput, Error>> {
        // 使用信号量限制并发
        let semaphore = Arc::new(
            tokio::sync::Semaphore::new(self.config.max_concurrency)
        );

        let futures: Vec<_> = tasks.into_iter().map(|task| {
            let sem = semaphore.clone();
            async move {
                let _permit = sem.acquire().await?;
                self.execute_task(parent_id, task).await
            }
        }).collect();

        futures::future::join_all(futures).await
    }
}
```

### 作业状态机

```
Pending -> InProgress -> Completed -> Submitted -> Accepted
                     \\                      \\--> Rejected
                      \\-> Failed
                      \\-> Stuck -> InProgress (recovery)
                               \\-> Failed
```

---

## NanoClaw - 容器队列管理

### Cron调度器

**File:** `nanoclaw/src/task-scheduler.ts`

```typescript
export interface ScheduledTask {
  id: string;
  schedule_type: 'cron' | 'interval' | 'once';
  schedule_value: string;
  next_run: string | null;
  enabled: boolean;
}

export function computeNextRun(task: ScheduledTask): string | null {
  if (task.schedule_type === 'cron') {
    const interval = CronExpressionParser.parse(
      task.schedule_value, { tz: TIMEZONE }
    );
    return interval.next().toISOString();
  }

  if (task.schedule_type === 'interval') {
    const ms = parseDuration(task.schedule_value);
    if (!ms) return null;

    let next = new Date(task.next_run!).getTime() + ms;
    const now = Date.now();
    while (next <= now) {
      next += ms;
    }
    return new Date(next).toISOString();
  }

  if (task.schedule_type === 'once') {
    return null; // One-shot
  }

  return null;
}
```

### 组队列并发控制

**File:** `nanoclaw/src/group-queue.ts`

```typescript
const MAX_CONCURRENT_CONTAINERS = 3;

interface GroupState {
  runningTaskId: string | null;
  pendingTasks: Array<{
    id: string;
    fn: () => Promise<void>;
    resolve: (value: void) => void;
    reject: (reason: Error) => void;
  }>;
}

export class GroupQueue {
  private groups = new Map<string, GroupState>();
  private activeCount = 0;
  private waitingGroups: string[] = [];

  async enqueueTask(
    groupJid: string,
    taskId: string,
    fn: () => Promise<void>
  ): Promise<void> {
    // 防止重复入队
    const state = this.groups.get(groupJid) || {
      runningTaskId: null, pendingTasks: []
    };
    if (state.runningTaskId === taskId) return;
    if (state.pendingTasks.some((t) => t.id === taskId)) return;

    return new Promise((resolve, reject) => {
      state.pendingTasks.push({ id: taskId, fn, resolve, reject });
      this.groups.set(groupJid, state);
      this._processQueue();
    });
  }

  private async _processQueue(): Promise<void> {
    if (this.activeCount >= MAX_CONCURRENT_CONTAINERS) return;

    const groupJid = this.waitingGroups.shift();
    if (!groupJid) return;

    const state = this.groups.get(groupJid);
    if (!state || state.pendingTasks.length === 0) return;

    const next = state.pendingTasks.shift()!;
    state.runningTaskId = next.id;
    this.activeCount++;

    try {
      await next.fn();
      next.resolve();
    } catch (err) {
      next.reject(err as Error);
    } finally {
      this.activeCount--;
      state.runningTaskId = null;
      if (state.pendingTasks.length > 0) {
        this.waitingGroups.push(groupJid);
      }
      this._processQueue();
    }
  }
}
```

---

## Codex - CSV批处理工作池

### 架构概览

Codex实现了CSV驱动的批量作业处理，支持并发控制和状态轮询。

```
CSV Input
    │
    ▼
┌─────────────────────────────────────┐
│      AgentJob (Batch Definition)    │
├─────────────────────────────────────┤
│  Items: AgentJobItem[]              │
│  - Pending                          │
│  - Running  ←── spawn threads       │
│  - Completed ←── poll status        │
│  - Failed                           │
└─────────────────────────────────────┘
```

### 工作池实现

**File:** `codex/codex-rs/core/src/tools/handlers/agent_jobs.rs`

```rust
const DEFAULT_AGENT_JOB_CONCURRENCY: usize = 16;
const MAX_AGENT_JOB_CONCURRENCY: usize = 64;
const STATUS_POLL_INTERVAL: Duration = Duration::from_millis(250);

async fn run_agent_job_loop(
    session: Arc<Session>,
    turn: Arc<TurnContext>,
    db: Arc<codex_state::StateRuntime>,
    job_id: String,
    options: JobRunnerOptions,
) -> anyhow::Result<()> {
    let cancel_requested = turn.cancel_requested();
    let mut active_items: HashMap<ThreadId, ActiveJobItem> = HashMap::new();

    loop {
        if cancel_requested.load(Ordering::Relaxed) {
            // 取消所有活动项
            for (thread_id, active) in &active_items {
                cancel_thread(&session, thread_id, &active.item.id).await.ok();
            }
            return Ok(());
        }

        // 启动待处理项（直到并发限制）
        if active_items.len() < options.max_concurrency {
            let slots = options.max_concurrency - active_items.len();
            let pending_items = db
                .list_agent_job_items(
                    job_id.as_str(),
                    Some(codex_state::AgentJobItemStatus::Pending),
                    Some(slots),
                )
                .await?;

            for item in pending_items {
                let thread_id = spawn_agent_for_item(&session, &item, &turn).await?;
                active_items.insert(
                    thread_id,
                    ActiveJobItem { item, spawned_at: Instant::now() },
                );
            }
        }

        // 收割已完成项
        let finished = poll_for_finished_threads(&session, &active_items).await?;
        for thread_id in finished {
            if let Some(active) = active_items.remove(&thread_id) {
                finalize_item(&db, &active.item, &thread_id).await?;
            }
        }

        // 检查是否全部完成
        if active_items.is_empty() {
            let remaining = db
                .list_agent_job_items(
                    job_id.as_str(),
                    Some(codex_state::AgentJobItemStatus::Pending),
                    Some(1),
                )
                .await?;
            if remaining.is_empty() {
                break;
            }
        }

        tokio::time::sleep(STATUS_POLL_INTERVAL).await;
    }

    Ok(())
}
```

---

## Gemini CLI - 事件驱动调度器

### 工具调用状态机

**File:** `gemini-cli/packages/core/src/scheduler/types.ts`

```typescript
export enum CoreToolCallStatus {
  Validating = 'validating',
  Scheduled = 'scheduled',
  Error = 'error',
  Success = 'success',
  Executing = 'executing',
  Cancelled = 'cancelled',
  AwaitingApproval = 'awaiting_approval',
}

export type ToolCall =
  | ValidatingToolCall
  | ScheduledToolCall
  | ErroredToolCall
  | SuccessfulToolCall
  | ExecutingToolCall
  | CancelledToolCall
  | WaitingToolCall;
```

### 调度器实现

**File:** `gemini-cli/packages/core/src/scheduler/scheduler.ts`

```typescript
export class Scheduler {
  private readonly state: SchedulerStateManager;
  private readonly executor: ToolExecutor;
  private isProcessing = false;
  private readonly requestQueue: SchedulerQueueItem[] = [];

  async schedule(
    request: ToolCallRequestInfo | ToolCallRequestInfo[],
    signal: AbortSignal,
  ): Promise<CompletedToolCall[]> {
    const requests = Array.isArray(request) ? request : [request];

    // 基于队列的批处理
    this.requestQueue.push(...requests.map(r => ({ request: r })));
    await this._processQueue(signal);

    return this.state.completedBatch;
  }

  private async _processQueue(signal: AbortSignal): Promise<void> {
    if (this.isProcessing) return;
    this.isProcessing = true;

    try {
      while (this.state.queueLength > 0 || this.state.isActive) {
        const shouldContinue = await this._processNextItem(signal);
        if (!shouldContinue) break;
        if (signal.aborted) break;
      }
    } finally {
      this.isProcessing = false;
    }
  }

  private async _processNextItem(signal: AbortSignal): Promise<boolean> {
    // 1. 处理所有'validating'调用（策略与确认）
    const validating = this.state.allActiveCalls.filter(
      c => c.status === CoreToolCallStatus.Validating
    );
    for (const call of validating) {
      await this._handleValidation(call, signal);
    }

    // 2. 执行scheduled调用（考虑并行/顺序）
    const scheduled = this.state.allActiveCalls.filter(
      c => c.status === CoreToolCallStatus.Scheduled
    );

    for (const call of scheduled) {
      const isParallel = this._isParallelizable(call.request);
      if (isParallel || scheduled.indexOf(call) === 0) {
        await this._executeCall(call, signal);
      }
    }

    // 3. 终结完成的调用
    const completed = this.state.allActiveCalls.filter(c =>
      c.status === CoreToolCallStatus.Success ||
      c.status === CoreToolCallStatus.Error
    );
    for (const call of completed) {
      this.state.finalizeCall(call.request.callId);
    }

    this.state.dequeue();
    return true;
  }

  private _isParallelizable(request: ToolCallRequestInfo): boolean {
    if (request.args) {
      const wait = request.args['wait_for_previous'];
      if (typeof wait === 'boolean') {
        return !wait;
      }
    }
    return true; // 默认并行
  }
}
```

### 状态管理器与尾调用

**File:** `gemini-cli/packages/core/src/scheduler/state-manager.ts`

```typescript
export class SchedulerStateManager {
  private readonly activeCalls = new Map<string, ToolCall>();
  private readonly queue: ToolCall[] = [];

  /**
   * 用新的调用替换当前活动调用，将新调用放入队列最前面
   * 用于Tail Calls - 链式执行而不终结原始调用
   */
  replaceActiveCallWithTailCall(callId: string, nextCall: ToolCall): void {
    if (this.activeCalls.has(callId)) {
      this.activeCalls.delete(callId);
      this.queue.unshift(nextCall);
      this.emitUpdate();
    }
  }
}
```

---

## Qwen Code - 策略感知调度器

**File:** `qwen-code/packages/core/src/core/coreToolScheduler.ts`

```typescript
export async function scheduleTools(
  context: AgentLoopContext,
  toolCallRequests: ToolCallRequestInfo[],
  policyChecker: PolicyChecker,
  confirmer: ToolConfirmer,
  modifier: ToolModifier,
  executor: ToolExecutor,
  signal: AbortSignal,
): Promise<ToolCallResult[]> {
  const results: ToolCallResult[] = [];

  for (const request of toolCallRequests) {
    if (signal.aborted) break;

    // 阶段1: 验证与策略检查
    const validation = await validateToolCall(context, request);
    if (!validation.valid) {
      results.push(createErrorResult(request, validation.error));
      continue;
    }

    // 阶段2: 用户确认（如需要）
    const confirmation = await confirmer.confirm(request, context);
    if (!confirmation.approved) {
      results.push(createCancelledResult(request, confirmation.reason));
      continue;
    }

    // 阶段3: 执行
    const executionResult = await executor.execute(request, context, signal);

    // 阶段4: 尾调用处理
    if (executionResult.tailCallRequest) {
      const tailResult = await scheduleTools(
        context,
        [executionResult.tailCallRequest],
        policyChecker,
        confirmer,
        modifier,
        executor,
        signal,
      );
      results.push(...tailResult);
    } else {
      results.push(executionResult);
    }
  }

  return results;
}
```

---

## Kimi CLI - 后台任务与心跳

### 任务模型

**File:** `kimi-cli/src/kimi_cli/background/models.py`

```python
type TaskStatus = Literal[
    "created", "starting", "running", "completed", "failed", "killed", "lost"
]
TERMINAL_TASK_STATUSES: tuple[TaskStatus, ...] = (
    "completed", "failed", "killed", "lost"
)

class TaskSpec(BaseModel):
    id: str
    kind: TaskKind  # "bash" | "agent"
    session_id: str
    description: str
    timeout_s: int | None = None

class TaskRuntime(BaseModel):
    status: TaskStatus = "created"
    worker_pid: int | None = None
    child_pid: int | None = None
    started_at: float | None = None
    heartbeat_at: float | None = None
    completed_at: float | None = None
    exit_code: int | None = None
    error: str | None = None
```

### Worker心跳循环

**File:** `kimi-cli/src/kimi_cli/background/worker.py`

```python
async def _run_task(task_id: str, spec: TaskSpec, store: TaskStore) -> None:
    current = TaskRuntime(status="starting", worker_pid=os.getpid())
    store.write_runtime(task_id, current)

    stop_event = asyncio.Event()
    current_control = TaskControl()

    async def _heartbeat_loop() -> None:
        """定期更新心跳时间戳"""
        while not stop_event.is_set():
            await asyncio.sleep(heartbeat_interval_ms / 1000)
            current.heartbeat_at = time.time()
            store.write_runtime(task_id, current)

    async def _control_loop() -> None:
        """监听控制命令（如kill）"""
        while not stop_event.is_set():
            await asyncio.sleep(control_poll_interval_ms / 1000)
            ctrl = store.read_control(task_id)
            if ctrl:
                current_control = ctrl
                if current_control.kill_requested_at is not None:
                    await _terminate_process(force=current_control.force)
                    stop_event.set()

    # 并行运行心跳、控制和主任务
    heartbeat_task = asyncio.create_task(_heartbeat_loop())
    control_task = asyncio.create_task(_control_loop())

    try:
        if spec.kind == "bash":
            result = await _run_bash_task(spec, current)
        else:
            result = await _run_agent_task(spec, current)

        current.status = "completed"
        current.completed_at = time.time()
    except Exception as e:
        current.status = "failed"
        current.error = str(e)
    finally:
        stop_event.set()
        await asyncio.gather(
            heartbeat_task, control_task, return_exceptions=True
        )
        store.write_runtime(task_id, current)
```

---

## Pi Mono - 转向/跟进队列

### 架构概览

Pi Mono实现了独特的双队列系统：**Steering Queue**（中途干预）和 **Follow-Up Queue**（后续处理）。

```
┌─────────────────────────────────────────────────────────────┐
│                     Agent Loop                              │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐      ┌──────────────────┐            │
│  │  Steering Queue  │      │ Follow-Up Queue  │            │
│  │  (中断当前执行)   │      │ (执行完成后处理)  │            │
│  ├──────────────────┤      ├──────────────────┤            │
│  │  - 用户中断      │      │  - 后续提示      │            │
│  │  - 紧急指令      │      │  - 补充问题      │            │
│  │  - 方向修正      │      │  - 相关任务      │            │
│  └────────┬─────────┘      └────────┬─────────┘            │
│           │                         │                      │
│           └───────────┬─────────────┘                      │
│                       ▼                                    │
│              ┌─────────────────┐                          │
│              │  Message Router │                          │
│              └────────┬────────┘                          │
│                       ▼                                    │
│         ┌─────────────────────────┐                       │
│         │  Tool Execution (并行/顺序) │                   │
│         └─────────────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

### Agent实现

**File:** `pi-mono/packages/agent/src/agent.ts`

```typescript
export class Agent {
    private steeringQueue: AgentMessage[] = [];
    private followUpQueue: AgentMessage[] = [];
    private steeringMode: "all" | "one-at-a-time";
    private followUpMode: "all" | "one-at-a-time";

    /**
     * 队列转向消息以中断Agent执行
     * 在当前工具执行后投递，跳过剩余工具
     */
    steer(m: AgentMessage) {
        this.steeringQueue.push(m);
    }

    /**
     * 队列跟进消息在Agent完成后处理
     * 仅在Agent无更多工具调用或转向消息时投递
     */
    followUp(m: AgentMessage) {
        this.followUpQueue.push(m);
    }

    private dequeueSteeringMessages(): AgentMessage[] {
        if (this.steeringMode === "one-at-a-time") {
            if (this.steeringQueue.length > 0) {
                const first = this.steeringQueue[0];
                this.steeringQueue = this.steeringQueue.slice(1);
                return [first];
            }
            return [];
        }
        const steering = this.steeringQueue.slice();
        this.steeringQueue = [];
        return steering;
    }
}
```

### Agent循环

**File:** `pi-mono/packages/agent/src/agent-loop.ts`

```typescript
async function runLoop(
    currentContext: AgentContext,
    newMessages: AgentMessage[],
    config: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink,
    streamFn?: StreamFn,
): Promise<void> {
    let firstTurn = true;
    let pendingMessages: AgentMessage[] =
        (await config.getSteeringMessages?.()) || [];

    // 外层循环：当队列中有跟进消息时继续
    while (true) {
        let hasMoreToolCalls = true;

        // 内层循环：处理工具调用和转向消息
        while (hasMoreToolCalls || pendingMessages.length > 0) {
            const contextMessages = [...currentContext.messages, ...pendingMessages];
            pendingMessages = [];

            const stream = await streamFn(...);

            for await (const event of stream) {
                if (event.type === "toolCall") {
                    // 并行或顺序执行
                    if (config.toolExecution === "parallel") {
                        await executeToolsParallel(event.toolCalls, ...);
                    } else {
                        await executeToolsSequential(event.toolCalls, ...);
                    }
                }
            }

            // 检查中途转向消息
            const steering = (await config.getSteeringMessages?.()) || [];
            if (steering.length > 0) {
                pendingMessages = steering;
                hasMoreToolCalls = false;
                break;
            }
        }

        // 检查跟进消息
        const followUpMessages = (await config.getFollowUpMessages?.()) || [];
        if (followUpMessages.length > 0) {
            pendingMessages = followUpMessages;
            continue;
        }
        break;
    }
}
```

---

## ZeroClaw - 流式调度器

### Cron调度器

**File:** `zeroclaw/src/cron/scheduler.rs`

```rust
pub async fn run(config: Config) -> Result<()> {
    let security = SecurityLayer::new(&config)?;
    let component = ComponentRegistry::new(&config).await?;

    let poll_secs = config.cron.poll_interval_secs;
    let max_concurrent = config.cron.max_concurrent_jobs;

    let mut interval = time::interval(Duration::from_secs(poll_secs));
    interval.set_missed_tick_behavior(time::MissedTickBehavior::Skip);

    loop {
        interval.tick().await;

        let jobs = due_jobs(&config, Utc::now())?;
        if jobs.is_empty() {
            continue;
        }

        info!("Found {} due jobs", jobs.len());

        // 使用buffer_unordered限制并发
        let mut in_flight = stream::iter(
            jobs.into_iter().map(|job| async move {
                execute_and_persist_job(&config, &security, &job, &component).await
            })
        ).buffer_unordered(max_concurrent);

        while let Some(result) = in_flight.next().await {
            match result {
                Ok(job_id) => info!("Job {} completed", job_id),
                Err(e) => error!("Job failed: {}", e),
            }
        }
    }
}
```

### Schedule工具

**File:** `zeroclaw/src/tools/schedule.rs`

```rust
#[async_trait]
impl Tool for ScheduleTool {
    fn name(&self) -> &str {
        "schedule"
    }

    fn description(&self) -> &str {
        "Manage scheduled jobs (create, list, cancel, pause, resume)"
    }

    async fn execute(&self, args: serde_json::Value) -> Result<ToolResult> {
        let action = args.get("action").and_then(|v| v.as_str())
            .unwrap_or("list");

        match action {
            "list" => self.handle_list(),
            "create" | "add" | "once" => {
                let approved = args.get("approved").and_then(|v| v.as_bool())
                    .unwrap_or(false);
                self.handle_create_like(action, &args, approved)
            }
            "cancel" | "remove" => {
                let id = args.get("id").and_then(|v| v.as_str()).unwrap_or("");
                Ok(self.handle_cancel(id))
            }
            "pause" => {
                let id = args.get("id").and_then(|v| v.as_str()).unwrap_or("");
                Ok(self.handle_pause_resume(id, true))
            }
            "resume" => {
                let id = args.get("id").and_then(|v| v.as_str()).unwrap_or("");
                Ok(self.handle_pause_resume(id, false))
            }
            _ => Err(anyhow::anyhow!("Unknown action: {}", action)),
        }
    }
}
```

---

## NanoBot - 异步Cron服务

**File:** `nanobot/nanobot/cron/service.py`

```python
class CronService:
    def __init__(self, store_path: Path):
        self._store_path = store_path
        self._store = self._load_store()
        self._timer_task: asyncio.Task | None = None

    async def start(self) -> None:
        self._arm_timer()

    def _arm_timer(self) -> None:
        """基于最早作业安排下次计时器滴答"""
        if self._timer_task:
            self._timer_task.cancel()

        next_wake = self._get_next_wake_ms()
        delay_ms = max(0, next_wake - _now_ms())

        async def tick() -> None:
            await asyncio.sleep(delay_ms / 1000)
            await self._on_timer()

        self._timer_task = asyncio.create_task(tick())

    async def _on_timer(self) -> None:
        now = _now_ms()
        due_jobs = [
            j for j in self._store.jobs
            if j.enabled and now >= j.state.next_run_at_ms
        ]

        for job in due_jobs:
            await self._execute_job(job)
            job.state.last_run_at_ms = now
            job.state.next_run_at_ms = self._compute_next_run(job)

        self._save_store()
        self._arm_timer()

    async def _execute_job(self, job: CronJob) -> None:
        try:
            result = await self._run_action(job.action)
            job.state.last_status = "success"
            job.state.last_result = result
        except Exception as e:
            job.state.last_status = "error"
            job.state.last_error = str(e)
            # 指数退避重试
            if job.state.consecutive_failures < job.config.max_retries:
                job.state.next_run_at_ms = _now_ms() + (
                    2 ** job.state.consecutive_failures * 1000
                )
```

---

## PicoClaw - ReAct工具循环

**File:** `picoclaw/pkg/tools/toolloop.go`

```go
func RunToolLoop(ctx context.Context, config ToolLoopConfig,
                 messages []providers.Message,
                 tools []Tool) (string, error) {
    maxIterations := config.MaxIterations
    if maxIterations == 0 {
        maxIterations = 25
    }

    iteration := 0
    for iteration < maxIterations {
        iteration++

        // 转换工具为provider格式
        providerToolDefs := make([]providers.ToolDefinition, len(tools))
        for i, tool := range tools {
            providerToolDefs[i] = providers.ToolDefinition{
                Name:        tool.Name(),
                Description: tool.Description(),
                Parameters:  tool.Parameters(),
            }
        }

        // 调用LLM
        response, err := config.Provider.Chat(ctx, messages, providerToolDefs, config.MaxTokens)
        if err != nil {
            return "", fmt.Errorf("llm call failed: %w", err)
        }

        // 无工具调用则完成
        if len(response.ToolCalls) == 0 {
            return response.Content, nil
        }

        // 并行执行工具调用
        toolResults := make([]ToolResult, len(response.ToolCalls))
        var wg sync.WaitGroup

        for i, tc := range response.ToolCalls {
            wg.Add(1)
            go func(idx int, tc providers.ToolCall) {
                defer wg.Done()

                tool := findTool(tools, tc.Name)
                if tool == nil {
                    toolResults[idx] = ToolResult{
                        ToolCallID: tc.ID,
                        Content:    fmt.Sprintf("Tool not found: %s", tc.Name),
                        IsError:    true,
                    }
                    return
                }

                result := tool.Execute(tc.Arguments)
                toolResults[idx] = ToolResult{
                    ToolCallID: tc.ID,
                    Content:    result.Content,
                    IsError:    result.IsError,
                }
            }(i, tc)
        }
        wg.Wait()

        // 构建下一轮消息
        messages = append(messages, providers.Message{
            Role:      providers.RoleAssistant,
            Content:   response.Content,
            ToolCalls: response.ToolCalls,
        })

        for _, tr := range toolResults {
            messages = append(messages, providers.Message{
                Role:       providers.RoleTool,
                ToolCallID: tr.ToolCallID,
                Content:    tr.Content,
                IsError:    tr.IsError,
            })
        }
    }

    return "", fmt.Errorf("max iterations (%d) reached", maxIterations)
}
```

---

## Mimiclaw - 嵌入式Cron

**File:** `mimiclaw/main/cron/cron_service.c`

```c
typedef enum {
    CRON_KIND_INTERVAL = 0,
    CRON_KIND_AT = 1,
} cron_job_kind_t;

typedef struct {
    char id[32];
    char channel[64];
    char message[256];
    cron_job_kind_t kind;
    time_t interval_s;
    time_t next_run;
    time_t last_run;
    bool enabled;
} cron_job_t;

static cron_job_t s_jobs[CRON_MAX_JOBS];
static int s_job_count = 0;
static TimerHandle_t s_cron_timer = NULL;

static void cron_timer_callback(TimerHandle_t xTimer)
{
    cron_process_due_jobs();
}

static void cron_process_due_jobs(void)
{
    time_t now = time(NULL);
    bool changed = false;

    for (int i = 0; i < s_job_count; i++) {
        cron_job_t *job = &s_jobs[i];
        if (!job->enabled) continue;
        if (job->next_run > now) continue;

        // 作业到期 - 通过消息总线触发
        mimi_msg_t msg;
        strncpy(msg.channel, job->channel, sizeof(msg.channel) - 1);
        msg.content = strdup(job->message);

        esp_err_t err = message_bus_push_inbound(&msg);
        if (err != ESP_OK) {
            ESP_LOGE(TAG, "Failed to push message: %d", err);
        }

        job->last_run = now;
        changed = true;

        if (job->kind == CRON_KIND_AT) {
            job->enabled = false; // One-shot
        } else {
            job->next_run = now + job->interval_s;
        }
    }

    if (changed) {
        cron_persist();
    }
}

esp_err_t cron_init(void)
{
    cron_load(); // 从JSON加载

    s_cron_timer = xTimerCreate(
        "cron",
        pdMS_TO_TICKS(1000), // 1秒滴答
        pdTRUE,              // 自动重载
        NULL,
        cron_timer_callback
    );

    if (s_cron_timer == NULL) {
        return ESP_ERR_NO_MEM;
    }

    xTimerStart(s_cron_timer, 0);
    return ESP_OK;
}
```

---

## CoPaw - APScheduler + ReAct

### Cron管理器

**File:** `CoPaw/src/copaw/app/crons/manager.py`

```python
class CronManager:
    def __init__(self, repo: BaseJobRepository, runner: Any,
                 channel_manager: Any, timezone: str = "UTC"):
        self._repo = repo
        self._runner = runner
        self._channel = channel_manager
        self._tz = timezone
        self._scheduler = AsyncIOScheduler(timezone=timezone)
        self._rt: dict[str, _Runtime] = {}

    async def start(self) -> None:
        specs = await self._repo.list_enabled()
        for spec in specs:
            await self._register_or_update(spec)
        self._scheduler.start()

    async def _register_or_update(self, spec: CronJobSpec) -> None:
        # 每个作业的并发信号量
        self._rt[spec.id] = _Runtime(
            sem=asyncio.Semaphore(spec.runtime.max_concurrency),
        )

        trigger = self._build_trigger(spec)

        self._scheduler.add_job(
            self._scheduled_callback,
            trigger=trigger,
            id=spec.id,
            args=[spec.id],
            misfire_grace_time=spec.runtime.misfire_grace_seconds,
            replace_existing=True,
        )

    async def _scheduled_callback(self, job_id: str) -> None:
        spec = await self._repo.get(job_id)
        if not spec or not spec.enabled:
            return

        rt = self._rt.get(spec.id)
        if not rt:
            return

        async with rt.sem:  # 并发控制
            await self._runner.run(spec)
            spec.runtime.last_run_at = utc_now()
            await self._repo.upsert(spec)
```

### ReAct Agent

**File:** `CoPaw/src/copaw/agents/react_agent.py`

```python
class CoPawAgent(ToolGuardMixin, ReActAgent):
    """CoPaw Agent with integrated tools, skills, and memory management."""

    def __init__(
        self,
        model: str | None = None,
        max_iters: int = 30,
        formatter: Formatter | None = None,
        namesake_strategy: NamesakeStrategy = NamesakeStrategy.ALWAYS_PREPEND,
    ):
        toolkit = self._create_toolkit(namesake_strategy=namesake_strategy)
        self._register_skills(toolkit)

        sys_prompt = self._build_sys_prompt()

        super().__init__(
            name="Friday",
            model=model,
            sys_prompt=sys_prompt,
            toolkit=toolkit,
            memory=InMemoryMemory(),
            formatter=formatter,
            max_iters=max_iters,
        )
```

---

## OpenClaw - 技能工具库

OpenClaw实现了基于Markdown的技能系统，技能可以注册为可调度工具。

```
skills/
├── skill-name/
│   ├── SKILL.md          # 技能定义、提示词、工具描述
│   ├── tools/
│   │   ├── tool1.py      # 工具实现
│   │   └── tool2.py
│   └── config.yaml       # 技能配置
```

技能在运行时动态加载并注册到工具调度器：

```python
# 伪代码 - OpenClaw技能加载
class SkillRegistry:
    def load_skill(self, skill_path: Path) -> Skill:
        skill_md = parse_skill_md(skill_path / "SKILL.md")
        for tool_def in skill_md.tools:
            self.tool_registry.register(
                name=tool_def.name,
                handler=load_tool_handler(skill_path / "tools" / tool_def.file),
                schema=tool_def.schema
            )
```

---

## OpenCode - 简单间隔调度器

**File:** `opencode/packages/opencode/src/scheduler/index.ts`

```typescript
export namespace Scheduler {
    export type Task = {
        id: string
        interval: number
        run: () => Promise<void>
        scope?: "instance" | "global"
    }

    const state = () => ({
        tasks: new Map<string, Task>(),
        timers: new Map<string, NodeJS.Timeout>(),
    })

    const shared = {
        tasks: new Map<string, Task>(),
        timers: new Map<string, NodeJS.Timeout>(),
    }

    export function register(task: Task) {
        const scope = task.scope ?? "instance"
        const entry = scope === "global" ? shared : state()

        if (entry.timers.has(task.id)) {
            clearInterval(entry.timers.get(task.id)!)
        }

        entry.tasks.set(task.id, task)

        const run = async () => {
            try {
                await task.run()
            } catch (err) {
                console.error(`Scheduled task ${task.id} failed:`, err)
            }
        }

        void run()
        const timer = setInterval(() => {
            void run()
        }, task.interval)

        entry.timers.set(task.id, timer)
    }

    export function unregister(id: string, scope?: "instance" | "global") {
        const s = scope ?? "instance"
        const entry = s === "global" ? shared : state()

        if (entry.timers.has(id)) {
            clearInterval(entry.timers.get(id)!)
            entry.timers.delete(id)
        }
        entry.tasks.delete(id)
    }
}
```

---

## Agent-Browser - 浏览器任务队列

**File:** `agent-browser/src/daemon.ts`

```typescript
interface Task {
  id: string;
  type: 'navigate' | 'click' | 'type' | 'screenshot' | 'evaluate' | 'scroll';
  payload: unknown;
  resolve: (value: unknown) => void;
  reject: (reason: Error) => void;
}

class BrowserDaemon {
  private taskQueue: Task[] = [];
  private isProcessing = false;
  private browser?: Browser;

  async enqueue<T>(type: Task['type'], payload: unknown): Promise<T> {
    return new Promise((resolve, reject) => {
      const task: Task = {
        id: crypto.randomUUID(),
        type,
        payload,
        resolve,
        reject,
      };
      this.taskQueue.push(task);
      this.processQueue();
    });
  }

  private async processQueue(): Promise<void> {
    if (this.isProcessing || this.taskQueue.length === 0) return;

    this.isProcessing = true;
    const task = this.taskQueue.shift()!;

    try {
      if (!this.browser) {
        this.browser = await this.launchBrowser();
      }

      const result = await this.executeTask(task);
      task.resolve(result);
    } catch (error) {
      task.reject(error instanceof Error ? error : new Error(String(error)));
    } finally {
      this.isProcessing = false;
      setImmediate(() => this.processQueue());
    }
  }
}
```

---

## 架构对比矩阵

### 工作流引擎功能对比

| 功能 | Edict | IronClaw | Codex | Gemini | Qwen | Kimi | Pi Mono | NanoClaw | ZeroClaw | NanoBot | CoPaw | PicoClaw | Mimiclaw |
|------|-------|----------|-------|--------|------|------|---------|----------|----------|---------|-------|----------|----------|
| 状态机 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| Cron调度 | ⚪ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ✅ | ✅ | ✅ | ✅ | ⚪ | ✅ |
| 后台任务 | ⚪ | ✅ | ✅ | ⚪ | ⚪ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| 并发控制 | ✅ | ✅ | ✅ | ✅ | ⚪ | ⚪ | ✅ | ✅ | ✅ | ⚪ | ✅ | ✅ | ⚪ |
| 队列系统 | ✅ | ✅ | ✅ | ✅ | ⚪ | ⚪ | ✅ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| 心跳监控 | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| 尾调用 | ⚪ | ⚪ | ⚪ | ✅ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| 策略检查 | ⚪ | ⚪ | ⚪ | ✅ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| 事件驱动 | ✅ | ⚪ | ⚪ | ✅ | ⚪ | ⚪ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| 持久化 | ✅ | ✅ | ✅ | ⚪ | ⚪ | ✅ | ⚪ | ✅ | ✅ | ✅ | ✅ | ⚪ | ✅ |

### 调度策略对比

| 项目 | 调度策略 | 并发模型 | 错误处理 |
|------|----------|----------|----------|
| Edict | Redis Streams | 信号量限制 | ACK+重试 |
| IronClaw | Job/Subtask层级 | Semaphore | 状态机转换 |
| Codex | CSV批处理 | Worker Pool | 轮询+取消 |
| Gemini CLI | 队列处理 | 并行/顺序可配置 | 状态机 |
| Qwen Code | 顺序策略检查 | 顺序 | 错误传播 |
| Kimi CLI | 后台进程 | 单进程 | 心跳检测 |
| Pi Mono | 双队列 | 并行/顺序 | 异常捕获 |
| NanoClaw | Cron + 组队列 | 3容器限制 | 重试 |
| ZeroClaw | Stream | buffer_unordered | 错误日志 |
| CoPaw | APScheduler | 信号量 | 异常捕获 |

---

## 推荐统一工作流架构

```yaml
# workflow-config.yaml
workflow:
  engine:
    type: "hybrid"  # state_machine | queue | event_driven

  scheduler:
    # Cron调度
    cron:
      enabled: true
      poll_interval: 60s
      timezone: "UTC"
      max_concurrent: 10

    # 队列配置
    queue:
      type: "redis"  # redis | memory | postgres
      concurrency: 16
      retry_policy:
        max_attempts: 3
        backoff: exponential

    # 批处理
    batch:
      enabled: true
      chunk_size: 100
      worker_pool: 8

  state_machine:
    states:
      - pending
      - validating
      - scheduled
      - executing
      - completed
      - failed
      - cancelled

    transitions:
      pending: [validating, cancelled]
      validating: [scheduled, failed]
      scheduled: [executing, cancelled]
      executing: [completed, failed, cancelled]

  error_handling:
    retry:
      enabled: true
      max_retries: 3
      delay: 1s
      max_delay: 60s

    dead_letter:
      enabled: true
      queue: "failed_jobs"
      ttl: 7d

  monitoring:
    heartbeat:
      enabled: true
      interval: 30s
      timeout: 120s

    metrics:
      - job_duration
      - queue_depth
      - error_rate
      - throughput
```

### 核心组件架构

```rust
// 推荐统一接口设计
#[async_trait]
pub trait WorkflowEngine: Send + Sync {
    async fn submit(&self, job: Job) -> Result<JobId>;
    async fn cancel(&self, id: JobId) -> Result<()>;
    async fn status(&self, id: JobId) -> Result<JobStatus>;
    async fn subscribe(&self, id: JobId) -> Result<Stream<JobEvent>>;
}

#[async_trait]
pub trait Scheduler: Send + Sync {
    async fn schedule(&self, spec: ScheduleSpec, job: Job) -> Result<()>;
    async fn unschedule(&self, id: ScheduleId) -> Result<()>;
    async fn list(&self) -> Result<Vec<ScheduledJob>>;
}

#[async_trait]
pub trait Queue: Send + Sync {
    async fn enqueue(&self, job: Job) -> Result<()>;
    async fn dequeue(&self) -> Result<Option<Job>>;
    async fn ack(&self, id: JobId) -> Result<()>;
    async fn nack(&self, id: JobId, requeue: bool) -> Result<()>;
}
```

### 分层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                      API Layer                                  │
│         REST / GraphQL / WebSocket / CLI                        │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                   Workflow Engine                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   Cron      │  │   Queue     │  │    State Machine        │ │
│  │ Scheduler   │  │  Manager    │  │      Engine             │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                  Execution Layer                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   Worker    │  │  Executor   │  │      Runner             │ │
│  │   Pool      │  │ (Tool/LLM)  │  │   (Agent Loop)          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                  Persistence Layer                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   State     │  │   Event     │  │       Queue             │ │
│  │   Store     │  │    Log      │  │    Backend              │ │
│  │ (Postgres)  │  │ (EventStore)│  │  (Redis/SQL)            │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```
