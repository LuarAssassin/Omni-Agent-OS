# Mimiclaw Architecture Decision Records

> Key architectural decisions and their rationale for the mimiclaw ESP32-S3 firmware project.

---

## ADR-001: ESP32-S3 as Hardware Platform

**Status**: Accepted

**Background**:
The project needed an embedded platform capable of running LLM API clients, WebSocket connections, and concurrent task scheduling while maintaining low power consumption and cost.

**Decision**:
Select ESP32-S3 (dual-core Xtensa LX7, 240MHz, 512KB SRAM + 8MB PSRAM) as the target hardware platform.

**Reasoning**:
1. **Sufficient compute**: 240MHz dual-core can handle TLS encryption, JSON parsing, and concurrent network operations
2. **PSRAM support**: External 8MB SPI RAM essential for buffering large LLM responses and search results
3. **WiFi integrated**: 2.4GHz 802.11 b/g/n built-in, no external networking chips needed
4. **Mature toolchain**: ESP-IDF v5.x provides robust HTTP/WebSocket/TLS libraries
5. **Cost-effective**: ~$3-5 per module in volume

**Trade-offs**:
- Limited RAM (512KB internal) requires careful stack management
- No hardware FPU impacts floating-point operations
- 2.4GHz-only WiFi may face congestion in dense environments

**Implementation**:
Partition layout designed with 2MB firmware, 6MB SPIFFS for file storage, and NVS for credentials persistence.

---

## ADR-002: FreeRTOS with Dual-Queue Message Bus

**Status**: Accepted

**Background**:
The system requires coordination between multiple concurrent operations: Telegram polling, Feishu WebSocket, cron scheduler, and agent processing. A decoupled communication mechanism was needed.

**Decision**:
Implement a message bus using FreeRTOS xQueue with separate inbound and outbound queues.

**Reasoning**:
1. **Decoupling**: Channels don't know about agent; agent doesn't know about channels
2. **Backpressure**: Fixed-size queues (20 messages each) prevent memory exhaustion
3. **Direction separation**: Inbound/outbound queues simplify flow control
4. **Timeout support**: `xQueueReceive` with timeouts enables non-blocking polling
5. **Battle-tested**: FreeRTOS queues are well-tested in ESP-IDF

**Trade-offs**:
- Queue depth limits must be tuned for expected message rates
- JSON message copying adds overhead (but acceptable for PSRAM availability)

**Implementation**:
`message_bus.c` provides 5-function interface: `init`, `push_inbound`, `pop_inbound`, `push_outbound`, `pop_outbound`. Message size limited to 256 bytes.

---

## ADR-003: ReAct Agent Loop Pattern

**Status**: Accepted

**Background**:
The system needs to process user messages through LLM reasoning and tool execution in a loop until a final response is generated.

**Decision**:
Implement the ReAct (Reasoning + Acting) pattern with an explicit agent loop that orchestrates LLM calls and tool execution.

**Reasoning**:
1. **Proven pattern**: ReAct is well-established for LLM agent systems
2. **Tool integration**: Natural fit for function calling APIs (Anthropic/OpenAI)
3. **Iterative refinement**: Multiple turns allow self-correction
4. **State management**: Session-based context maintains conversation history
5. **Observability**: Each step (think/act/observe) is loggable

**Trade-offs**:
- Infinite loops possible if tool keeps returning errors (mitigated by max turn limit)
- Context window grows with each turn (managed by history truncation)

**Implementation**:
`agent_loop.c` implements the main loop: build context → call LLM → parse response → execute tools → loop or respond. Max 10 turns per session to prevent infinite loops.

---

## ADR-004: Static Tool Registration with JSON Schema

**Status**: Accepted

**Background**:
Tools must be discoverable by the LLM at runtime, requiring a mechanism to expose tool schemas and dispatch to tool implementations.

**Decision**:
Use static tool registration where tools are registered at compile-time with their JSON schemas, and the registry builds a cached JSON array for LLM consumption.

**Reasoning**:
1. **No dynamic loading**: ESP32 doesn't support dynamic library loading
2. **Memory efficiency**: Schema strings stored in flash (RODATA), not RAM
3. **Caching**: JSON tool list built once at init, reused for every LLM call
4. **Type safety**: Function pointer dispatch ensures correct signatures
5. **Simplicity**: 3-function interface (init, get_tools_json, execute)

**Trade-offs**:
- Adding a tool requires modifying `tool_registry.c` (macro-based self-registration explored but rejected for simplicity)
- Schema strings consume flash space (~2KB for 10 tools)

**Implementation**:
`tool_registry.c` manages tool array, builds JSON on init. Each tool provides `execute(input_json, output, output_size)` function.

---

## ADR-005: HTTP CONNECT and SOCKS5 Proxy Support

**Status**: Accepted

**Background**:
Corporate and restricted network environments require proxy tunneling for HTTPS connections. The system must support both HTTP CONNECT and SOCKS5 proxies.

**Decision**:
Implement a proxy abstraction layer that handles HTTP CONNECT and SOCKS5 negotiation, providing transparent TLS tunneling.

**Reasoning**:
1. **Enterprise compatibility**: HTTP CONNECT widely supported by corporate proxies
2. **Flexibility**: SOCKS5 supports UDP (future-proofing) and authentication
3. **Transparency**: Callers use same API regardless of proxy configuration
4. **ESP-IDF integration**: Works with `esp_tls` for certificate validation
5. **Compile-time config**: Proxy settings in `mimi_secrets.h` (not runtime)

**Trade-offs**:
- Each tool must check `http_proxy_is_enabled()` and branch (ideally would be transparent)
- Proxy adds connection latency (~1 RTT for handshake)

**Implementation**:
`http_proxy.c` provides `proxy_conn_t` abstraction with `open/write/read/close` operations. Handles CONNECT handshake and SOCKS5 authentication.

---

## ADR-006: FNV1a64 for Message Deduplication

**Status**: Accepted

**Background**:
Telegram long-polling and Feishu event streams may deliver duplicate messages. The system must deduplicate without storing full message content.

**Decision**:
Use FNV1a64 hash for message ID deduplication with a 64-entry circular buffer.

**Reasoning**:
1. **Fast computation**: FNV1a is simple and efficient on ESP32
2. **Low collision risk**: 64-bit hash space sufficient for message volume
3. **Memory efficient**: 64-entry buffer = 512 bytes vs. KB for string storage
4. **Good distribution**: FNV1a provides uniform hash distribution
5. **Adequate window**: 64 messages covers typical retry windows

**Trade-offs**:
- Hash collision possible (extremely rare with 64-bit)
- Fixed buffer size may miss duplicates outside window

**Implementation**:
`fnv1a64()` function in `telegram_bot.c` and `feishu_bot.c`. Circular buffer stores recent hashes. New messages rejected if hash exists in buffer.

---

## ADR-007: JSONL for Session History Storage

**Status**: Accepted

**Background**:
Conversation history must persist across agent turns but be efficiently appendable and parseable with limited RAM.

**Decision**:
Store session history as JSON Lines (JSONL) format - one JSON object per line, no enclosing array.

**Reasoning**:
1. **Append-only**: New turns simply append a line (no re-serialization)
2. **Streaming parse**: Can read line-by-line without loading full history
3. **Human readable**: Plain text for debugging
4. **Standard format**: JSONL is widely supported
5. **Memory efficient**: Context builder can load only recent N messages

**Trade-offs**:
- No random access (must scan from start for specific turn)
- Slightly larger than binary format (acceptable trade-off)

**Implementation**:
History file stored in SPIFFS. `context_builder.c` loads last `MIMI_AGENT_MAX_HISTORY` lines (default 10) to build context.

---

## ADR-008: SPIFFS for Persistent File Storage

**Status**: Accepted

**Background**:
The system needs persistent storage for user files, agent memory, and session history on the ESP32.

**Decision**:
Use SPIFFS (SPI Flash File System) with a 6MB partition for file operations.

**Reasoning**:
1. **Wear leveling**: SPIFFS handles flash wear leveling automatically
2. **Posix API**: Standard file operations via `fopen/fread/fwrite`
3. **Directory support**: Simple directory structure supported
4. **Integrated**: Built into ESP-IDF, no external dependencies
5. **Adequate capacity**: 6MB sufficient for text files and small configs

**Trade-offs**:
- Not suitable for large files (>100KB fragments)
- No journaling (power loss may corrupt)
- Limited to 256-character pathnames

**Implementation**:
`/spiffs/` prefix for all file paths. `tool_files.c` implements read/write/edit/list operations with path validation.

---

## ADR-009: Dual Channel Support (Telegram + Feishu)

**Status**: Accepted

**Background**:
Users may prefer different messaging platforms. Telegram is popular globally; Feishu (Lark) is common in Chinese enterprises.

**Decision**:
Support both Telegram (HTTP long-polling) and Feishu (WebSocket event stream) as independent channel implementations.

**Reasoning**:
1. **User choice**: Users can enable one or both channels
2. **Protocol diversity**: Demonstrates HTTP and WebSocket patterns
3. **Compile-time flags**: `MIMI_ENABLE_TELEGRAM` and `MIMI_ENABLE_FEISHU` for binary size control
4. **Shared abstraction**: Both push to same message bus
5. **Market coverage**: Covers international and Chinese markets

**Trade-offs**:
- Code duplication between channels (deduplication logic, retry logic)
- Feishu implementation significantly more complex (WebSocket + protobuf)
- Maintenance burden of two channel implementations

**Implementation**:
`channels/telegram/telegram_bot.c` (560 lines) for HTTP polling. `channels/feishu/feishu_bot.c` (990 lines) for WebSocket + protobuf events.

---

## ADR-010: Cron-Based Autonomous Task Scheduling

**Status**: Accepted

**Background**:
The agent should be able to schedule and execute tasks autonomously (e.g., periodic reports, reminders) without user interaction.

**Decision**:
Implement a cron-like scheduler using `esp_timer` for one-shot and recurring task execution.

**Reasoning**:
1. **Familiar syntax**: Users understand cron-like scheduling
2. **Lightweight**: `esp_timer` is already included, minimal overhead
3. **Autonomous operation**: Jobs trigger agent turns independently
4. **Persistent storage**: Jobs survive reboot if stored in NVS (future enhancement)
5. **Flexible intervals**: Supports both "every N seconds" and "at timestamp"

**Trade-offs**:
- No persistence across reboots (current limitation)
- Timer accuracy limited to millisecond range
- Maximum concurrent jobs limited by timer resources

**Implementation**:
`tool_cron.c` provides `cron_add`, `cron_list`, `cron_remove` tools. Jobs inject messages into message bus when fired.

---

## ADR-011: Policy-Based GPIO Access Control

**Status**: Accepted

**Background**:
GPIO pins can control hardware but must be restricted to prevent damage to the ESP32 or connected devices.

**Decision**:
Implement compile-time GPIO allowlist with policy enforcement in `tool_gpio.c`.

**Reasoning**:
1. **Safety**: Prevents accidental use of reserved pins (SPI flash, USB, etc.)
2. **Compile-time**: Policy defined in `mimi_config.h`, no runtime overhead
3. **Clear errors**: Policy violations return helpful hints about allowed pins
4. **Flexible**: Users can customize allowlist for their hardware
5. **Read/write separation**: Separate policies for input vs output

**Trade-offs**:
- Policy check exposed to callers (should be hidden inside tool)
- No runtime policy modification

**Implementation**:
`gpio_policy.c` validates pins against `MIMI_GPIO_ALLOWED_INPUTS` and `MIMI_GPIO_ALLOWED_OUTPUTS` arrays. Returns descriptive error on violation.

---

## ADR-012: Tavily Primary with Brave Fallback for Search

**Status**: Accepted

**Background**:
Web search requires an API provider. Tavily offers AI-optimized results but is newer; Brave Search is established but less AI-focused.

**Decision**:
Use Tavily as primary search provider with Brave Search as fallback when Tavily is unavailable or unconfigured.

**Reasoning**:
1. **Tavily optimized for LLMs**: Returns structured results with AI-generated summaries
2. **Brave fallback**: Established service with generous free tier
3. **Runtime selection**: Chooses based on which API key is configured
4. **Error resilience**: Falls back gracefully on network/API errors
5. **Proxy support**: Both APIs work through HTTP CONNECT/SOCKS5

**Trade-offs**:
- Two different response formats to parse
- Different rate limits to manage
- More code to maintain

**Implementation**:
`tool_web_search.c` tries Tavily first if `MIMI_SECRET_TAVILY_KEY` configured, otherwise Brave Search. Formats results as markdown for LLM consumption.

---

## ADR-013: NVS for Credentials with Build-Time Fallback

**Status**: Accepted

**Background**:
WiFi credentials and API keys need storage. Options: build-time secrets only, runtime configuration, or hybrid approach.

**Decision**:
Use hybrid approach: NVS (Non-Volatile Storage) for WiFi credentials with build-time fallback; build-time secrets for API keys.

**Reasoning**:
1. **WiFi flexibility**: Users can configure WiFi without recompiling
2. **API key security**: Tokens compiled into binary (flash encryption can protect)
3. **Priority**: NVS credentials override build-time for WiFi
4. **Migration path**: WiFi credentials can be updated at runtime
5. **ESP-IDF native**: NVS is designed for exactly this use case

**Trade-offs**:
- API keys require reflash to update (acceptable for embedded device)
- NVS limited to key-value pairs (no complex structures)

**Implementation**:
`wifi_manager.c` checks NVS first, falls back to `MIMI_SECRET_WIFI_SSID/PASSWORD`. API keys use `MIMI_SECRET_*` defines directly.

---

## ADR-014: Context Truncation for History Management

**Status**: Accepted

**Background**:
LLM context windows are limited (e.g., 4K-200K tokens). Conversation history grows indefinitely and must be managed.

**Decision**:
Implement history truncation that keeps only the most recent N turns (default 10) plus system prompt for LLM context.

**Reasoning**:
1. **Fixed overhead**: Bounded context size prevents memory exhaustion
2. **Recency bias**: Recent turns are most relevant for current query
3. **Configurable**: `MIMI_AGENT_MAX_HISTORY` allows tuning
4. **Simple algorithm**: No complex importance scoring needed
5. **Works with JSONL**: Easy to implement line-based truncation

**Trade-offs**:
- Old context lost (may cause confusion in long conversations)
- No summarization of discarded history
- Fixed count not adaptive to message length

**Implementation**:
`context_builder.c` loads last N lines from history file, prepends system prompt, appends current message. Total context sent to LLM.

---

## ADR-015: Telegram Date Header for Time Synchronization

**Status**: Accepted

**Background**:
ESP32 has no RTC battery backup. System time resets on power loss. Cron scheduling requires accurate time.

**Decision**:
Use Telegram Bot API server date header (`Date: Mon, 01 Jan 2024 00:00:00 GMT`) for initial time synchronization.

**Reasoning**:
1. **No additional dependencies**: Uses existing Telegram HTTP connection
2. **Reliable**: Telegram servers are NTP-synchronized
3. **Low overhead**: Single HEAD request on first connection
4. **Sufficient accuracy**: HTTP date has ~1 second precision, adequate for cron
5. **Fallback available**: `get_current_time` tool fetches time explicitly if needed

**Trade-offs**:
- Requires Telegram channel enabled or explicit time fetch
- No timezone handling (all times UTC)
- Drift between syncs (ESP32 oscillator not calibrated)

**Implementation**:
`tool_get_time.c` sends HEAD request to `api.telegram.org`, parses `Date` header, sets system time with `settimeofday()`.

---

## ADR-016: Anthropic Messages API as Primary LLM Interface

**Status**: Accepted

**Background**:
Multiple LLM APIs exist with different formats. The system needs a primary interface that can be adapted to others.

**Decision**:
Use Anthropic Messages API as the primary interface with OpenAI-compatible adapters.

**Reasoning**:
1. **Tool use support**: Native support for function calling
2. **Structured output**: Clean JSON format for tool inputs
3. **Streaming ready**: Messages API supports streaming (future enhancement)
4. **OpenAI compatibility**: Many providers offer OpenAI-compatible endpoints
5. **Claude quality**: Claude models excel at following instructions and tool use

**Trade-offs**:
- Anthropic API has stricter rate limits
- May need adapters for other providers
- API key required (no free tier)

**Implementation**:
`llm_client.c` sends POST to `/v1/messages` with tool schemas. Parses `content` array for text or `tool_use` blocks.

---

## Summary

| ADR | Decision | Impact |
|-----|----------|--------|
| ADR-001 | ESP32-S3 hardware | Platform capabilities and constraints |
| ADR-002 | FreeRTOS message bus | Inter-task communication architecture |
| ADR-003 | ReAct agent loop | Core reasoning and action pattern |
| ADR-004 | Static tool registration | Tool discovery and dispatch mechanism |
| ADR-005 | HTTP CONNECT/SOCKS5 proxy | Enterprise network compatibility |
| ADR-006 | FNV1a64 deduplication | Message deduplication strategy |
| ADR-007 | JSONL history storage | Session persistence format |
| ADR-008 | SPIFFS file system | Persistent storage architecture |
| ADR-009 | Telegram + Feishu channels | Multi-platform communication |
| ADR-010 | Cron task scheduler | Autonomous operation capability |
| ADR-011 | GPIO policy control | Hardware access safety |
| ADR-012 | Tavily/Brave search | Web search capability |
| ADR-013 | NVS credentials | Configuration management |
| ADR-014 | Context truncation | LLM context window management |
| ADR-015 | Telegram time sync | System time initialization |
| ADR-016 | Anthropic Messages API | LLM integration primary interface |

---

> *Architecture decisions documented for mimiclaw ESP32-S3 firmware project*
