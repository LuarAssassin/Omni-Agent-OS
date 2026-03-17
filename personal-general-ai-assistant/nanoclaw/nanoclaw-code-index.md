# NanoClaw 代码索引

> 快速导航与代码结构参考

---

## 目录

1. [源码结构总览](#源码结构总览)
2. [文件导航](#文件导航)
3. [核心类型定义](#核心类型定义)
4. [函数与类索引](#函数与类索引)
5. [关键代码路径](#关键代码路径)
6. [模块依赖图](#模块依赖图)

---

## 源码结构总览

```
src/
├── index.ts              # 主入口 - 编排器 (~650行)
├── config.ts             # 配置常量 (~70行)
├── types.ts              # 类型定义 (~110行)
├── db.ts                 # 数据库操作 (~700行)
├── logger.ts             # 日志配置 (~20行)
├── router.ts             # 消息路由 (~50行)
├── ipc.ts                # IPC通信 (~450行)
├── group-queue.ts        # 群组队列 (~370行)
├── container-runner.ts   # 容器运行 (~710行)
├── container-runtime.ts  # 容器运行时 (~130行)
├── task-scheduler.ts     # 任务调度 (~280行)
├── mount-security.ts     # 挂载安全 (~420行)
├── credential-proxy.ts   # 凭证代理 (~130行)
├── sender-allowlist.ts   # 发送者白名单 (~130行)
├── remote-control.ts     # 远程控制 (~230行)
├── group-folder.ts       # 群组文件夹 (~50行)
├── env.ts                # 环境变量解析 (~40行)
└── channels/
    ├── registry.ts       # 频道注册表 (~30行)
    └── *.ts              # 各频道实现
```

---

## 文件导航

### A. 核心编排层

#### `src/index.ts` - 主编排器

**职责**: 系统主入口，协调所有子系统运行

**导出**:
```typescript
// 无命名导出，副作用启动
```

**核心状态**:
| 变量 | 类型 | 用途 |
|------|------|------|
| `lastTimestamp` | `string` | 全局消息游标（上次处理的消息ID） |
| `sessions` | `Record<string, string>` | group folder → session ID 映射 |
| `registeredGroups` | `Record<string, RegisteredGroup>` | 已注册群组缓存 |
| `lastAgentTimestamp` | `Record<string, string>` | 每群组已处理游标 |
| `channels` | `Channel[]` | 已连接频道实例列表 |
| `queue` | `GroupQueue` | 全局消息队列实例 |

**关键函数**:

```typescript
// 启动消息循环
async function startMessageLoop(): Promise<void>

// 处理单个群组消息
async function processGroupMessages(
  group: RegisteredGroup,
  messages: NewMessage[]
): Promise<void>

// 运行Agent容器
async function runAgent(
  group: RegisteredGroup,
  messages: NewMessage[],
  taskContext?: TaskContext
): Promise<void>

// 崩溃恢复 - 检测未处理消息
async function recoverPendingMessages(): Promise<void>
```

**代码量**: ~650行（核心编排逻辑）

---

#### `src/config.ts` - 配置常量

**职责**: 集中管理所有可配置参数

**关键导出**:

```typescript
// 基础配置
export const ASSISTANT_NAME = 'NanoClaw';
export const POLL_INTERVAL = 1000;           // 消息轮询间隔(ms)
export const SCHEDULER_POLL_INTERVAL = 60000; // 任务调度轮询(ms)

// 并发控制
export const MAX_CONCURRENT_CONTAINERS = 5;  // 全局并发容器数

// 超时设置
export const CONTAINER_TIMEOUT = 10 * 60 * 1000;  // 10分钟
export const IDLE_TIMEOUT = 5 * 60 * 1000;        // 5分钟空闲
export const TASK_CLOSE_DELAY_MS = 10_000;        // 任务后关闭延迟

// 路径配置
export const STORE_DIR = path.join(process.cwd(), 'store');
export const GROUPS_DIR = path.join(process.cwd(), 'groups');
export const DATA_DIR = path.join(process.cwd(), 'data');
export const IPC_DIR = path.join(DATA_DIR, 'ipc');

// 安全配置路径
export const MOUNT_ALLOWLIST_PATH = path.join(
  os.homedir(), '.config/nanoclaw/mount-allowlist.json'
);
export const SENDER_ALLOWLIST_PATH = path.join(
  os.homedir(), '.config/nanoclaw/sender-allowlist.json'
);

// 触发模式
export const TRIGGER_PATTERN = new RegExp(
  `^@${escapeRegex(ASSISTANT_NAME)}\\b`, 'i'
);
```

---

#### `src/types.ts` - 类型定义

**职责**: 核心TypeScript接口和类型定义

**关键接口**:

```typescript
// 频道接口 - 所有消息通道实现此接口
export interface Channel {
  readonly name: string;
  connect(onMessage: OnInboundMessage): Promise<void>;
  sendMessage(jid: string, text: string, options?: MessageOptions): Promise<void>;
  ownsJid(jid: string): boolean;
  setTyping(jid: string, typing: boolean): Promise<void>;
  syncGroups(): Promise<Array<{ jid: string; name: string; is_group: boolean }>>;
}

// 已注册群组
export interface RegisteredGroup {
  folder: string;
  isMain: boolean;
  requiresTrigger: boolean;
  containerConfig?: {
    additionalMounts?: AdditionalMount[];
    env?: Record<string, string>;
  };
}

// 消息结构
export interface NewMessage {
  id: string;
  chat_jid: string;
  sender: string;
  content: string;
  timestamp: string;
  is_from_me: boolean;
  is_bot_message: boolean;
}

// 定时任务
export interface ScheduledTask {
  id: string;
  group_folder: string;
  prompt: string;
  schedule_type: 'cron' | 'interval' | 'once';
  schedule_value: string;
  next_run_at: string;
  last_run_at?: string;
  is_paused: boolean;
  context_mode: 'group' | 'isolated';
  created_at: string;
}

// 额外挂载配置
export interface AdditionalMount {
  hostPath: string;
  containerPath: string;
  readonly?: boolean;
}

// 挂载安全
export interface MountAllowlist {
  allowedRoots: string[];
  blockedPatterns: string[];
  nonMainReadOnly: boolean;
}
```

---

### B. 数据层

#### `src/db.ts` - 数据库操作

**职责**: SQLite数据库封装，所有持久化操作

**关键函数**:

```typescript
// ========== 消息操作 ==========
export function getNewMessages(
  chatJid: string,
  afterTimestamp: string,
  limit?: number
): NewMessage[];

export function getMessagesSince(
  chatJid: string,
  since: Date,
  limit?: number
): NewMessage[];

export function storeMessage(msg: NewMessage): void;

export function markMessagesProcessed(
  chatJid: string,
  upToTimestamp: string
): void;

// ========== 群组管理 ==========
export function getRegisteredGroup(folder: string): RegisteredGroup | undefined;
export function setRegisteredGroup(group: RegisteredGroup): void;
export function getAllRegisteredGroups(): RegisteredGroup[];

// ========== 会话管理 ==========
export function getSession(groupFolder: string): string | undefined;
export function setSession(groupFolder: string, sessionId: string): void;
export function getAllSessions(): Record<string, string>;

// ========== 任务调度 ==========
export function createTask(task: Omit<ScheduledTask, 'id' | 'created_at'>): string;
export function getDueTasks(now?: Date): ScheduledTask[];
export function updateTaskAfterRun(taskId: string, nextRunAt: Date): void;
export function pauseTask(taskId: string): void;
export function resumeTask(taskId: string): void;
export function cancelTask(taskId: string): void;
export function updateTask(
  taskId: string,
  updates: Partial<Pick<ScheduledTask, 'prompt' | 'schedule_value'>>
): void;

// ========== 任务日志 ==========
export function logTaskRun(
  taskId: string,
  success: boolean,
  output?: string,
  error?: string
): void;

// ========== 路由器状态 ==========
export function getRouterState(key: string): string | undefined;
export function setRouterState(key: string, value: string): void;

// ========== 迁移 ==========
export function runMigrations(): void;
export function migrateFromJson(): void;
```

**Schema版本**: v1（支持context_mode, is_bot_message等列）

---

### C. 容器层

#### `src/container-runner.ts` - 容器运行器

**职责**: 深模块典范 - 复杂挂载逻辑隐藏在简单接口后

**核心导出**:

```typescript
// 容器输入结构
export interface ContainerInput {
  messages: Array<{
    role: 'user' | 'assistant';
    content: string;
  }>;
  systemPrompt?: string;
  cwd: string;
  allowedTools?: string[];
}

// 容器输出结构
export interface ContainerOutput {
  response?: string;
  sessionId?: string;
  error?: string;
}

// 主函数 - 运行容器Agent
export async function runContainerAgent(
  group: RegisteredGroup,
  input: ContainerInput,
  registerProcess: (pid: number) => void,
  onOutput: (chunk: string) => Promise<void>
): Promise<ContainerOutput>;
```

**内部函数**:

```typescript
// 构建挂载配置（核心安全逻辑）
function buildVolumeMounts(group: RegisteredGroup): MountConfig[];

// 启动凭证代理
async function startProxy(): Promise<{ port: number; stop: () => void }>;

// 解析流式输出
function parseOutputLine(line: string): OutputLine | null;

// 写入任务快照
function writeTasksSnapshot(groupFolder: string): void;

// 写入群组快照
function writeGroupsSnapshot(groupFolder: string): void;
```

**输出标记**:
```
---NANOCLAW_OUTPUT_START---
{json content}
---NANOCLAW_OUTPUT_END---
```

---

#### `src/container-runtime.ts` - 容器运行时

**职责**: 容器运行时抽象与平台适配

**关键导出**:

```typescript
// 运行时命令
export const CONTAINER_RUNTIME_BIN = 'docker';

// 容器内访问主机的域名
export const CONTAINER_HOST_GATEWAY = 'host.docker.internal';

// 代理绑定地址（平台相关）
export const PROXY_BIND_HOST: string;
// macOS/WSL: '127.0.0.1' (Docker Desktop VM路由)
// Linux: docker0网桥IP

// 清理孤立容器
export async function cleanupOrphanedContainers(): Promise<void>;
```

**平台检测逻辑**:
```typescript
if (process.platform === 'darwin' || isWSL()) {
  PROXY_BIND_HOST = '127.0.0.1';  // Docker Desktop
} else {
  PROXY_BIND_HOST = getDockerBridgeIP();  // 裸机Linux
}
```

---

### D. 队列与调度

#### `src/group-queue.ts` - 群组队列

**职责**: 全局并发控制、消息排序、指数退避重试

**类定义**:

```typescript
export class GroupQueue {
  // 全局配置
  private maxConcurrent: number;
  private baseRetryMs: number;
  private maxRetries: number;

  // 状态
  private runningContainers: Map<string, GroupContainerState>;
  private pendingMessages: Map<string, NewMessage[]>;
  private pendingTasks: Map<string, TaskContext[]>;

  // 核心方法
  constructor(maxConcurrent?: number);

  // 消息入队
  enqueueMessageCheck(groupFolder: string): void;

  // 任务入队
  enqueueTask(groupFolder: string, task: TaskContext): void;

  // 注册运行中容器
  registerProcess(
    groupFolder: string,
    pid: number,
    stdin: Writable
  ): void;

  // 通知空闲
  notifyIdle(groupFolder: string): void;

  // 关闭标准输入
  closeStdin(groupFolder: string): void;

  // 发送消息到容器
  sendMessage(groupFolder: string, content: string): void;

  // 获取活跃容器数
  getActiveCount(): number;
}

// 容器状态
interface GroupContainerState {
  pid: number;
  stdin: Writable;
  active: boolean;
  idleWaiting: boolean;
  retryCount: number;
}
```

**重试逻辑**:
```typescript
delayMs = BASE_RETRY_MS * Math.pow(2, retryCount - 1)
// 1次: 2000ms, 2次: 4000ms, 3次: 8000ms...
```

---

#### `src/task-scheduler.ts` - 任务调度器

**职责**: 定时任务调度，防止漂移

**关键导出**:

```typescript
// 启动调度器
export function startTaskScheduler(
  onTaskDue: (task: ScheduledTask) => void
): void;

// 停止调度器
export function stopTaskScheduler(): void;

// 计算下次运行时间（内部）
function computeNextRun(
  scheduleType: 'cron' | 'interval' | 'once',
  scheduleValue: string,
  lastRun?: Date,
  now?: Date
): Date | null;
```

**漂移防止**:
```typescript
// 基于上次实际运行时间计算下次
const nextRun = lastRun
  ? new Date(lastRun.getTime() + intervalMs)
  : new Date(now.getTime() + intervalMs);
```

---

### E. 通信层

#### `src/ipc.ts` - IPC处理器

**职责**: 文件系统IPC，处理容器到主机的指令

**核心函数**:

```typescript
// 启动IPC监听
export function startIpcWatcher(
  onMessage: (chatJid: string, text: string) => void,
  onTask: (task: IpcTask) => void,
  onGroupRefresh: () => void,
  onGroupRegister: (group: RegisteredGroup) => void
): void;

// 停止监听
export function stopIpcWatcher(): void;

// 处理任务指令（内部）
function processTaskIpc(
  groupFolder: string,
  task: IpcTask,
  callbacks: TaskCallbacks
): void;

// 权限检查（内部）
function checkAuth(
  groupFolder: string,
  requestingGroup: string
): boolean;
```

**指令集**:
| 指令 | 权限 | 功能 |
|------|------|------|
| `schedule_task` | 本组 | 创建定时任务 |
| `pause_task` | 本组 | 暂停任务 |
| `resume_task` | 本组 | 恢复任务 |
| `cancel_task` | 本组 | 取消任务 |
| `update_task` | 本组 | 更新任务 |
| `refresh_groups` | 任意 | 刷新群组列表 |
| `register_group` | 主组 | 注册新群组 |

**IPC目录结构**:
```
data/ipc/{groupFolder}/
├── input/      # 容器输入
├── output/     # 容器输出
├── messages/   # 发送给用户的消息
└── tasks/      # 任务指令
```

---

#### `src/router.ts` - 消息路由

**职责**: 消息格式化与出站路由

**导出函数**:

```typescript
// 格式化消息为XML
export function formatMessages(
  messages: NewMessage[],
  timezone?: string
): string;

// XML转义
export function escapeXml(text: string): string;

// 去除内部标签
export function stripInternalTags(text: string): string;

// 格式化出站消息
export function formatOutbound(text: string): string;

// 查找频道
export function findChannel(
  channels: Channel[],
  jid: string
): Channel | undefined;
```

---

### F. 安全层

#### `src/mount-security.ts` - 挂载安全

**职责**: 容器挂载路径验证与访问控制

**关键导出**:

```typescript
// 验证结果
export interface MountValidationResult {
  allowed: boolean;
  reason?: string;
}

// 加载允许列表配置
export function loadAllowlist(): MountAllowlist;

// 验证挂载路径
export function validateMount(
  hostPath: string,
  config: MountAllowlist,
  isMainGroup: boolean
): MountValidationResult;

// 解析符号链接
export function getRealPath(p: string): string;

// 生成配置模板
export function generateAllowlistTemplate(): string;
```

**默认阻止模式**:
```typescript
const DEFAULT_BLOCKED_PATTERNS = [
  '**/.ssh/**',
  '**/.gnupg/**',
  '**/.aws/**',
  '**/.env',
  '**/.env.*',
  '**/node_modules/**',
  '**/.git/**',
];
```

---

#### `src/credential-proxy.ts` - 凭证代理

**职责**: 容器与Anthropic API之间的凭证代理

**关键导出**:

```typescript
// 启动代理
export async function startCredentialProxy(
  port: number,
  host: string
): Promise<Server>;

// 检测认证模式
export function detectAuthMode(): 'api-key' | 'oauth';

// 停止代理
export function stopCredentialProxy(): Promise<void>;
```

**认证模式**:
- **API Key模式**: 从`.env`读取`ANTHROPIC_API_KEY`，注入请求头
- **OAuth模式**: 从`.env`读取`CLAUDE_CODE_OAUTH_TOKEN`，交换临时密钥

---

#### `src/sender-allowlist.ts` - 发送者白名单

**职责**: 发送者过滤与权限控制

**关键导出**:

```typescript
// 配置结构
export interface SenderAllowlistConfig {
  default: ChatAllowlistEntry;
  chats: Record<string, ChatAllowlistEntry>;
  logDenied: boolean;
}

export interface ChatAllowlistEntry {
  mode: 'trigger' | 'drop';
  senders: string[];
}

// 加载配置
export function loadSenderAllowlist(): SenderAllowlistConfig;

// 检查发送者权限
export function isSenderAllowed(
  config: SenderAllowlistConfig,
  chatJid: string,
  sender: string
): boolean;

// 是否应该丢弃消息
export function shouldDropMessage(
  config: SenderAllowlistConfig,
  chatJid: string,
  sender: string
): boolean;

// 是否允许触发
export function isTriggerAllowed(
  config: SenderAllowlistConfig,
  chatJid: string,
  sender: string
): boolean;
```

**模式说明**:
- `trigger`: 允许列表中的发送者可触发bot
- `drop`: 丢弃非允许列表发送者的所有消息

---

### G. 频道层

#### `src/channels/registry.ts` - 频道注册表

**职责**: 自注册工厂模式实现

**关键导出**:

```typescript
// 频道工厂类型
export type ChannelFactory = (opts: ChannelOpts) => Channel | null;

// 注册频道
export function registerChannel(
  name: string,
  factory: ChannelFactory
): void;

// 获取频道工厂
export function getChannelFactory(
  name: string
): ChannelFactory | undefined;

// 获取所有已注册频道名
export function getRegisteredChannelNames(): string[];
```

**使用示例**:
```typescript
// channels/whatsapp.ts
import { registerChannel } from './registry.js';

registerChannel('whatsapp', (opts) => {
  if (!hasCredentials()) return null;  // 未配置，跳过
  return new WhatsAppChannel(opts);
});
```

---

### H. 工具层

#### `src/remote-control.ts` - 远程控制

**职责**: Claude Code远程控制会话管理

**关键导出**:

```typescript
// 会话信息
export interface RemoteControlSession {
  pid: number;
  url: string;
  startedBy: string;
  startedInChat: string;
  startedAt: string;
}

// 启动会话
export async function startRemoteControl(
  sender: string,
  chatJid: string,
  cwd: string
): Promise<{ ok: true; url: string } | { ok: false; error: string }>;

// 停止会话
export function stopRemoteControl():
  | { ok: true }
  | { ok: false; error: string };

// 获取活跃会话
export function getActiveSession(): RemoteControlSession | null;

// 恢复会话（启动时调用）
export function restoreRemoteControl(): void;
```

---

#### `src/group-folder.ts` - 群组文件夹

**职责**: 群组文件夹名称验证与路径解析

**关键导出**:

```typescript
// 文件夹名格式：字母数字开头，可包含连字符下划线
const GROUP_FOLDER_PATTERN = /^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$/;

// 保留名
const RESERVED_FOLDERS = new Set(['global']);

// 验证
export function isValidGroupFolder(name: string): boolean;
export function assertValidGroupFolder(name: string): void;

// 路径解析
export function resolveGroupFolderPath(name: string): string;
export function resolveGroupIpcPath(name: string): string;
```

---

#### `src/env.ts` - 环境变量

**职责**: `.env`文件解析，防止秘密泄露

**关键导出**:

```typescript
// 读取指定键的环境变量
export function readEnvFile(keys: string[]): Record<string, string>;

// 示例
const secrets = readEnvFile(['ANTHROPIC_API_KEY', 'CLAUDE_CODE_OAUTH_TOKEN']);
// 仅返回请求的键，避免加载全部到process.env
```

---

#### `src/logger.ts` - 日志

**职责**: Pino日志配置

**导出**:

```typescript
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: {
    target: 'pino-pretty',
    options: { colorize: true }
  }
});
```

---

## 核心类型定义

### 消息流类型

```typescript
// 入站消息（数据库存储）
interface NewMessage {
  id: string;              // 消息唯一ID
  chat_jid: string;        // 聊天JID
  sender: string;          // 发送者
  content: string;         // 内容
  timestamp: string;       // ISO时间戳
  is_from_me: boolean;     // 是否自己发送
  is_bot_message: boolean; // 是否bot消息（防循环）
}

// 容器输入格式
interface ContainerInput {
  messages: Array<{
    role: 'user' | 'assistant';
    content: string;
  }>;
  systemPrompt?: string;
  cwd: string;
  allowedTools?: string[];
}

// 任务上下文
interface TaskContext {
  taskId: string;
  prompt: string;
  contextMode: 'group' | 'isolated';
}
```

### 安全类型

```typescript
// 挂载配置
interface AdditionalMount {
  hostPath: string;        // 主机路径（绝对）
  containerPath: string;   // 容器内路径
  readonly?: boolean;      // 是否只读
}

// 挂载验证结果
interface MountValidationResult {
  allowed: boolean;
  reason?: string;         // 拒绝原因
}
```

---

## 函数与类索引

### 类索引

| 类 | 文件 | 职责 |
|----|------|------|
| `GroupQueue` | `group-queue.ts` | 全局并发控制与消息队列 |

### 核心函数索引

| 函数 | 文件 | 用途 |
|------|------|------|
| `runContainerAgent` | `container-runner.ts` | 运行容器Agent（深模块） |
| `startMessageLoop` | `index.ts` | 启动消息循环 |
| `processGroupMessages` | `index.ts` | 处理群组消息 |
| `startIpcWatcher` | `ipc.ts` | 启动IPC监听 |
| `startTaskScheduler` | `task-scheduler.ts` | 启动任务调度 |
| `validateMount` | `mount-security.ts` | 验证挂载路径 |
| `startCredentialProxy` | `credential-proxy.ts` | 启动凭证代理 |
| `startRemoteControl` | `remote-control.ts` | 启动远程控制 |
| `registerChannel` | `channels/registry.ts` | 注册频道 |

---

## 关键代码路径

### 1. 消息处理流程

```
Channel.receive()
    ↓
db.storeMessage()
    ↓
MessageLoop.poll() ──→ db.getNewMessages()
    ↓
queue.enqueueMessageCheck()
    ↓
queue.processQueue() ──→ 并发控制检查
    ↓
runContainerAgent()
    ↓
Channel.sendMessage()
```

### 2. 容器启动流程

```
runContainerAgent(group, input)
    ↓
buildVolumeMounts(group) ──→ 安全校验
    ↓
startCredentialProxy() ──→ 端口分配
    ↓
spawn(docker run ...)
    ↓
stream parsing ──→ 标记解析
    ↓
return ContainerOutput
```

### 3. 任务调度流程

```
startTaskScheduler()
    ↓
setInterval() ──→ 每分钟检查
    ↓
db.getDueTasks()
    ↓
computeNextRun() ──→ 防漂移计算
    ↓
onTaskDue() ──→ queue.enqueueTask()
```

### 4. IPC指令流程

```
Container.writeTask()
    ↓
data/ipc/{group}/tasks/{timestamp}.json
    ↓
ipc.watch() ──→ processTaskIpc()
    ↓
checkAuth() ──→ 权限验证
    ↓
db.createTask() / pauseTask() / etc.
```

---

## 模块依赖图

### 顶层依赖

```
index.ts (编排器)
├── config.ts
├── types.ts
├── logger.ts
├── db.ts
├── router.ts
├── group-queue.ts
│   ├── config.ts
│   ├── logger.ts
│   └── types.ts
├── container-runner.ts
│   ├── config.ts
│   ├── logger.ts
│   ├── types.ts
│   ├── mount-security.ts
│   ├── credential-proxy.ts
│   └── container-runtime.ts
├── ipc.ts
│   ├── config.ts
│   ├── logger.ts
│   ├── types.ts
│   └── db.ts
├── task-scheduler.ts
│   ├── config.ts
│   ├── logger.ts
│   └── types.ts
└── channels/registry.ts
    └── types.ts
```

### 安全模块依赖

```
mount-security.ts
├── types.ts
└── config.ts (MOUNT_ALLOWLIST_PATH)

credential-proxy.ts
├── env.ts
└── logger.ts

sender-allowlist.ts
├── config.ts (SENDER_ALLOWLIST_PATH)
└── types.ts
```

### 工具模块

```
remote-control.ts
├── config.ts (DATA_DIR)
└── logger.ts

group-folder.ts
├── config.ts (GROUPS_DIR, IPC_DIR)
└── types.ts

env.ts (独立)

logger.ts (独立，被所有模块依赖)
```

---

## 代码复杂度分布

```
src/index.ts              ████████████████████  ~650行
src/container-runner.ts   ███████████████████  ~700行
src/db.ts                 ███████████████████  ~700行
src/ipc.ts                ██████████████       ~450行
src/group-queue.ts        ████████████         ~370行
src/mount-security.ts     ████████████         ~420行
src/task-scheduler.ts     █████████            ~280行
src/remote-control.ts     ████████             ~230行
src/credential-proxy.ts   █████                ~130行
src/sender-allowlist.ts   █████                ~130行
src/container-runtime.ts  █████                ~130行
src/types.ts              ████                 ~110行
src/router.ts             ██                   ~50行
src/config.ts             ██                   ~70行
src/group-folder.ts       ██                   ~50行
src/env.ts                █                    ~40行
src/logger.ts             █                    ~20行
src/channels/registry.ts  █                    ~30行
```

---

## 快速导航参考

### 按功能查找

| 功能 | 文件 |
|------|------|
| 修改轮询间隔 | `config.ts` - `POLL_INTERVAL` |
| 修改并发数 | `config.ts` - `MAX_CONCURRENT_CONTAINERS` |
| 修改容器超时 | `config.ts` - `CONTAINER_TIMEOUT` |
| 修改触发词 | `config.ts` - `ASSISTANT_NAME` |
| 添加挂载白名单 | `~/.config/nanoclaw/mount-allowlist.json` |
| 配置发送者过滤 | `~/.config/nanoclaw/sender-allowlist.json` |
| 查看日志输出 | 依赖 `LOG_LEVEL` 环境变量 |
| 添加新频道 | `channels/*.ts` + `registerChannel()` |
| 调试容器问题 | `container-runner.ts` + `ipc.ts` |
| 调试任务调度 | `task-scheduler.ts` |

### 按问题查找

| 问题 | 检查文件 |
|------|----------|
| 消息不触发 | `index.ts` - `TRIGGER_PATTERN` |
| 容器启动失败 | `container-runner.ts`, `container-runtime.ts` |
| 挂载被拒绝 | `mount-security.ts` |
| 认证失败 | `credential-proxy.ts` |
| 消息发送失败 | `router.ts`, 各 `channels/*.ts` |
| 任务不执行 | `task-scheduler.ts`, `db.ts` |
| IPC指令无效 | `ipc.ts` - `checkAuth()` |
| 内存/并发问题 | `group-queue.ts` |

---

*代码索引完成。详见 `nanoclaw-analysis.md` 了解设计哲学，查看 `nanoclaw-adr.md` 了解架构决策。*
