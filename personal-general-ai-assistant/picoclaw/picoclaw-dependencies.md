# PicoClaw 依赖清单

> 生成时间: 2026-03-17

---

## 直接依赖 (35个)

### CLI 与 TUI
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/spf13/cobra` | v1.10.2 | CLI 框架，命令解析 |
| `github.com/rivo/tview` | v0.42.0 | TUI 组件库 |
| `github.com/gdamore/tcell/v2` | v2.13.8 | 终端控制库 |
| `github.com/chzyer/readline` | v1.5.1 | 交互式命令行输入 |

### LLM 提供商 SDK
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/anthropics/anthropic-sdk-go` | v1.22.1 | Anthropic Claude API |
| `github.com/openai/openai-go/v3` | v3.22.0 | OpenAI API |
| `github.com/github/copilot-sdk/go` | v0.1.23 | GitHub Copilot API |

### MCP (Model Context Protocol)
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/modelcontextprotocol/go-sdk` | v1.3.1 | MCP 协议实现 |

### 消息通道集成
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/mymmrac/telego` | v1.6.0 | Telegram Bot API |
| `github.com/bwmarrin/discordgo` | v0.29.0 | Discord Bot |
| `github.com/tencent-connect/botgo` | v0.2.1 | QQ Bot 框架 |
| `github.com/larksuite/oapi-sdk-go/v3` | v3.5.3 | 飞书开放平台 SDK |
| `github.com/open-dingtalk/dingtalk-stream-sdk-go` | v0.9.1 | 钉钉 Stream SDK |
| `github.com/slack-go/slack` | v0.17.3 | Slack API |
| `maunium.net/go/mautrix` | v0.26.3 | Matrix 客户端库 |
| `go.mau.fi/whatsmeow` | v0.0.0-20260219150138 | WhatsApp 协议库 |
| `github.com/ergochat/irc-go` | v0.5.0 | IRC 协议库 |

### 数据存储
| 依赖 | 版本 | 用途 |
|------|------|------|
| `modernc.org/sqlite` | v1.46.1 | 纯 Go SQLite 实现 |
| `google.golang.org/protobuf` | v1.36.11 | Protocol Buffers |

### 网络与 WebSocket
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/gorilla/websocket` | v1.5.3 | WebSocket 客户端/服务器 |
| `golang.org/x/oauth2` | v0.35.0 | OAuth2 认证 |
| `golang.org/x/time` | v0.14.0 | 速率限制器 |

### 日志与工具
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/rs/zerolog` | v1.34.0 | 高性能结构化日志 |
| `github.com/caarlos0/env/v11` | v11.3.1 | 环境变量解析 |
| `github.com/google/uuid` | v1.6.0 | UUID 生成 |

### Markdown 与文件处理
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/gomarkdown/markdown` | v0.0.0-20260217112301 | Markdown 渲染 |
| `github.com/h2non/filetype` | v1.1.3 | 文件类型检测 |

### 定时任务与二维码
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/adhocore/gronx` | v1.19.6 | Cron 表达式解析 |
| `github.com/mdp/qrterminal/v3` | v3.2.1 | 终端二维码生成 |

### JSON 处理
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/tidwall/gjson` | v1.18.0 | JSON 快速查询 |
| `github.com/tidwall/sjson` | v1.2.5 | JSON 修改 |

### HTTP 客户端
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/go-resty/resty/v2` | v2.17.1 | HTTP 客户端 |
| `github.com/valyala/fasthttp` | v1.69.0 | 高性能 HTTP |

### 测试
| 依赖 | 版本 | 用途 |
|------|------|------|
| `github.com/stretchr/testify` | v1.11.1 | 测试断言库 |

---

## 主要间接依赖

| 依赖 | 版本 | 说明 |
|------|------|------|
| `github.com/coder/websocket` | v1.8.14 | WebSocket (MCP 使用) |
| `golang.org/x/crypto` | v0.48.0 | 加密算法 |
| `golang.org/x/net` | v0.51.0 | 网络扩展 |
| `golang.org/x/sys` | v0.41.0 | 系统调用 |
| `golang.org/x/term` | v0.40.0 | 终端控制 |
| `golang.org/x/text` | v0.34.0 | 文本处理 |
| `github.com/davecgh/go-spew` | v1.1.1 | 测试输出格式化 |
| `github.com/pmezard/go-difflib` | v1.0.0 | 差异计算 |
| `gopkg.in/yaml.v3` | v3.0.1 | YAML 解析 |
| `filippo.io/edwards25519` | v1.1.1 | 曲线加密 |

---

## 依赖关系图

```
picoclaw
├── CLI层
│   ├── cobra (spf13)
│   ├── tview + tcell (TUI)
│   └── readline
│
├── LLM层
│   ├── anthropic-sdk-go
│   ├── openai-go/v3
│   └── copilot-sdk
│
├── MCP层
│   └── go-sdk (modelcontextprotocol)
│       └── coder/websocket
│
├── 通道层
│   ├── telego (Telegram)
│   ├── discordgo (Discord)
│   ├── botgo (QQ)
│   ├── oapi-sdk-go/v3 (飞书)
│   ├── dingtalk-stream-sdk-go (钉钉)
│   ├── slack-go/slack (Slack)
│   ├── mautrix (Matrix)
│   ├── whatsmeow (WhatsApp)
│   └── irc-go (IRC)
│
├── 存储层
│   ├── sqlite (modernc)
│   └── protobuf
│
├── 网络层
│   ├── gorilla/websocket
│   ├── golang.org/x/oauth2
│   ├── resty/v2
│   └── fasthttp
│
└── 工具层
    ├── zerolog (日志)
    ├── gomarkdown (Markdown)
    ├── gjson/sjson (JSON)
    ├── gronx (Cron)
    └── uuid
```

---

## 依赖更新建议

### 稳定依赖（无需频繁更新）
- `cobra` - 成熟 CLI 框架
- `zerolog` - 日志库稳定
- `sqlite` - 数据库驱动稳定

### 需关注更新
- `whatsmeow` - WhatsApp 协议频繁变更
- `anthropic-sdk-go` - API 更新较快
- `modelcontextprotocol/go-sdk` - MCP 协议演进中
- `oapi-sdk-go` - 飞书 API 持续更新

### 安全相关
- `golang.org/x/crypto` - 加密库需及时更新
- `oauth2` - 认证相关安全更新
