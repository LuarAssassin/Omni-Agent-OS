# Qwen Code 关键代码索引

> 快速导航到项目中的重要代码文件

---

## 入口与启动

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| CLI 主入口 | `packages/cli/index.ts` | ~50 |
| 应用启动器 | `packages/cli/src/gemini.tsx` | ~500 |
| 核心库导出 | `packages/core/src/index.ts` | ~250+ 导出 |
| SDK 入口 | `packages/sdk-typescript/src/index.ts` | ~100 |

---

## CLI 包核心 (`packages/cli/src/`)

### 命令系统

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 命令总线 | `commands/index.ts` | ~200 |
| Help 命令 | `commands/help.ts` | ~150 |
| Model 命令 | `commands/model.ts` | ~200 |
| Clear 命令 | `commands/clear.ts` | ~100 |
| Settings 命令 | `commands/settings.ts` | ~300 |
| MCP 命令 | `commands/mcp.ts` | ~250 |
| Extensions 命令 | `commands/extensions/` | ~400 |

### UI 组件

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 主应用 | `ui/App.tsx` | ~300 |
| 聊天面板 | `ui/ChatPanel.tsx` | ~400 |
| 输入组件 | `ui/Input.tsx` | ~250 |
| 消息渲染 | `ui/Message.tsx` | ~300 |
| 侧边栏 | `ui/Sidebar.tsx` | ~200 |
| 工具调用显示 | `ui/ToolCall.tsx` | ~250 |
| 主题系统 | `ui/ThemeProvider.tsx` | ~150 |

### 配置与服务

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 配置加载 | `config/load.ts` | ~300 |
| 配置验证 | `config/validation.ts` | ~200 |
| 配置迁移 | `config/migration.ts` | ~250 |
| 认证服务 | `services/auth.ts` | ~300 |
| 会话服务 | `services/session.ts` | ~200 |

### 非交互模式

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 非交互主逻辑 | `nonInteractive/index.ts` | ~300 |
| 命令执行 | `nonInteractive/run.ts` | ~200 |

---

## Core 包核心 (`packages/core/src/`)

### LLM 客户端 (`core/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 客户端接口 | `core/llm-client.ts` | ~100 |
| OpenAI 客户端 | `core/openai.ts` | ~400 |
| Anthropic 客户端 | `core/anthropic.ts` | ~400 |
| Gemini 客户端 | `core/gemini.ts` | ~350 |
| Qwen 客户端 | `core/qwen.ts` | ~300 |
| Ollama 客户端 | `core/ollama.ts` | ~250 |
| 流式处理 | `core/streaming.ts` | ~200 |

### 工具系统 (`tools/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 工具接口 | `tools/types.ts` | ~100 |
| 工具注册表 | `tools/registry.ts` | ~400 |
| 基础工具类 | `tools/base-tool.ts` | ~200 |
| 读取文件 | `tools/ReadFile.ts` | ~250 |
| 写入文件 | `tools/WriteFile.ts` | ~250 |
| 编辑文件 | `tools/EditFile.ts` | ~400 |
| 列出目录 | `tools/ListDir.ts` | ~200 |
| 文件搜索 | `tools/Glob.ts` | ~150 |
| 代码搜索 | `tools/Grep.ts` | ~300 |
| Shell 执行 | `tools/RunShellCommand.ts` | ~400 |
| 网页搜索 | `tools/WebSearch.ts` | ~500 |
| 浏览器快照 | `tools/BrowserSnapshot.ts` | ~350 |
| 调用子代理 | `tools/CallSubagent.ts` | ~300 |
| 发送消息 | `tools/SendMessage.ts` | ~150 |
| MCP 工具包装 | `tools/ReadMcpResource.ts` | ~200 |
| 工具循环 | `tools/tool-loop.ts` | ~300 |

### MCP 系统 (`mcp/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| MCP 管理器 | `mcp/manager.ts` | ~500 |
| 服务器管理 | `mcp/server.ts` | ~300 |
| 工具发现 | `mcp/discovery.ts` | ~250 |
| 工具转换 | `mcp/tools.ts` | ~200 |

### 扩展系统 (`extension/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 扩展接口 | `extension/types.ts` | ~100 |
| 扩展管理器 | `extension/manager.ts` | ~400 |
| 市场集成 | `extension/marketplace.ts` | ~300 |
| 加载器 | `extension/loader.ts` | ~250 |

### 配置系统 (`config/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 配置类型 | `config/types.ts` | ~300 |
| 配置加载 | `config/load.ts` | ~400 |
| 配置验证 | `config/validation.ts` | ~300 |
| 配置迁移 | `config/migration.ts` | ~350 |
| 设置管理 | `config/settings.ts` | ~250 |

### 服务层 (`services/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 文件发现 | `services/fileDiscovery.ts` | ~300 |
| Git 集成 | `services/git.ts` | ~400 |
| 会话管理 | `services/session.ts` | ~350 |

### IDE 集成 (`ide/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| IDE 接口 | `ide/types.ts` | ~100 |
| VS Code 集成 | `ide/vscode.ts` | ~300 |
| JetBrains 集成 | `ide/jetbrains.ts` | ~250 |
| Zed 集成 | `ide/zed.ts` | ~200 |
| 通用集成 | `ide/generic.ts` | ~150 |

### 模型配置 (`models/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 模型注册表 | `models/registry.ts` | ~400 |
| 模型路由 | `models/router.ts` | ~300 |
| 模型配置 | `models/config.ts` | ~250 |

### 子代理 (`subagents/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 子代理接口 | `subagents/types.ts` | ~100 |
| 子代理管理器 | `subagents/manager.ts` | ~350 |
| 任务委托 | `subagents/delegate.ts` | ~250 |

### 技能系统 (`skills/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 技能接口 | `skills/types.ts` | ~100 |
| 技能管理器 | `skills/manager.ts` | ~300 |
| 技能加载 | `skills/loader.ts` | ~200 |
| 技能搜索 | `skills/search.ts` | ~250 |

### 钩子系统 (`hooks/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 钩子接口 | `hooks/types.ts` | ~100 |
| 钩子管理器 | `hooks/manager.ts` | ~300 |
| 生命周期钩子 | `hooks/lifecycle.ts` | ~200 |

### 遥测 (`telemetry/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 遥测接口 | `telemetry/types.ts` | ~100 |
| 遥测收集 | `telemetry/collector.ts` | ~200 |
| 遥测发送 | `telemetry/sender.ts` | ~150 |

### LSP 支持 (`lsp/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| LSP 客户端 | `lsp/client.ts` | ~300 |
| LSP 工具 | `lsp/tools.ts` | ~200 |

### 工具函数 (`utils/`)

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 文件工具 | `utils/file.ts` | ~200 |
| 字符串工具 | `utils/string.ts` | ~150 |
| 类型工具 | `utils/types.ts` | ~100 |
| 异步工具 | `utils/async.ts` | ~100 |

---

## 集成测试

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 测试配置 | `integration-tests/vitest.config.ts` | ~100 |
| 全局设置 | `integration-tests/globalSetup.ts` | ~150 |
| 文件系统测试 | `integration-tests/file-system.test.ts` | ~300 |
| Shell 命令测试 | `integration-tests/run_shell_command.test.ts` | ~250 |
| 编辑操作测试 | `integration-tests/edit.test.ts` | ~300 |
| MCP 测试 | `integration-tests/mcp_server_cyclic_schema.test.ts` | ~200 |
| 配置迁移测试 | `integration-tests/settings-migration.test.ts` | ~200 |
| SDK 测试 | `integration-tests/sdk-typescript/*.test.ts` | ~300 |

---

## 构建脚本

| 功能 | 文件路径 | 行数估算 |
|------|----------|----------|
| 构建配置 | `esbuild.config.js` | ~100 |
| 构建主脚本 | `scripts/build.js` | ~200 |
| 包构建 | `scripts/build_package.js` | ~150 |
| 打包脚本 | `scripts/bundle.js` | ~150 |

---

## 配置文件

| 功能 | 文件路径 | 描述 |
|------|----------|------|
| 根 package.json | `package.json` | Workspaces 配置 |
| TypeScript 配置 | `tsconfig.json` | 基础 TS 配置 |
| Vitest 配置 | `vitest.config.ts` | 测试配置 |
| ESLint 配置 | `eslint.config.js` | Lint 规则 |
| Docker 配置 | `Dockerfile` | 容器镜像定义 |

---

## 技能文件 (`.qwen/`)

| 功能 | 文件路径 | 描述 |
|------|----------|------|
| 更新配置技能 | `.qwen/skills/update-config/` | 配置更新技能 |
| 简化技能 | `.qwen/skills/simplify/` | 代码简化技能 |
| 循环技能 | `.qwen/skills/loop/` | 循环任务技能 |
| Claude API 技能 | `.qwen/skills/claude-api/` | Claude API 技能 |

---

## 关键接口定义

### 工具接口

```typescript
// packages/core/src/tools/types.ts
interface Tool {
  name: string
  description: string
  parameters: JSONSchema
  execute(args: any): Promise<ToolResult>
}

interface ToolResult {
  content: string
  isError?: boolean
}
```

### LLM 客户端接口

```typescript
// packages/core/src/core/llm-client.ts
interface LLMClient {
  name: string
  complete(params: CompletionParams): Promise<CompletionResult>
  stream(params: CompletionParams): AsyncIterable<StreamChunk>
}

interface CompletionParams {
  model: string
  messages: Message[]
  tools?: Tool[]
  maxTokens?: number
  temperature?: number
}
```

### 扩展接口

```typescript
// packages/core/src/extension/types.ts
interface Extension {
  id: string
  name: string
  version: string
  activate(context: ExtensionContext): void
  deactivate(): void
}

interface ExtensionContext {
  registerCommand(command: Command): void
  registerTool(tool: Tool): void
}
```

---

## 测试文件索引

### CLI 包测试

- `packages/cli/src/**/*.test.ts`
- `packages/cli/src/**/*.test.tsx`

### Core 包测试

- `packages/core/src/**/*.test.ts`
- `packages/core/src/tools/*.test.ts`
- `packages/core/src/config/*.test.ts`

### SDK 测试

- `packages/sdk-typescript/**/*.test.ts`

### 运行测试

```bash
# 全部测试
pnpm test

# 单元测试
pnpm test:unit

# 集成测试
pnpm test:e2e

# 带覆盖率
pnpm test -- --coverage
```

---

*索引生成完成。如需补充特定文件，请告知。*
