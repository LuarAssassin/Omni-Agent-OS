# Gemini CLI 架构文档

> Google 开源的 Gemini AI 命令行工具深度架构分析
> 基于代码版本: 0.35.0-nightly.20260313

## 文档导航

| 文档 | 内容 |
|------|------|
| [gemini-cli-analysis.md](./gemini-cli-analysis.md) | 全面架构分析（模块、交互、设计哲学） |
| [gemini-cli-code-index.md](./gemini-cli-code-index.md) | 代码导航索引（按包、模块组织） |
| [gemini-cli-dependencies.md](./gemini-cli-dependencies.md) | 依赖分析（外部包、内部模块关系） |
| [gemini-cli-adr.md](./gemini-cli-adr.md) | 架构决策记录（设计选择、权衡） |

## 快速概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        Gemini CLI                                │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │     CLI      │  │  A2A Server  │  │     SDK      │          │
│  │   @google/   │  │   @google/   │  │   @google/   │          │
│  │  gemini-cli  │  │gemini-cli-   │  │ gemini-cli-  │          │
│  │              │  │  a2a-server  │  │     sdk      │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         └─────────────────┴─────────────────┘                   │
│                           │                                     │
│         ┌─────────────────▼─────────────────┐                   │
│         │         CORE Package              │                   │
│         │      @google/gemini-cli-core      │                   │
│         └───────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

## 核心架构特点

1. **Monorepo 结构**: npm workspaces 管理 6 个包
2. **分层设计**: CLI → Core → SDK 清晰分层
3. **事件驱动**: MessageBus + EventEmitter 架构
4. **A2A 协议**: 支持 Agent-to-Agent 通信
5. **MCP 集成**: Model Context Protocol 工具扩展
6. **策略引擎**: PolicyEngine 控制工具执行权限

## 技术栈

- **Runtime**: Node.js >= 20
- **Language**: TypeScript (ES Modules)
- **AI SDK**: @google/genai
- **UI**: Ink (React for Terminal)
- **Testing**: Vitest

## 关键目录

```
gemini-cli/
├── packages/
│   ├── cli/              # CLI 入口与 TUI
│   ├── core/             # 核心逻辑（工具、调度、策略）
│   ├── sdk/              # 程序化 API
│   ├── a2a-server/       # A2A 协议服务器
│   ├── devtools/         # 开发工具
│   └── test-utils/       # 测试工具
├── docs/                 # 官方文档
└── scripts/              # 构建脚本
```

## 核心概念

| 概念 | 说明 |
|------|------|
| **Scheduler** | 工具调用编排器，管理工具生命周期 |
| **MessageBus** | 事件总线，连接调度器与 UI |
| **PolicyEngine** | 策略引擎，控制工具执行权限 |
| **HookSystem** | 钩子系统，支持扩展 |
| **ToolRegistry** | 工具注册表，管理内置/MCP/发现工具 |
| **GeminiClient** | AI 客户端，处理 LLM 交互 |

---

*文档生成时间: 2026-03-17*
*分析范围: packages/cli, core, sdk, a2a-server*
