# Codex 项目文档集

> 全面深入的 OpenAI Codex CLI 项目分析文档
> 生成时间: 2026-03-17

---

## 文档列表

| 文档 | 描述 | 大小 |
|------|------|------|
| [`codex-analysis.md`](./codex-analysis.md) | **主分析报告** - 项目全景、架构、目录结构 | ~20KB |
| [`codex-dependencies.md`](./codex-dependencies.md) | **依赖清单** - Rust/TypeScript/Python 依赖详解 | ~8KB |
| [`codex-code-index.md`](./codex-code-index.md) | **代码索引** - 关键文件快速导航 | ~12KB |
| [`codex-adr.md`](./codex-adr.md) | **架构决策** - 关键 ADR 记录 | ~10KB |
| [`codex-design-philosophy.md`](./codex-design-philosophy.md) | **设计哲学** - Ousterhout 视角的架构评估 | ~8KB |

---

## 快速导航

### 如果你是...

**项目新手**
→ 先看 [`codex-analysis.md`](./codex-analysis.md) 的「项目概述」和「架构设计」章节

**需要定位代码**
→ 查阅 [`codex-code-index.md`](./codex-code-index.md) 按功能查找文件

**需要了解依赖**
→ 查看 [`codex-dependencies.md`](./codex-dependencies.md)

**想了解设计决策**
→ 阅读 [`codex-adr.md`](./codex-adr.md)

**关注设计质量与复杂度**
→ 阅读 [`codex-design-philosophy.md`](./codex-design-philosophy.md)

---

## 项目关键信息

```yaml
项目: Codex CLI
来源: OpenAI (github.com/openai/codex)
用途: 本地运行的 AI 编码代理，支持 TUI/IDE/桌面应用
主语言: Rust (codex-rs)
代码量: ~638,000 行 Rust (1,359 个 .rs 文件)
Workspace: 70+ Cargo crates
支持平台: macOS 12+, Ubuntu 20.04+/Debian 10+, Windows 11 (WSL2)
核心特性: SQ/EQ 协议、沙箱执行、MCP 集成、技能系统、多模态
```

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         UI 层                                        │
│  TUI (ratatui)  │  App Server (JSON-RPC)  │  IDE 插件  │  桌面 App   │
├─────────────────────────────────────────────────────────────────────┤
│                         CLI 入口 (cli/)                               │
│  codex [prompt] │ exec │ login │ mcp │ apply │ resume │ cloud │ ...  │
├─────────────────────────────────────────────────────────────────────┤
│                         Core 引擎 (core/)                             │
│  CodexThread │ Agent │ Protocol │ Config │ MCP │ Exec │ Sandbox     │
├─────────────────────────────────────────────────────────────────────┤
│                        协议层 (protocol/)                             │
│  Op (SQ) │ EventMsg (EQ) │ Session │ Task │ Turn                     │
├─────────────────────────────────────────────────────────────────────┤
│                        外部集成                                       │
│  Responses API │ MCP Servers │ ChatGPT Connectors │ Ollama/LMStudio  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码路径

| 功能 | 路径 |
|------|------|
| 主入口 | `codex-rs/cli/src/main.rs` |
| Core 库 | `codex-rs/core/src/lib.rs` |
| 协议定义 | `codex-rs/protocol/src/protocol.rs` |
| TUI | `codex-rs/tui/` |
| 执行引擎 | `codex-rs/exec/` |
| 沙箱 | `codex-rs/linux-sandbox/`, `codex-rs/windows-sandbox-rs/` |
| MCP 服务器 | `codex-rs/mcp-server/` |
| 技能系统 | `codex-rs/skills/` |
| TypeScript SDK | `sdk/typescript/` |
| Python SDK | `sdk/python/` |

---

## 构建命令

```bash
# 克隆并进入工作区
git clone https://github.com/openai/codex.git
cd codex/codex-rs

# 构建
cargo build

# 运行 TUI
cargo run --bin codex -- "explain this codebase to me"

# 非交互模式
cargo run --bin codex -- exec "fix the bug in src/main.rs"

# 使用 justfile
just codex "your prompt"
just test
just fmt
```

---

## 配置文件位置

```
~/.codex/
├── config.toml          # 主配置
├── log/                 # 日志目录
│   └── codex-tui.log
└── (sqlite_home)        # 状态数据库 (CODEX_SQLITE_HOME)
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
