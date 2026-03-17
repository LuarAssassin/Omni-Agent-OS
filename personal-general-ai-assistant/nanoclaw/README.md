# NanoClaw 项目文档集

> 全面深入的 NanoClaw 项目分析文档
> 生成时间: 2026-03-17

---

## 文档列表

| 文档 | 描述 | 大小 |
|------|------|------|
| [`nanoclaw-analysis.md`](./nanoclaw-analysis.md) | **主分析报告** - 项目全景分析 | ~20KB |
| [`nanoclaw-dependencies.md`](./nanoclaw-dependencies.md) | **依赖清单** - 核心依赖详解 | ~5KB |
| [`nanoclaw-code-index.md`](./nanoclaw-code-index.md) | **代码索引** - 关键文件快速导航 | ~8KB |
| [`nanoclaw-adr.md`](./nanoclaw-adr.md) | **架构决策** - 关键 ADR | ~10KB |

---

## 快速导航

### 如果你是...

**项目新手**
→ 先看 [`nanoclaw-analysis.md`](./nanoclaw-analysis.md) 的「项目概述」和「架构设计」章节

**需要定位代码**
→ 查阅 [`nanoclaw-code-index.md`](./nanoclaw-code-index.md) 按功能查找文件

**想了解设计决策**
→ 阅读 [`nanoclaw-adr.md`](./nanoclaw-adr.md)

**需要了解依赖**
→ 查看 [`nanoclaw-dependencies.md`](./nanoclaw-dependencies.md)

---

## 项目关键信息

```yaml
项目: NanoClaw
用途: 个人 Claude 助手，支持多通道消息和定时任务
语言: TypeScript (Node.js 20+)
代码量: ~8,000 行 (30+ 个 TypeScript 文件)
架构: 单一 Node.js 进程 + 容器化 Agent 执行
支持平台: macOS (Apple Silicon), Linux, Windows (WSL)
消息通道: WhatsApp, Telegram, Discord, Slack, Gmail
核心特性: 容器隔离、技能系统、定时任务、MCP 支持
```

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        HOST (Node.js)                        │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Channels   │  │ SQLite (DB)  │  │   Scheduler  │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                 │               │
│         └─────────────────┼─────────────────┘               │
│                           │                                 │
│         ┌─────────────────┘                                 │
│         ▼                                                   │
│  ┌──────────────────┐      ┌──────────────────┐            │
│  │  Message Loop    │─────▶│   Group Queue    │            │
│  └──────────────────┘      └────────┬─────────┘            │
│                                     │                       │
├─────────────────────────────────────┼───────────────────────┤
│                                     │  CONTAINER (Linux VM) │
│                                     ▼                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  Agent Runner                        │  │
│  │  - Claude Agent SDK                                  │  │
│  │  - File operations (Read/Write/Edit/Glob/Grep)      │  │
│  │  - Web tools (WebSearch/WebFetch)                   │  │
│  │  - Bash (safe - sandboxed!)                         │  │
│  │  - MCP servers (nanoclaw scheduler)                 │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 关键代码路径

| 功能 | 路径 |
|------|------|
| 主入口 | `src/index.ts` |
| 配置常量 | `src/config.ts` |
| 数据库操作 | `src/db.ts` |
| 容器运行器 | `src/container-runner.ts` |
| 容器运行时 | `src/container-runtime.ts` |
| 消息队列 | `src/group-queue.ts` |
| IPC 通信 | `src/ipc.ts` |
| 消息路由 | `src/router.ts` |
| 任务调度器 | `src/task-scheduler.ts` |
| 频道注册表 | `src/channels/registry.ts` |
| 挂载安全 | `src/mount-security.ts` |

---

## 构建命令

```bash
# 安装依赖
npm install

# 开发模式 (热重载)
npm run dev

# 编译 TypeScript
npm run build

# 运行
npm start

# 测试
npm test

# 格式化
npm run format

# 类型检查
npm run typecheck

# 构建容器
./container/build.sh
```

---

## 目录结构

```
nanoclaw/
├── src/                    # 主程序源码
│   ├── index.ts           # 主入口 (orchestrator)
│   ├── channels/          # 频道系统
│   ├── *.ts               # 核心模块
├── container/             # 容器相关
│   ├── Dockerfile         # Agent 容器镜像
│   ├── agent-runner/      # 容器内运行的代码
│   └── skills/            # 容器内可用技能
├── setup/                 # 安装脚本 (TypeScript)
├── docs/                  # 文档
├── groups/                # 群组文件夹 (gitignored)
├── store/                 # 数据存储 (gitignored)
├── data/                  # 运行时数据 (gitignored)
└── .claude/skills/        # Claude Code 技能
```

---

## 设计理念 (软件设计哲学)

本项目体现了 John Ousterhout 《A Philosophy of Software Design》中的核心原则：

| 原则 | 体现 |
|------|------|
**Small Enough to Understand** | 单一进程，~8000行代码，无微服务
**Security Through Isolation** | 容器隔离而非应用层权限检查
**Deep Modules** | 简洁接口 (Channel, DB)，复杂实现隐藏
**Information Hiding** | 容器封装 Agent 执行细节
**Pull Complexity Down** | 队列系统内部处理并发复杂度

---

*文档生成完成。如需补充特定模块的分析，请告知。*
