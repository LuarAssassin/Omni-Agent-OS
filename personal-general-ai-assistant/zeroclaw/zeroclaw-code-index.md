# ZeroClaw 代码索引

> 按功能组织的代码文件导航

---

## 核心 Trait 定义

| Trait | 文件路径 | 描述 |
|-------|---------|------|
| `Provider` | `src/providers/traits.rs` | LLM Provider 接口 |
| `Channel` | `src/channels/traits.rs` | 消息通道接口 |
| `Tool` | `src/tools/traits.rs` | 工具接口 |
| `Memory` | `src/memory/traits.rs` | 记忆存储接口 |
| `Sandbox` | `src/security/traits.rs` | 沙盒接口 |
| `RuntimeAdapter` | `src/runtime/traits.rs` | 运行时适配器接口 |
| `Peripheral` | `src/peripherals/traits.rs` | 硬件外设接口 |
| `Observer` | `src/observability/traits.rs` | 可观察性接口 |

---

## Agent 核心

| 功能 | 文件 | 关键结构/函数 |
|------|------|--------------|
| Agent 状态机 | `src/agent/agent.rs` | `Agent`, `AgentBuilder` |
| 主循环 | `src/agent/loop_.rs` | `run()`, `process_message()` |
| 消息分发 | `src/agent/dispatcher.rs` | `Dispatcher` |
| 意图分类 | `src/agent/classifier.rs` | `Classifier` |
| 提示词构建 | `src/agent/prompt.rs` | `PromptBuilder` |
| 记忆加载 | `src/agent/memory_loader.rs` | `MemoryLoader` |

---

## Provider (LLM 接口)

| Provider | 文件 | 特性标志 |
|----------|------|---------|
| Anthropic (Claude) | `src/providers/anthropic.rs` | - |
| OpenAI | `src/providers/openai.rs` | - |
| DeepSeek | `src/providers/deepseek.rs` | - |
| Azure | `src/providers/azure.rs` | - |
| Ollama | `src/providers/ollama.rs` | - |
| Resilient 包装 | `src/providers/resilient.rs` | 重试/熔断/降级 |

---

## Channel (消息通道)

| Channel | 文件 | 特性标志 |
|---------|------|---------|
| Telegram | `src/channels/telegram.rs` | - |
| Discord | `src/channels/discord.rs` | - |
| Slack | `src/channels/slack.rs` | - |
| WhatsApp | `src/channels/whatsapp.rs` | `whatsapp-web` |
| Matrix | `src/channels/matrix.rs` | `channel-matrix` |
| Nostr | `src/channels/nostr.rs` | `channel-nostr` |
| Email | `src/channels/email.rs` | - |
| IRC | `src/channels/irc.rs` | - |
| iMessage | `src/channels/imessage.rs` | - |
| SMS | `src/channels/sms.rs` | - |

---

## Tool (工具)

| 工具 | 文件 | 描述 |
|------|------|------|
| Shell | `src/tools/shell.rs` | 命令执行 |
| File Read | `src/tools/file_read.rs` | 文件读取 |
| File Write | `src/tools/file_write.rs` | 文件写入 |
| File Edit | `src/tools/file_edit.rs` | 文件编辑 |
| Browser Open | `src/tools/browser_open.rs` | 浏览器打开 |
| Browser Delegate | `src/tools/browser_delegate.rs` | 浏览器代理 |
| Memory Store | `src/tools/memory_store.rs` | 记忆存储 |
| Memory Recall | `src/tools/memory_recall.rs` | 记忆检索 |
| Memory Forget | `src/tools/memory_forget.rs` | 记忆删除 |
| Delegate | `src/tools/delegate.rs` | 委派 Agent |
| Swarm | `src/tools/swarm.rs` | Swarm 编排 |
| MCP Client | `src/tools/mcp_client.rs` | MCP 客户端 |
| MCP Tool | `src/tools/mcp_tool.rs` | MCP 工具 |
| Cron Add | `src/tools/cron_add.rs` | 添加定时任务 |
| Cron List | `src/tools/cron_list.rs` | 列出定时任务 |
| Cron Remove | `src/tools/cron_remove.rs` | 删除定时任务 |
| Cron Run | `src/tools/cron_run.rs` | 执行定时任务 |
| HTTP Request | `src/tools/http_request.rs` | HTTP 请求 |
| Web Search | `src/tools/web_search_tool.rs` | 网络搜索 |
| Web Fetch | `src/tools/web_fetch.rs` | 网页获取 |
| Screenshot | `src/tools/screenshot.rs` | 截图 |
| Glob Search | `src/tools/glob_search.rs` | 文件搜索 |
| Content Search | `src/tools/content_search.rs` | 内容搜索 |
| Knowledge | `src/tools/knowledge_tool.rs` | 知识库 |
| Notion | `src/tools/notion_tool.rs` | Notion 集成 |
| Git Operations | `src/tools/git_operations.rs` | Git 操作 |
| Cloud Ops | `src/tools/cloud_ops.rs` | 云操作 |
| LinkedIn | `src/tools/linkedin.rs` | LinkedIn 集成 |
| Microsoft 365 | `src/tools/microsoft365/` | M365 集成 |
| Hardware Board Info | `src/tools/hardware_board_info.rs` | 硬件信息 |
| Hardware Memory | `src/tools/hardware_memory_*.rs` | 硬件内存操作 |
| SOP 工具 | `src/tools/sop_*.rs` | 标准操作流程 |
| Report Templates | `src/tools/report_templates.rs` | 报告模板 |

---

## Memory (记忆后端)

| 后端 | 文件 | 特性标志 |
|------|------|---------|
| SQLite | `src/memory/sqlite.rs` | 默认 |
| PostgreSQL | `src/memory/postgres.rs` | `memory-postgres` |
| Qdrant | `src/memory/qdrant.rs` | `memory-qdrant` |
| Markdown | `src/memory/markdown.rs` | - |
| Lucid | `src/memory/lucid.rs` | - |
| None | `src/memory/none.rs` | - |
| 工厂 | `src/memory/mod.rs:72` | `create_memory()` |

### 记忆相关功能

| 功能 | 文件 |
|------|------|
| Chunker | `src/memory/chunker.rs` |
| Embeddings | `src/memory/embeddings.rs` |
| Vector Merge | `src/memory/vector.rs` |
| Knowledge Graph | `src/memory/knowledge_graph.rs` |
| Hygiene | `src/memory/hygiene.rs` |
| Consolidation | `src/memory/consolidation.rs` |
| Snapshot | `src/memory/snapshot.rs` |
| Response Cache | `src/memory/response_cache.rs` |

---

## Security (安全)

| 组件 | 文件 | 描述 |
|------|------|------|
| SecurityPolicy | `src/security/policy.rs` | 安全策略定义 |
| Sandbox Trait | `src/security/traits.rs` | 沙盒接口 |
| Docker Sandbox | `src/security/docker.rs` | Docker 沙盒 |
| Firejail Sandbox | `src/security/firejail.rs` | Firejail 沙盒 |
| Bubblewrap Sandbox | `src/security/bubblewrap.rs` | Bubblewrap 沙盒 |
| Landlock Sandbox | `src/security/landlock.rs` | Landlock 沙盒 |
| 沙盒检测 | `src/security/detect.rs` | `create_sandbox()` |
| Secret Store | `src/security/secrets.rs` | 加密存储 |
| Pairing | `src/security/pairing.rs` | 设备配对 |
| Audit | `src/security/audit.rs` | 审计日志 |
| Estop | `src/security/estop.rs` | 急停机制 |
| Prompt Guard | `src/security/prompt_guard.rs` | 提示注入防御 |
| Domain Matcher | `src/security/domain_matcher.rs` | 域匹配 |
| Leak Detector | `src/security/leak_detector.rs` | 泄露检测 |
| Nevis IAM | `src/security/nevis.rs` | IAM 集成 |
| IAM Policy | `src/security/iam_policy.rs` | IAM 策略 |
| Workspace Boundary | `src/security/workspace_boundary.rs` | 工作空间边界 |
| Vulnerability | `src/security/vulnerability.rs` | 漏洞检查 |
| Playbook | `src/security/playbook.rs` | 安全手册 |
| OTP | `src/security/otp.rs` | 一次性密码 |

---

## Runtime (运行时)

| 运行时 | 文件 | 描述 |
|--------|------|------|
| Native | `src/runtime/native.rs` | 原生运行时 |
| Docker | `src/runtime/docker.rs` | Docker 运行时 |
| 工厂 | `src/runtime/mod.rs:12` | `create_runtime()` |

---

## Peripheral (硬件外设)

| 外设 | 文件 | 描述 |
|------|------|------|
| STM32 Nucleo F401RE | `src/peripherals/nucleo_f401re.rs` | STM32 开发板 |
| Raspberry Pi GPIO | `src/peripherals/rpi_gpio.rs` | RPi GPIO |
| ESP32 | `src/peripherals/esp32.rs` | ESP32 开发板 |
| Arduino | `src/peripherals/arduino.rs` | Arduino 开发板 |

---

## Observability (可观察性)

| 后端 | 文件 | 特性标志 |
|------|------|---------|
| Log | `src/observability/log.rs` | 默认 |
| Verbose | `src/observability/verbose.rs` | - |
| Prometheus | `src/observability/prometheus.rs` | `observability-prometheus` |
| OpenTelemetry | `src/observability/otel.rs` | `observability-otel` |
| Noop | `src/observability/noop.rs` | - |
| Multi | `src/observability/multi.rs` | 多后端聚合 |
| 工厂 | `src/observability/mod.rs:28` | `create_observer()` |

---

## Gateway (网关)

| 功能 | 文件 | 描述 |
|------|------|------|
| Webhook Server | `src/gateway/mod.rs` | HTTP/WebSocket 服务器 |
| Channel 路由 | `src/gateway/` | 通道消息路由 |

---

## Config (配置)

| 功能 | 文件 | 描述 |
|------|------|------|
| Config 结构 | `src/config/schema.rs` | 50+ 配置项 |
| 工作空间 | `src/config/workspace.rs` | 工作空间管理 |
| Trait | `src/config/traits.rs` | 配置接口 |

---

## Skills (技能)

| 功能 | 文件 | 描述 |
|------|------|------|
| Skill 管理 | `src/skills/mod.rs` | 技能加载/安装/审计 |
| Audit | `src/skills/audit.rs` | 技能安全审计 |

---

## RAG (检索增强)

| 功能 | 文件 | 描述 |
|------|------|------|
| Hardware RAG | `src/rag/mod.rs` | 硬件数据手册检索 |

---

## Hardware Discovery

| 功能 | 文件 | 描述 |
|------|------|------|
| USB 发现 | `src/hardware/discover.rs` | USB 设备枚举 |
| 设备内省 | `src/hardware/introspect.rs` | 设备信息查询 |
| 注册表 | `src/hardware/registry.rs` | 设备注册表 |

---

## 工具函数

| 功能 | 文件 |
|------|------|
| 通用工具 | `src/util.rs` |
| 多模态处理 | `src/multimodal.rs` |

---

## CLI 入口

| 文件 | 描述 |
|------|------|
| `src/main.rs` | CLI 入口 (~98KB) |
| `src/lib.rs` | 库导出 + 命令枚举 |

---

## 子模块组织

```
src/
├── agent/          # Agent 核心
├── approval/       # 审批流程
├── auth/           # 认证
├── channels/       # 消息通道
├── config/         # 配置管理
├── cost/           # 成本追踪
├── cron/           # 定时任务
├── daemon/         # 守护进程
├── doctor/         # 诊断工具
├── gateway/        # 网关服务
├── hands/          # 手部操作
├── hardware/       # 硬件发现
├── health/         # 健康检查
├── heartbeat/      # 心跳
├── hooks/          # Git hooks
├── identity/       # 身份管理
├── integrations/   # 第三方集成
├── memory/         # 记忆系统
├── migration/      # 数据迁移
├── multimodal.rs   # 多模态
├── nodes/          # 节点管理
├── observability/  # 可观察性
├── onboard/        # 初始化向导
├── peripherals/    # 硬件外设
├── providers/      # LLM Provider
├── rag/            # RAG 检索
├── runtime/        # 运行时
├── security/       # 安全子系统
├── service/        # 服务管理
├── skills/         # 技能系统
├── tools/          # 工具层
├── tunnel/         # 隧道
└── util.rs         # 通用工具
```

---

## 关键工厂函数索引

| 函数 | 位置 | 返回值 |
|------|------|--------|
| `create_memory` | `src/memory/mod.rs:72` | `Arc<dyn Memory>` |
| `create_sandbox` | `src/security/detect.rs` | `Box<dyn Sandbox>` |
| `create_runtime` | `src/runtime/mod.rs:12` | `Box<dyn RuntimeAdapter>` |
| `create_observer` | `src/observability/mod.rs:28` | `Box<dyn Observer>` |
| `default_tools` | `src/tools/mod.rs:87` | `Vec<Box<dyn Tool>>` |
| `all_tools` | `src/tools/mod.rs:150` | `Vec<Box<dyn Tool>>` |
| `create_channel` | `src/channels/mod.rs` | `Box<dyn Channel>` |
| `create_provider` | `src/providers/mod.rs` | `Box<dyn Provider>` |

---

## CLI 命令定义

| 命令枚举 | 定义位置 | 子命令 |
|---------|---------|--------|
| `GatewayCommands` | `src/lib.rs:78` | Start, Restart, GetPaircode |
| `ServiceCommands` | `src/lib.rs:144` | Install, Start, Stop, Restart, Status, Uninstall |
| `ChannelCommands` | `src/lib.rs:161` | List, Start, Doctor, Add, Remove, BindTelegram, Send |
| `SkillCommands` | `src/lib.rs:235` | List, Audit, Install, Remove |
| `MigrateCommands` | `src/lib.rs:257` | Openclaw |
| `CronCommands` | `src/lib.rs:272` | List, Add, AddAt, AddEvery, Once, Remove, Update, Pause, Resume |
| `MemoryCommands` | `src/lib.rs:399` | List, Get, Stats, Clear |
| `IntegrationCommands` | `src/lib.rs:438` | Info |
| `HardwareCommands` | `src/lib.rs:449` | Discover, Introspect, Info |
| `PeripheralCommands` | `src/lib.rs:493` | List, Add, Flash, SetupUnoQ, FlashNucleo |

---

*最后更新: 2026-03-17*
