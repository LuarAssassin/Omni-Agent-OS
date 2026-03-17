# Codex 依赖清单

> 生成时间: 2026-03-17

---

## 内部 Crate 依赖 (70+)

### 核心层
| Crate | 路径 | 用途 |
|------|------|------|
| `codex-cli` | cli/ | 主入口、子命令分发 |
| `codex-core` | core/ | 引擎、Agent、Config、Tools |
| `codex-protocol` | protocol/ | Op/EventMsg 协议定义 |
| `codex-exec` | exec/ | 执行引擎 |
| `codex-execpolicy` | execpolicy/ | 执行策略 |
| `codex-tui` | tui/ | TUI 界面 |
| `codex-tui-app-server` | tui_app_server/ | App Server |
| `codex-state` | state/ | SQLite 状态存储 |

### 沙箱与安全
| Crate | 路径 | 用途 |
|------|------|------|
| `codex-linux-sandbox` | linux-sandbox/ | Landlock 沙箱 |
| `codex-windows-sandbox` | windows-sandbox-rs/ | Windows Sandbox |
| `codex-process-hardening` | process-hardening/ | 进程加固 |
| `codex-secrets` | secrets/ | 密钥管理 |
| `codex-keyring-store` | keyring-store/ | 系统密钥环 |

### 集成与连接
| Crate | 路径 | 用途 |
|------|------|------|
| `codex-mcp-server` | mcp-server/ | MCP 服务器 |
| `codex-rmcp-client` | rmcp-client/ | RMCP 客户端 |
| `codex-connectors` | connectors/ | ChatGPT 连接器 |
| `codex-backend-client` | backend-client/ | 后端 API |
| `codex-login` | login/ | 认证 |
| `codex-ollama` | ollama/ | Ollama 集成 |
| `codex-lmstudio` | lmstudio/ | LM Studio 集成 |

### 工具与协议
| Crate | 路径 | 用途 |
|------|------|------|
| `codex-skills` | skills/ | 技能系统 |
| `codex-file-search` | file-search/ | 文件搜索 |
| `codex-app-server` | app-server/ | App Server |
| `codex-app-server-protocol` | app-server-protocol/ | 协议类型 |
| `codex-apply-patch` | apply-patch/ | 补丁应用 |
| `codex-hooks` | hooks/ | 钩子系统 |

### 工具函数 (utils/)
| Crate | 用途 |
|------|------|
| `codex-utils-git` | Git 操作 |
| `codex-utils-pty` | PTY 终端 |
| `codex-utils-stream-parser` | 流解析 |
| `codex-utils-absolute-path` | 路径处理 |
| `codex-utils-cache` | 缓存 |
| `codex-utils-pty` | 伪终端 |

---

## 外部 Rust 依赖 (主要)

### CLI 与 TUI
| 依赖 | 版本 | 用途 |
|------|------|------|
| `clap` | 4 | CLI 解析 |
| `clap_complete` | 4 | Shell 补全 |
| `ratatui` | 0.29 | TUI 渲染 |
| `crossterm` | 0.28 | 终端控制 |
| `portable-pty` | 0.9 | 跨平台 PTY |

### 异步与网络
| 依赖 | 版本 | 用途 |
|------|------|------|
| `tokio` | 1 | 异步运行时 |
| `reqwest` | 0.12 | HTTP 客户端 |
| `tokio-tungstenite` | 0.28 | WebSocket |
| `async-channel` | 2.3 | 异步通道 |
| `crossbeam-channel` | 0.5 | 跨线程通道 |

### 序列化与存储
| 依赖 | 版本 | 用途 |
|------|------|------|
| `serde` | 1 | 序列化 |
| `serde_json` | 1 | JSON |
| `toml` | 0.9 | TOML |
| `sqlx` | 0.8 | SQLite |

### MCP 与协议
| 依赖 | 版本 | 用途 |
|------|------|------|
| `rmcp` | 0.15 | MCP 协议 |

### 沙箱与安全
| 依赖 | 版本 | 用途 |
|------|------|------|
| `landlock` | 0.4 | Linux 沙箱 |
| `seccompiler` | 0.5 | Seccomp |
| `keyring` | 3.6 | 密钥环 |

### 工具库
| 依赖 | 版本 | 用途 |
|------|------|------|
| `regex` | 1.12 | 正则 |
| `tree-sitter` | 0.25 | 代码解析 |
| `bm25` | 2.3 | BM25 搜索 |
| `starlark` | 0.13 | 策略脚本 |
| `schemars` | 0.8 | JSON Schema |

---

## pnpm 工作区 (Node.js)

| 包 | 路径 | 用途 |
|------|------|------|
| `codex-monorepo` | 根 | 根 package.json |
| `codex-cli` | codex-cli/ | npm 发布 CLI |
| `@openai/codex-sdk` | sdk/typescript/ | TypeScript SDK |
| `shell-tool-mcp` | shell-tool-mcp/ | Shell MCP |
| `responses-api-proxy` | codex-rs/responses-api-proxy/npm | API 代理 |

---

## Python 依赖

| 依赖 | 用途 |
|------|------|
| `pydantic` | 数据模型 |
| `hatchling` | 构建 |
| `pytest` | 测试 |
| `datamodel-code-generator` | 代码生成 |

---

## 依赖关系图

```
codex-cli
├── codex-tui
│   ├── codex-core
│   ├── codex-protocol
│   └── codex-rmcp-client
├── codex-exec
│   ├── codex-core
│   └── codex-execpolicy
├── codex-core
│   ├── codex-protocol
│   ├── codex-linux-sandbox
│   ├── codex-mcp-server
│   ├── codex-skills
│   ├── codex-state
│   └── ...
├── codex-login
└── codex-state
```

---

## 依赖更新建议

### 稳定依赖（无需频繁更新）
- `clap` - 成熟 CLI 框架
- `serde` - 序列化标准
- `tokio` - 异步运行时

### 需关注更新
- `rmcp` - MCP 协议演进
- `ratatui` - TUI 定制分支 (nornagon)
- `tokio-tungstenite` - OpenAI fork

### 安全相关
- `keyring` - 认证存储
- `landlock` - 沙箱权限
