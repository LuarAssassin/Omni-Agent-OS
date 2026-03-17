# IronClaw 代码索引

> 关键文件快速导航 | 按功能区域组织

---

## 导航速查

| 功能区域 | 文件数 | 说明 |
|---------|-------|------|
| [Agent 核心](#agent-核心) | 20+ | 消息路由、调度、会话管理 |
| [通道系统](#通道系统) | 25+ | 多通道输入 (REPL/HTTP/Web/WASM) |
| [工具系统](#工具系统) | 30+ | 内置工具、WASM、MCP |
| [数据库层](#数据库层) | 15+ | 双后端抽象 (PostgreSQL + libSQL) |
| [LLM 层](#llm-层) | 10+ | 多提供商抽象 |
| [安全层](#安全层) | 8 | 纵深防御体系 |
| [配置系统](#配置系统) | 12+ | 多源配置管理 |
| [工作空间](#工作空间) | 8+ | 持久记忆系统 |
| [沙箱系统](#沙箱系统) | 10+ | Docker + WASM 隔离 |
| [基础设施](#基础设施) | 15+ | 错误处理、日志、工具函数 |

---

## Agent 核心

> 位于 `src/agent/`

### 核心循环引擎

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `agent_loop.rs` | 主 Agent 循环 (Legacy) | `agent_loop()`, `TurnResult` |
| `agentic_loop.rs` | 通用 Agentic 循环引擎 | `run_agentic_loop()`, `LoopDelegate` |
| `agent.rs` | Agent 主结构体 | `Agent`, `AgentConfig` |
| `types.rs` | Agent 核心类型 | `Message`, `Role`, `Conversation` |

### 调度与执行

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `dispatcher.rs` | 消息分发器 | `Dispatcher`, `DispatchResult` |
| `scheduler.rs` | 任务调度器 | `Scheduler`, `JobHandle`, `SubtaskHandle` |
| `session.rs` | 会话管理 | `SessionManager`, `Session` |
| `reasoning.rs` | LLM 推理协调 | `ReasoningEngine`, `ReasonContext` |
| `turn.rs` | 回合管理 | `TurnManager`, `TurnState` |

### 专用 Delegates

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `chat.rs` | 聊天模式 Delegate | `ChatDelegate` |
| `job.rs` | 后台任务 Delegate | `JobDelegate` |
| `container.rs` | 容器模式 Delegate | `ContainerDelegate` |

### 自我修复

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `stuck_detector.rs` | 卡死检测 | `StuckDetector`, `StuckReason` |
| `self_repair.rs` | 自我修复逻辑 | `SelfRepair`, `RecoveryStrategy` |
| `heartbeat.rs` | 心跳系统 | `Heartbeat`, `HeartbeatConfig` |

---

## 通道系统

> 位于 `src/channels/`

### 核心抽象

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `channel.rs` | Channel trait 定义 | `Channel`, `IncomingMessage`, `OutgoingResponse` |
| `manager.rs` | 通道管理器 | `ChannelManager`, `MultiChannelStream` |

### CLI/REPL 通道

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `cli/mod.rs` | TUI 主模块 | `CliChannel`, `start_tui()` |
| `cli/ui.rs` | Ratatui UI 组件 | `ChatUI`, `render_frame()` |
| `cli/events.rs` | 事件处理 | `EventHandler`, `InputEvent` |
| `repl.rs` | 简单 REPL | `ReplChannel` |

### Web 通道

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `web/mod.rs` | Web 网关主模块 | `WebChannel`, `start_web_gateway()` |
| `web/server.rs` | Axum 服务器 | `create_app()`, `run_server()` |
| `web/routes.rs` | HTTP 路由 | `chat_routes()`, `api_routes()` |
| `web/sse.rs` | SSE 流支持 | `SseStream`, `broadcast_event()` |
| `web/static/` | 静态文件 (React UI) | - |

### HTTP/Webhook 通道

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `http.rs` | HTTP Webhook | `HttpChannel`, `WebhookHandler` |
| `webhook_server.rs` | 统一 Webhook 服务器 | `WebhookServer`, `register_routes()` |

### WASM 通道

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `wasm/mod.rs` | WASM 通道主模块 | `WasmChannel`, `WasmChannelSetup` |
| `wasm/runtime.rs` | WASM 运行时 | `WasmRuntime`, `execute_channel()` |
| `wasm/wrapper.rs` | Channel trait 包装 | `WasmChannelWrapper` |
| `wasm/setup.rs` | 通道初始化 | `setup_wasm_channels()` |
| `wasm/capabilities.rs` | 通道能力定义 | `ChannelCapabilities` |
| `wasm/bundled.rs` | 捆绑通道发现 | `discover_bundled_channels()` |

---

## 工具系统

> 位于 `src/tools/`

### 核心抽象

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `tool.rs` | Tool trait 定义 | `Tool`, `ToolOutput`, `ToolError` |
| `registry.rs` | 工具注册表 | `ToolRegistry`, `ToolMetadata` |
| `rate_limiter.rs` | 速率限制器 | `SlidingWindowRateLimiter` |

### 内置工具

| 文件 | 描述 | 工具名称 |
|------|------|---------|
| `builtin/echo.rs` | 回显工具 | `echo` |
| `builtin/time.rs` | 时间工具 | `time`, `datetime` |
| `builtin/json.rs` | JSON 处理 | `json_parse`, `json_query` |
| `builtin/http.rs` | HTTP 请求 | `http_get`, `http_post` |
| `builtin/web_fetch.rs` | 网页获取 | `web_fetch` |
| `builtin/file.rs` | 文件操作 | `file_read`, `file_write` |
| `builtin/shell.rs` | Shell 执行 | `shell` |
| `builtin/memory.rs` | 记忆操作 | `memory_search`, `memory_write` |
| `builtin/message.rs` | 消息发送 | `message_user` |
| `builtin/job.rs` | 任务管理 | `job_submit`, `job_status` |
| `builtin/routine.rs` | 例行任务 | `routine_schedule` |
| `builtin/extension_tools.rs` | 扩展工具 | `extension_install` |
| `builtin/skill_tools.rs` | 技能工具 | `skill_list`, `skill_search` |
| `builtin/secrets_tools.rs` | 密钥工具 | `secret_get`, `secret_set` |

### WASM 沙箱

> 位于 `src/tools/wasm/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `runtime.rs` | WASM 运行时 | `WasmRuntime`, `CompiledModule` |
| `wrapper.rs` | Tool trait 包装 | `WasmTool` |
| `host.rs` | Host 函数 | `host_functions()`, `log_fn()` |
| `limits.rs` | 资源限制 | `FuelLimiter`, `MemoryLimiter` |
| `allowlist.rs` | 网络白名单 | `EndpointAllowlist`, `is_allowed()` |
| `credential_injector.rs` | 凭证注入 | `inject_credentials()` |
| `loader.rs` | 模块加载 | `load_wasm_tools()`, `discover_tools()` |
| `rate_limiter.rs` | 工具级限流 | `WasmRateLimiter` |
| `storage.rs` | 线性内存持久化 | `MemoryStorage` |
| `error.rs` | WASM 错误类型 | `WasmError` |

### MCP (Model Context Protocol)

> 位于 `src/tools/mcp/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `protocol.rs` | JSON-RPC 协议 | `McpRequest`, `McpResponse` |
| `client.rs` | MCP 客户端 | `McpClient`, `call_tool()` |
| `factory.rs` | 客户端工厂 | `create_client_from_config()` |
| `session.rs` | 会话管理 | `McpSession`, `session_id` |

### 工具构建

> 位于 `src/tools/builder/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `core.rs` | 构建核心 | `BuildRequirement`, `SoftwareType` |
| `templates.rs` | 项目模板 | `ProjectTemplate`, `render_template()` |
| `testing.rs` | 测试集成 | `TestHarness` |
| `validation.rs` | WASM 验证 | `validate_wasm()` |

---

## 数据库层

> 位于 `src/db/`

### 核心抽象

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | Database trait | `Database` (+ 7 子 trait) |
| `error.rs` | 数据库错误 | `DatabaseError` |

### 子 Trait 定义

| Trait | 所在文件 | 用途 |
|-------|---------|------|
| `ConversationStore` | `stores/conversation.rs` | 对话存储 |
| `JobStore` | `stores/job.rs` | 任务存储 |
| `SandboxStore` | `stores/sandbox.rs` | 沙箱存储 |
| `RoutineStore` | `stores/routine.rs` | 例行任务存储 |
| `ToolFailureStore` | `stores/tool_failure.rs` | 工具失败记录 |
| `SettingsStore` | `stores/settings.rs` | 用户设置存储 |
| `WorkspaceStore` | `stores/workspace.rs` | 工作空间存储 |

### PostgreSQL 后端

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `postgres/mod.rs` | PostgreSQL 主模块 | `PostgresBackend` |
| `postgres/pool.rs` | 连接池 | `create_pool()` |
| `postgres/migrations.rs` | 迁移管理 | `run_migrations()` |

### libSQL 后端

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `libsql/mod.rs` | libSQL 主模块 | `LibSqlBackend` |
| `libsql/local.rs` | 本地数据库 | `LocalLibSql` |
| `libsql/remote.rs` | 远程 Turso | `RemoteLibSql` |

### 迁移文件

> 位于 `migrations/`

| 文件 | 描述 |
|------|------|
| `001_init.sql` | 初始表结构 |
| `002_add_sessions.sql` | 会话表 |
| `003_add_vector_support.sql` | pgvector 支持 |
| `004_add_routines.sql` | 例行任务表 |

---

## LLM 层

> 位于 `src/llm/`

### 核心抽象

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | LLM 模块根 | `LlmProvider`, `ChatCompletion` |
| `provider.rs` | Provider trait | `LlmProvider::complete()` |
| `types.rs` | 共享类型 | `Message`, `Role`, `ToolCall` |
| `error.rs` | LLM 错误 | `LlmError` |

### 提供商实现

| 文件 | 描述 | 提供商 |
|------|------|--------|
| `providers/nearai.rs` | NEAR AI | `nearai` |
| `providers/openai.rs` | OpenAI | `openai` |
| `providers/anthropic.rs` | Anthropic | `anthropic` |
| `providers/ollama.rs` | Ollama | `ollama` |
| `providers/bedrock.rs` | AWS Bedrock | `bedrock` |
| `providers/tinfoil.rs` | Tinfoil | `tinfoil` |
| `providers/openai_compatible.rs` | 兼容端点 | `openai_compatible` |

### 装饰器/包装器

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `decorators/retry.rs` | 重试装饰器 | `RetryLlmProvider` |
| `decorators/fallback.rs` | 降级装饰器 | `FallbackLlmProvider` |
| `decorators/cache.rs` | 缓存装饰器 | `CachingLlmProvider` |

### 工厂与配置

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `factory.rs` | 提供商工厂 | `create_provider()` |
| `config.rs` | LLM 配置 | `LlmConfig`, `ProviderConfig` |

---

## 安全层

> 位于 `src/safety/`

### 核心组件

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | SafetyLayer 主结构 | `SafetyLayer`, `SafetyConfig` |
| `sanitizer.rs` | 输入消毒 | `InputSanitizer`, `sanitize()` |
| `validator.rs` | 输入验证 | `InputValidator`, `validate()` |
| `policy.rs` | 策略引擎 | `PolicyEngine`, `PolicyRule` |

### 泄露检测

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `leak_detector.rs` | 密钥泄露检测 | `LeakDetector`, `SecretPattern` |
| `credential_detect.rs` | HTTP 凭证检测 | `CredentialDetector` |
| `patterns.rs` | 检测模式 | `LEAK_PATTERNS`, `API_KEY_PATTERNS` |

---

## 配置系统

> 位于 `src/config/`

### 核心配置

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | Config 主结构 | `Config`, `Config::from_db()` |
| `agent.rs` | Agent 配置 | `AgentConfig` |
| `llm.rs` | LLM 配置 | `LlmConfig` |
| `channels.rs` | 通道配置 | `ChannelsConfig` |
| `database.rs` | 数据库配置 | `DatabaseConfig` |
| `sandbox.rs` | 沙箱配置 | `SandboxConfig` |
| `secrets.rs` | 密钥配置 | `SecretsConfig` |

### 配置加载

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `builder.rs` | 配置构建 | `ConfigBuilder` |
| `helpers.rs` | 配置辅助 | `get_env_or_default()` |

---

## 工作空间

> 位于 `src/workspace/`

### 核心模块

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | Workspace 主结构 | `Workspace`, `WorkspaceConfig` |
| `memory.rs` | 记忆操作 | `MemoryDocument`, `MemoryStore` |
| `search.rs` | 混合搜索 | `hybrid_search()`, `rrf_merge()` |
| `identity.rs` | 身份文件 | `IdentityFile`, `load_identity()` |

### 存储后端

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `storage/fts.rs` | 全文搜索 | `FtsIndex` |
| `storage/vector.rs` | 向量存储 | `VectorIndex` |
| `storage/hybrid.rs` | 混合存储 | `HybridStore` |

---

## 沙箱系统

> 位于 `src/sandbox/`

### Docker 沙箱

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `config.rs` | 沙箱配置 | `SandboxConfig`, `SandboxPolicy` |
| `manager.rs` | 沙箱管理 | `SandboxManager` |
| `container.rs` | 容器运行 | `ContainerRunner` |

### 网络代理

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `proxy/mod.rs` | 代理主模块 | `NetworkProxy` |
| `proxy/allowlist.rs` | 域名白名单 | `DomainAllowlist` |
| `proxy/credential.rs` | 凭证代理 | `CredentialProxy` |
| `proxy/tunnel.rs` | CONNECT 隧道 | `ConnectTunnel` |

---

## 基础设施

### 错误处理

> 位于 `src/error.rs`

| 类型 | 描述 |
|------|------|
| `Error` | 全局错误枚举 |
| `AgentError` | Agent 错误 |
| `ToolError` | 工具错误 |
| `DatabaseError` | 数据库错误 |
| `LlmError` | LLM 错误 |
| `ConfigError` | 配置错误 |

### 上下文管理

> 位于 `src/context/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | 上下文管理 | `JobContext`, `JobState` |
| `manager.rs` | 上下文管理器 | `ContextManager` |

### 密钥管理

> 位于 `src/secrets/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | 密钥管理 | `SecretsStore`, `Secret` |
| `crypto.rs` | AES-256-GCM 加密 | `encrypt()`, `decrypt()` |
| `keychain.rs` | 系统密钥链 | `KeychainStorage` |

### 隧道

> 位于 `src/tunnel/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | Tunnel trait | `Tunnel`, `create_tunnel()` |
| `cloudflare.rs` | Cloudflare Tunnel | `CloudflareTunnel` |
| `ngrok.rs` | Ngrok 隧道 | `NgrokTunnel` |
| `tailscale.rs` | Tailscale | `TailscaleTunnel` |

### 可观测性

> 位于 `src/observability/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | 可观测性接口 | `Observer`, `Event` |
| `log.rs` | 日志后端 | `LogObserver` |
| `noop.rs` | 空实现 | `NoopObserver` |

### 钩子系统

> 位于 `src/hooks/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | Hook trait | `Hook`, `HookPoint` |
| `6 个生命周期点` | BeforeInbound, BeforeToolCall, BeforeOutbound, OnSessionStart, OnSessionEnd, TransformResponse |

---

## 入口与启动

### 主入口

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `main.rs` | 程序入口 | `main()` |
| `lib.rs` | 库根 | 模块声明, `prelude` |
| `app.rs` | 应用启动 | `App::run()`, `setup_channels()` |

### 启动与引导

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `bootstrap.rs` | 目录解析 | `base_dir()`, `load_env()` |
| `boot_screen.rs` | 启动画面 | `print_boot_screen()` |
| `service.rs` | 系统服务 | `install_service()`, `daemonize()` |
| `tracing_fmt.rs` | 日志格式 | `TracingFormatter` |

### CLI

> 位于 `src/cli/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `mod.rs` | CLI 主结构 | `Cli`, `Command` |
| `config.rs` | 配置命令 | `ConfigCommand` |
| `tool.rs` | 工具命令 | `ToolCommand` |
| `registry.rs` | 注册表命令 | `RegistryCommand` |
| `memory.rs` | 记忆命令 | `MemoryCommand` |
| `mcp.rs` | MCP 命令 | `McpCommand` |
| `pairing.rs` | 配对命令 | `PairingCommand` |
| `service.rs` | 服务命令 | `ServiceCommand` |
| `doctor.rs` | 诊断命令 | `DoctorCommand` |
| `status.rs` | 状态命令 | `StatusCommand` |

---

## 扩展与注册表

> 位于 `src/registry/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `manifest.rs` | 扩展清单 | `ExtensionManifest`, `ArtifactSpec` |
| `catalog.rs` | 注册表目录 | `RegistryCatalog`, `load_catalog()` |
| `installer.rs` | 安装器 | `RegistryInstaller`, `install()` |

---

## 编排器 (Orchestrator)

> 位于 `src/orchestrator/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `api.rs` | HTTP API | `create_app()`, `routes()` |
| `auth.rs` | 认证 | `JobTokenStore`, `validate_token()` |
| `job_manager.rs` | 任务管理 | `JobManager`, `ContainerLifecycle` |

---

## Worker

> 位于 `src/worker/`

| 文件 | 描述 | 关键导出 |
|------|------|---------|
| `container.rs` | 容器内工作 | `ContainerDelegate` |
| `job.rs` | 后台任务 | `JobDelegate` |
| `claude_bridge.rs` | Claude Code 桥接 | `ClaudeBridge` |
| `proxy_llm.rs` | LLM 代理 | `ProxyLlmProvider` |

---

## 测试文件

### 集成测试

> 位于 `tests/`

| 文件 | 描述 |
|------|------|
| `integration_test.rs` | 集成测试 |
| `workspace_test.rs` | 工作空间测试 |
| `heartbeat_test.rs` | 心跳测试 |
| `websocket_gateway_test.rs` | WebSocket 测试 |
| `pairing_test.rs` | 配对测试 |

### E2E 测试

> 位于 `tests/e2e/`

| 文件 | 描述 |
|------|------|
| `test_chat.py` | 聊天流程 E2E |
| `test_tools.py` | 工具调用 E2E |
| `test_memory.py` | 记忆系统 E2E |

---

## 代码统计

```
语言         文件数    代码行数    注释行数
Rust         200+      ~177,000    ~25,000
SQL          15        ~800        ~200
TOML         1         ~150        ~50
```

---

## 快速定位指南

### 按功能查找

| 想实现... | 查看... |
|----------|--------|
| 添加新通道 | `src/channels/channel.rs`, `src/channels/wasm/` |
| 添加新工具 | `src/tools/builtin/`, `src/tools/tool.rs` |
| 添加 LLM 提供商 | `src/llm/providers/`, `src/llm/factory.rs` |
| 修改 Agent 循环 | `src/agent/agentic_loop.rs` |
| 修改数据库 | `src/db/mod.rs`, `src/db/stores/` |
| 修改安全策略 | `src/safety/policy.rs`, `src/safety/patterns.rs` |
| 修改配置加载 | `src/config/mod.rs`, `src/config/builder.rs` |

### 按模式查找

| 设计模式 | 示例位置 |
|---------|---------|
| Trait 抽象 | `src/db/mod.rs` (Database), `src/llm/provider.rs` (LlmProvider) |
| 子 trait 分解 | `src/db/mod.rs` (7 个 Store trait) |
| 装饰器模式 | `src/llm/decorators/` |
| 通用引擎 + Delegate | `src/agent/agentic_loop.rs` |
| 状态机 | `src/context/mod.rs` (JobState) |
| 工厂模式 | `src/llm/factory.rs`, `src/config/mod.rs` |

---

*本索引基于 IronClaw v0.17.0 代码库生成*
