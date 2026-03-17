# NanoClaw 项目深度分析

> 基于 John Ousterhout《A Philosophy of Software Design》的设计理念分析

---

## 目录

1. [项目概述](#项目概述)
2. [设计理念分析](#设计理念分析)
3. [架构设计](#架构设计)
4. [核心模块分析](#核心模块分析)
5. [安全模型](#安全模型)
6. [设计模式与最佳实践](#设计模式与最佳实践)
7. [复杂度管理](#复杂度管理)

---

## 项目概述

### 项目定位

NanoClaw 是一个**个人 Claude 助手**，通过多通道消息系统（WhatsApp、Telegram、Discord、Slack、Gmail 等）与用户交互，使用容器化隔离执行 AI 代理任务。

### 核心特征

| 特征 | 实现方式 |
|------|----------|
| **轻量级** | 单一 Node.js 进程，~8,000 行代码 |
| **容器隔离** | 每个群组在独立容器中运行，文件系统隔离 |
| **多通道** | 自注册频道系统，按需加载 |
| **无配置** | 通过代码修改而非配置文件定制 |
| **技能系统** | 通过 git 分支合并添加功能 |

### 项目统计

```
代码量:     ~8,000 行 TypeScript
文件数:     30+ 个 .ts 文件
依赖数:     6 个核心依赖
架构:       单一进程 + 容器化代理
运行时:     Node.js 20+
```

---

## 设计理念分析

### 与 Ousterhout 原则的映射

#### 1. Small Enough to Understand

**原则**: 系统应小到可以被理解。

**NanoClaw 实践**:
- 单一 Node.js 进程，无微服务架构
- 核心逻辑集中在 `src/index.ts` (~650 行)
- 每个模块职责单一，边界清晰
- 无配置文件的"配置蔓延"

> "If you want to understand the full NanoClaw codebase, just ask Claude Code to walk you through it."

#### 2. Security Through Isolation

**原则**: 安全应通过隔离实现，而非应用层权限检查。

**NanoClaw 实践**:
```
┌─────────────────────────────────────┐
│  HOST (Node.js)                     │
│  ├─ 消息路由                        │
│  ├─ 频道管理                        │
│  └─ 队列调度                        │
└──────────────┬──────────────────────┘
               │ spawn container
               ▼
┌─────────────────────────────────────┐
│  CONTAINER (Linux VM)               │
│  ├─ Claude Agent SDK                │
│  ├─ 隔离文件系统                    │
│  └─ 仅挂载显式允许的路径            │
└─────────────────────────────────────┘
```

- 代理在容器中运行，非主机
- 挂载安全: 仅显式配置的路径可访问
- 凭证代理: 容器通过代理访问 API，不直接接触密钥

#### 3. Deep Modules (深模块)

**原则**: 简单接口，复杂实现。

**NanoClaw 实践**:

| 模块 | 接口复杂度 | 实现复杂度 | 深度 |
|------|------------|------------|------|
| `GroupQueue` | `enqueue()`, `sendMessage()` | 366 行，并发控制、重试逻辑 | 深 |
| `runContainerAgent()` | 单一函数调用 | 708 行，挂载构建、流处理 | 深 |
| `db.ts` | 简洁查询函数 | 698 行，Schema 管理、迁移 | 深 |
| `Channel` | `connect()`, `sendMessage()` | 各频道独立实现 | 深 |

#### 4. Information Hiding (信息隐藏)

**原则**: 封装设计决策，暴露简化接口。

**关键隐藏点**:
- **容器生命周期**: `GroupQueue` 隐藏了容器的启动、停止、重启逻辑
- **挂载配置**: `container-runner.ts` 隐藏了复杂的挂载规则计算
- **并发控制**: 队列系统内部处理全局并发限制，对外透明
- **重试逻辑**: 指数退避重试完全封装在队列内部

```typescript
// 使用者只需调用简单接口
queue.enqueueMessageCheck(chatJid);

// 内部隐藏了:
// - 并发限制检查
// - 重试计数管理
// - 退避时间计算
// - 进程注册与清理
```

#### 5. Pull Complexity Downward (向下拉复杂度)

**原则**: 不可避免复杂度应由模块内部承担，而非推给调用者。

**NanoClaw 实践**:

| 复杂度 | 处理方式 | 受益方 |
|--------|----------|--------|
| 容器超时管理 | `container-runner.ts` 内部处理 | 调用者无需关心 |
| 消息格式化 | `router.ts` 统一处理 XML 转义 | 频道实现简化 |
| 任务调度计算 | `task-scheduler.ts` 计算下次运行时间 | 任务创建者只需提供表达式 |
| 挂载安全校验 | `mount-security.ts` 集中验证 | 配置者只需提供路径 |

---

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         HOST PROCESS                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Channel A  │  │  Channel B  │  │      Task Scheduler     │  │
│  │ (WhatsApp)  │  │ (Telegram)  │  │                         │  │
│  └──────┬──────┘  └──────┬──────┘  └────────────┬────────────┘  │
│         │                │                      │               │
│         └────────────────┼──────────────────────┘               │
│                          │                                      │
│                          ▼                                      │
│              ┌─────────────────────┐                            │
│              │     Message Loop    │                            │
│              │   (src/index.ts)    │                            │
│              └──────────┬──────────┘                            │
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │     Group Queue     │                            │
│              │  (src/group-queue)  │                            │
│              │                     │                            │
│              │  ┌───────────────┐  │                            │
│              │  │   Group A     │  │                            │
│              │  │  (container)  │  │                            │
│              │  └───────────────┘  │                            │
│              │  ┌───────────────┐  │                            │
│              │  │   Group B     │  │                            │
│              │  │  (container)  │  │                            │
│              │  └───────────────┘  │                            │
│              └─────────────────────┘                            │
├─────────────────────────────────────────────────────────────────┤
│                         DATA LAYER                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │    SQLite DB    │  │   IPC Files     │  │   Group Files   │  │
│  │                 │  │                 │  │                 │  │
│  │  - messages     │  │  - input/       │  │  - CLAUDE.md    │  │
│  │  - groups       │  │  - output/      │  │  - skills/      │  │
│  │  - tasks        │  │  - tasks/       │  │  - .claude/     │  │
│  │  - sessions     │  │  - messages/    │  │                 │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 消息流分析

```
┌─────────┐    ┌──────────┐    ┌─────────────┐    ┌──────────────┐
│ Channel │───▶│ Store in │───▶│ Message Loop│───▶│ Group Queue  │
│         │    │ SQLite   │    │ Poll        │    │ (per group)  │
└─────────┘    └──────────┘    └─────────────┘    └──────┬───────┘
                                                         │
                              ┌──────────────────────────┘
                              │ spawn / pipe
                              ▼
                    ┌───────────────────┐
                    │     Container     │
                    │  (Claude Agent)   │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  Response Stream  │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  Channel Send     │
                    └───────────────────┘
```

### 容器隔离模型

```
┌─────────────────────────────────────────────────────────────┐
│                         HOST                                 │
│  groups/                                                    │
│  ├── group-a/          ┌───────────────────────────────┐    │
│  │   ├── CLAUDE.md     │ Container A                   │    │
│  │   ├── .claude/      │ ├─ 仅挂载 group-a/            │    │
│  │   └── ...           │ ├─ 独立 Claude session        │    │
│  │                     │ └─ 网络隔离                   │    │
│  ├── group-b/          └───────────────────────────────┘    │
│  │   ├── CLAUDE.md     ┌───────────────────────────────┐    │
│  │   ├── .claude/      │ Container B                   │    │
│  │   └── ...           │ ├─ 仅挂载 group-b/            │    │
│  │                     │ ├─ 独立 Claude session        │    │
│  └── group-c/          └───────────────────────────────┘    │
│      ├── CLAUDE.md    ┌───────────────────────────────┐     │
│      └── ...          │ Container C (main)            │     │
│                       │ ├─ 挂载所有 groups/           │     │
│                       │ ├─ 管理权限                   │     │
│                       │ └─ 任务调度可见性             │     │
│                       └───────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心模块分析

### 1. 编排器 (src/index.ts)

**职责**: 系统主入口，协调所有子系统

**核心状态**:
```typescript
let lastTimestamp = '';                    // 全局消息游标
let sessions: Record<string, string> = {}; // group folder -> session ID
let registeredGroups: Record<string, RegisteredGroup> = {};
let lastAgentTimestamp: Record<string, string> = {}; // per-group 游标
```

**关键设计决策**:
- **双游标机制**: `lastTimestamp` (全局已见) + `lastAgentTimestamp` (已处理)
- 崩溃恢复时通过对比两个游标识别未处理消息
- 错误时回滚 `lastAgentTimestamp` 实现重试，避免重复发送

**复杂度管理**:
- 消息循环与处理逻辑分离
- 通过 `GroupQueue` 委托并发管理

### 2. 频道注册表 (src/channels/registry.ts)

**设计**: 自注册工厂模式

```typescript
const factories = new Map<string, ChannelFactory>();

export function registerChannel(name: string, factory: ChannelFactory) {
  factories.set(name, factory);
}
```

**优势**:
- 零配置: 频道通过 barrel import 自注册
- 可选加载: 无凭证时工厂返回 null，自动跳过
- 扩展性: 新频道只需调用 `registerChannel()`

### 3. 容器运行器 (src/container-runner.ts)

**深度模块典范**: 复杂挂载逻辑隐藏在简单接口后

**挂载构建复杂度**:
```typescript
function buildVolumeMounts(group: RegisteredGroup): MountConfig[] {
  // 1. 基础挂载 (skill、ipc、logs)
  // 2. 主群组特殊挂载 (项目根只读、data 目录)
  // 3. 额外挂载 (group 配置、allowlist 校验)
  // 4. 安全阴影 (.env -> /dev/null)
  // 5. Claude 配置 (session、settings)
}
```

**流式输出处理**:
```typescript
// 解析容器输出的标记流
const output = await runContainerAgent(
  group,
  input,
  registerProcess,
  onOutput  // 流式回调
);

// 内部处理:
// - ---NANOCLAW_OUTPUT_START--- / END--- 标记
// - session ID 更新
// - 错误检测与超时处理
```

### 4. 群组队列 (src/group-queue.ts)

**核心职责**:
- 全局并发控制 (默认 5 容器)
- 每群组消息排序保证
- 指数退避重试
- IPC 文件系统通信

**状态机**:
```
         ┌─────────────┐
         │   PENDING   │
         └──────┬──────┘
                │ start
                ▼
         ┌─────────────┐
         │   RUNNING   │
         └──────┬──────┘
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
┌───────────────┐ ┌─────────────┐
│    SUCCESS    │ │    ERROR    │
└───────────────┘ └──────┬──────┘
                         │ retry < max
                         ▼
                  ┌─────────────┐
                  │   RETRYING  │
                  └─────────────┘
```

**IPC 设计**:
```
data/ipc/{groupFolder}/
├── input/     # 发送给容器的消息
├── output/    # 容器输出
├── tasks/     # 任务指令
└── messages/  # 发送给用户的回复
```

### 5. 数据库层 (src/db.ts)

**Schema 设计**:

| 表 | 用途 |
|----|------|
| `chats` | 群组/聊天元数据 |
| `messages` | 消息历史 |
| `scheduled_tasks` | 定时任务定义 |
| `task_run_logs` | 任务执行记录 |
| `router_state` | 路由器状态 KV |
| `sessions` | Claude session ID |
| `registered_groups` | 已注册群组 |

**迁移策略**:
- JSON 文件迁移到 SQLite
- Schema 版本管理
- 列添加的向后兼容

### 6. IPC 处理器 (src/ipc.ts)

**授权模型**:
```typescript
function checkAuth(groupFolder: string, requestingGroup: string): boolean {
  if (requestingGroup === MAIN_GROUP) return true;  // 主群组可访问所有
  return requestingGroup === groupFolder;           // 否则只能访问自己
}
```

**指令集**:
- `schedule_task`, `pause_task`, `resume_task`, `cancel_task`, `update_task`
- `refresh_groups`, `register_group`

### 7. 任务调度器 (src/task-scheduler.ts)

**调度类型**:
- `cron`: Cron 表达式
- `interval`: 固定间隔 (秒)
- `once`: 一次性任务

**漂移防止**:
```typescript
// 基于上次运行时间计算下次运行时间
const nextRun = lastRun
  ? new Date(lastRun.getTime() + intervalMs)
  : new Date(now.getTime() + intervalMs);
```

---

## 安全模型

### 多层防御

```
┌─────────────────────────────────────────┐
│  Layer 1: Mount Security                │
│  - 路径 allowlist 校验                  │
│  - 符号链接解析                         │
│  - 禁止访问敏感目录                     │
├─────────────────────────────────────────┤
│  Layer 2: Container Isolation           │
│  - Linux 容器隔离                       │
│  - 仅显式挂载可见                       │
│  - 无主机网络访问                       │
├─────────────────────────────────────────┤
│  Layer 3: Credential Proxy              │
│  - 容器不持有真实 API key               │
│  - 通过代理转发请求                     │
│  - 请求审计与限流                       │
├─────────────────────────────────────────┤
│  Layer 4: IPC Authorization             │
│  - 主群组隔离检查                       │
│  - 跨组操作限制                         │
├─────────────────────────────────────────┤
│  Layer 5: Sender Allowlist              │
│  - 消息发送者过滤                       │
│  - 触发词权限控制                       │
└─────────────────────────────────────────┘
```

### 挂载安全实现

```typescript
// src/mount-security.ts
export function validateMount(path: string, allowedPatterns: string[]): boolean {
  const resolved = resolveSymlinks(realpathSync(path));
  for (const pattern of allowedPatterns) {
    if (minimatch(resolved, pattern)) return true;
  }
  return false;
}
```

---

## 设计模式与最佳实践

### 1. 自注册模式 (Self-Registration)

**应用**: 频道系统

```typescript
// channels/whatsapp.ts
import { registerChannel } from './registry.js';

registerChannel('whatsapp', (opts) => {
  if (!hasCredentials()) return null;
  return new WhatsAppChannel(opts);
});
```

**优势**: 零配置、插件化、延迟加载

### 2. 工厂模式 (Factory Pattern)

**应用**: 频道创建

```typescript
type ChannelFactory = (opts: ChannelOpts) => Channel | null;
```

工厂返回 null 表示"未配置，跳过"，避免启动失败。

### 3. 队列模式 (Queue Pattern)

**应用**: `GroupQueue`

- 保证每群组消息顺序
- 全局并发限制
- 背压与重试

### 4. 流式处理 (Streaming)

**应用**: 容器输出

```typescript
for await (const line of readLines(proc.stdout)) {
  const parsed = parseOutput(line);
  await onOutput(parsed);  // 实时流式输出
}
```

### 5. 文件系统 IPC

**应用**: 容器与主机通信

优势:
- 无网络依赖
- 持久化（崩溃后可恢复）
- 简单调试（直接查看文件）

---

## 复杂度管理

### 避免的复杂度

| 反模式 | NanoClaw 做法 |
|--------|---------------|
| 配置蔓延 | 代码即配置 |
| 微服务复杂性 | 单一进程 |
| ORM 抽象泄漏 | 直接 SQL |
| 复杂权限系统 | 容器隔离 |
| 插件架构 | 技能系统 (git 分支) |

### 有意引入的复杂度

| 复杂度 | 位置 | 理由 |
|--------|------|------|
| 双游标消息追踪 | `index.ts` | 崩溃恢复可靠性 |
| 指数退避重试 | `group-queue.ts` | 系统韧性 |
| 复杂挂载规则 | `container-runner.ts` | 安全性 |
| Schema 迁移 | `db.ts` | 数据持久性 |

### 代码复杂度分布

```
src/index.ts          ████████████████████  ~650 行 (编排)
src/container-runner.ts ███████████████████  ~700 行 (容器)
src/db.ts             ███████████████████  ~700 行 (数据)
src/ipc.ts            ██████████████       ~450 行 (通信)
src/group-queue.ts    ████████████         ~370 行 (队列)
src/task-scheduler.ts █████████            ~280 行 (调度)
src/router.ts         ██                   ~50 行  (路由)
src/config.ts         ██                   ~70 行  (配置)
```

---

## 总结

### 设计亮点

1. **深模块设计**: 复杂实现隐藏在简洁接口后
2. **安全优先**: 多层防御，隔离优于权限
3. **可理解性**: 小而专注的代码库
4. **可维护性**: 代码即配置，无配置蔓延
5. **可扩展性**: 技能系统实现功能扩展

### 可改进点

1. **错误处理**: 部分错误处理可更统一
2. **测试覆盖**: 可增加单元测试覆盖
3. **文档**: 关键算法（如队列）可添加更多注释

### Ousterhout 原则契合度

| 原则 | 契合度 | 说明 |
|------|--------|------|
| Deep Modules | ★★★★★ | 核心设计哲学 |
| Information Hiding | ★★★★★ | 容器、队列均为典范 |
| Pull Complexity Down | ★★★★☆ | 大部分复杂度下沉 |
| Define Errors Out | ★★★☆☆ | 部分异常可进一步消除 |
| Design It Twice | ★★★★☆ | 架构设计清晰 |
| Comments | ★★★☆☆ | 可增加更多设计注释 |

---

*分析完成。详细 ADR 请参见 nanoclaw-adr.md，代码导航请参见 nanoclaw-code-index.md。*
