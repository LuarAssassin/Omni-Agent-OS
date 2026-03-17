# Pi-Mono 代码索引

> 关键文件快速导航
> 项目: pi-mono
> 生成时间: 2026-03-17

---

## 快速入口

| 目的 | 文件路径 |
|------|----------|
| 项目配置 | `package.json` |
| 根 README | `README.md` |
| 贡献指南 | `CONTRIBUTING.md` |
| Agent 指南 | `AGENTS.md` |

---

## 按包索引

### @mariozechner/pi-ai (`packages/ai/`)

#### 核心入口
| 文件 | 说明 |
|------|------|
| `src/index.ts` | 主导出，类型和函数 |
| `src/types.ts` | 核心类型定义 (Model, Message, Tool 等) |
| `src/stream.ts` | 流式处理函数 `streamSimple` |
| `src/models.ts` | 模型注册表 |
| `src/api-registry.ts` | API 注册表 |
| `src/cli.ts` | CLI 入口 (`pi-ai` 命令) |
| `src/oauth.ts` | OAuth 流程入口 |

#### 提供商实现 (`src/providers/`)
| 文件 | 说明 |
|------|------|
| `anthropic.ts` | Anthropic Messages API |
| `openai-completions.ts` | OpenAI Completions API |
| `openai-responses.ts` | OpenAI Responses API |
| `openai-codex-responses.ts` | GitHub Copilot/Codex |
| `azure-openai-responses.ts` | Azure OpenAI |
| `google.ts` | Google Gemini API |
| `google-vertex.ts` | Google Vertex AI |
| `google-gemini-cli.ts` | Google Gemini CLI 集成 |
| `mistral.ts` | Mistral API |
| `amazon-bedrock.ts` | AWS Bedrock |
| `register-builtins.ts` | 注册内置提供商 |

#### 工具函数 (`src/utils/`)
| 文件 | 说明 |
|------|------|
| `event-stream.ts` | 事件流处理 |
| `json-parse.ts` | JSON 流式解析 |
| `validation.ts` | 输入验证 |
| `hash.ts` | 哈希工具 |
| `oauth/types.ts` | OAuth 类型定义 |
| `oauth/anthropic.ts` | Anthropic OAuth |
| `oauth/github-copilot.ts` | GitHub Copilot OAuth |
| `oauth/google-*.ts` | Google OAuth 实现 |
| `oauth/openai-codex.ts` | OpenAI Codex OAuth |

---

### @mariozechner/pi-agent-core (`packages/agent/`)

| 文件 | 说明 |
|------|------|
| `src/index.ts` | 主导出 |
| `src/agent.ts` | **Agent 类核心** - 状态管理和事件系统 |
| `src/agent-loop.ts` | Agent 执行循环 |
| `src/types.ts` | Agent 类型定义 |
| `src/proxy.ts` | 代理工具 |
| `test/agent.test.ts` | Agent 单元测试 |
| `test/agent-loop.test.ts` | Agent 循环测试 |
| `test/e2e.test.ts` | 端到端测试 |

---

### @mariozechner/pi-tui (`packages/tui/`)

| 文件 | 说明 |
|------|------|
| `src/index.ts` | 主导出 |
| `src/tui.ts` | **TUI 核心** - Container, Component 基类 |
| `src/terminal.ts` | 终端抽象和实现 |
| `src/keys.ts` | 键盘输入处理 (Kitty 协议支持) |
| `src/keybindings.ts` | 编辑器键绑定管理 |
| `src/fuzzy.ts` | 模糊匹配 |
| `src/terminal-image.ts` | 终端图片渲染 |
| `src/stdin-buffer.ts` | 输入缓冲 |

#### 组件 (`src/components/`)
| 文件 | 说明 |
|------|------|
| `box.ts` | 布局容器 |
| `text.ts` | 文本组件 |
| `input.ts` | 单行输入 |
| `editor.ts` | 多行编辑器 |
| `markdown.ts` | Markdown 渲染 |
| `image.ts` | 图片组件 |
| `select-list.ts` | 选择列表 |
| `settings-list.ts` | 设置列表 |
| `loader.ts` | 加载动画 |
| `cancellable-loader.ts` | 可取消加载 |
| `spacer.ts` | 间距组件 |
| `truncated-text.ts` | 截断文本 |

---

### @mariozechner/pi-coding-agent (`packages/coding-agent/`)

#### CLI 入口
| 文件 | 说明 |
|------|------|
| `src/cli.ts` | **主 CLI 入口** (`pi` 命令) |
| `src/cli/args.ts` | 参数解析 |
| `src/cli/session-picker.ts` | 会话选择器 |
| `src/cli/config-selector.ts` | 配置选择器 |
| `src/cli/file-processor.ts` | 文件处理 |
| `src/cli/list-models.ts` | 模型列表 |

#### 核心 (`src/core/`)
| 文件 | 说明 |
|------|------|
| `index.ts` | 核心模块导出 |
| `agent-session.ts` | **AgentSession 类** - 会话管理 |
| `session-manager.ts` | 会话持久化管理 |
| `settings-manager.ts` | 设置管理 |
| `model-registry.ts` | 模型注册表 |
| `model-resolver.ts` | 模型解析 |
| `bash-executor.ts` | Bash 命令执行 |
| `event-bus.ts` | 事件总线 |
| `system-prompt.ts` | 系统提示词构建 |
| `prompt-templates.ts` | 提示词模板 |
| `slash-commands.ts` | 斜杠命令 |
| `skills.ts` | 技能系统 |
| `resource-loader.ts` | 资源加载 |
| `defaults.ts` | 默认配置 |
| `config.ts` | 配置管理 |
| `auth-storage.ts` | 认证存储 |
| `diagnostics.ts` | 诊断信息 |
| `timings.ts` | 计时工具 |
| `sdk.ts` | SDK 入口 |

#### 工具 (`src/core/tools/`)
| 文件 | 说明 |
|------|------|
| `index.ts` | 工具注册 |
| `read.ts` | 文件读取工具 |
| `write.ts` | 文件写入工具 |
| `edit.ts` | 文件编辑工具 |
| `edit-diff.ts` | 差异编辑工具 |
| `bash.ts` | Bash 执行工具 |
| `find.ts` | 文件查找工具 |
| `grep.ts` | 文本搜索工具 |
| `ls.ts` | 目录列表工具 |
| `truncate.ts` | 文本截断工具 |
| `path-utils.ts` | 路径工具 |

#### 扩展系统 (`src/core/extensions/`)
| 文件 | 说明 |
|------|------|
| `index.ts` | 扩展系统导出 |
| `types.ts` | 扩展类型定义 |
| `loader.ts` | 扩展加载器 |
| `runner.ts` | 扩展运行器 |
| `wrapper.ts` | 扩展包装器 |

#### 上下文压缩 (`src/core/compaction/`)
| 文件 | 说明 |
|------|------|
| `index.ts` | 压缩功能导出 |
| `compaction.ts` | 压缩主逻辑 |
| `branch-summarization.ts` | 分支摘要 |
| `utils.ts` | 压缩工具函数 |

#### 导出 HTML (`src/core/export-html/`)
| 文件 | 说明 |
|------|------|
| `index.ts` | HTML 导出功能 |
| `tool-renderer.ts` | 工具渲染 |
| `ansi-to-html.ts` | ANSI 转 HTML |
| `template.js` | HTML 模板 |

#### 交互模式 (`src/modes/interactive/`)
| 文件 | 说明 |
|------|------|
| `interactive.ts` | 交互模式主逻辑 |
| `theme/theme.ts` | 主题定义 |
| `components/` | 交互组件 |

---

### @mariozechner/pi-web-ui (`packages/web-ui/`)

#### 主组件
| 文件 | 说明 |
|------|------|
| `src/index.ts` | 主导出 |
| `src/ChatPanel.ts` | **主聊天面板** |
| `src/AgentInterface.ts` | Agent 接口组件 |
| `src/Messages.ts` | 消息容器 |
| `src/MessageList.ts` | 消息列表 |
| `src/MessageEditor.ts` | 消息编辑器 |
| `src/Input.ts` | 输入组件 |
| `src/AttachmentTile.ts` | 附件展示 |
| `src/ConsoleBlock.ts` | 控制台输出 |
| `src/ThinkingBlock.ts` | 思考过程展示 |

#### 对话框 (`src/dialogs/`)
| 文件 | 说明 |
|------|------|
| `SettingsDialog.ts` | 设置对话框 |
| `ModelSelector.ts` | 模型选择器 |
| `SessionListDialog.ts` | 会话列表 |
| `ApiKeyPromptDialog.ts` | API Key 输入 |
| `CustomProviderDialog.ts` | 自定义提供商 |
| `AttachmentOverlay.ts` | 附件预览 |
| `PersistentStorageDialog.ts` | 持久化存储 |
| `ProvidersModelsTab.ts` | 提供商/模型标签页 |

#### 存储层 (`src/storage/`)
| 文件 | 说明 |
|------|------|
| `app-storage.ts` | 应用存储接口 |
| `store.ts` | Store 基类 |
| `backends/indexeddb-storage-backend.ts` | IndexedDB 后端 |
| `stores/sessions-store.ts` | 会话存储 |
| `stores/settings-store.ts` | 设置存储 |
| `stores/provider-keys-store.ts` | API Key 存储 |
| `stores/custom-providers-store.ts` | 自定义提供商存储 |

#### 工具组件 (`src/tools/`)
| 文件 | 说明 |
|------|------|
| `index.ts` | 工具导出 |
| `renderer-registry.ts` | 渲染器注册表 |
| `extract-document.ts` | 文档提取 |
| `javascript-repl.ts` | JS REPL |
| `artifacts/HtmlArtifact.ts` | HTML 产物 |
| `artifacts/ImageArtifact.ts` | 图片产物 |
| `artifacts/MarkdownArtifact.ts` | Markdown 产物 |
| `artifacts/SvgArtifact.ts` | SVG 产物 |
| `artifacts/TextArtifact.ts` | 文本产物 |
| `renderers/BashRenderer.ts` | Bash 渲染 |
| `renderers/CalculateRenderer.ts` | 计算渲染 |
| `renderers/DefaultRenderer.ts` | 默认渲染 |
| `renderers/GetCurrentTimeRenderer.ts` | 时间渲染 |

---

### @mariozechner/pi-mom (`packages/mom/`)

| 文件 | 说明 |
|------|------|
| `src/main.ts` | **Slack Bot 入口** |
| `src/slack.ts` | Slack Bot 实现 |
| `src/agent.ts` | Agent 运行器 |
| `src/store.ts` | 频道状态存储 |
| `src/events.ts` | 事件监听 |
| `src/context.ts` | 上下文创建 |
| `src/sandbox.ts` | 沙盒配置 |
| `src/tools/*.ts` | Mom 专用工具 |

---

### @mariozechner/pi-pods (`packages/pods/`)

| 文件 | 说明 |
|------|------|
| `src/cli.ts` | **CLI 入口** (`pi-pods`) |
| `src/config.ts` | 配置管理 |
| `src/types.ts` | 类型定义 |
| `src/ssh.ts` | SSH 执行 |
| `src/model-configs.ts` | 预定义模型配置 |
| `src/commands/pods.ts` | Pod 管理命令 |
| `src/commands/models.ts` | 模型管理命令 |
| `src/commands/prompt.ts` | Agent 对话命令 |

---

## 按功能索引

### Agent 核心功能
| 功能 | 相关文件 |
|------|----------|
| Agent 类 | `packages/agent/src/agent.ts` |
| Agent 循环 | `packages/agent/src/agent-loop.ts` |
| AgentSession | `packages/coding-agent/src/core/agent-session.ts` |
| 状态管理 | `packages/agent/src/types.ts` |
| 事件系统 | `packages/agent/src/types.ts`, `coding-agent/src/core/event-bus.ts` |
| 消息队列 | `packages/agent/src/agent.ts` (steeringQueue, followUpQueue) |

### LLM Provider 支持
| 提供商 | 文件 |
|--------|------|
| OpenAI | `packages/ai/src/providers/openai-*.ts` |
| Anthropic | `packages/ai/src/providers/anthropic.ts` |
| Google | `packages/ai/src/providers/google*.ts` |
| Mistral | `packages/ai/src/providers/mistral.ts` |
| Bedrock | `packages/ai/src/providers/amazon-bedrock.ts` |
| GitHub Copilot | `packages/ai/src/providers/openai-codex-responses.ts` |

### 工具系统
| 工具 | 文件 |
|------|------|
| read | `packages/coding-agent/src/core/tools/read.ts` |
| write | `packages/coding-agent/src/core/tools/write.ts` |
| edit | `packages/coding-agent/src/core/tools/edit.ts` |
| bash | `packages/coding-agent/src/core/tools/bash.ts` |
| find | `packages/coding-agent/src/core/tools/find.ts` |
| grep | `packages/coding-agent/src/core/tools/grep.ts` |
| ls | `packages/coding-agent/src/core/tools/ls.ts` |

### TUI 组件
| 组件 | 文件 |
|------|------|
| TUI 核心 | `packages/tui/src/tui.ts` |
| 终端抽象 | `packages/tui/src/terminal.ts` |
| Box | `packages/tui/src/components/box.ts` |
| Editor | `packages/tui/src/components/editor.ts` |
| Markdown | `packages/tui/src/components/markdown.ts` |
| Image | `packages/tui/src/components/image.ts` |
| SelectList | `packages/tui/src/components/select-list.ts` |

### Web UI 组件
| 组件 | 文件 |
|------|------|
| ChatPanel | `packages/web-ui/src/ChatPanel.ts` |
| Messages | `packages/web-ui/src/Messages.ts` |
| Input | `packages/web-ui/src/Input.ts` |
| Settings | `packages/web-ui/src/dialogs/SettingsDialog.ts` |
| Artifacts | `packages/web-ui/src/components/ArtifactsPanel.ts` |

---

## 配置文件

| 文件 | 说明 |
|------|------|
| `tsconfig.json` | 根 TypeScript 配置 |
| `tsconfig.base.json` | 共享 TS 配置 |
| `biome.json` | Biome (Linter/Formatter) 配置 |
| `package.json` | 根 package.json |
| `.husky/pre-commit` | Git 钩子 |

---

## 脚本

| 文件 | 说明 |
|------|------|
| `scripts/sync-versions.js` | 版本同步 |
| `scripts/release.mjs` | 发布脚本 |
| `scripts/check-browser-smoke.mjs` | 浏览器冒烟测试 |
| `test.sh` | 测试入口 |
| `pi-test.sh` | 从源码运行 pi |

---

*如需快速定位代码，使用 `grep -r "pattern" packages/*/src/` 搜索*
