# Kimi CLI 项目文档

> 分析时间: 2026-03-17
> 项目路径: `/Users/luarassassin/reference-projects-me/kimi-cli`
> Python 版本: >=3.12

---

## 文档索引

| 文档 | 说明 |
|------|------|
| [kimi-cli-analysis.md](./kimi-cli-analysis.md) | 深度分析报告，包含架构设计、代码质量评估 |
| [kimi-cli-adr.md](./kimi-cli-adr.md) | 架构决策记录 (Architecture Decision Records) |
| [kimi-cli-code-index.md](./kimi-cli-code-index.md) | 代码文件索引与模块说明 |
| [kimi-cli-dependencies.md](./kimi-cli-dependencies.md) | 依赖分析与外部库说明 |

---

## 项目简介

**Kimi CLI** 是由 Moonshot AI 开发的 AI Agent 命令行工具，帮助开发者在终端中完成软件开发任务。它是 Kimi Code 的 CLI 版本，支持代码阅读编辑、Shell 命令执行、网页搜索和自主任务规划。

### 核心特性

| 特性 | 描述 |
|------|------|
| **多 UI 模式** | Shell 交互模式、Print 非交互模式、ACP Server 模式 |
| **MCP 支持** | 原生支持 Model Context Protocol，可扩展外部工具 |
| **技能系统** | 基于 Markdown 的层级化技能管理 |
| **Agent 系统** | 可配置的 Agent 角色与系统提示词 |
| **多 LLM 支持** | Kimi、OpenAI、Anthropic、Gemini 等多种提供商 |
| **后台任务** | 支持长时间运行的后台任务 |
| **Web 界面** | 提供 Web UI 作为备选交互方式 |

### 项目统计

| 指标 | 数值 |
|------|------|
| Python 源文件数 | ~167 个 |
| 代码总行数 | ~33,000+ 行 |
| 核心依赖数 | 39 个 |
| 测试文件数 | 200+ 个 |
| 文档文件 (KLIP) | 16 个 |

---

## 快速导航

- [架构分析](./kimi-cli-analysis.md#二架构设计)
- [设计哲学分析](./kimi-cli-analysis.md#四软件设计哲学分析)
- [核心模块](./kimi-cli-code-index.md)
- [依赖清单](./kimi-cli-dependencies.md)
