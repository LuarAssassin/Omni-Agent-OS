# OpenCode 代码索引

> 关键文件快速导航
> 生成时间: 2026-03-17
> 基于软件设计哲学: 信息隐藏、模块深度、关注点分离

---

## 目录

1. [入口与启动](#一入口与启动)
2. [CLI 命令层](#二cli-命令层)
3. [Agent 系统](#三agent-系统)
4. [工具集 (Tools)](#四工具集-tools)
5. [配置系统](#五配置系统)
6. [会话管理](#六会话管理)
7. [模型提供商](#七模型提供商)
8. [MCP 协议](#八mcp-协议)
9. [存储与数据库](#九存储与数据库)
10. [权限系统](#十权限系统)
11. [LSP 支持](#十一lsp-支持)
12. [事件总线](#十二事件总线)
13. [核心基础设施](#十三核心基础设施)
14. [UI 层](#十四ui-层)

---

## 一、入口与启动

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `packages/opencode/src/index.ts` | CLI 主入口 | CLI 配置、命令注册、数据库初始化 |
| `packages/opencode/src/start.ts` | 启动逻辑 | 环境检查、版本检测 |
| `packages/opencode/package.json` | 包配置 | 依赖清单、脚本定义 |

**依赖关系**:
```
index.ts → cli/cmd/* (命令注册)
         → storage/migrate.ts (数据库迁移)
         → version.ts (版本检查)
```

---

## 二、CLI 命令层

路径: `packages/opencode/src/cli/cmd/`

### 核心命令

| 文件 | 命令 | 职责 | 深度评级 |
|------|------|------|----------|
| `run.ts` | `run` | 执行单条命令并退出 | ★★★★☆ |
| `agent.ts` | `agent` | 启动代理交互模式 | ★★★★☆ |
| `mcp.ts` | `mcp` | MCP 服务器管理 | ★★★★☆ |
| `tui.ts` | `tui` | 终端 UI 模式 | ★★★☆☆ |
| `serve.ts` | `serve` | 启动服务端 | ★★★★☆ |
| `web.ts` | `web` | Web 界面模式 | ★★★☆☆ |
| `session.ts` | `session` | 会话管理 (list/resume/rm) | ★★★★☆ |
| `db.ts` | `db` | 数据库管理 | ★★★★☆ |
| `config.ts` | `config` | 配置查看与编辑 | ★★★★☆ |
| `pr.ts` | `pr` | Pull Request 操作 | ★★★☆☆ |
| `stats.ts` | `stats` | 使用统计 | ★★★☆☆ |
| `debug.ts` | `debug` | 调试工具 | ★★★☆☆ |

### 命令辅助模块

| 文件 | 职责 |
|------|------|
| `cli/util.ts` | 命令行工具函数 |
| `cli/prompt.ts` | 交互式提示封装 |
| `cli/version.ts` | 版本检查与更新 |

**设计模式**: 每个命令是一个深度模块，封装完整的命令处理逻辑。

---

## 三、Agent 系统

路径: `packages/opencode/src/agent/`

### 核心文件

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `agent.ts` | Agent 定义与管理 | `Agent.Info`, `Agent.get()`, `PermissionNext` |

### Prompt 模板

路径: `packages/opencode/src/agent/prompt/`

| 文件 | 用途 |
|------|------|
| `build.txt` | Build Agent 系统提示词 |
| `plan.txt` | Plan Agent 系统提示词 |
| `explore.txt` | Explore Agent 系统提示词 |
| `general.txt` | General Agent 系统提示词 |
| `compaction.txt` | Compaction Agent 提示词 |
| `title.txt` | Title Agent 提示词 |
| `summary.txt` | Summary Agent 提示词 |

### Agent 类型速查

```typescript
// 7 种内置 Agent 类型
| Agent        | 模式      | 用途           | 权限特点               |
|--------------|-----------|----------------|------------------------|
| build        | primary   | 默认构建代理   | 允许编辑、提问、计划   |
| plan         | primary   | 计划模式       | 只读，仅规划           |
| general      | subagent  | 通用研究       | 禁用 todo 工具         |
| explore      | subagent  | 代码库探索     | 只允许搜索、读取       |
| compaction   | primary   | 会话压缩       | 无权限（内部使用）     |
| title        | primary   | 生成标题       | 无权限（内部使用）     |
| summary      | primary   | 生成摘要       | 无权限（内部使用）     |
```

---

## 四、工具集 (Tools)

路径: `packages/opencode/src/tool/`

### 核心定义

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `tool.ts` | Tool 基础定义 | `Tool.define()`, `Tool.execute()` |

### 工具实现 (25+ 工具)

#### 文件操作

| 文件 | 工具名 | 功能 |
|------|--------|------|
| `read.ts` | `read` | 读取文件/目录，支持 offset/limit |
| `write.ts` | `write` | 写入文件，带 LSP 诊断检查 |
| `edit.ts` | `edit` | 编辑文件（搜索替换） |
| `glob.ts` | `glob` | 文件模式匹配搜索 |
| `grep.ts` | `grep` | 内容搜索 |

#### 系统操作

| 文件 | 工具名 | 功能 |
|------|--------|------|
| `bash.ts` | `bash` | 执行 shell 命令 |
| `bash-command.ts` | - | Bash 命令解析辅助 |
| `file-upload.ts` | `file_upload` | 文件上传处理 |

#### 网络/Web

| 文件 | 工具名 | 功能 |
|------|--------|------|
| `web-search.ts` | `web_search` | Web 搜索 |
| `web-fetch.ts` | `web_fetch` | URL 内容获取 |
| `open-url.ts` | `open_url` | 打开 URL |

#### 代码智能

| 文件 | 工具名 | 功能 |
|------|--------|------|
| `lsp-diagnostics.ts` | `lsp_diagnostics` | LSP 诊断信息 |
| `tree-sitter.ts` | `tree_sitter` | 语法树解析 |

#### 开发工具

| 文件 | 工具名 | 功能 |
|------|--------|------|
| `npm.ts` | `npm` | npm 操作 |
| `github.ts` | `github` | GitHub API 操作 |
| `git-commit.ts` | `git_commit` | Git 提交 |

#### MCP 相关

| 文件 | 工具名 | 功能 |
|------|--------|------|
| `mcp.ts` | `mcp` | MCP 工具调用 |
| `external-directory.ts` | - | 外部目录访问检查 |

#### 其他工具

| 文件 | 工具名 | 功能 |
|------|--------|------|
| `todo.ts` | `todo` | 待办事项管理 |
| `agent.ts` | `agent` | 子代理调用 |
| `ask-user.ts` | `ask_user` | 用户确认提示 |
| `task.ts` | `task` | 任务管理 |
| `skill.ts` | `skill` | 技能调用 |
| `remember.ts` | `remember` | 记忆存储 |
| `cron.ts` | `cron` | 定时任务 |
| `exit-plan-mode.ts` | `exit_plan_mode` | 退出计划模式 |

### 工具描述文件

每个工具对应 `.txt` 描述文件（如 `read.txt`, `write.txt`），用于 LLM 理解工具用途。

---

## 五、配置系统

路径: `packages/opencode/src/config/`

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `config.ts` | 核心配置管理 | `Config.get()`, `Config.set()` |
| `tui.ts` | TUI 配置 | TUI 相关配置项 |
| `permission.ts` | 权限配置 | 权限规则定义 |
| `provider.ts` | 提供商配置 | 模型提供商配置 |

**配置层级** (由高到低):
1. Managed config (企业部署)
2. Inline config (环境变量)
3. `.opencode/` 目录配置
4. Project config (`opencode.json`)
5. Custom config (环境变量指定)
6. Global config (`~/.config/opencode/`)
7. Remote `.well-known/opencode`

---

## 六、会话管理

路径: `packages/opencode/src/session/`

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `session.ts` | 会话核心 | `Session` 类型，会话 CRUD |
| `message.ts` | 消息管理 | `Message` 类型，消息处理 |
| `context.ts` | 上下文管理 | 会话上下文处理 |
| `instruction.ts` | 指令提示 | `InstructionPrompt` |

---

## 七、模型提供商

路径: `packages/opencode/src/provider/`

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `provider.ts` | 提供商基础 | `Provider` 接口定义 |
| `router.ts` | 模型路由 | 提供商选择逻辑 |

### 提供商适配器 (20+)

路径: `packages/opencode/src/provider/`

| 文件 | 提供商 | 依赖 |
|------|--------|------|
| `anthropic.ts` | Claude (Anthropic) | `@ai-sdk/anthropic` |
| `openai.ts` | OpenAI GPT | `@ai-sdk/openai` |
| `google.ts` | Google Gemini | `@ai-sdk/google` |
| `groq.ts` | Groq | `@ai-sdk/groq` |
| `mistral.ts` | Mistral | `@ai-sdk/mistral` |
| `azure.ts` | Azure OpenAI | `@ai-sdk/azure` |
| `cohere.ts` | Cohere | `@ai-sdk/cohere` |
| `bedrock.ts` | AWS Bedrock | `@ai-sdk/bedrock` |
| `vertex.ts` | Google Vertex | `@ai-sdk/vertex` |
| `xai.ts` | xAI Grok | `@ai-sdk/xai` |
| `perplexity.ts` | Perplexity | `@ai-sdk/perplexity` |
| `togetherai.ts` | Together AI | `@ai-sdk/togetherai` |
| `deepinfra.ts` | Deep Infra | `@ai-sdk/deepinfra` |
| `cerebras.ts` | Cerebras | `@ai-sdk/cerebras` |
| `ollama.ts` | Ollama (本地) | - |
| `lmstudio.ts` | LM Studio | - |

---

## 八、MCP 协议

路径: `packages/opencode/src/mcp/`

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `index.ts` | MCP 管理 | `MCP`, `Status` 类型 |
| `client.ts` | MCP 客户端 | 客户端连接管理 |
| `auth.ts` | 认证管理 | OAuth 认证 |
| `oauth-callback.ts` | OAuth 回调 | 回调处理 |

**传输方式**: stdio | SSE | HTTP

---

## 九、存储与数据库

路径: `packages/opencode/src/storage/`

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `db.ts` | 数据库连接 | `Client` (延迟初始化) |
| `migrate.ts` | 迁移管理 | 数据库迁移逻辑 |

### Schema 定义

路径: `packages/opencode/src/storage/schema/`

| 文件 | 表 | 用途 |
|------|-----|------|
| `session.sql.ts` | `sessions` | 会话数据 |
| `message.sql.ts` | `messages` | 消息数据 |
| `mcp.sql.ts` | `mcps` | MCP 服务器配置 |
| `file.sql.ts` | `files` | 文件记录 |
| `todo.sql.ts` | `todos` | 待办事项 |
| `stats.sql.ts` | `stats` | 使用统计 |
| `skill.sql.ts` | `skills` | 技能数据 |
| `task.sql.ts` | `tasks` | 任务数据 |

---

## 十、权限系统

路径: `packages/opencode/src/permission/`

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `next.ts` | 权限核心 | `PermissionNext`, `Ruleset` |
| `ruleset.ts` | 规则集 | 权限规则定义 |

**权限级别**: `allow` | `ask` | `deny`

---

## 十一、LSP 支持

路径: `packages/opencode/src/lsp/`

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `lsp.ts` | LSP 主入口 | `LSP` 命名空间 |
| `client.ts` | LSP 客户端 | 客户端连接 |
| `server.ts` | 服务器管理 | LSP 服务器生命周期 |
| `language.ts` | 语言支持 | 语言检测 |
| `diagnostic.ts` | 诊断处理 | `Diagnostic` 类型 |

---

## 十二、事件总线

路径: `packages/opencode/src/bus/`

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `bus.ts` | 事件总线 | `Bus`, `GlobalBus` |
| `event.ts` | 事件定义 | `Event` 类型 |

**使用模式**:
```typescript
// 发布事件
Bus.publish(File.Event.Edited, { file: filepath })

// 订阅事件
Bus.subscribe(File.Event.Edited, handler)
```

---

## 十三、核心基础设施

### 项目实例

路径: `packages/opencode/src/project/`

| 文件 | 职责 |
|------|------|
| `instance.ts` | 项目实例管理 |
| `worktree.ts` | 工作树处理 |

### 文件系统

路径: `packages/opencode/src/file/`

| 文件 | 职责 |
|------|------|
| `file.ts` | 文件操作封装 |
| `time.ts` | 文件时间戳管理 |
| `watcher.ts` | 文件监听 |

### 工具库

路径: `packages/opencode/src/util/`

| 文件 | 职责 |
|------|------|
| `filesystem.ts` | 文件系统工具 |
| `diff.ts` | 差异计算 |
| `format.ts` | 格式化工具 |

### 控制平面

路径: `packages/opencode/src/control-plane/`

| 文件 | 职责 |
|------|------|
| `workspace.ts` | 工作空间管理 |
| `router.ts` | 请求路由 |
| `sse.ts` | SSE 连接管理 |

---

## 十四、UI 层

### TUI (Terminal UI)

路径: `packages/opencode/src/components/`

| 文件 | 组件 |
|------|------|
| `app.tsx` | 主应用组件 |
| `chat.tsx` | 聊天界面 |
| `input.tsx` | 输入组件 |
| `message.tsx` | 消息渲染 |
| `sidebar.tsx` | 侧边栏 |

### Web 应用

路径: `packages/app/`

| 目录 | 用途 |
|------|------|
| `src/` | SolidJS 源码 |
| `e2e/` | Playwright E2E 测试 |

### 桌面应用

路径: `packages/desktop/`

| 目录 | 用途 |
|------|------|
| `src/` | Tauri 桌面应用 |

---

## 代码导航技巧

### 按功能定位

| 需求 | 起始文件 |
|------|----------|
| 添加新 CLI 命令 | `src/cli/cmd/*.ts` → 参考现有命令模式 |
| 添加新工具 | `src/tool/tool.ts` → `src/tool/{name}.ts` |
| 修改 Agent 行为 | `src/agent/agent.ts` + `src/agent/prompt/*.txt` |
| 添加提供商 | `src/provider/*.ts` → 参考 anthropic.ts |
| 修改配置 | `src/config/config.ts` + schema 定义 |
| 数据库变更 | `src/storage/schema/*.sql.ts` + `src/storage/migrate.ts` |

### 关键类型定义

```typescript
// Agent 配置
Agent.Info: z.object({ name, mode, permissions, prompt, ... })

// 工具定义
Tool.define(name, { description, parameters, execute })

// 权限规则
Ruleset: Record<string, "allow" | "ask" | "deny" | Ruleset>

// MCP 状态
Status: "connected" | "disabled" | "failed" | ...
```

### 设计模式索引

| 模式 | 示例位置 |
|------|----------|
| Namespace 模式 | `export namespace Agent {}` |
| Lazy 初始化 | `const Client = lazy(() => ...)` |
| Zod Schema | `const Info = z.object({...})` |
| Effect 组合 | `Effect.gen(function* () {...})` |
| 事件总线 | `Bus.publish()` / `Bus.subscribe()` |

---

## 文件命名规范

- **文件**: kebab-case (`oauth-callback.ts`)
- **类型**: PascalCase (`Agent.Info`)
- **函数**: camelCase (`getDefaultModel()`)
- **常量**: SCREAMING_SNAKE_CASE
- **命名空间**: 单数名词 (`Config`, `Agent`)

---

*索引生成完成。如需补充特定模块的代码路径，请告知。*
