# PicoClaw 项目深度分析报告

> 分析时间: 2026-03-17
> 项目路径: `/Users/luarassassin/reference-projects-me/picoclaw`
> Go 版本: 1.25.7

---

## 一、项目概述

### 1.1 项目定位

**PicoClaw** 是一个超轻量级个人 AI 助手，由 Sipeed 团队开发，完全使用 Go 语言构建。项目灵感来源于 [nanobot](https://github.com/HKUDS/nanobot)，但通过 AI 自举（self-bootstrapping）的方式从底层重构。

### 1.2 核心特性

| 特性 | 描述 |
|------|------|
| **极致轻量** | 可在 $10 硬件上运行，内存占用 < 10MB |
| **快速启动** | 1秒内完成启动 |
| **多平台** | 支持 x86_64, ARM64, ARMv7, MIPS LE, RISC-V, LoongArch, Darwin, Windows |
| **多通道** | 支持 15+ 消息平台（Telegram、Discord、QQ、飞书、钉钉等） |
| **MCP 支持** | 原生支持 Model Context Protocol |
| **技能系统** | 基于 Markdown 的层级化技能管理 |
| **模型路由** | 智能切换轻量/重量级模型 |

### 1.3 项目统计

| 指标 | 数值 |
|------|------|
| Go 源文件总数 | 355 个 |
| 代码总行数 | ~72,000 行 |
| 直接依赖数 | 35 个 |
| 间接依赖数 | 60+ 个 |
| 测试文件数 | 150+ 个 |

---

## 二、架构设计

### 2.1 分层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                           展示层                                     │
│  ┌──────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │ CLI (Cobra)  │  │ TUI (tview)     │  │ Web Console (React)     │ │
│  │  picoclaw    │  │ picoclaw-       │  │ Web Backend + Frontend  │ │
│  └──────────────┘  │ launcher-tui    │  └─────────────────────────┘ │
│                    └─────────────────┘                             │
├─────────────────────────────────────────────────────────────────────┤
│                           命令层 (cmd/)                              │
│  onboard │ auth │ agent │ gateway │ cron │ skills │ migrate | status│
├─────────────────────────────────────────────────────────────────────┤
│                           核心包 (pkg/)                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  agent  │ │channels │ │providers│ │  tools  │ │ skills  │       │
│  │ (代理核)│ │(消息通) │ │(LLM接口)│ │ (工具)  │ │(技能管) │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  bus    │ │ config  │ │  auth   │ │ memory  │ │ session │       │
│  │(消息总线)│ │(配置管) │ │(认证管) │ │(记忆存) │ │(会话管) │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  mcp    │ │ routing │ │ commands│ │  state  │ │  voice  │       │
│  │(MCP管)  │ │(路由)   │ │(内置命令)│ │(状态)   │ │(语音)   │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
├─────────────────────────────────────────────────────────────────────┤
│                           数据层                                     │
│  JSONL │ SQLite │ FileSystem │ Memory                               │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件关系

```
User Message
     │
     ▼
┌────────────┐
│  Channel   │◄──────► Channel Manager (15+ 平台适配)
└─────┬──────┘
      │
      ▼
┌────────────┐
│ MessageBus │◄──────► Pub/Sub 消息路由
└─────┬──────┘
      │
      ▼
┌────────────┐
│AgentInstance│
│ ┌─────────┐│
│ │Context  ││◄──────► Skills + MCP Tools + History
│ │ Builder ││
│ └────┬────┘│
│      │     │
│      ▼     │
│ ┌─────────┐│
│ │Provider ││◄──────► LLM API (OpenAI/Anthropic/DeepSeek/...)
│ │ (Router)││
│ └────┬────┘│
│      │     │
│      ▼     │
│ ┌─────────┐│
│ │Tool Loop││◄──────► Tool Registry (File/Shell/Web/...)
│ │(max 20) ││
│ └─────────┘│
└────────────┘
      │
      ▼
Response → Channel Send
```

---

## 三、目录结构详解

### 3.1 完整目录树

```
picoclaw/
├── assets/                          # 静态资源（Logo等）
│
├── cmd/                             # 可执行程序入口
│   ├── picoclaw/                    # 主 CLI 程序
│   │   ├── internal/                # 内部命令实现
│   │   │   ├── agent/               # 代理交互命令
│   │   │   │   ├── command.go       # agent 命令定义
│   │   │   │   └── helpers.go       # 辅助函数
│   │   │   ├── auth/                # 认证管理 (OAuth/PKCE)
│   │   │   │   ├── command.go
│   │   │   │   ├── login.go
│   │   │   │   ├── logout.go
│   │   │   │   ├── status.go
│   │   │   │   ├── models.go
│   │   │   │   └── *_test.go
│   │   │   ├── cron/                # 定时任务管理
│   │   │   │   ├── command.go
│   │   │   │   ├── add.go
│   │   │   │   ├── list.go
│   │   │   │   ├── remove.go
│   │   │   │   ├── enable.go
│   │   │   │   └── disable.go
│   │   │   ├── gateway/             # 网关服务命令
│   │   │   ├── migrate/             # 数据迁移命令
│   │   │   ├── onboard/             # 初始化向导
│   │   │   ├── skills/              # 技能管理命令
│   │   │   ├── status/              # 状态查询
│   │   │   ├── version/             # 版本信息
│   │   │   └── helpers.go           # 通用辅助函数
│   │   ├── main.go                  # 主入口（带 ASCII Logo）
│   │   └── main_test.go
│   │
│   └── picoclaw-launcher-tui/       # TUI 启动器（Web 控制台入口）
│       ├── internal/
│       │   ├── ui/                  # UI 组件
│       │   │   ├── app.go
│       │   │   ├── menu.go
│       │   │   ├── model.go
│       │   │   ├── style.go
│       │   │   ├── channel.go
│       │   │   ├── gateway_posix.go
│       │   │   └── gateway_windows.go
│       │   └── config/
│       │       └── store.go
│       └── main.go
│
├── config/                          # 配置文件示例
│
├── docker/                          # Docker 配置
│   ├── Dockerfile                   # 精简版（Alpine）
│   ├── Dockerfile.full              # 完整版（含 Node.js 24）
│   └── docker-compose.yml
│
├── docs/                            # 文档
│   ├── channels/                    # 各通道配置文档
│   │   ├── telegram.md
│   │   ├── discord.md
│   │   ├── qq.md
│   │   ├── feishu.md
│   │   ├── dingtalk.md
│   │   └── ...
│   └── *.md
│
├── pkg/                             # 核心包（业务逻辑）
│   ├── agent/                       # 代理核心逻辑
│   │   ├── instance.go              # AgentInstance 定义
│   │   ├── loop.go                  # 主循环逻辑
│   │   ├── context.go               # 上下文构建器
│   │   ├── summarize.go             # 消息摘要
│   │   └── split.go                 # 消息分片
│   │
│   ├── auth/                        # OAuth/PKCE 认证
│   │   ├── pkce.go                  # PKCE 流程实现
│   │   └── token.go                 # Token 管理
│   │
│   ├── bus/                         # 消息总线
│   │   ├── bus.go                   # Pub/Sub 实现
│   │   └── bus_test.go
│   │
│   ├── channels/                    # 消息通道（15+ 平台）
│   │   ├── manager.go               # 通道管理器
│   │   ├── base.go                  # 基础通道接口
│   │   ├── split.go                 # 消息分片
│   │   ├── telegram.go              # Telegram 适配
│   │   ├── discord.go               # Discord 适配
│   │   ├── qq.go                    # QQ 适配
│   │   ├── feishu.go                # 飞书适配
│   │   ├── dingtalk.go              # 钉钉适配
│   │   ├── slack.go                 # Slack 适配
│   │   ├── matrix.go                # Matrix 适配
│   │   ├── line.go                  # LINE 适配
│   │   ├── wechat.go                # 微信企业号适配
│   │   ├── irc.go                   # IRC 适配
│   │   ├── whatsapp.go              # WhatsApp 适配
│   │   ├── imessage.go              # iMessage 适配
│   │   ├── webhook.go               # Webhook 适配
│   │   └── placeholder.go           # 占位消息管理
│   │
│   ├── commands/                    # 内置命令系统
│   │   ├── command.go               # 命令接口
│   │   ├── builtin.go               # /help, /switch, /list 等
│   │   └── executor.go              # 命令执行器
│   │
│   ├── config/                      # 配置管理
│   │   ├── config.go                # 配置结构定义
│   │   ├── load.go                  # 配置加载
│   │   ├── validation.go            # 配置校验
│   │   └── migration.go             # 配置迁移
│   │
│   ├── cron/                        # 定时任务
│   │   ├── cron.go                  # 定时任务管理
│   │   └── scheduler.go             # 调度器
│   │
│   ├── devices/                     # 设备管理
│   │   └── device.go
│   │
│   ├── health/                      # 健康检查
│   │   └── health.go
│   │
│   ├── heartbeat/                   # 心跳服务
│   │   └── heartbeat.go
│   │
│   ├── identity/                    # 身份管理
│   │   └── identity.go
│   │
│   ├── logger/                      # 结构化日志 (Zerolog)
│   │   └── logger.go
│   │
│   ├── mcp/                         # MCP 管理器
│   │   ├── manager.go               # MCP 服务器管理
│   │   ├── tools.go                 # MCP 工具发现
│   │   └── discovery.go             # BM25 工具发现
│   │
│   ├── media/                       # 媒体存储
│   │   └── media.go
│   │
│   ├── memory/                      # 记忆存储
│   │   ├── memory.go
│   │   ├── jsonl.go                 # JSONL 存储后端
│   │   └── migration.go             # 数据迁移
│   │
│   ├── migrate/                     # 数据迁移
│   │   └── migrate.go
│   │
│   ├── providers/                   # LLM 提供商
│   │   ├── provider.go              # 提供商接口
│   │   ├── factory.go               # 工厂模式
│   │   ├── fallback.go              # 降级策略
│   │   ├── openai.go                # OpenAI 实现
│   │   ├── anthropic.go             # Anthropic 实现
│   │   ├── deepseek.go              # DeepSeek 实现
│   │   ├── gemini.go                # Gemini 实现
│   │   ├── github_copilot.go        # GitHub Copilot
│   │   ├── codex_cli.go             # Codex CLI
│   │   └── ollama.go                # Ollama 实现
│   │
│   ├── routing/                     # 模型路由
│   │   ├── router.go                # 智能路由
│   │   └── scoring.go               # 消息评分
│   │
│   ├── session/                     # 会话管理
│   │   ├── session.go               # 会话接口
│   │   └── manager.go               # JSON 存储后端
│   │
│   ├── skills/                      # 技能系统
│   │   ├── skills.go                # 技能管理
│   │   ├── loader.go                # 技能加载
│   │   └── search.go                # BM25 搜索
│   │
│   ├── state/                       # 状态管理
│   │   └── state.go
│   │
│   ├── tools/                       # 工具集（~1,144 行 web.go）
│   │   ├── registry.go              # 工具注册表
│   │   ├── read_file.go             # 文件读取
│   │   ├── write_file.go            # 文件写入
│   │   ├── edit_file.go             # 文件编辑
│   │   ├── list_dir.go              # 目录列表
│   │   ├── exec.go                  # Shell 执行
│   │   ├── web.go                   # 网页搜索（~1,144 行）
│   │   ├── mcp_tool.go              # MCP 工具包装
│   │   └── subagent.go              # 子代理调用
│   │
│   ├── utils/                       # 工具函数
│   │   ├── bm25.go                  # BM25 算法
│   │   ├── download.go
│   │   ├── http_retry.go
│   │   ├── media.go
│   │   ├── skills.go
│   │   ├── string.go
│   │   └── zip.go
│   │
│   └── voice/                       # 语音处理
│       └── transcriber.go
│
├── scripts/                         # 脚本工具
│
├── web/                             # Web 控制台
│   ├── backend/                     # Go 后端
│   │   ├── main.go
│   │   ├── embed.go                 # 静态资源嵌入
│   │   ├── middleware/              # 中间件
│   │   ├── launcherconfig/          # 启动器配置
│   │   └── utils/                   # 工具函数
│   │
│   └── frontend/                    # React + Vite 前端
│       ├── src/
│       ├── index.html
│       └── vite.config.ts
│
├── workspace/                       # 工作区示例
│   ├── skills/                      # 示例技能
│   ├── memory/                      # 记忆存储
│   ├── AGENTS.md                    # 代理定义
│   ├── SOUL.md                      # 系统提示词
│   └── USER.md                      # 用户配置
│
├── go.mod                           # Go 模块定义
├── go.sum                           # 依赖校验
├── Makefile                         # 构建脚本
├── .golangci.yaml                   # Lint 配置
├── .goreleaser.yaml                 # 发布配置
└── README.md                        # 项目文档
```

---

## 四、核心代码深度分析

### 4.1 AgentInstance 结构

```go
type AgentInstance struct {
    ID                        string              // 代理唯一标识
    Name                      string              // 代理名称
    Model                     string              // 主模型（如 "claude-sonnet-4-6"）
    Fallbacks                 []string            // 降级模型列表
    Workspace                 string              // 工作目录
    MaxIterations             int                 // 最大工具迭代次数（默认20）
    MaxTokens                 int                 // 最大 Token 数（默认8192）
    Temperature               float64             // 温度参数（默认0.7）
    ThinkingLevel             ThinkingLevel       // 思考级别（none/small/medium/large）
    ContextWindow             int                 // 上下文窗口大小
    SummarizeMessageThreshold int                 // 消息摘要阈值（默认20条）
    SummarizeTokenPercent     int                 // 摘要 Token 百分比（默认75%）
    Provider                  providers.LLMProvider
    Sessions                  session.SessionStore
    ContextBuilder            *ContextBuilder
    Tools                     *tools.ToolRegistry
    Subagents                 *config.SubagentsConfig
    SkillsFilter              []string
    Candidates                []providers.FallbackCandidate
    Router                    *routing.Router     // 模型路由器
    LightCandidates           []providers.FallbackCandidate // 轻量模型候选
}
```

### 4.2 消息处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      消息处理生命周期                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 接收 (Receive)                                               │
│     ├── Channel.Receive() 从平台 API 获取消息                     │
│     ├── 消息标准化（转换为 internal Message 结构）                 │
│     └── MessageBus.Publish() 发布到总线                          │
│                                                                 │
│  2. 路由 (Route)                                                 │
│     ├── AgentRegistry 查找目标代理                                │
│     ├── 如果是 /command 则走内置命令流程                          │
│     └── 否则进入 Agent Loop                                       │
│                                                                 │
│  3. 上下文构建 (Build Context)                                    │
│     ├── ContextBuilder.Build()                                   │
│     ├── 加载 Skills（BM25 搜索过滤）                              │
│     ├── 加载 MCP Tools（按需发现）                                │
│     ├── 加载 Session History                                     │
│     ├── 添加 System Prompt (SOUL.md)                             │
│     └── 最终 Prompt 组装                                         │
│                                                                 │
│  4. 模型调用 (LLM Call)                                           │
│     ├── Provider.Complete()                                      │
│     ├── 支持降级策略（主模型失败→Fallback模型）                    │
│     ├── 支持模型路由（轻量/重量级自动切换）                        │
│     └── 返回 LLMResponse                                         │
│                                                                 │
│  5. 工具循环 (Tool Loop)                                          │
│     ├── 解析 Tool Call 请求                                      │
│     ├── 工具权限校验（Read/Write 限制）                           │
│     ├── 执行 Tool（File/Shell/Web/Subagent/MCP）                 │
│     ├── 将结果追加到上下文                                        │
│     └── 循环直到无工具调用或达最大迭代次数                         │
│                                                                 │
│  6. 响应 (Response)                                               │
│     ├── 格式化最终响应                                            │
│     ├── SessionStore.Save() 保存会话                              │
│     └── Channel.Send() 发送回用户                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 工具注册表设计

```go
type ToolRegistry struct {
    tools   map[string]*ToolEntry
    mu      sync.RWMutex
    version atomic.Uint64  // 用于缓存失效
}

type ToolEntry struct {
    Tool   Tool            // 工具接口
    IsCore bool            // 是否为核心工具（显示在系统提示中）
    TTL    int             // 生存时间（用于 MCP 工具）
}

// 核心工具（始终可用）
- read_file    // 读取文件
- write_file   // 写入文件
- edit_file    // 编辑文件
- list_dir     // 列出目录
- exec         // 执行命令

// MCP 工具（动态发现）
- 通过 BM25 搜索或正则匹配发现
- 设置 TTL 控制有效期
- 定期 TickTTL() 递减，过期自动隐藏
```

### 4.4 MCP 发现机制

```
MCP Discovery Enabled
         │
         ▼
┌─────────────────┐
│ 消息内容分析     │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
BM25搜索   正则匹配
(关键词)   (模式)
    │         │
    └────┬────┘
         ▼
PromoteTools()
    │
    ▼
设置 TTL（默认3轮对话）
    │
    ▼
工具在 TTL 期间可用
    │
    ▼
TickTTL() 每轮递减
    │
    ▼
过期隐藏 → 下次需重新发现
```

### 4.5 模型路由策略

```go
// RouterConfig 配置
type RouterConfig struct {
    LightModel string  // 轻量模型（如 "haiku"）
    Threshold  float64 // 切换阈值（0.0-1.0）
}

// 评分维度
1. 消息长度（短消息→轻量模型）
2. 复杂度（简单问候→轻量模型）
3. 上下文深度（新话题→轻量模型）
4. 历史模式（类似问题→参考历史选择）

// 决策流程
if score < threshold {
    use LightCandidates  // 轻量模型
} else {
    use Candidates       // 重量级模型
}
```

---

## 五、依赖分析

### 5.1 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/spf13/cobra` | v1.10.2 | CLI 框架 |
| `github.com/rivo/tview` | v0.42.0 | TUI 界面框架 |
| `github.com/gdamore/tcell/v2` | v2.13.8 | 终端控制 |
| `github.com/anthropics/anthropic-sdk-go` | v1.22.1 | Anthropic API |
| `github.com/openai/openai-go/v3` | v3.22.0 | OpenAI API |
| `github.com/modelcontextprotocol/go-sdk` | v1.3.1 | MCP 协议 |
| `modernc.org/sqlite` | v1.46.1 | SQLite 嵌入式数据库 |
| `github.com/rs/zerolog` | v1.34.0 | 结构化日志 |

### 5.2 通道集成依赖

| 依赖 | 用途 |
|------|------|
| `github.com/mymmrac/telego` | Telegram Bot |
| `github.com/bwmarrin/discordgo` | Discord Bot |
| `github.com/tencent-connect/botgo` | QQ Bot |
| `github.com/larksuite/oapi-sdk-go/v3` | 飞书开放平台 |
| `github.com/open-dingtalk/dingtalk-stream-sdk-go` | 钉钉 Stream |
| `github.com/slack-go/slack` | Slack API |
| `maunium.net/go/mautrix` | Matrix 协议 |
| `go.mau.fi/whatsmeow` | WhatsApp 协议 |
| `github.com/ergochat/irc-go` | IRC 协议 |

### 5.3 工具依赖

| 依赖 | 用途 |
|------|------|
| `github.com/gomarkdown/markdown` | Markdown 渲染 |
| `github.com/h2non/filetype` | 文件类型检测 |
| `github.com/google/uuid` | UUID 生成 |
| `github.com/gorilla/websocket` | WebSocket |
| `golang.org/x/oauth2` | OAuth2 认证 |
| `github.com/adhocore/gronx` | Cron 表达式解析 |
| `github.com/chzyer/readline` | 交互式输入 |

---

## 六、接口与 API

### 6.1 CLI 命令

| 命令 | 描述 | 关键参数 |
|------|------|----------|
| `picoclaw agent` | 与代理交互 | `-m, --message` 非交互模式 |
| `picoclaw auth` | 认证管理 | `login`, `logout`, `status` |
| `picoclaw gateway` | 启动网关 | `--port`, `--host` |
| `picoclaw cron` | 定时任务 | `list`, `add`, `remove`, `enable`, `disable` |
| `picoclaw skills` | 技能管理 | `list`, `install`, `remove`, `search`, `show` |
| `picoclaw migrate` | 数据迁移 | - |
| `picoclaw onboard` | 初始化向导 | `--defaults` |
| `picoclaw status` | 状态检查 | - |
| `picoclaw version` | 版本信息 | - |

### 6.2 HTTP API (Gateway)

| 端点 | 方法 | 描述 |
|------|------|------|
| `/health` | GET | 健康检查 |
| `/webhook/{channel}` | POST | 通道 Webhook 回调 |
| `/v1/chat/completions` | POST | OpenAI 兼容 API |

### 6.3 内置聊天命令

| 命令 | 功能 |
|------|------|
| `/help` | 显示帮助信息 |
| `/switch <agent>` | 切换到指定代理 |
| `/list` | 列出可用代理 |
| `/start` | 开始新会话 |
| `/clear` | 清除当前上下文 |
| `/check` | 检查配置状态 |

---

## 七、配置系统

### 7.1 配置文件结构

```json
{
  "agents": {
    "defaults": { /* 代理默认配置 */ },
    "list": [ /* 多代理定义 */ ]
  },
  "model_list": [ /* 模型列表（支持负载均衡） */ ],
  "channels": { /* 各通道配置 */ },
  "tools": {
    "mcp": { /* MCP 服务器配置 */ },
    "web": { /* 搜索配置 */ },
    "exec": { /* 执行限制 */ }
  },
  "heartbeat": { /* 心跳配置 */ },
  "gateway": { /* 网关配置 */ }
}
```

### 7.2 环境变量支持

| 变量 | 说明 |
|------|------|
| `PICOCLAW_HOME` | 主目录路径 |
| `PICOCLAW_CONFIG` | 配置文件路径 |
| `PICOCLAW_WORKSPACE` | 默认工作区 |
| `DEBUG` | 调试模式 |

---

## 八、构建与部署

### 8.1 Makefile 命令

```bash
# 构建
make build              # 当前平台
make build-all          # 全平台
make build-linux-arm    # ARM32（树莓派 Zero 2 W）
make build-linux-arm64  # ARM64
make build-linux-mipsle # MIPS32 LE

# 安装
make install            # 安装到 ~/.local/bin
make uninstall          # 卸载

# 测试
make test               # 运行测试
make check              # 完整检查 (vet + fmt + test)

# Docker
make docker-build       # 精简版镜像
make docker-build-full  # 完整版镜像
make docker-run         # 运行网关
```

### 8.2 跨平台支持

| 平台 | 架构 | 说明 |
|------|------|------|
| Linux | x86_64 | 标准服务器 |
| Linux | ARM64 | 树莓派 4/5 |
| Linux | ARMv7 | 树莓派 Zero 2 W |
| Linux | MIPS LE | Ingenic X2600 等 |
| Linux | RISC-V | VisionFive 等 |
| macOS | ARM64 | Apple Silicon |
| Windows | x86_64 | 标准桌面 |

---

## 九、测试策略

### 9.1 测试分布

| 包路径 | 测试文件数 | 主要测试内容 |
|--------|-----------|--------------|
| `cmd/picoclaw/internal/*` | 30+ | 命令逻辑测试 |
| `pkg/agent` | 10+ | 代理循环、上下文构建 |
| `pkg/channels` | 15+ | 通道管理、消息处理 |
| `pkg/providers` | 12+ | LLM 提供商接口 |
| `pkg/tools` | 20+ | 工具功能测试 |
| `pkg/config` | 8+ | 配置加载、迁移 |
| `pkg/utils` | 10+ | 工具函数测试 |

### 9.2 测试命令

```bash
# 全部测试
go test ./...

# 带覆盖率
go test -cover ./...

# 详细输出
go test -v ./...

# 特定包测试
go test ./pkg/agent/...
```

---

## 十、设计模式应用

### 10.1 使用的模式

| 模式 | 应用位置 | 说明 |
|------|----------|------|
| **工厂模式** | `providers/factory.go` | 根据配置创建不同 LLM 提供商 |
| **策略模式** | `providers/fallback.go` | 降级策略选择 |
| **注册表模式** | `tools/registry.go` | 工具动态注册/发现 |
| **发布订阅** | `bus/bus.go` | 消息总线解耦 |
| **命令模式** | `commands/*.go` | 内置命令封装 |
| **代理模式** | `channels/manager.go` | 通道统一管理 |
| **Builder模式** | `agent/context.go` | 上下文逐步构建 |

### 10.2 并发设计

```go
// 通道管理器 - 每个通道独立 goroutine
type Manager struct {
    workers map[string]*channelWorker  // 每个通道一个 worker
    // ...
}

// 工具注册表 - 读写锁保护
type ToolRegistry struct {
    tools map[string]*ToolEntry
    mu    sync.RWMutex
}

// 会话存储 - 原子操作
var rrCounter atomic.Uint64  // 轮询计数器
```

---

## 十一、代码质量

### 11.1 Lint 配置

`.golangci.yaml` 启用了 60+ 个 linter：
- `errcheck` - 错误检查
- `gosimple` - 代码简化建议
- `govet` - 标准 vet 检查
- `ineffassign` - 无效赋值检测
- `staticcheck` - 静态分析
- `unused` - 未使用代码检测

### 11.2 代码风格

- 行长度限制：120 字符
- 使用 `gofumpt` 严格格式化
- 使用 `goimports` 管理导入
- 使用 `golines` 自动换行

### 11.3 潜在改进点

1. **测试覆盖**：MCP 发现、模型路由等复杂逻辑测试不足
2. **错误处理**：部分地方使用 `log.Fatalf` 过于激进
3. **配置验证**：缺少统一校验框架
4. **文档**：内部 API 文档需补充
5. **并发安全**：部分 map 访问需更严格锁保护

---

## 十二、项目亮点

### 12.1 技术创新

1. **AI 自举架构**：代码库通过 AI 代理自身驱动的架构迁移完成
2. **极致性能优化**：<10MB 内存占用，99% 比竞品更轻量
3. **MCP 原生支持**：紧跟行业标准协议
4. **BM25 工具发现**：基于相关性动态发现 MCP 工具
5. **智能模型路由**：根据消息复杂度自动切换模型

### 12.2 工程实践

1. **清晰分层**：cmd → internal → pkg，依赖单向
2. **接口抽象**：LLMProvider、Channel、Tool 等接口隔离
3. **依赖注入**：构造函数注入，便于测试
4. **模块化设计**：高内聚低耦合
5. **完善测试**：150+ 测试文件覆盖核心逻辑

---

## 十三、总结

PicoClaw 是一个架构精良、功能丰富的个人 AI 助手项目。它展示了 Go 语言在资源受限场景下的优势，同时保持了代码的可读性和可维护性。

### 核心优势

- **性能极致**：<10MB 内存，支持低端硬件
- **生态丰富**：15+ 消息平台原生支持
- **架构先进**：MCP 协议、技能系统、模型路由
- **工程成熟**：完善的测试、构建、部署流程

### 适用场景

- 个人 AI 助手（低功耗设备）
- 多平台消息聚合处理
- AI 驱动的自动化工作流
- 边缘计算场景

---

*文档生成完成。如需深入分析特定模块，请告知。*
