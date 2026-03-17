# Mimiclaw Dependencies

> External dependencies, component relationships, and system requirements

---

## System Dependencies

### Hardware Requirements

| Component | Minimum Spec | Notes |
|-----------|--------------|-------|
| MCU | ESP32-S3 | Dual-core Xtensa LX7, 240MHz |
| SRAM | 512KB | Internal RAM |
| PSRAM | 8MB | External SPI RAM (required for large buffers) |
| Flash | 8MB+ | For firmware and SPIFFS storage |
| WiFi | 2.4GHz 802.11 b/g/n | STA mode only |

### Partition Layout

```
# partitions.csv
# Name,   Type, SubType, Offset,  Size,    Flags
nvs,      data, nvs,     0x9000,  0x6000,
phy_init, data, phy,     0xf000,  0x1000,
factory,  app,  factory, 0x10000, 0x200000,
spiffs,   data, spiffs,  0x210000,0x5F0000,
```

| Partition | Size | Purpose |
|-----------|------|---------|
| nvs | 24KB | Non-volatile storage for credentials, offsets |
| phy_init | 4KB | WiFi PHY initialization data |
| factory | 2MB | Main firmware image |
| spiffs | ~6MB | File system for persistent storage |

---

## ESP-IDF Framework Dependencies

### Required Components

| Component | Version | Purpose |
|-----------|---------|---------|
| `nvs_flash` | v5.x+ | Non-volatile storage API |
| `esp_wifi` | v5.x+ | WiFi STA/AP mode |
| `esp_netif` | v5.x+ | Network interface abstraction |
| `esp_http_client` | v5.x+ | HTTP/HTTPS client |
| `esp_http_server` | v5.x+ | HTTP server (WebSocket support) |
| `esp_https_ota` | v5.x+ | Over-the-air firmware updates |
| `esp_event` | v5.x+ | Event loop and handlers |
| `esp_timer` | v5.x+ | High-resolution timers |
| `esp_websocket_client` | v5.x+ | WebSocket client (Feishu) |
| `esp_driver_gpio` | v5.x+ | GPIO control |
| `esp-tls` | v5.x+ | TLS/SSL abstraction |
| `app_update` | v5.x+ | Application update utilities |

### SDK Configuration (`sdkconfig`)

Key configuration options:

```
# WiFi
CONFIG_ESP_WIFI_STATIC_RX_BUFFER_NUM=10
CONFIG_ESP_WIFI_DYNAMIC_RX_BUFFER_NUM=32
CONFIG_ESP_WIFI_DYNAMIC_TX_BUFFER_NUM=32
CONFIG_ESP_WIFI_TX_BA_WIN=6
CONFIG_ESP_WIFI_RX_BA_WIN=6

# PSRAM
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y

# HTTP Client
CONFIG_ESP_HTTP_CLIENT_ENABLE_HTTPS=y
CONFIG_ESP_HTTP_CLIENT_ENABLE_BASIC_AUTH=y

# WebSocket
CONFIG_ESP_WS_CLIENT_ENABLE_DYNAMIC_BUFFER=y
```

---

## Third-Party Libraries

### Core Libraries

| Library | License | Purpose | Location |
|---------|---------|---------|----------|
| **cJSON** | MIT | JSON parsing/generation | ESP-IDF built-in |
| **esp_crt_bundle** | Apache 2.0 | CA certificate bundle | ESP-IDF built-in |

### Library Usage Details

#### cJSON

Used throughout for JSON operations:
- Tool schema generation (`tool_registry.c`)
- LLM request/response parsing (`llm_client.c`, `agent_loop.c`)
- Channel message formatting (Telegram, Feishu)

```c
// Typical usage pattern
cJSON *obj = cJSON_CreateObject();
cJSON_AddStringToObject(obj, "key", "value");
char *str = cJSON_PrintUnformatted(obj);
cJSON_Delete(obj);
```

Memory considerations:
- cJSON uses heap allocation
- Large JSON responses (search results) require PSRAM
- Always delete objects to prevent leaks

---

## External API Dependencies

### LLM Providers

| Provider | API Type | Endpoint Pattern | Authentication |
|----------|----------|------------------|----------------|
| Anthropic Claude | HTTP POST | `https://api.anthropic.com/v1/messages` | `x-api-key` header |
| OpenAI (compatible) | HTTP POST | `https://api.openai.com/v1/chat/completions` | `Authorization: Bearer` header |

**Request Format**: Anthropic Messages API
```json
{
  "model": "claude-3-opus-20240229",
  "max_tokens": 1024,
  "system": "system prompt...",
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
  ],
  "tools": [...]
}
```

**Response Format**:
```json
{
  "content": [
    {"type": "text", "text": "..."},
    {"type": "tool_use", "name": "...", "input": {...}}
  ]
}
```

### Communication Channels

#### Telegram Bot API

| Aspect | Details |
|--------|---------|
| Protocol | HTTPS long-polling |
| Endpoint | `https://api.telegram.org/bot<token>/getUpdates` |
| Polling interval | 30 seconds (configurable) |
| Timeout | 30 seconds (long-polling) |
| Message format | JSON |
| Rate limits | 30 messages/second |

Key API methods used:
- `getUpdates` - Poll for messages
- `sendMessage` - Send responses

#### Feishu/Lark Open Platform

| Aspect | Details |
|--------|---------|
| Protocol | WebSocket event stream |
| Authentication | Tenant access token |
| Message format | Protocol Buffer + JSON event payload |
| Heartbeat | Client-initiated ping/pong |
| Reconnection | Exponential backoff |

Key API endpoints:
- `https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal` - Token refresh
- `wss://ws.feishu.cn/passport/` - WebSocket connection

### Search APIs

#### Tavily Search (Preferred)

| Aspect | Details |
|--------|---------|
| Endpoint | `https://api.tavily.com/search` |
| Method | POST |
| Authentication | `api_key` in request body |
| Max results | 10 (configurable) |
| Include answer | Yes (LLM-generated summary) |

Request format:
```json
{
  "api_key": "tvly-...",
  "query": "search query",
  "max_results": 5,
  "include_answer": true
}
```

#### Brave Search (Fallback)

| Aspect | Details |
|--------|---------|
| Endpoint | `https://api.search.brave.com/res/v1/web/search` |
| Method | GET |
| Authentication | `X-Subscription-Token` header |
| Max results | 20 (configurable) |

Request format:
```
GET /res/v1/web/search?q=query&count=5
X-Subscription-Token: BSA...
```

### Time Service

| Service | Endpoint | Method |
|---------|----------|--------|
| Telegram Date Header | `api.telegram.org` | HEAD |

Used for initial system time synchronization (ESP32 has no RTC battery).

---

## Proxy Support

### Supported Proxy Types

| Type | Protocol | Authentication | Use Case |
|------|----------|----------------|----------|
| HTTP CONNECT | HTTP/1.1 | Basic, None | Corporate proxies |
| SOCKS5 | SOCKS v5 | Username/Password, None | General proxying |

### Proxy Configuration

Compile-time configuration in `mimi_secrets.h`:

```c
#define MIMI_PROXY_TYPE      MIMI_PROXY_HTTP_CONNECT  // or MIMI_PROXY_SOCKS5
#define MIMI_PROXY_HOST      "proxy.company.com"
#define MIMI_PROXY_PORT      8080
#define MIMI_PROXY_USERNAME  "user"  // optional
#define MIMI_PROXY_PASSWORD  "pass"  // optional
```

### TLS Over Proxy

All HTTPS traffic is tunneled through the proxy:
1. Client connects to proxy
2. HTTP CONNECT or SOCKS5 handshake
3. TLS handshake through tunnel
4. Normal HTTPS request/response

---

## Build System Dependencies

### CMake

| Tool | Minimum Version | Purpose |
|------|-----------------|---------|
| CMake | 3.16 | Build configuration |
| Ninja | 1.10 | Build execution |
| Python | 3.8 | ESP-IDF tools |

### ESP-IDF Toolchain

| Tool | Version | Purpose |
|------|---------|---------|
| `xtensa-esp32s3-elf-gcc` | esp-12.2.0 | C compiler |
| `xtensa-esp32s3-elf-ld` | esp-12.2.0 | Linker |
| `esptool.py` | v4.x | Flash tool |

### Build Commands

```bash
# Setup
. $HOME/esp/esp-idf/export.sh

# Configure (menuconfig)
idf.py menuconfig

# Build
idf.py build

# Flash
idf.py flash

# Monitor
idf.py monitor

# Full cycle
idf.py build flash monitor
```

---

## Dependency Graph

### Module-to-Dependency Mapping

```
┌─────────────────────┬────────────────────────────────────────────────────────┐
│ Module              │ Dependencies                                           │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ agent_loop          │ cJSON, esp_http_client, esp-tls, FreeRTOS, PSRAM      │
│ llm_client          │ esp_http_client, esp-tls, proxy, cJSON                │
│ context_builder     │ cJSON                                                  │
│ message_bus         │ FreeRTOS (xQueue)                                      │
│ telegram_bot        │ esp_http_client, esp-tls, proxy, cJSON, NVS           │
│ feishu_bot          │ esp_websocket_client, esp-tls, proxy, cJSON, NVS      │
│ http_proxy          │ esp-tls, lwip sockets                                  │
│ tool_registry       │ cJSON                                                  │
│ tool_web_search     │ esp_http_client, esp-tls, proxy, cJSON, PSRAM         │
│ tool_get_time       │ esp_http_client, esp-tls, proxy                       │
│ tool_files          │ SPIFFS, vfs                                            │
│ tool_cron           │ esp_timer, FreeRTOS, cJSON                            │
│ tool_gpio           │ esp_driver_gpio                                        │
│ wifi_manager        │ esp_wifi, esp_netif, esp_event, NVS                   │
└─────────────────────┴────────────────────────────────────────────────────────┘
```

### External Service Dependencies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Mimiclaw Firmware                                 │
│                                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │
│  │   LLM API   │    │  Telegram   │    │   Feishu    │    │   Search    │   │
│  │ (Required)  │    │  (Optional) │    │  (Optional) │    │  (Optional) │   │
│  │             │    │             │    │             │    │             │   │
│  │ Anthropic   │    │ api.telegram│    │open.feishu.cn│   │api.tavily   │   │
│  │ OpenAI      │    │    .org     │    │             │    │api.brave    │   │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘   │
│         │                  │                  │                  │          │
│         └──────────────────┴──────────────────┴──────────────────┘          │
│                                      │                                       │
│                              ┌───────┴───────┐                               │
│                              │  HTTP/HTTPS   │                               │
│                              │  WebSocket    │                               │
│                              └───────────────┘                               │
│                                      │                                       │
│                              ┌───────┴───────┐                               │
│                              │  Proxy (Opt)  │                               │
│                              │ HTTP CONNECT  │                               │
│                              │   SOCKS5      │                               │
│                              └───────────────┘                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Optional Features

### Feature Flags

| Feature | Configuration | Dependencies | Size Impact |
|---------|---------------|--------------|-------------|
| Telegram Channel | `MIMI_ENABLE_TELEGRAM` | esp_http_client | ~15KB |
| Feishu Channel | `MIMI_ENABLE_FEISHU` | esp_websocket_client | ~25KB |
| Web Search | `MIMI_ENABLE_SEARCH` | esp_http_client | ~5KB |
| GPIO Tools | `MIMI_ENABLE_GPIO` | esp_driver_gpio | ~3KB |
| Cron Scheduler | `MIMI_ENABLE_CRON` | esp_timer | ~8KB |
| HTTP Proxy | `MIMI_ENABLE_PROXY` | esp-tls | ~10KB |
| OTA Updates | `MIMI_ENABLE_OTA` | esp_https_ota | ~20KB |

### Memory Requirements by Feature

| Feature | Static RAM | PSRAM | Notes |
|---------|------------|-------|-------|
| Base system | 32KB | 64KB | Core agent loop |
| + LLM context | +8KB | +256KB | Context buffer |
| + Web search | +2KB | +128KB | Response buffer |
| + Telegram | +4KB | 0 | Deduplication cache |
| + Feishu | +8KB | +32KB | Event buffering |

---

## Security Considerations

### Credential Storage

| Credential | Storage | Encryption | Notes |
|------------|---------|------------|-------|
| WiFi Password | NVS | None | ESP-IDF default |
| Bot Tokens | Build-time | N/A | In `mimi_secrets.h` |
| API Keys | Build-time | N/A | In `mimi_secrets.h` |
| Proxy Password | Build-time | N/A | In `mimi_secrets.h` |

### TLS Configuration

- CA verification: Enabled (using esp_crt_bundle)
- Certificate pinning: Not implemented
- TLS version: 1.2+
- Cipher suites: ESP-IDF defaults

### Network Security

- No mDNS/Bonjour exposure
- No open ports (outbound connections only)
- WebSocket connections authenticated with tokens

---

## Version Compatibility

### Tested ESP-IDF Versions

| Version | Status | Notes |
|---------|--------|-------|
| v5.0.x | Compatible | Minimum supported |
| v5.1.x | Recommended | Tested, stable |
| v5.2.x | Compatible | Tested |
| v5.3.x | Compatible | Latest features |

### Breaking Changes History

| ESP-IDF Version | Change Required |
|-----------------|-----------------|
| v5.0 | GPIO API changed from `gpio_pad_select_gpio` to `gpio_reset_pin` |
| v5.1 | WebSocket client API updated (buffer handling) |

---

> *Dependencies documented for mimiclaw ESP32-S3 firmware project*
