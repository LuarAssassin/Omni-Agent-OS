# PicoClaw 关键代码索引

> 快速导航到项目中的重要代码文件

---

## 入口与命令

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 主入口 | `cmd/picoclaw/main.go` | 72 |
| 命令总线 | `cmd/picoclaw/internal/helpers.go` | ~100 |
| Agent 命令 | `cmd/picoclaw/internal/agent/command.go` | ~200 |
| Auth 命令 | `cmd/picoclaw/internal/auth/command.go` | ~100 |
| Gateway 命令 | `cmd/picoclaw/internal/gateway/command.go` | ~150 |
| Skills 命令 | `cmd/picoclaw/internal/skills/command.go` | ~100 |
| Cron 命令 | `cmd/picoclaw/internal/cron/command.go` | ~100 |

---

## 核心 Agent

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| AgentInstance 定义 | `pkg/agent/instance.go` | 331 |
| 主循环 | `pkg/agent/loop.go` | ~400 |
| 上下文构建 | `pkg/agent/context.go` | ~500 |
| 消息摘要 | `pkg/agent/summarize.go` | ~200 |
| 消息分片 | `pkg/agent/split.go` | ~150 |

---

## 配置系统

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 配置结构 | `pkg/config/config.go` | ~1,500 |
| 配置加载 | `pkg/config/load.go` | ~300 |
| 配置校验 | `pkg/config/validation.go` | ~200 |
| 配置迁移 | `pkg/config/migration.go` | ~400 |

---

## 消息通道

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 通道管理器 | `pkg/channels/manager.go` | ~500 |
| 基础接口 | `pkg/channels/base.go` | ~200 |
| 消息分片 | `pkg/channels/split.go` | ~200 |
| Telegram | `pkg/channels/telegram.go` | ~400 |
| Discord | `pkg/channels/discord.go` | ~300 |
| QQ | `pkg/channels/qq.go` | ~300 |
| 飞书 | `pkg/channels/feishu.go` | ~400 |
| 钉钉 | `pkg/channels/dingtalk.go` | ~400 |
| Slack | `pkg/channels/slack.go` | ~300 |
| Matrix | `pkg/channels/matrix.go` | ~300 |
| WhatsApp | `pkg/channels/whatsapp.go` | ~400 |
| LINE | `pkg/channels/line.go` | ~300 |
| IRC | `pkg/channels/irc.go` | ~250 |
| Webhook | `pkg/channels/webhook.go` | ~200 |

---

## LLM 提供商

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 提供商接口 | `pkg/providers/provider.go` | ~200 |
| 工厂模式 | `pkg/providers/factory.go` | ~200 |
| 降级策略 | `pkg/providers/fallback.go` | ~300 |
| OpenAI | `pkg/providers/openai.go` | ~400 |
| Anthropic | `pkg/providers/anthropic.go` | ~400 |
| DeepSeek | `pkg/providers/deepseek.go` | ~300 |
| Gemini | `pkg/providers/gemini.go` | ~300 |
| GitHub Copilot | `pkg/providers/github_copilot.go` | ~400 |
| Codex CLI | `pkg/providers/codex_cli.go` | ~300 |
| Ollama | `pkg/providers/ollama.go` | ~300 |

---

## 工具系统

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 工具注册表 | `pkg/tools/registry.go` | ~300 |
| 工具接口 | `pkg/tools/types.go` | ~58 |
| 文件读取 | `pkg/tools/read_file.go` | ~200 |
| 文件写入 | `pkg/tools/write_file.go` | ~200 |
| 文件编辑 | `pkg/tools/edit_file.go` | ~400 |
| 目录列表 | `pkg/tools/list_dir.go` | ~150 |
| Shell 执行 | `pkg/tools/exec.go` | ~400 |
| 网页搜索 | `pkg/tools/web.go` | 1,139 |
| MCP 工具 | `pkg/tools/mcp_tool.go` | ~300 |
| 子代理 | `pkg/tools/subagent.go` | ~361 |
| 工具循环 | `pkg/tools/toolloop.go` | ~179 |

---

## MCP 系统

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| MCP 管理器 | `pkg/mcp/manager.go` | ~500 |
| 工具发现 | `pkg/mcp/discovery.go` | ~300 |
| MCP 工具 | `pkg/mcp/tools.go` | ~200 |

---

## 技能系统

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 技能管理 | `pkg/skills/skills.go` | ~400 |
| 技能加载 | `pkg/skills/loader.go` | ~300 |
| 技能搜索 | `pkg/skills/search.go` | ~250 |

---

## 消息总线

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 消息总线 | `pkg/bus/bus.go` | ~300 |
| 消息类型 | `pkg/bus/message.go` | ~100 |

---

## 会话与记忆

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 会话接口 | `pkg/session/session.go` | ~100 |
| 会话管理器 | `pkg/session/manager.go` | ~300 |
| JSONL 存储 | `pkg/memory/jsonl.go` | ~300 |
| 记忆迁移 | `pkg/memory/migration.go` | ~200 |

---

## 模型路由

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 路由器 | `pkg/routing/router.go` | ~300 |
| 评分算法 | `pkg/routing/scoring.go` | ~200 |

---

## 内置命令

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| 命令接口 | `pkg/commands/command.go` | ~100 |
| 内置命令 | `pkg/commands/builtin.go` | ~300 |
| 执行器 | `pkg/commands/executor.go` | ~200 |

---

## 认证系统

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| PKCE 流程 | `pkg/auth/pkce.go` | ~200 |
| Token 管理 | `pkg/auth/token.go` | ~200 |

---

## 工具函数

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| BM25 算法 | `pkg/utils/bm25.go` | ~272 |
| HTTP 重试 | `pkg/utils/http_retry.go` | ~60 |
| 文件下载 | `pkg/utils/download.go` | ~93 |
| 字符串工具 | `pkg/utils/string.go` | ~67 |
| ZIP 处理 | `pkg/utils/zip.go` | ~121 |
| 媒体处理 | `pkg/utils/media.go` | ~158 |

---

## Web 控制台

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| Web 后端主入口 | `web/backend/main.go` | ~300 |
| 静态资源嵌入 | `web/backend/embed.go` | ~100 |
| 中间件 | `web/backend/middleware/middleware.go` | ~150 |
| 访问控制 | `web/backend/middleware/access_control.go` | ~100 |

---

## TUI 启动器

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| TUI 主入口 | `cmd/picoclaw-launcher-tui/main.go` | ~100 |
| 应用逻辑 | `cmd/picoclaw-launcher-tui/internal/ui/app.go` | ~400 |
| 菜单组件 | `cmd/picoclaw-launcher-tui/internal/ui/menu.go` | ~300 |
| 数据模型 | `cmd/picoclaw-launcher-tui/internal/ui/model.go` | ~200 |
| 样式定义 | `cmd/picoclaw-launcher-tui/internal/ui/style.go` | ~100 |

---

## 测试文件索引

### 核心测试
- `pkg/agent/*_test.go` - Agent 核心测试
- `pkg/channels/*_test.go` - 通道测试
- `pkg/providers/*_test.go` - 提供商测试
- `pkg/tools/*_test.go` - 工具测试

### 命令测试
- `cmd/picoclaw/internal/agent/*_test.go`
- `cmd/picoclaw/internal/auth/*_test.go`
- `cmd/picoclaw/internal/cron/*_test.go`
- `cmd/picoclaw/internal/skills/*_test.go`

### 工具函数测试
- `pkg/utils/*_test.go`
- `pkg/config/*_test.go`
