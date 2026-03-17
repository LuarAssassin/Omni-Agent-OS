# ZeroClaw 项目文档中心

> ZeroClaw — Rust 优先的高性能 AI Agent 运行时

---

## 文档索引

| 文档 | 描述 |
|------|------|
| [zeroclaw-analysis.md](./zeroclaw-analysis.md) | 项目深度分析报告（架构、代码、设计哲学） |
| [zeroclaw-code-index.md](./zeroclaw-code-index.md) | 代码文件导航与功能索引 |
| [zeroclaw-dependencies.md](./zeroclaw-dependencies.md) | 依赖分析与特性标志 |
| [zeroclaw-adr.md](./zeroclaw-adr.md) | 架构决策记录 (ADRs) |

---

## 项目速览

**ZeroClaw** 是一个 Rust 优先的自主 Agent 运行时，针对性能、效率、稳定性、可扩展性、可持续性和安全性进行了优化。

### 核心特性

| 特性 | 描述 |
|------|------|
| **极致性能** | Rust 原生实现，~8.8MB 二进制文件 |
| **Trait 驱动架构** | 通过 Trait 实现实现核心可扩展性 |
| **模块化设计** | 可替换的 Provider、Channel、Tool、Memory 后端 |
| **多记忆后端** | SQLite (默认)、Postgres、Qdrant、Markdown、Lucid |
| **沙盒后端** | Docker、Firejail、Bubblewrap、Landlock (特性门控) |
| **20+ 通道** | Telegram、Discord、Slack、WhatsApp、Matrix、Nostr 等 |
| **MCP 支持** | Model Context Protocol 集成 |
| **多 Agent 支持** | Delegate Agent 和 Swarm 编排 |
| **硬件集成** | STM32 Nucleo、Raspberry Pi GPIO、ESP32、Arduino |

### 技术栈

- **语言**: Rust (Edition 2021)
- **运行时**: Tokio (异步)
- **Web 框架**: Axum
- **CLI**: clap
- **序列化**: serde
- **数据库**: rusqlite, sqlx
- **HTTP 客户端**: reqwest

---

## 快速导航

### 按模块浏览

| 模块 | 路径 | 核心 Trait |
|------|------|-----------|
| Agent | `src/agent/` | `Agent` |
| Channel | `src/channels/` | `Channel` |
| Provider | `src/providers/` | `Provider` |
| Tool | `src/tools/` | `Tool` |
| Memory | `src/memory/` | `Memory` |
| Security | `src/security/` | `Sandbox`, `SecurityPolicy` |
| Runtime | `src/runtime/` | `RuntimeAdapter` |
| Peripheral | `src/peripherals/` | `Peripheral` |
| Observability | `src/observability/` | `Observer` |

### 关键工厂函数

| 工厂 | 位置 | 用途 |
|------|------|------|
| `create_memory` | `src/memory/mod.rs:72` | 创建记忆后端 |
| `create_sandbox` | `src/security/detect.rs` | 创建沙盒后端 |
| `create_runtime` | `src/runtime/mod.rs:12` | 创建运行时适配器 |
| `create_observer` | `src/observability/mod.rs:28` | 创建可观察性后端 |
| `all_tools` | `src/tools/mod.rs:150` | 创建工具集 |

---

## 软件设计哲学应用

本文档集结合《A Philosophy of Software Design》(John Ousterhout) 的核心理念：

1. **深度模块 (Deep Modules)** - ZeroClaw 的 Trait 抽象提供了强大的功能，但接口简洁
2. **信息隐藏 (Information Hiding)** - 每个模块通过特性标志和工厂模式隐藏实现细节
3. **通用性设计 (General-Purpose Design)** - Trait 系统允许任意组合和扩展
4. **错误处理 (Error Handling)** - 使用 `anyhow` 进行上下文传播，明确定义错误边界
5. **策略与机制分离** - 策略 (SecurityPolicy) 与机制 (Sandbox 实现) 清晰分离

---

## 项目统计

| 指标 | 数值 |
|------|------|
| Rust 源文件 | ~150+ 个 |
| 代码总行数 | ~50,000+ 行 |
| 直接依赖 | 80+ 个 |
| 特性标志 | 20+ 个 |
| 核心 Trait | 8 个 |
| 工具数量 | 40+ 个 |

---

*分析时间: 2026-03-17*
*项目路径: `/Users/luarassassin/reference-projects-me/zeroclaw`*
