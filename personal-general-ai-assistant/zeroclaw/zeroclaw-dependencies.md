# ZeroClaw 依赖分析

> 特性标志与依赖关系详解

---

## 直接依赖清单

### 核心运行时

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| tokio | ^1 | 异步运行时 | 是 |
| tokio-util | ^0.7 | Tokio 工具 | 是 |
| async-trait | ^0.1 | async Trait 支持 | 是 |
| futures | ^0.3 | Future 工具 | 是 |

### Web 与 HTTP

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| axum | ^0.8 | Web 框架 | 是 |
| axum-server | ^0.7 | HTTP 服务器 | 是 |
| reqwest | ^0.12 | HTTP 客户端 | 是 |
| tower | ^0.5 | 中间件抽象 | 是 |
| tower-http | ^0.6 | HTTP 中间件 | 是 |
| hyper | ^1 | HTTP 底层 | 是 |
| hyper-util | ^0.1 | HTTP 工具 | 是 |

### 序列化与配置

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| serde | ^1 | 序列化框架 | 是 |
| serde_json | ^1 | JSON 支持 | 是 |
| toml | ^0.8 | TOML 配置 | 是 |
| config | ^0.15 | 配置管理 | 是 |

### CLI 与终端

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| clap | ^4 | CLI 解析 | 是 |
| dialoguer | ^0.11 | 交互式提示 | 是 |
| indicatif | ^0.17 | 进度条 | 是 |
| console | ^0.15 | 终端控制 | 是 |
| colored | ^2 | 颜色输出 | 是 |
| syntect | ^5 | 语法高亮 | 否 |

### 数据库与存储

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| rusqlite | ^0.34 | SQLite | 是 |
| sqlx | ^0.8 | SQL 异步 | postgres 特性 |
| deadpool | ^0.12 | 连接池 | postgres 特性 |

### 安全与加密

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| ring | ^0.17 | 加密原语 | 是 |
| rustls | ^0.23 | TLS | 是 |
| rustls-pemfile | ^2 | PEM 解析 | 是 |
| aes-gcm | ^0.11 | AEAD 加密 | 是 |
| pbkdf2 | ^0.12 | 密钥派生 | 是 |
| sha2 | ^0.10 | 哈希 | 是 |
| hmac | ^0.12 | HMAC | 是 |
| base64 | ^0.22 | Base64 | 是 |
| hex | ^0.4 | Hex 编码 | 是 |

### 日志与追踪

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| tracing | ^0.1 | 结构化日志 | 是 |
| tracing-subscriber | ^0.3 | 日志订阅 | 是 |
| tracing-appender | ^0.2 | 日志追加 | 是 |
| log | ^0.4 | 日志兼容 | 是 |

### 错误处理

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| anyhow | ^1 | 错误处理 | 是 |
| thiserror | ^2 | 错误定义 | 是 |

### 工具与集合

| 依赖 | 版本 | 用途 | 必需 |
|------|------|------|------|
| regex | ^1 | 正则表达式 | 是 |
| lazy_static | ^1 | 延迟初始化 | 是 |
| once_cell | ^1 | 单次初始化 | 是 |
| chrono | ^0.4 | 日期时间 | 是 |
| uuid | ^1 | UUID 生成 | 是 |
| rand | ^0.8 | 随机数 | 是 |
| itertools | ^0.14 | 迭代器工具 | 是 |
| bitflags | ^2 | 位标志 | 是 |

---

## 特性标志详解

### Channel 特性

| 特性 | 依赖 | 描述 |
|------|------|------|
| `channel-matrix` | matrix-sdk | Matrix 协议支持 |
| `channel-nostr` | nostr-sdk | Nostr 协议支持 |
| `whatsapp-web` | whatsapp-web-rs | WhatsApp Web 支持 |

### Memory 特性

| 特性 | 依赖 | 描述 |
|------|------|------|
| `memory-postgres` | sqlx, sqlx-postgres | PostgreSQL 后端 |
| `memory-qdrant` | qdrant-client | Qdrant 向量后端 |

### Sandbox 特性

| 特性 | 依赖 | 描述 |
|------|------|------|
| `sandbox-bubblewrap` | - | Bubblewrap 沙盒 |
| `sandbox-landlock` | landlock | Landlock LSM 沙盒 |

### Observability 特性

| 特性 | 依赖 | 描述 |
|------|------|------|
| `observability-prometheus` | prometheus | Prometheus 指标 |
| `observability-otel` | opentelemetry | OpenTelemetry 追踪 |

### Browser 特性

| 特性 | 依赖 | 描述 |
|------|------|------|
| `browser-native` | headless_chrome | 原生浏览器自动化 |

### RAG 特性

| 特性 | 依赖 | 描述 |
|------|------|------|
| `rag-pdf` | pdf-extract | PDF 文本提取 |

---

## 依赖关系图

```
zeroclaw
├── Core Runtime
│   ├── tokio (async runtime)
│   ├── async-trait
│   └── futures
│
├── Web Stack
│   ├── axum (web framework)
│   ├── axum-server
│   ├── tower (middleware)
│   ├── tower-http
│   ├── hyper (HTTP)
│   └── reqwest (HTTP client)
│
├── Data Layer
│   ├── serde (serialization)
│   ├── serde_json
│   ├── toml (config)
│   ├── rusqlite [default]
│   ├── sqlx [feature: memory-postgres]
│   └── qdrant-client [feature: memory-qdrant]
│
├── Security
│   ├── ring (crypto)
│   ├── aes-gcm (encryption)
│   ├── pbkdf2 (key derivation)
│   ├── rustls (TLS)
│   └── landlock [feature: sandbox-landlock]
│
├── Observability
│   ├── tracing (logging)
│   ├── prometheus [feature: observability-prometheus]
│   └── opentelemetry [feature: observability-otel]
│
└── Channels [optional]
    ├── matrix-sdk [feature: channel-matrix]
    ├── nostr-sdk [feature: channel-nostr]
    └── whatsapp-web-rs [feature: whatsapp-web]
```

---

## 特性组合建议

### 最小构建

```bash
cargo build --release --no-default-features
# 仅包含: SQLite, 无沙盒, Log 可观察性
```

### 推荐构建

```bash
cargo build --release \
  --features "memory-postgres channel-matrix sandbox-bubblewrap observability-prometheus"
```

### 完整构建

```bash
cargo build --release --all-features
# 包含所有可选依赖
```

### 服务器部署

```bash
cargo build --release \
  --features "memory-postgres memory-qdrant sandbox-landlock observability-otel"
```

### 开发构建

```bash
cargo build --release \
  --features "memory-postgres sandbox-bubblewrap browser-native rag-pdf"
```

---

## 依赖大小分析

| 构建类型 | 估计大小 | 说明 |
|---------|---------|------|
| 最小构建 | ~5MB | 仅核心功能 |
| 默认构建 | ~8.8MB | SQLite + 基础功能 |
| 推荐构建 | ~12MB | + Postgres + Prometheus |
| 完整构建 | ~20MB | 所有特性 |

---

## 安全依赖审计

### 加密库

| 库 | 用途 | 审计状态 |
|---|------|---------|
| ring | 底层加密 | 广泛使用，定期审计 |
| rustls | TLS | Rustls 项目维护 |
| aes-gcm | AEAD | RustCrypto 组织 |

### 建议

- 使用 `cargo audit` 定期检查依赖漏洞
- 启用 `dependabot` 自动更新
- 最小化特性标志减少攻击面

---

*最后更新: 2026-03-17*
