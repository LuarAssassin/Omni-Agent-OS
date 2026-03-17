# OpenClaw 代码索引

> 关键文件快速导航与功能定位

---

## 目录

1. [入口文件](#一入口文件)
2. [Gateway 控制平面](#二gateway-控制平面)
3. [ACP 协议层](#三acp-协议层)
4. [配置系统](#四配置系统)
5. [Agent 系统](#五agent-系统)
6. [通道系统](#六通道系统)
7. [插件系统](#七插件系统)
8. [CLI 命令](#八cli-命令)
9. [扩展插件](#九扩展插件)

---

## 一、入口文件

| 文件 | 功能 | 行数 |
|------|------|------|
| `openclaw.mjs` | CLI 启动脚本 | ~30 |
| `src/index.ts` | 库导出入口 | ~50 |
| `src/cli-entry.ts` | CLI 入口点 | ~100 |

---

## 二、Gateway 控制平面

### 2.1 核心服务器

| 文件 | 功能 | 行数 | 设计评价 |
|------|------|------|----------|
| `src/gateway/server.impl.ts` | Gateway 主实现 | 1065 | ⭐⭐⭐⭐☆ 功能完整，文件偏长 |
| `src/gateway/server.ts` | 服务器导出 | 4 | ⭐⭐⭐⭐⭐ 简洁接口 |
| `src/gateway/server/close-reason.ts` | 关闭原因处理 | ~50 | ⭐⭐⭐⭐☆ 职责单一 |

### 2.2 服务器方法

| 文件 | 功能 |
|------|------|
| `src/gateway/server-methods/tools-catalog.ts` | 工具目录管理 |
| `src/gateway/server-methods/sessions.ts` | 会话管理方法 |
| `src/gateway/server-methods/channels.ts` | 通道管理方法 |
| `src/gateway/server-methods/agents.ts` | Agent 管理方法 |

### 2.3 工具调用

| 文件 | 功能 |
|------|------|
| `src/gateway/tools-invoke-http.ts` | HTTP 工具调用 |
| `src/gateway/tools-invoke-http.test.ts` | 工具调用测试 |

---

## 三、ACP 协议层

| 文件 | 功能 | 行数 | 设计评价 |
|------|------|------|----------|
| `src/acp/server.ts` | ACP 服务器入口 | 262 | ⭐⭐⭐⭐⭐ 接口简洁 |
| `src/acp/client.ts` | ACP 客户端 | ~200 | ⭐⭐⭐⭐☆ 连接管理 |
| `src/acp/types.ts` | ACP 类型定义 | ~100 | ⭐⭐⭐⭐⭐ 类型清晰 |

---

## 四、配置系统

### 4.1 核心 IO

| 文件 | 功能 | 行数 |
|------|------|------|
| `src/config/io.ts` | 配置读写核心 | ~500 |
| `src/config/types.ts` | 类型聚合导出 | 36 |
| `src/config/validation.ts` | 配置验证 | ~400 |

### 4.2 类型定义（分层设计）

| 文件 | 功能 |
|------|------|
| `src/config/types.base.ts` | 基础类型 |
| `src/config/types.agents.ts` | Agent 配置 |
| `src/config/types.channels.ts` | 通道配置 |
| `src/config/types.plugins.ts` | 插件配置 |
| `src/config/types.gateway.ts` | Gateway 配置 |
| `src/config/types.auth.ts` | 认证配置 |
| `src/config/types.secrets.ts` | 密钥配置 |
| `src/config/types.tools.ts` | 工具配置 |

### 4.3 遗留配置迁移

| 文件 | 功能 |
|------|------|
| `src/config/legacy-migrate.ts` | 配置迁移主逻辑 |
| `src/config/legacy.migrations.ts` | 迁移规则定义 |
| `src/config/legacy.rules.ts` | 遗留规则检测 |

### 4.4 环境处理

| 文件 | 功能 |
|------|------|
| `src/config/env-substitution.ts` | 环境变量替换 |
| `src/config/env-preserve.ts` | 环境变量保留 |
| `src/config/env-vars.ts` | 环境变量定义 |

---

## 五、Agent 系统

### 5.1 核心模块

| 文件 | 功能 | 行数 |
|------|------|------|
| `src/agents/acp-spawn.ts` | ACP Agent 启动 | ~300 |
| `src/agents/agent-paths.ts` | Agent 路径管理 | ~100 |
| `src/agents/agent-scope.ts` | Agent 作用域 | ~150 |

### 5.2 认证配置 (auth-profiles/)

| 文件 | 功能 |
|------|------|
| `src/agents/auth-profiles.ts` | 认证配置主模块 |
| `src/agents/auth-profiles/store.ts` | 配置存储 |
| `src/agents/auth-profiles/types.ts` | 类型定义 |
| `src/agents/auth-profiles/oauth.ts` | OAuth 处理 |
| `src/agents/auth-profiles/order.ts` | 优先级排序 |

### 5.3 Bash 工具

| 文件 | 功能 |
|------|------|
| `src/agents/bash-tools.ts` | Bash 工具主入口 |
| `src/agents/bash-tools.exec.ts` | 命令执行 |
| `src/agents/bash-tools.process.ts` | 进程管理 |
| `src/agents/bash-process-registry.ts` | 进程注册表 |

### 5.4 补丁应用

| 文件 | 功能 |
|------|------|
| `src/agents/apply-patch.ts` | 代码补丁应用 |
| `src/agents/apply-patch-update.ts` | 补丁更新 |

---

## 六、通道系统

### 6.1 通道核心

| 文件 | 功能 |
|------|------|
| `src/channels/plugins/catalog.ts` | 通道目录管理 |
| `src/channels/plugins/load.ts` | 通道加载器 |
| `src/channels/plugins/types.ts` | 通道类型定义 |

### 6.2 通道配置

| 文件 | 功能 |
|------|------|
| `src/channels/channel-config.ts` | 通道配置管理 |
| `src/channels/allowlist-match.ts` | 白名单匹配 |
| `src/channels/mention-gating.ts` | 提及门控 |
| `src/channels/command-gating.ts` | 命令门控 |

### 6.3 消息操作

| 文件 | 功能 |
|------|------|
| `src/channels/plugins/message-actions.ts` | 消息操作处理 |
| `src/channels/plugins/actions/telegram.ts` | Telegram 操作 |
| `src/channels/plugins/actions/discord.ts` | Discord 操作 |
| `src/channels/plugins/actions/signal.ts` | Signal 操作 |

### 6.4 归一化处理

| 文件 | 功能 |
|------|------|
| `src/channels/plugins/normalize/telegram.ts` | Telegram 数据归一化 |
| `src/channels/plugins/normalize/discord.ts` | Discord 数据归一化 |
| `src/channels/plugins/normalize/shared.ts` | 共享归一化逻辑 |

### 6.5 引导配置

| 文件 | 功能 |
|------|------|
| `src/channels/plugins/onboarding/telegram.ts` | Telegram 引导 |
| `src/channels/plugins/onboarding/discord.ts` | Discord 引导 |
| `src/channels/plugins/onboarding/slack.ts` | Slack 引导 |

---

## 七、插件系统

### 7.1 插件核心

| 文件 | 功能 |
|------|------|
| `src/plugins/discovery.ts` | 插件发现 |
| `src/plugins/manifest.ts` | 清单解析 |
| `src/plugins/types.ts` | 插件类型 |
| `src/plugins/tools.ts` | 工具插件 |

### 7.2 注册表

| 文件 | 功能 |
|------|------|
| `src/plugins/registry.ts` | 插件注册表 |
| `src/plugins/load.ts` | 插件加载 |

---

## 八、CLI 命令

### 8.1 命令入口

| 文件 | 功能 |
|------|------|
| `src/commands/index.ts` | 命令聚合 |
| `src/commands/gateway.ts` | Gateway 命令 |
| `src/commands/channels.ts` | 通道命令 |
| `src/commands/agents.ts` | Agent 命令 |

### 8.2 具体命令

| 文件 | 功能 |
|------|------|
| `src/commands/gateway-run.ts` | Gateway 启动 |
| `src/commands/channels-status.ts` | 通道状态 |
| `src/commands/config-set.ts` | 配置设置 |
| `src/commands/doctor.ts` | 诊断修复 |

---

## 九、扩展插件

### 9.1 通道扩展

| 扩展 | 路径 | 主要依赖 |
|------|------|----------|
| Telegram | `extensions/telegram/` | telegraf |
| Discord | `extensions/discord/` | discord.js |
| Slack | `extensions/slack/` | @slack/web-api |
| Signal | `extensions/signal/` | @whiskeysockets/signale |
| WhatsApp | `extensions/whatsapp/` | @whiskeysockets/baileys |
| Matrix | `extensions/matrix/` | matrix-js-sdk |
| IRC | `extensions/irc/` | irc-framework |
| Feishu | `extensions/feishu/` | @larksuiteoapi/node-sdk |
| MS Teams | `extensions/msteams/` | @microsoft/microsoft-graph-client |
| Google Chat | `extensions/googlechat/` | googleapis |
| Line | `extensions/line/` | @line/bot-sdk |
| Zalo | `extensions/zalo/` | zca-js |
| Twitch | `extensions/twitch/` | @twurple/auth |
| Nostr | `extensions/nostr/` | nostr-tools |
| Mattermost | `extensions/mattermost/` | @mattermost/client |
| BlueBubbles | `extensions/bluebubbles/` | (HTTP API) |
| Nextcloud Talk | `extensions/nextcloud-talk/` | (HTTP API) |
| Synology Chat | `extensions/synology-chat/` | (HTTP API) |
| Tlon | `extensions/tlon/` | @tloncorp/api |

### 9.2 功能扩展

| 扩展 | 路径 | 功能 |
|------|------|------|
| ACPX | `extensions/acpx/` | ACP 扩展协议 |
| Copilot Proxy | `extensions/copilot-proxy/` | GitHub Copilot 代理 |
| Diagnostics OTel | `extensions/diagnostics-otel/` | OpenTelemetry 诊断 |
| Diffs | `extensions/diffs/` | 代码差异处理 |
| Google Gemini CLI Auth | `extensions/google-gemini-cli-auth/` | Gemini CLI 认证 |
| LLM Task | `extensions/llm-task/` | LLM 任务调度 |
| Lobster | `extensions/lobster/` | (内部功能) |
| Memory Core | `extensions/memory-core/` | 记忆系统核心 |
| Memory LanceDB | `extensions/memory-lancedb/` | LanceDB 向量存储 |
| Minimax Portal Auth | `extensions/minimax-portal-auth/` | Minimax 认证 |
| Open Prose | `extensions/open-prose/` | 文本处理 |
| Voice Call | `extensions/voice-call/` | 语音通话 |

---

## 十、测试文件索引

### 10.1 配置测试

| 文件 | 功能 |
|------|------|
| `src/config/io.validation-fails-closed.test.ts` | IO 验证失败处理 |
| `src/config/config.env-vars.test.ts` | 环境变量测试 |
| `src/config/config.legacy-config-detection.*.test.ts` | 遗留配置检测 |

### 10.2 Agent 测试

| 文件 | 功能 |
|------|------|
| `src/agents/acp-spawn.test.ts` | ACP 启动测试 |
| `src/agents/bash-tools.exec.test.ts` | Bash 执行测试 |
| `src/agents/auth-profiles.*.test.ts` | 认证配置测试 |

### 10.3 Gateway 测试

| 文件 | 功能 |
|------|------|
| `src/gateway/tools-invoke-http.test.ts` | 工具调用测试 |
| `src/gateway/server-methods/tools-catalog.test.ts` | 工具目录测试 |

---

## 十一、关键路径速查

### 启动流程

```
openclaw.mjs
  → src/cli-entry.ts
    → src/commands/gateway.ts
      → src/commands/gateway-run.ts
        → src/gateway/server.impl.ts:startGatewayServer()
```

### 配置加载

```
任意代码调用
  → src/config/io.ts:loadConfig()
    → src/config/io.ts:readConfigFileSnapshot()
      → src/config/validation.ts:validateConfigObject()
```

### 消息处理

```
Channel 收到消息
  → src/channels/plugins/message-actions.ts
    → Gateway 路由
      → src/agents/acp-spawn.ts (如需要 Agent 处理)
        → @mariozechner/pi-agent-core
```

### 插件加载

```
Gateway 启动
  → src/plugins/discovery.ts:discoverOpenClawPlugins()
    → 扫描 extensions/ 目录
      → src/plugins/load.ts:loadPlugin()
        → jiti 转译加载
```

---

*索引完成 - 2026-03-17*
