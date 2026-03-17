# OpenCode 项目文档集

> 全面深入的 OpenCode 项目分析文档
> 生成时间: 2026-03-17
> 分析依据: 软件设计哲学 (A Philosophy of Software Design)

---

## 文档列表

| 文档 | 描述 | 大小 |
|------|------|------|
| [`opencode-analysis.md`](./opencode-analysis.md) | **主分析报告** - 项目全景分析、架构设计、代码质量评估 | ~25KB |
| [`opencode-dependencies.md`](./opencode-dependencies.md) | **依赖清单** - 核心依赖详解与设计分析 | ~8KB |
| [`opencode-code-index.md`](./opencode-code-index.md) | **代码索引** - 关键文件快速导航 | ~10KB |
| [`opencode-adr.md`](./opencode-adr.md) | **架构决策** - 关键设计决策与软件设计哲学评估 | ~12KB |

---

## 快速导航

### 如果你是...

**项目新手**
→ 先看 [`opencode-analysis.md`](./opencode-analysis.md) 的「项目概述」和「架构设计」章节

**需要定位代码**
→ 查阅 [`opencode-code-index.md`](./opencode-code-index.md) 按功能查找文件

**需要了解依赖**
→ 查看 [`opencode-dependencies.md`](./opencode-dependencies.md)

**想了解设计决策**
→ 阅读 [`opencode-adr.md`](./opencode-adr.md)

---

## 项目关键信息

```yaml
项目: OpenCode
定位: 开源 AI 编程助手（Claude Code 替代品）
语言: TypeScript
运行时: Bun 1.3.10+
代码量: ~176,000 行 (940+ TypeScript 文件)
架构: Monorepo (Turborepo)
核心框架:
  - CLI: yargs
  - TUI: SolidJS + @opentui
  - DB: SQLite + Drizzle ORM
  - 函数式: Effect
核心特性:
  - 100% 开源
  - 多提供商支持 (Claude/OpenAI/Google/本地模型)
  - 内置 LSP 支持
  - MCP (Model Context Protocol) 原生支持
  - 客户端/服务器架构
  - TUI/Web/Desktop 多界面
```

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              展示层                                          │
│  ┌──────────────┐  ┌─────────────────┐  ┌─────────────────────────────────┐ │
│  │ CLI (yargs)  │  │ TUI (SolidJS)   │  │ Web (SolidJS + Vite)            │ │
│  │  Terminal    │  │ @opentui/core   │  │ packages/app                    │ │
│  └──────────────┘  └─────────────────┘  └─────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ Desktop (Tauri) - packages/desktop                                      ││
│  └─────────────────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────────────────┤
│                              命令层 (cli/cmd/)                               │
│  agent │ mcp │ tui │ run │ serve │ web │ pr │ session │ db │ debug │ stats  │
├─────────────────────────────────────────────────────────────────────────────┤
│                              核心层 (src/)                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │  agent  │ │ session │ │provider │ │  tool   │ │   mcp   │ │  skill  │   │
│  │ (代理)  │ │(会话)   │ │(模型)   │ │ (工具)  │ │(MCP)    │ │(技能)   │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │   bus   │ │ config  │ │ project │ │  file   │ │  lsp    │ │  pty    │   │
│  │(事件总线)│ │(配置)   │ │(项目)   │ │(文件)   │ │(LSP)    │ │(终端)   │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
├─────────────────────────────────────────────────────────────────────────────┤
│                              数据层                                          │
│  SQLite │ Drizzle ORM │ JSON │ FileSystem                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码路径

| 功能 | 路径 |
|------|------|
| 主入口 | `packages/opencode/src/index.ts` |
| CLI 命令 | `packages/opencode/src/cli/cmd/` |
| Agent 核心 | `packages/opencode/src/agent/agent.ts` |
| 配置系统 | `packages/opencode/src/config/tui.ts` |
| 数据库 | `packages/opencode/src/storage/db.ts` |
| MCP 管理 | `packages/opencode/src/mcp/index.ts` |
| 会话管理 | `packages/opencode/src/session/` |
| 工具集 | `packages/opencode/src/tool/` |
| 权限系统 | `packages/opencode/src/permission/` |

---

## 构建命令

```bash
# 安装依赖
bun install

# 开发模式
bun dev              # 运行 TUI
bun dev:desktop      # 运行桌面应用
bun dev:web          # 运行 Web 应用

# 类型检查
bun typecheck

# 测试
bun test

# 构建
./packages/opencode/script/build.ts --single
```

---

## 配置文件位置

```
~/.config/opencode/
├── config.json          # 主配置
├── tui.json            # TUI 配置
└── ...

.opencode/               # 项目级配置
├── agent/              # 代理定义
├── skill/              # 技能文件
└── ...
```

---

## 软件设计哲学评分

基于 John Ousterhout 的《A Philosophy of Software Design》:

| 维度 | 评分 | 说明 |
|------|------|------|
| 模块深度 | ★★★★☆ | 大多数模块接口简洁，但存在部分浅模块 |
| 信息隐藏 | ★★★★☆ | 配置系统、数据库访问控制良好 |
| 错误处理 | ★★★☆☆ | 使用 Effect 但部分地方仍显复杂 |
| 命名质量 | ★★★★☆ | 总体清晰，但部分命名略显抽象 |
| 注释质量 | ★★★☆☆ | 核心逻辑注释较少 |
| 一致性 | ★★★★★ | 代码风格高度一致 |

---

*文档生成完成。如需深入分析特定模块，请告知。*
