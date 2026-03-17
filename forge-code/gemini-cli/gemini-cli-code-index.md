# Gemini CLI 代码索引

> 按包和模块组织的代码导航指南

## 目录

- [Package Overview](#package-overview)
- [Core Package](#core-package)
- [SDK Package](#sdk-package)
- [A2A Server Package](#a2a-server-package)
- [CLI Package](#cli-package)
- [Test Utils Package](#test-utils-package)

---

## Package Overview

| 包名 | 路径 | 主要职责 | 对外暴露 |
|------|------|----------|----------|
| `@google/gemini-cli-core` | `packages/core/` | 核心逻辑、工具执行、调度、策略 | 100+ 导出 |
| `@google/gemini-cli-sdk` | `packages/sdk/` | 程序化 API | 5 核心导出 |
| `@google/gemini-cli-a2a-server` | `packages/a2a-server/` | A2A 协议服务器 | 3 核心导出 |
| `@google/gemini-cli` | `packages/cli/` | CLI 入口与 TUI | CLI 命令 |
| `@google/gemini-cli-test-utils` | `packages/test-utils/` | 测试工具 | 测试辅助 |

---

## Core Package

### 入口点

**文件**: `packages/core/src/index.ts`

核心包导出 200+ 成员，按功能模块组织：

### 1. Scheduler 模块 (`src/scheduler/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `scheduler.ts` | 工具调度器主类 | `Scheduler`, `SchedulerOptions` |
| `types.ts` | 状态机类型定义 | `CoreToolCallStatus`, `ToolCall`, `CompletedToolCall` |
| `state-manager.ts` | 状态管理 | `SchedulerStateManager` |
| `tool-executor.ts` | 工具执行器 | `ToolExecutor` |
| `tool-modifier.ts` | 工具修改处理器 | `ToolModificationHandler` |
| `confirmation.ts` | 确认逻辑 | `resolveConfirmation` |
| `policy.ts` | 策略检查 | `checkPolicy`, `updatePolicy` |

**状态机**: `Validating → Scheduled → Executing → Success/Error/Cancelled/AwaitingApproval`

### 2. MessageBus 模块 (`src/confirmation-bus/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `message-bus.ts` | 消息总线 | `MessageBus` |
| `types.ts` | 消息类型 | `MessageBusType`, `ToolConfirmationRequest` |

### 3. ToolRegistry 模块 (`src/tools/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `tool-registry.ts` | 工具注册表 | `ToolRegistry`, `DiscoveredTool`, `DiscoveredMCPTool` |
| `tools.ts` | 基础工具定义 | `BaseDeclarativeTool`, `BaseToolInvocation`, `Kind` |
| `tool-error.ts` | 工具错误 | `ToolErrorType` |
| `tool-names.ts` | 工具名称常量 | `DISCOVERED_TOOL_PREFIX`, `TOOL_LEGACY_ALIASES` |
| `read-file.ts` | 读取文件工具 | `ReadFileTool` |
| `edit.ts` | 编辑文件工具 | `EditTool` |
| `write-file.ts` | 写入文件工具 | `WriteFileTool` |
| `ls.ts` | 列出目录工具 | `LsTool` |
| `grep.ts` | 搜索内容工具 | `GrepTool` |
| `glob.ts` | 文件匹配工具 | `GlobTool` |
| `shell.ts` | Shell 执行工具 | `ShellTool` |
| `web-search.ts` | 网页搜索工具 | `WebSearchTool` |
| `web-fetch.ts` | 网页获取工具 | `WebFetchTool` |
| `mcp-client.ts` | MCP 客户端 | `MCPClientTool` |
| `mcp-tool.ts` | MCP 工具 | `DiscoveredMCPTool` |
| `memoryTool.ts` | 记忆工具 | `MemoryTool` |
| `ask-user.ts` | 询问用户工具 | `AskUserTool` |
| `write-todos.ts` | 待办事项工具 | `WriteTodosTool` |

### 4. PolicyEngine 模块 (`src/policy/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `policy-engine.ts` | 策略引擎 | `PolicyEngine` |
| `types.ts` | 策略类型 | `PolicyDecision`, `ApprovalMode`, `PolicyRule` |
| `toml-loader.ts` | TOML 配置加载 | `TomlPolicyLoader` |
| `config.ts` | 策略配置 | `PolicyConfig` |
| `integrity.ts` | 完整性检查 | `PolicyIntegrity` |

### 5. GeminiClient 模块 (`src/core/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `client.ts` | AI 客户端 | `GeminiClient` |
| `geminiChat.ts` | 聊天管理 | `GeminiChat` |
| `turn.ts` | 回合管理 | `Turn`, `GeminiEventType`, `CompressionStatus` |
| `prompts.ts` | 系统提示词 | `getCoreSystemPrompt` |
| `tokenLimits.ts` | 令牌限制 | `tokenLimit` |
| `geminiRequest.ts` | 请求处理 | `partListUnionToString` |
| `contentGenerator.ts` | 内容生成 | `ContentGenerator` |
| `loggingContentGenerator.ts` | 日志内容生成 | `LoggingContentGenerator` |
| `recordingContentGenerator.ts` | 录制内容生成 | `RecordingContentGenerator` |

### 6. HookSystem 模块 (`src/hooks/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `index.ts` | 钩子系统入口 | `HookSystem`, `HookRegistry`, `HookRunner` |
| `types.ts` | 钩子类型 | `HookType`, `BeforeAgentHookOutput`, `AfterAgentHookOutput` |
| `hookSystem.ts` | 钩子系统实现 | `HookSystem` 实现 |

**事件类型**: `SessionStart`, `SessionEnd`, `BeforeAgent`, `AfterAgent`, `BeforeModel`, `AfterModel`, `BeforeTool`, `AfterTool`

### 7. Services 模块 (`src/services/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `chatCompressionService.ts` | 聊天压缩 | `ChatCompressionService` |
| `loopDetectionService.ts` | 循环检测 | `LoopDetectionService` |
| `gitService.ts` | Git 服务 | `GitService` |
| `shellExecutionService.ts` | Shell 执行 | `ShellExecutionService` |
| `sandboxManager.ts` | 沙盒管理 | `SandboxManager` |
| `fileDiscoveryService.ts` | 文件发现 | `FileDiscoveryService` |
| `fileSystemService.ts` | 文件系统 | `FileSystemService` |
| `contextManager.ts` | 上下文管理 | `ContextManager` |
| `chatRecordingService.ts` | 聊天录制 | `ChatRecordingService`, `ResumedSessionData` |
| `executionLifecycleService.ts` | 生命周期 | `ExecutionLifecycleService` |
| `keychainService.ts` | 密钥链 | `KeychainService` |

### 8. Config 模块 (`src/config/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `config.ts` | 主配置 | `Config` |
| `storage.ts` | 存储 | `Storage` |
| `agent-loop-context.ts` | 代理上下文 | `AgentLoopContext` |
| `memory.ts` | 记忆配置 | `MemoryConfig` |
| `models.ts` | 模型配置 | `ModelConfig`, `getDisplayString`, `resolveModel` |
| `constants.ts` | 常量 | 各种配置常量 |
| `defaultModelConfigs.ts` | 默认模型 | 默认模型配置 |

### 9. Utils 模块 (`src/utils/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `errors.ts` | 错误处理 | `FatalError`, `getErrorMessage`, `isAbortError` |
| `retry.ts` | 重试逻辑 | `retryWithBackoff` |
| `session.ts` | 会话 | `createSessionId`, `sessionId` |
| `events.ts` | 事件 | `coreEvents`, `CoreEvent` |
| `gitUtils.ts` | Git 工具 | Git 辅助函数 |
| `shell-utils.ts` | Shell 工具 | Shell 辅助函数 |
| `fileUtils.ts` | 文件工具 | 文件操作辅助 |
| `paths.ts` | 路径 | `homedir`, `tmpdir`, 路径辅助 |
| `checkpointUtils.ts` | 检查点 | 检查点相关工具 |
| `safeJsonStringify.ts` | JSON 安全序列化 | `safeJsonStringify` |

### 10. MCP 模块 (`src/mcp/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `oauth-provider.ts` | OAuth 提供器 | `MCPOAuthProvider` |
| `oauth-token-storage.ts` | Token 存储 | `MCPOAuthTokenStorage` |
| `oauth-utils.ts` | OAuth 工具 | `OAuthUtils` |

### 11. IDE 模块 (`src/ide/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `ide-client.ts` | IDE 客户端 | `IDEClient` |
| `ideContext.ts` | IDE 上下文 | `ideContextStore` |
| `detect-ide.ts` | IDE 检测 | `IDE_DEFINITIONS`, `isCloudShell` |
| `types.ts` | IDE 类型 | `IdeContext`, `File` |

### 12. Telemetry 模块 (`src/telemetry/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `index.ts` | 遥测入口 | 遥测函数 |
| `types.ts` | 遥测类型 | `ToolCallEvent`, `LlmRole` |
| `loggers.ts` | 日志器 | `logBillingEvent`, `logToolCall` |
| `constants.ts` | 遥测常量 | `GeminiCliOperation` |
| `trace.ts` | 追踪 | `runInDevTraceSpan` |

### 13. Skills 模块 (`src/skills/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `skillManager.ts` | 技能管理 | `SkillManager` |
| `skillLoader.ts` | 技能加载 | `SkillLoader` |

---

## SDK Package

### 入口点

**文件**: `packages/sdk/src/index.ts`

导出 5 个核心成员：

| 导出 | 来源文件 | 说明 |
|------|----------|------|
| `GeminiCliAgent` | `agent.ts` | SDK 代理主类 |
| `GeminiCliSession` | `session.ts` | 会话管理 |
| `tool` | `tool.ts` | 工具定义 DSL |
| `Skill` | `skills.ts` | 技能定义 |
| `SessionContext` | `types.ts` | 会话上下文类型 |

### 核心文件

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `agent.ts` | 代理管理 | `GeminiCliAgent` |
| `session.ts` | 会话实现 | `GeminiCliSession` |
| `tool.ts` | 工具定义 | `tool()`, `Tool`, `ToolDefinition`, `ModelVisibleError` |
| `skills.ts` | 技能管理 | `Skill` |
| `types.ts` | 类型定义 | `SessionContext`, `GeminiCliAgentOptions` |

---

## A2A Server Package

### 入口点

**文件**: `packages/a2a-server/src/index.ts`

导出 3 个核心成员：

| 导出 | 来源文件 | 说明 |
|------|----------|------|
| `AgentExecutor` | `agent/executor.ts` | A2A 代理执行器 |
| `A2AApp` | `http/app.ts` | A2A HTTP 应用 |
| `A2ARequest` | `types.ts` | A2A 请求类型 |

### 模块详情

#### Agent 模块 (`src/agent/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `executor.ts` | 代理执行器 | `AgentExecutor` |
| `task.ts` | 任务管理 | `Task` 类 |

**Task 生命周期**: `submitted → working → (input-required → working) → completed/failed/cancelled`

#### HTTP 模块 (`src/http/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `app.ts` | Express 应用 | `A2AApp` |
| `server.ts` | 服务器启动 | 服务器启动函数 |
| `requestStorage.ts` | 请求存储 | 异步存储 |

#### Commands 模块 (`src/commands/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `command-registry.ts` | 命令注册表 | `CommandRegistry` |
| `extensions.ts` | 扩展命令 | 扩展相关命令 |
| `init.ts` | 初始化命令 | 初始化逻辑 |
| `memory.ts` | 记忆命令 | 记忆管理 |
| `restore.ts` | 恢复命令 | 恢复逻辑 |
| `types.ts` | 命令类型 | 命令类型定义 |

#### Config 模块 (`src/config/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `config.ts` | A2A 配置 | `A2AConfig` |
| `extension.ts` | 扩展配置 | 扩展配置类型 |
| `settings.ts` | 设置 | 设置管理 |

#### Persistence 模块 (`src/persistence/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `gcs.ts` | GCS 持久化 | Google Cloud Storage |

---

## CLI Package

### 入口点

**文件**: `packages/cli/index.ts`

CLI 入口，处理全局异常和退出清理。

### 核心模块

#### 主模块 (`src/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `gemini.ts` | CLI 主入口 | `main()` 函数 |

#### ACP 模块 (`src/acp/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `acpClient.ts` | ACP 客户端 | `ACPClient` |
| `acpErrors.ts` | ACP 错误 | `ACPError` |
| `commandHandler.ts` | 命令处理 | `CommandHandler` |
| `commands/` | ACP 命令 | 各种 ACP 命令 |

#### Commands 模块 (`src/commands/`)

**Extensions 子模块** (`src/commands/extensions/`)

| 文件 | 职责 |
|------|------|
| `configure.ts` | 配置扩展 |
| `disable.ts` | 禁用扩展 |
| `enable.ts` | 启用扩展 |
| `install.ts` | 安装扩展 |
| `link.ts` | 链接扩展 |
| `list.ts` | 列出扩展 |
| `new.ts` | 创建新扩展 |
| `uninstall.ts` | 卸载扩展 |
| `update.ts` | 更新扩展 |
| `validate.ts` | 验证扩展 |
| `utils.ts` | 扩展工具 |

**MCP 子模块** (`src/commands/mcp/`)

| 文件 | 职责 |
|------|------|
| `mcp.ts` | MCP 主命令 |
| `add.ts` | 添加 MCP |
| `list.ts` | 列出 MCP |
| `remove.ts` | 移除 MCP |
| `enableDisable.ts` | 启用/禁用 MCP |

**Skills 子模块** (`src/commands/skills/`)

| 文件 | 职责 |
|------|------|
| `install.ts` | 安装技能 |
| `uninstall.ts` | 卸载技能 |
| `list.ts` | 列出技能 |
| `enable.ts` | 启用技能 |
| `disable.ts` | 禁用技能 |
| `link.ts` | 链接技能 |

#### Config 模块 (`src/config/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `auth.ts` | 认证 | 认证逻辑 |

#### UI 组件 (`src/components/`)

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `App.tsx` | 主应用组件 | React Ink 应用 |
| `Chat.tsx` | 聊天组件 | 聊天界面 |
| `StreamingContent.tsx` | 流式内容 | 流式渲染 |
| `ConfirmationPrompt.tsx` | 确认提示 | 用户确认 UI |
| `MultiConfirmation.tsx` | 多确认 | 批量确认 |
| `ModelSelector.tsx` | 模型选择 | 模型选择 UI |

---

## Test Utils Package

### 文件结构

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `index.ts` | 测试工具入口 | 测试辅助函数 |
| `mocks/` | 模拟对象 | 各种 mock |

---

## 依赖关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                           CLI Package                            │
│                     (UI + Commands + Entry)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                         SDK Package                              │
│                   (Programmatic API)                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Core Package                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │Scheduler │ │MessageBus│ │ToolRegis-│ │PolicyEng-│            │
│  │          │ │          │ │  try     │ │  ine     │            │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │GeminiCli-│ │HookSystem│ │Services  │ │Config    │            │
│  │  ent     │ │          │ │          │ │          │            │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘            │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     A2A Server Package                           │
│                    (A2A Protocol Server)                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 快速导航

### 按功能查找

| 功能 | 文件路径 |
|------|----------|
| 添加新工具 | `core/src/tools/tools.ts` (基础类) |
| 修改调度逻辑 | `core/src/scheduler/scheduler.ts` |
| 修改策略规则 | `core/src/policy/policy-engine.ts` |
| 添加钩子事件 | `core/src/hooks/types.ts`, `core/src/hooks/hookSystem.ts` |
| 修改聊天压缩 | `core/src/services/chatCompressionService.ts` |
| 修改循环检测 | `core/src/services/loopDetectionService.ts` |
| 添加 CLI 命令 | `cli/src/commands/` |
| 修改 UI 组件 | `cli/src/components/` |
| SDK 工具定义 | `sdk/src/tool.ts` |

---

*代码版本: 0.35.0-nightly.20260313*
*索引生成时间: 2026-03-17*
