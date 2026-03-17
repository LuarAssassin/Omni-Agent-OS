# Qwen Code 项目文档集

> 全面深入的 Qwen Code 项目分析文档
> 生成时间: 2026-03-17

---

## 文档列表

| 文档 | 描述 | 大小 |
|------|------|------|
| [`qwen-code-analysis.md`](./qwen-code-analysis.md) | **主分析报告** - 项目全景分析 | ~25KB |
| [`qwen-code-code-index.md`](./qwen-code-code-index.md) | **代码索引** - 关键文件快速导航 | ~10KB |
| [`qwen-code-adr.md`](./qwen-code-adr.md) | **架构决策** - 关键 ADR | ~12KB |
| [`qwen-code-design-review.md`](./qwen-code-design-review.md) | **设计评审** - 基于《软件设计哲学》的深度分析 | ~15KB |

---

## 快速导航

### 如果你是...

**项目新手**
→ 先看 [`qwen-code-analysis.md`](./qwen-code-analysis.md) 的「项目概述」和「架构设计」章节

**需要定位代码**
→ 查阅 [`qwen-code-code-index.md`](./qwen-code-code-index.md) 按功能查找文件

**想了解设计决策**
→ 阅读 [`qwen-code-adr.md`](./qwen-code-adr.md)

**关注代码质量和设计**
→ 参考 [`qwen-code-design-review.md`](./qwen-code-design-review.md)

---

## 项目关键信息

```yaml
项目: Qwen Code
用途: AI 驱动的代码编辑器 CLI
语言: TypeScript 5.x + React 19.x
代码量: ~160,000+ 行 (TypeScript/TSX)
架构: Monorepo (npm workspaces)
包数量: 8 个核心包
支持模型: OpenAI, Anthropic Claude, Google Gemini, Qwen, Ollama
核心特性: 工具系统、MCP 协议、扩展系统、多 IDE 集成
```

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        展示层                                        │
│  CLI (Ink/React)  │  Web UI (React)  │  VS Code Extension           │
│  Zed Extension    │  SDK (TypeScript)│                              │
├─────────────────────────────────────────────────────────────────────┤
│                        命令层 (packages/cli/)                        │
│  gemini.tsx │ commands/ │ ui/ │ services/ │ config/ │ utils/       │
├─────────────────────────────────────────────────────────────────────┤
│                        核心层 (packages/core/)                       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  core/  │ │ tools/  │ │ services│ │extension│ │  mcp/   │       │
│  │(LLM客户端)│ │(工具系统)│ │(文件/Git)│ │(扩展系统) │ │(MCP协议) │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  ide/   │ │ models/ │ │ subagent│ │  hooks/ │ │telemetry│       │
│  │(IDE集成) │ │(模型配置)│ │(子代理)  │ │(钩子)   │ │(遥测)   │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
├─────────────────────────────────────────────────────────────────────┤
│                        数据层                                        │
│  JSON Config │ File System │ SQLite │ Memory                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码路径

| 功能 | 路径 |
|------|------|
| CLI 主入口 | `packages/cli/index.ts` |
| 应用启动 | `packages/cli/src/gemini.tsx` |
| 核心库导出 | `packages/core/src/index.ts` |
| 工具系统 | `packages/core/src/tools/` |
| MCP 实现 | `packages/core/src/mcp/` |
| 扩展系统 | `packages/core/src/extension/` |
| 配置管理 | `packages/core/src/config/` |
| LLM 客户端 | `packages/core/src/core/` |

---

## 构建命令

```bash
# 安装依赖
pnpm install

# 构建所有包
npm run build

# 构建 Rust CLI (native)
pnpm build:native

# 运行测试
pnpm test

# 代码格式化检查
pnpm format:check

# 类型检查
pnpm typecheck
```

---

## 配置文件位置

```
~/.qwen/
├── settings.json          # 用户全局配置
├── auth.json              # 认证信息
└── sessions/              # 会话存储

./.qwen/                   # 项目级配置
├── settings.json          # 项目配置
├── SOUL.md                # 系统提示词
└── skills/                # 技能文件
```

---

## 相关项目

根据 CLAUDE.md，同一仓库下的其他项目：

| 项目 | 技术栈 | 用途 |
|------|--------|------|
| `agent-browser/` | TypeScript + Rust | 浏览器自动化 CLI |
| `CoPaw/` | Python + TypeScript | 个人 AI 助手（多通道） |
| `edict/` | Python + React | 多智能体编排系统 |
| `picoclaw/` | Go | 超轻量级个人 AI 助手 |

---

*文档生成完成。如需补充特定模块的分析，请告知。*
