# PicoClaw 项目文档集

> 全面深入的 PicoClaw 项目分析文档
> 生成时间: 2026-03-17

---

## 文档列表

| 文档 | 描述 | 大小 |
|------|------|------|
| [`picoclaw-analysis.md`](./picoclaw-analysis.md) | **主分析报告** - 项目全景分析 | ~15KB |
| [`picoclaw-dependencies.md`](./picoclaw-dependencies.md) | **依赖清单** - 35+ 直接依赖详解 | ~6KB |
| [`picoclaw-code-index.md`](./picoclaw-code-index.md) | **代码索引** - 关键文件快速导航 | ~8KB |
| [`picoclaw-adr.md`](./picoclaw-adr.md) | **架构决策** - 14 个关键 ADR | ~10KB |

---

## 快速导航

### 如果你是...

**项目新手**
→ 先看 [`picoclaw-analysis.md`](./picoclaw-analysis.md) 的「项目概述」和「架构设计」章节

**需要定位代码**
→ 查阅 [`picoclaw-code-index.md`](./picoclaw-code-index.md) 按功能查找文件

**需要了解依赖**
→ 查看 [`picoclaw-dependencies.md`](./picoclaw-dependencies.md)

**想了解设计决策**
→ 阅读 [`picoclaw-adr.md`](./picoclaw-adr.md)

---

## 项目关键信息

```yaml
项目: PicoClaw
用途: 超轻量级个人 AI 助手
语言: Go 1.25.7
代码量: ~72,000 行 (355 个 Go 文件)
内存占用: <10MB
支持平台: Linux(x86_64/ARM64/ARMv7/MIPS/RISC-V/LoongArch), macOS(ARM64), Windows
消息通道: 15+ (Telegram/Discord/QQ/飞书/钉钉/Slack/Matrix/WhatsApp/LINE/iMessage/IRC/...)
核心特性: MCP 协议、BM25 工具发现、智能模型路由、技能系统
```

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        展示层                                 │
│  CLI (Cobra)  │  TUI (tview)  │  Web Console (React)        │
├─────────────────────────────────────────────────────────────┤
│                        命令层 (cmd/)                          │
│  agent │ auth │ gateway │ cron │ skills │ migrate │ status  │
├─────────────────────────────────────────────────────────────┤
│                        核心包 (pkg/)                          │
│  agent │ channels │ providers │ tools │ skills │ bus │ mcp   │
├─────────────────────────────────────────────────────────────┤
│                        数据层                                 │
│  JSONL │ SQLite │ FileSystem                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 关键代码路径

| 功能 | 路径 |
|------|------|
| 主入口 | `cmd/picoclaw/main.go` |
| Agent 核心 | `pkg/agent/instance.go`, `pkg/agent/loop.go` |
| 配置管理 | `pkg/config/config.go` |
| 工具注册表 | `pkg/tools/registry.go` |
| MCP 管理器 | `pkg/mcp/manager.go` |
| 消息总线 | `pkg/bus/bus.go` |
| 通道管理 | `pkg/channels/manager.go` |
| 模型路由 | `pkg/routing/router.go` |

---

## 构建命令

```bash
# 基础构建
make build

# 跨平台构建
make build-linux-arm      # 树莓派 Zero
make build-linux-arm64    # ARM64 服务器
make build-linux-mipsle   # MIPS 设备

# Docker 部署
make docker-build
make docker-run

# 测试
make test
make check
```

---

## 配置文件位置

```
~/.picoclaw/
├── config.json          # 主配置
├── workspace/           # 默认工作区
│   ├── sessions/        # 会话存储 (JSONL)
│   └── skills/          # 技能文件 (Markdown)
└── agents/              # 代理定义
```

---

## 相关项目

根据 CLAUDE.md，同一仓库下的其他项目：

| 项目 | 技术栈 | 用途 |
|------|--------|------|
| `agent-browser/` | TypeScript + Rust | 浏览器自动化 CLI |
| `CoPaw/` | Python + TypeScript | 个人 AI 助手（多通道） |
| `edict/` | Python + React | 多智能体编排系统 |
| `pinchtab/` | Go + React | HTTP 浏览器控制服务器 |

---

*文档生成完成。如需补充特定模块的分析，请告知。*
