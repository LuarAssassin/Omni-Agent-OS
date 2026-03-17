# IronClaw 依赖清单

> 完整依赖分析 | 生成时间: 2026-03-17

---

## 核心依赖概览

| 类别 | 数量 | 说明 |
|------|------|------|
| 核心异步运行时 | 3 | Tokio 生态，提供全异步 I/O |
| Web 框架与 HTTP | 8 | Axum + Hyper，构建多通道服务 |
| LLM/AI 集成 | 2 | rig-core 多提供商抽象 |
| 数据库 | 8 | PostgreSQL + libSQL 双后端 |
| WASM 沙箱 | 3 | Wasmtime 组件模型 |
| 安全/加密 | 7 | AES-GCM、Argon2、密钥派生 |
| 序列化/配置 | 6 | Serde、TOML、YAML |
| 终端/CLI | 5 | Clap、Ratatui、Rustyline |
| 工具库 | 15+ | UUID、Chrono、Regex 等 |

---

## 详细依赖列表

### 核心异步运行时

| 名称 | 版本 | 用途 |
|------|------|------|
| tokio | 1.x | 异步运行时，full feature 启用所有组件 |
| tokio-stream | 0.1.x | 流式处理，支持同步原语的流转换 |
| futures | 0.3.x | 标准 Future 扩展与工具函数 |
| async-trait | 0.1.x | 支持 async fn in trait (Rust < 1.75 兼容) |

### Web 框架与 HTTP

| 名称 | 版本 | 用途 |
|------|------|------|
| axum | 0.8.x | Web 框架，ws feature 启用 WebSocket 支持 |
| tower | 0.5.x | 中间件抽象层，服务组合 |
| tower-http | 0.6.x | HTTP 专用中间件 (trace, cors, set-header) |
| hyper | 1.5.x | 底层 HTTP 实现，server/http1/http2 |
| hyper-util | 0.1.x | Hyper 工具集 |
| http-body-util | 0.1.x | HTTP Body 工具 |
| reqwest | 0.12.x | HTTP 客户端，rustls-tls 启用 TLS |
| bytes | 1.x | 零拷贝字节缓冲区 |

### LLM/AI 集成

| 名称 | 版本 | 用途 |
|------|------|------|
| rig-core | 0.30.x | 多提供商 LLM 抽象层 (OpenAI, Anthropic, Ollama 等) |
| aws-sdk-bedrockruntime | 1.x | AWS Bedrock 原生 Converse API (可选 feature) |
| aws-config | 1.x | AWS 凭证与区域配置 |
| aws-smithy-types | 1.x | AWS SDK 共享类型 |

### 数据库 (双后端架构)

| 名称 | 版本 | 用途 |
|------|------|------|
| deadpool-postgres | 0.14.x | PostgreSQL 连接池 |
| tokio-postgres | 0.7.x | 异步 PostgreSQL 驱动 |
| postgres-types | 0.2.x | PostgreSQL 类型映射 |
| refinery | 0.8.x | 数据库迁移管理 |
| tokio-postgres-rustls | 0.13.x | PostgreSQL TLS 支持 |
| pgvector | 0.4.x | PostgreSQL 向量扩展支持 |
| libsql | 0.6.x | libSQL/Turso 嵌入式数据库 (可选) |
| rust_decimal | 1.x | 精确十进制数，数据库友好 |

### WASM 沙箱

| 名称 | 版本 | 用途 |
|------|------|------|
| wasmtime | 28.x | WASM 运行时，component-model 启用组件 |
| wasmtime-wasi | 28.x | WASI 支持 |
| wasmparser | 0.220.x | WASM 二进制验证与解析 |

### 安全与加密

| 名称 | 版本 | 用途 |
|------|------|------|
| aes-gcm | 0.10.x | AES-256-GCM 对称加密 (Secrets 管理) |
| hkdf | 0.12.x | 密钥派生函数 |
| hmac | 0.12.x | HMAC 消息认证 |
| sha2 | 0.10.x | SHA-256 哈希 |
| blake3 | 1.x | 高速哈希 (用于密钥指纹) |
| rand | 0.8.x | 密码学安全随机数 |
| subtle | 2.x | 常量时间比较 (防时序攻击) |
| secrecy | 0.10.x | 敏感值封装，防意外泄露 |
| ed25519-dalek | 2.2.x | Ed25519 签名验证 |
| hex | 0.4.x | 十六进制编解码 |

### 序列化与配置

| 名称 | 版本 | 用途 |
|------|------|------|
| serde | 1.x | 序列化框架 |
| serde_json | 1.x | JSON 序列化 |
| serde_yml | 0.0.12 | YAML 序列化 (SKILL.md frontmatter) |
| toml | 0.8.x | TOML 配置解析 |
| dotenvy | 0.15.x | .env 文件加载 |
| json5 | 0.4.x | JSON5 解析 (OpenClaw 导入，可选) |

### 核心类型

| 名称 | 版本 | 用途 |
|------|------|------|
| uuid | 1.x | UUID v4/v5 生成与序列化 |
| chrono | 0.4.x | 日期时间处理，serde 支持 |
| chrono-tz | 0.10.x | 时区数据库 |
| iana-time-zone | 0.1.x | 系统时区检测 |
| semver | 1.x | 语义化版本解析 |

### 终端与 CLI

| 名称 | 版本 | 用途 |
|------|------|------|
| clap | 4.x | 命令行参数解析，derive + env |
| clap_complete | 4.5.x | Shell 补全生成 |
| crossterm | 0.28.x | 跨平台终端控制 |
| rustyline | 17.x | REPL 行编辑，文件历史 |
| termimad | 0.34.x | Markdown 终端渲染 |

### 文件系统与工具

| 名称 | 版本 | 用途 |
|------|------|------|
| dirs | 6.x | 系统目录路径 (~/.ironclaw) |
| fs4 | 0.6.x | 文件锁与高级文件操作 |
| tar | 0.4.x | TAR 归档处理 |
| flate2 | 1.x | Gzip 压缩 |
| zip | 2.x | ZIP 归档处理 |
| tempfile | 3.x | 临时文件/目录 (dev) |

### 文本处理

| 名称 | 版本 | 用途 |
|------|------|------|
| regex | 1.x | 正则表达式 (模式检测) |
| aho-corasick | 1.x | 多模式字符串匹配 (高性能) |
| base64 | 0.22.x | Base64 编解码 |
| mime_guess | 2.0.x | MIME 类型猜测 |

### 调度与定时

| 名称 | 版本 | 用途 |
|------|------|------|
| cron | 0.13.x | Cron 表达式解析 (Routine 调度) |

### 容器与 Docker

| 名称 | 版本 | 用途 |
|------|------|------|
| bollard | 0.18.x | Docker API 客户端 |

### 文档处理

| 名称 | 版本 | 用途 |
|------|------|------|
| pdf-extract | 0.7.x | PDF 文本提取 |
| html-to-markdown-rs | 2.3.x | HTML 转 Markdown (可选 feature) |
| readabilityrs | 0.1.2 | 文章正文提取 (可选 feature) |

### 日志与追踪

| 名称 | 版本 | 用途 |
|------|------|------|
| tracing | 0.1.x | 结构化日志框架 |
| tracing-subscriber | 0.3.x | 日志订阅器，env-filter + json |

### URL 处理

| 名称 | 版本 | 用途 |
|------|------|------|
| url | 2.x | URL 解析 |
| urlencoding | 2.x | URL 编码/解码 |
| open | 5.x | 系统默认打开 URL |

### 缓存与集合

| 名称 | 版本 | 用途 |
|------|------|------|
| lru | 0.16.x | LRU 缓存实现 |

### 错误处理

| 名称 | 版本 | 用途 |
|------|------|------|
| thiserror | 2.x | 强类型错误派生宏 |
| anyhow | 1.x | 动态错误处理 (主要用于测试/二进制) |

---

## 平台特定依赖

### macOS

| 名称 | 版本 | 用途 |
|------|------|------|
| security-framework | 3.x | macOS Keychain 访问 |

### Linux

| 名称 | 版本 | 用途 |
|------|------|------|
| secret-service | 4.x | GNOME Keyring/KWallet 集成 |
| zbus | 4.x | D-Bus 通信 |

---

## 开发依赖

| 名称 | 版本 | 用途 |
|------|------|------|
| tokio-test | 0.4.x | Tokio 测试工具 |
| tracing-test | 0.2.x | 日志断言测试 |
| tokio-tungstenite | 0.26.x | WebSocket 测试客户端 |
| testcontainers-modules | 0.11.x | PostgreSQL 测试容器 |
| pretty_assertions | 1.x | 美观的测试 diff |
| insta | 1.46.x | 快照测试 |

---

## Feature 标志

| Feature | 默认 | 说明 |
|---------|------|------|
| postgres | ✓ | PostgreSQL 后端支持 |
| libsql | ✓ | libSQL/Turso 后端支持 |
| html-to-markdown | ✓ | HTML 转换工具 |
| bedrock | ✗ | AWS Bedrock 支持 |
| import | ✗ | OpenClaw 配置导入 |
| integration | ✗ | 集成测试 (PostgreSQL) |

---

## 依赖设计哲学分析

### 1. 异步优先

所有 I/O 操作使用 Tokio 异步运行时，避免阻塞线程。数据库驱动、HTTP 客户端、文件系统操作全部异步化。

### 2. 安全第一

- **secrecy**: 防止密钥意外打印到日志
- **subtle**: 常量时间比较防时序攻击
- **blake3**: 高速哈希用于密钥指纹
- **aes-gcm**: 标准认证加密

### 3. 模块化依赖

Feature flags 控制可选功能：
- 数据库后端可单独选择 (postgres/libsql)
- AWS Bedrock 是可选 feature
- HTML 转换可禁用

### 4. 最小依赖原则

- 优先使用标准库功能
- 避免重复功能的 crate (如只选一个 HTTP 客户端)
- dev-dependencies 严格分离

---

*本清单基于 Cargo.toml 生成，总计 60+ 直接依赖*
