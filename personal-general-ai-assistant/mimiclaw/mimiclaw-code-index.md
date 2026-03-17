# Mimiclaw Code Index

> Source file navigation and module organization guide

---

## Directory Structure

```
mimiclaw/
├── main/
│   ├── agent/           # ReAct agent loop implementation
│   ├── bus/             # Message bus (FreeRTOS xQueue)
│   ├── channels/        # Communication channels (Telegram, Feishu)
│   ├── proxy/           # HTTP CONNECT / SOCKS5 proxy
│   ├── tools/           # Tool implementations
│   ├── wifi/            # WiFi management
│   ├── mimi_config.h    # Build-time configuration
│   ├── mimi_secrets.h   # Sensitive configuration (gitignored)
│   └── CMakeLists.txt   # ESP-IDF build config
├── partitions.csv       # SPIFFS partition table
├── sdkconfig            # ESP-IDF SDK configuration
└── CMakeLists.txt       # Project root
```

---

## Core Modules

### Agent Layer (`agent/`)

| File | Lines | Purpose |
|------|-------|---------|
| `agent_loop.c` | ~360 | ReAct (Reasoning + Acting) main loop, tool orchestration, session management |
| `agent_loop.h` | ~25 | Public interface for agent lifecycle |
| `context_builder.c` | ~80 | LLM prompt construction (system + history + user message) |
| `context_builder.h` | ~20 | Context building interface |
| `llm_client.c` | ~150 | HTTP client for LLM API (Claude/OpenAI compatible) |
| `llm_client.h` | ~30 | LLM client interface |

**Key Design Pattern**: The agent loop implements the ReAct pattern where each turn consists of:
1. Build context → 2. Call LLM → 3. Parse response → 4. Execute tools → 5. Loop or respond

---

### Message Bus (`bus/`)

| File | Lines | Purpose |
|------|-------|---------|
| `message_bus.c` | ~60 | Dual-queue message bus (inbound/outbound) using FreeRTOS xQueue |
| `message_bus.h` | ~35 | Push/pop interface with timeout support |

**Design Strength**: Deep module - simple 5-function interface hides FreeRTOS queue complexity

---

### Communication Channels (`channels/`)

#### Telegram (`channels/telegram/`)

| File | Lines | Purpose |
|------|-------|---------|
| `telegram_bot.c` | ~560 | Long-polling bot implementation, message deduplication |
| `telegram_bot.h` | ~40 | Public interface for Telegram channel |

**Key Features**:
- FNV1a64-based message deduplication (64-entry cache)
- Update offset persistence to NVS (throttled writes)
- Markdown/Plain text fallback on send failure

#### Feishu (`channels/feishu/`)

| File | Lines | Purpose |
|------|-------|---------|
| `feishu_bot.c` | ~990 | WebSocket event stream implementation, protobuf parsing |
| `feishu_bot.h` | ~50 | Public interface for Feishu channel |

**Key Features**:
- Protocol Buffer varint parser for WebSocket frames
- Tenant token management with automatic refresh
- Event-driven architecture with WebSocket lifecycle management

---

### Network Proxy (`proxy/`)

| File | Lines | Purpose |
|------|-------|---------|
| `http_proxy.c` | ~350 | HTTP CONNECT and SOCKS5 tunneling, TLS over proxy |
| `http_proxy.h` | ~45 | Connection abstraction (`proxy_conn_t`) |

**Design Strength**: Deep module - callers use `proxy_conn_open()` without knowing proxy type

**Key Abstractions**:
```c
proxy_conn_t *proxy_conn_open(const char *host, int port, uint32_t timeout_ms);
int proxy_conn_write(proxy_conn_t *conn, const char *data, int len);
int proxy_conn_read(proxy_conn_t *conn, char *buf, int len, uint32_t timeout_ms);
void proxy_conn_close(proxy_conn_t *conn);
```

---

### Tool System (`tools/`)

#### Registry

| File | Lines | Purpose |
|------|-------|---------|
| `tool_registry.c` | ~240 | Central tool registration, JSON schema generation |
| `tool_registry.h` | ~40 | Tool registration interface |

**Design Pattern**: Static registration with JSON schema caching. Tools self-describe via schema.

#### Individual Tools

| File | Lines | Purpose | Complexity |
|------|-------|---------|------------|
| `tool_web_search.c` | ~530 | Tavily/Brave search with proxy support | High |
| `tool_get_time.c` | ~180 | Fetch time from api.telegram.org | Medium |
| `tool_files.c` | ~280 | SPIFFS read/write/edit/list operations | Medium |
| `tool_cron.c` | ~320 | Cron job scheduling for autonomous tasks | High |
| `tool_gpio.c` | ~195 | GPIO control with policy-based access | Medium |

**Tool Pattern**: Each tool implements:
```c
esp_err_t tool_{name}_execute(const char *input_json, char *output, size_t output_size);
```

---

### WiFi Management (`wifi/`)

| File | Lines | Purpose |
|------|-------|---------|
| `wifi_manager.c` | ~270 | STA mode, auto-reconnect, credential management |
| `wifi_manager.h` | ~50 | WiFi manager interface |

**Key Features**:
- Exponential backoff retry: 1s → 2s → 4s → 8s → ... (capped at 30s)
- Credentials priority: NVS overrides > build-time secrets
- Event-driven state management with FreeRTOS event groups

---

### Configuration

| File | Purpose |
|------|---------|
| `mimi_config.h` | Build-time configuration (timeouts, buffer sizes, feature flags) |
| `mimi_secrets.h` | Sensitive data (tokens, API keys, WiFi credentials) - gitignored |

---

## Module Dependencies

```
┌─────────────────────────────────────────────────────────────────┐
│                        Application Layer                         │
│    ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐   │
│    │   Telegram  │    │   Feishu    │    │   Cron (tools)  │   │
│    │   Channel   │    │   Channel   │    │   Scheduler     │   │
│    └──────┬──────┘    └──────┬──────┘    └────────┬────────┘   │
│           │                  │                     │            │
│           └──────────────────┴─────────────────────┘            │
│                              │                                  │
│                       ┌──────┴──────┐                           │
│                       │ Message Bus │◄─────────────────────┐    │
│                       └──────┬──────┘                      │    │
│                              │                             │    │
│           ┌──────────────────┼──────────────────┐          │    │
│           │                  │                  │          │    │
│    ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐   │    │
│    │ Agent Loop  │◄──►│ Tool Registry│    │   System    │   │    │
│    │  (ReAct)    │    │              │    │   Services  │   │    │
│    └──────┬──────┘    └──────┬──────┘    └─────────────┘   │    │
│           │                  │                            │    │
│           │    ┌─────────────┼─────────────┐               │    │
│           │    │             │             │               │    │
│           │  ┌─▼─┐      ┌────▼───┐    ┌────▼────┐          │    │
│           └──►LLM│      │ Tools  │    │  GPIO   │          │    │
│                │  │      │(8 impl)│    │  Files  │          │    │
│                └──┘      │  Cron  │    │  Time   │          │    │
│                          │  Search│    │  etc    │          │    │
│                          └────┬───┘    └────┬────┘          │    │
│                               │             │               │    │
│                               └─────────────┘               │    │
│                                     │                       │    │
│                              ┌──────┴──────┐                │    │
│                              │  HTTP Proxy │◄───────────────┘    │
│                              │  (tunnel)   │                     │
│                              └──────┬──────┘                     │
│                                     │                            │
│                              ┌──────┴──────┐                     │
│                              │ WiFi Manager│                     │
│                              │  (STA mode) │                     │
│                              └─────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## File Size Distribution

| Size Range | Count | Files |
|------------|-------|-------|
| < 100 lines | 15 | Headers, simple interfaces |
| 100-300 lines | 8 | `wifi_manager.c`, `tool_gpio.c`, `tool_get_time.c`, `tool_files.c` |
| 300-600 lines | 4 | `agent_loop.c`, `telegram_bot.c`, `http_proxy.c`, `tool_web_search.c` |
| > 900 lines | 1 | `feishu_bot.c` (complex WebSocket + protobuf implementation) |

**Total C Source**: ~3,800 lines (excluding headers and configuration)

---

## Key Abstractions by Layer

### Layer 1: Hardware/RTOS
- FreeRTOS tasks, queues, event groups
- ESP-IDF WiFi, HTTP, GPIO APIs
- SPIFFS file system

### Layer 2: System Services
- **WiFi Manager**: Connection abstraction with retry logic
- **HTTP Proxy**: Transparent tunneling (HTTP CONNECT / SOCKS5)
- **Message Bus**: Inter-task communication

### Layer 3: Agent Infrastructure
- **Tool Registry**: Discovery and dispatch
- **LLM Client**: API abstraction
- **Channels**: Multi-platform messaging

### Layer 4: Application
- **Agent Loop**: ReAct orchestration
- **Tools**: Domain-specific capabilities

---

## Navigation Quick Reference

### Finding Implementation Details

| Want to understand... | Look at... |
|-----------------------|------------|
| How messages flow between channels and agent | `bus/message_bus.c`, `agent/agent_loop.c:process_inbound_queue()` |
| How tools are discovered by LLM | `tools/tool_registry.c:build_tools_json()` |
| How proxy tunneling works | `proxy/http_proxy.c:proxy_conn_open()` |
| How message deduplication works | `channels/telegram/telegram_bot.c:fnv1a64()`, `channels/feishu/feishu_bot.c` |
| How WiFi reconnection works | `wifi/wifi_manager.c:event_handler()` |
| How file paths are validated | `tools/tool_files.c` |
| How cron schedules tasks | `tools/tool_cron.c` |

---

## Build System

| File | Purpose |
|------|---------|
| `CMakeLists.txt` (root) | Project declaration, SDK configuration |
| `main/CMakeLists.txt` | Source file list, component dependencies |
| `partitions.csv` | SPIFFS partition sizing |
| `sdkconfig` | ESP-IDF feature flags |

---

> *Code index for mimiclaw ESP32-S3 firmware project*
