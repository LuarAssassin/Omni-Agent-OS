# IronClaw 项目文档集

> 全面深入的 IronClaw 项目分析文档
> 生成时间: 2026-03-17

---

## 文档列表

| 文档 | 描述 | 大小 |
|------|------|------|
| [`ironclaw-analysis.md`](./ironclaw-analysis.md) | **主分析报告** - 项目全景分析，软件设计哲学深度解读 | ~30KB |
| [`ironclaw-dependencies.md`](./ironclaw-dependencies.md) | **依赖清单** - 60+ 直接依赖详解 | ~10KB |
| [`ironclaw-code-index.md`](./ironclaw-code-index.md) | **代码索引** - 关键文件快速导航 | ~12KB |
| [`ironclaw-adr.md`](./ironclaw-adr.md) | **架构决策** - 20 个关键 ADR | ~15KB |

---

## 快速导航

### 如果你是...

**项目新手**
→ 先看 [`ironclaw-analysis.md`](./ironclaw-analysis.md) 的「项目概述」和「架构设计」章节

**需要定位代码**
→ 查阅 [`ironclaw-code-index.md`](./ironclaw-code-index.md) 按功能查找文件

**需要了解依赖**
→ 查看 [`ironclaw-dependencies.md`](./ironclaw-dependencies.md)

**想了解设计决策**
→ 阅读 [`ironclaw-adr.md`](./ironclaw-adr.md)

---

## 项目关键信息

```yaml
项目: IronClaw
用途: 安全个人 AI 助手
作者: NEAR AI <support@near.ai>
语言: Rust 2024 Edition (要求 Rust 1.92+)
版本: 0.17.0
代码量: ~177,000 行 (Rust)
架构: 多通道输入 + Agentic 循环 + WASM/Docker 双沙箱
安全模型: Defense in Depth (WASM + Docker + Safety Layer)
数据库: 双后端 (PostgreSQL + libSQL/Turso)
LLM 支持: NEAR AI, OpenAI, Anthropic, Ollama, Bedrock 等
消息通道: REPL, HTTP, Web Gateway, WASM Channels, Signal
核心特性: MCP 协议、动态工具构建、混合搜索 (FTS + Vector)、技能系统
```

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        交互层 (Channels)                         │
│  ┌────────┐ ┌────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │  REPL  │ │ HTTP   │ │ WASM Channels│ │ Web Gateway  │        │
│  │ (TUI)  │ │Webhook │ │(Telegram等)  │ │(React+SSE)   │        │
│  └───┬────┘ └───┬────┘ └──────┬───────┘ └──────┬───────┘        │
│      └─────────┴─────────────┬┴────────────────┘                │
└──────────────────────────────┼──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                        Agent 核心层                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │
│  │ Message Router │──│  LLM Reasoning │──│ Action Executor│     │
│  └────────────────┘  └───────┬────────┘  └───────┬────────┘     │
│         ▲                    │                   │              │
│         │         ┌──────────┴───────────────────┴──────────┐   │
│         │         ▼                                         ▼   │
│  ┌──────┴─────────────┐                         ┌───────────────┴───┐
│  │   Safety Layer     │                         │    Self-Repair    │
│  │ - Input Sanitizer  │                         │ - Stuck Detection │
│  │ - Leak Detector    │                         │ - Tool Recovery   │
│  │ - Policy Engine    │                         └───────────────────┘
│  └────────────────────┘                                         │
└─────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                        执行层                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Built-in     │  │ MCP Tools    │  │ WASM Sandbox │          │
│  │ Tools        │  │              │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Docker Sandbox (Orchestrator)               │   │
│  │     ┌───────────────┐  ┌───────────────┐                │   │
│  │     │ Worker / CC   │  │ Job Containers│                │   │
│  │     └───────────────┘  └───────────────┘                │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键代码路径

| 功能 | 路径 |
|------|------|
| 主入口 | `src/main.rs` |
| Agent 核心循环 | `src/agent/agent_loop.rs`, `src/agent/agentic_loop.rs` |
| 配置管理 | `src/config/mod.rs` |
| 工具注册表 | `src/tools/registry.rs`, `src/tools/tool.rs` |
| MCP 客户端 | `src/tools/mcp/` |
| WASM 运行时 | `src/tools/wasm/` |
| 数据库抽象 | `src/db/mod.rs` |
| LLM 提供商 | `src/llm/mod.rs` |
| 安全层 | `src/safety/mod.rs` |
| Web 网关 | `src/channels/web/` |

---

## 构建命令

```bash
# 格式化代码
cargo fmt

# 运行 lint（零警告策略）
cargo clippy --all --benches --tests --examples --all-features

# 运行单元测试
cargo test

# 运行集成测试（需要 PostgreSQL）
cargo test --features integration

# 带日志运行
RUST_LOG=ironclaw=debug cargo run

# 构建发布版本
cargo build --release
```

---

## 配置文件位置

```
~/.ironclaw/
├── .env                 # 启动环境变量
├── settings.json        # 用户设置
├── workspace/           # 默认工作区
│   ├── sessions/        # 会话存储
│   ├── skills/          # 技能文件 (SKILL.md)
│   ├── AGENTS.md        # 代理定义
│   ├── MEMORY.md        # 持久记忆
│   └── HEARTBEAT.md     # 心跳配置
├── agents/              # 代理定义目录
└── extensions/          # WASM 扩展
```

---

## 安全模型

IronClaw 采用**纵深防御 (Defense in Depth)** 策略：

```
WASM ──► Allowlist ──► Leak Scan ──► Credential ──► Execute ──► Leak Scan ──► WASM
         Validator     (request)     Injector       Request     (response)
```

1. **WASM 沙箱** - 不可信代码在隔离环境中运行
2. **能力权限** - 显式授权 HTTP、密钥、工具调用权限
3. **密钥保护** - AES-256-GCM 加密，主机边界注入
4. **泄露检测** - 扫描请求/响应中的密钥外泄
5. **提示注入防御** - 多层检测与净化

---

## 相关项目

| 项目 | 技术栈 | 用途 |
|------|--------|------|
| `agent-browser/` | TypeScript + Rust | 浏览器自动化 CLI |
| `CoPaw/` | Python + TypeScript | 个人 AI 助手（多通道） |
| `edict/` | Python + React | 多智能体编排系统 |
| `pinchtab/` | Go + React | HTTP 浏览器控制服务器 |
| `picoclaw/` | Go | 超轻量级 AI 助手 |

---

*文档生成完成。如需补充特定模块的分析，请告知。*
