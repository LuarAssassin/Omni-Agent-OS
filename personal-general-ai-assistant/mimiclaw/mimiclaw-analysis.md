# Mimiclaw Design Analysis

> Analysis based on John Ousterhout's *A Philosophy of Software Design*

---

## I. Module Depth Analysis

### Deep Modules (Well-Designed)

#### 1. Message Bus (`bus/message_bus.c`)

**Interface Surface**: 5 functions
- `message_bus_init()`, `message_bus_push_inbound()`, `message_bus_pop_inbound()`, `message_bus_push_outbound()`, `message_bus_pop_outbound()`

**Implementation Complexity**: FreeRTOS queue management, dual-queue separation (inbound/outbound), timeout handling, queue creation with size limits

**Depth Assessment**: **Deep Module** ✓

The message bus presents an extremely simple interface (just push/pop operations) while encapsulating the complexity of FreeRTOS xQueue management, including proper queue sizing, timeout parameters, and direction separation. Users don't need to understand FreeRTOS queue internals—they just push messages in and pop them out.

```c
// Interface is trivially simple
esp_err_t message_bus_push_inbound(const char *msg);
esp_err_t message_bus_pop_inbound(char *buf, size_t len, uint32_t timeout_ms);
```

**Key Design Strength**: The common case (sending/receiving a message) requires minimal cognitive load. The complexity of queue management is completely hidden.

---

#### 2. HTTP Proxy (`proxy/http_proxy.c`)

**Interface Surface**: 5 functions
- `http_proxy_init()`, `http_proxy_is_enabled()`, `proxy_conn_open()`, `proxy_conn_write()`, `proxy_conn_read()`, `proxy_conn_close()`

**Implementation Complexity**: HTTP CONNECT handshake, SOCKS5 negotiation, TLS over tunnel, chunked header parsing, state machine for proxy protocol

**Depth Assessment**: **Deep Module** ✓

This is an excellent example of a deep module. The interface abstracts away the entire complexity of proxy tunneling:

```c
// Opening a connection through proxy looks identical to direct connection
proxy_conn_t *conn = proxy_conn_open("api.telegram.org", 443, 10000);
```

Behind this simple call lies:
- Proxy protocol detection (HTTP CONNECT vs SOCKS5)
- TCP connection to proxy server
- Authentication negotiation
- Tunnel establishment
- TLS handshake over the tunnel

**Information Hiding**: Callers have no visibility into whether they're connecting directly or through a proxy. This is proper abstraction—the policy decision (use proxy or not) is hidden, and the mechanism is encapsulated.

---

#### 3. Tool Registry (`tools/tool_registry.c`)

**Interface Surface**: 3 functions
- `tool_registry_init()`, `tool_registry_get_tools_json()`, `tool_registry_execute()`

**Implementation Complexity**: Static tool array management, JSON schema generation, cJSON object construction, tool lookup by name

**Depth Assessment**: **Deep Module** ✓

The registry hides significant complexity:
- 10 tools registered with JSON schemas
- Schema generation from C structures
- JSON caching for efficiency
- Tool dispatch by string name

The interface is beautifully simple:
```c
const char *tool_registry_get_tools_json(void);  // Get all tool schemas for LLM
esp_err_t tool_registry_execute(const char *name, const char *input_json, char *output, size_t output_size);
```

**Key Strength**: Adding a new tool requires zero changes to callers—the registry handles discovery automatically.

---

### Shallow Modules (Needs Improvement)

#### 1. Feishu Bot (`channels/feishu/feishu_bot.c`)

**Interface Surface**: 4 functions with complex interactions
- `feishu_init()`, `feishu_start()`, `feishu_send_reply()`, `feishu_is_connected()`

**Implementation Complexity**: WebSocket management, protobuf frame parsing, tenant token refresh, message deduplication

**Depth Assessment**: **Shallow Module** 🚩

**Problems Identified**:

1. **Implementation Leakage**: Callers must understand the WebSocket event loop pattern
2. **Conjoined Methods**: Understanding initialization requires understanding the entire connection state machine
3. **High Cognitive Load**: The module exposes too many internal concepts:
   - Event stream handling
   - Token refresh timing
   - WebSocket frame parsing
   - Message deduplication strategy (FNV1a64)

**Evidence of Shallowness**:
```c
// init function has complex side effects and dependencies
void feishu_start(void) {
    // Creates FreeRTOS tasks
    // Manages WebSocket lifecycle
    // Handles event parsing
    // Manages token refresh timers
}
```

The module should hide the WebSocket implementation detail. Callers shouldn't care that Feishu uses WebSocket—they should just send/receive messages.

---

#### 2. Telegram Bot (`channels/telegram/telegram_bot.c`)

**Interface Surface**: 4 functions with similar complexity to Feishu

**Depth Assessment**: **Shallow Module** 🚩

**Problems**:
1. **Temporal Decomposition**: The module is organized around the execution flow (poll → parse → handle → send) rather than information hiding
2. **Pass-Through Pattern**: `telegram_send_message()` is essentially a pass-through to HTTP client with minimal added value
3. **Exposed Internals**: Update offset management, message ID deduplication are all visible

**Recommendation**: Consider merging common channel logic into a unified transport abstraction. Both Telegram and Feishu share:
- Message deduplication
- Outbound message queueing
- Connection state management
- JSON message formatting

---

## II. Information Hiding Assessment

### Well-Hidden Information

| Module | Hidden Information | Benefit |
|--------|-------------------|---------|
| `http_proxy.c` | Proxy protocol (HTTP vs SOCKS5), TLS tunnel setup | Callers use uniform API regardless of network configuration |
| `message_bus.c` | FreeRTOS queue handles, buffer sizes | Portability; could swap to different queue implementation |
| `tool_registry.c` | Tool storage format, JSON generation | Tools can be added without changing discovery mechanism |
| `wifi_manager.c` | Retry logic, exponential backoff, event handling | Users just call `wifi_manager_start()` and wait for connection |

### Information Leakage Issues

#### 1. Agent Loop Context Exposure (`agent/context_builder.c`)

The context builder exposes too much about how the LLM prompt is constructed:

```c
// Leaks the internal structure of context
size_t build_context(const agent_turn_t *turn,
                     const char *system_prompt,
                     const char *history_json,
                     const char *user_message,
                     char *out_buf, size_t out_size);
```

Callers must understand the three-part context structure (system + history + message). A deeper design would accept a context specification and handle formatting internally.

#### 2. GPIO Tool Policy Leakage (`tools/tool_gpio.c`)

```c
// Policy check is exposed to caller
if (!gpio_policy_pin_is_allowed(pin)) {
    if (gpio_policy_pin_forbidden_hint(pin, output, output_size)) {
        return ESP_ERR_INVALID_ARG;
    }
}
```

The policy checking should happen inside `gpio_write_execute()`, not be exposed to the caller. This is information leakage about which pins are allowed.

---

## III. Complexity Direction Analysis

### Complexity Pulled Downward ✓

#### HTTP Proxy Connection (`proxy/http_proxy.c:proxy_conn_open()`)

The proxy connection management absorbs significant complexity:
- Proxy type detection
- Handshake protocol differences (CONNECT vs SOCKS5)
- Error handling for various proxy failures
- TLS setup over established tunnel

Callers get a simple: `open → write → read → close` flow.

#### Tool Execution (`tools/tool_registry.c:tool_registry_execute()`)

Complexity absorbed:
- Tool name lookup
- JSON parsing per tool
- Output buffer management
- Error translation

Callers just provide the name and input JSON.

### Complexity Pushed Upward 🚩

#### File Operation Tools (`tools/tool_files.c`)

File operations expose path validation to callers:
```c
// Caller must understand path format requirements
const char *base = MIMI_SPIFFS_BASE;  // Implementation detail leaked
```

The `read_file` tool requires paths starting with `/spiffs/`. This complexity should be handled internally—if a user provides `memory/foo.txt`, the tool should prepend the base path automatically.

#### Cron Tool (`tools/tool_cron.c`)

The cron tool exposes the internal job structure:
```c
// Caller must understand job_id format (8-character hex)
tool_cron_remove_execute() requires parsing job_id strings
```

This is a case where complexity could be better hidden. Job IDs could be opaque handles rather than formatted strings.

---

## IV. Error Handling: "Define Errors Out of Existence"

### Well-Handled: WiFi Manager

The WiFi manager implements excellent error masking:

```c
// Disconnection is not an error—it's handled internally
static void event_handler(...) {
    if (event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_count < MIMI_WIFI_MAX_RETRY) {
            // Auto-retry with backoff—no error propagated
            vTaskDelay(pdMS_TO_TICKS(delay_ms));
            esp_wifi_connect();
            s_retry_count++;
        }
    }
}
```

Transient failures (disconnects) are handled at the low level. Only permanent failure after all retries becomes visible.

### Needs Improvement: Tool Execution

Tool execution could apply "define errors out of existence" better:

```c
// Current: Returns error for unknown tool
if (err != ESP_OK) {
    snprintf(output, output_size, "Error: unknown tool '%s'", name);
    return ESP_ERR_NOT_FOUND;
}

// Better: Return a "noop" result that the agent can handle gracefully
```

### Web Search Error Handling

The web search tool has reasonable error masking:
- Missing API key → Returns informative message in output (not error code)
- Network failure → Retries handled at lower level
- JSON parse failure → Returns "Error: Failed to parse" message

However, it still propagates `ESP_FAIL` which callers must handle. Better would be to always return a valid result string explaining what happened.

---

## V. Red Flags Identified

### 1. Pass-Through Methods

**Location**: `channels/telegram/telegram_bot.c:telegram_send_message()`

```c
static esp_err_t telegram_send_message(const char *chat_id, const char *text, ...) {
    // ... setup config ...
    esp_http_client_handle_t client = esp_http_client_init(&config);
    // ... perform request ...
    esp_http_client_cleanup(client);
    return err;
}
```

This method primarily forwards parameters to `esp_http_client`. It adds retry logic, but the abstraction is thin. The channel should expose higher-level operations like `send_reply()` that hide the HTTP details entirely.

### 2. Temporal Decomposition

**Location**: `agent/agent_loop.c`

The agent loop is organized around the ReAct execution flow:
```c
agent_loop_init()
agent_loop_start()
agent_loop_run_once()  // Think → Act → Observe
```

While this follows the ReAct pattern, it exposes the temporal structure. A deeper design might expose:
```c
agent_process_message(msg)  // Handles full turn internally
```

### 3. Repetition

**Pattern**: Proxy HTTP requests repeated in multiple tools

Both `tool_web_search.c` and `tool_get_time.c` implement nearly identical proxy request logic:
- Open proxy connection
- Format HTTP headers
- Write request
- Read response
- Parse headers
- Extract body

**Recommendation**: Move this into `http_proxy.c` as a `proxy_http_request()` function.

### 4. Special-General Mixture

**Location**: `tools/tool_registry.c`

The registry mixes general-purpose registration logic with specific tool initialization:
```c
tool_registry_init() {
    // General: registry setup
    s_tool_count = 0;

    // Specific: each tool's init
    tool_web_search_init();
    tool_gpio_init();
    // ... 8 more tool inits ...

    // General: build JSON cache
    build_tools_json();
}
```

Tools should self-register via a macro or constructor pattern rather than the registry knowing about each tool.

---

## VI. Naming Quality Assessment

### Good Names ✓

| Name | Why It's Good |
|------|---------------|
| `proxy_conn_t` | Clear: a connection through a proxy |
| `tool_registry_execute()` | Verb-noun; clear purpose |
| `wifi_manager_wait_connected()` | Precise: waits until connected |
| `http_proxy_is_enabled()` | Clear boolean intent |

### Needs Improvement 🚩

| Name | Issue | Suggestion |
|------|-------|------------|
| `sb` (in `tool_web_search.c`) | Too short, no meaning | `search_buf` |
| `tmp` (ubiquitous) | No information | `response_buf`, `header_buf` |
| `nvs` (local handles) | Too generic | `nvs_handle`, `nvs_cfg` |
| `MIMI_SECRET_SEARCH_KEY` | Ambiguous: which search? | `MIMI_SECRET_BRAVE_KEY` |

---

## VII. Design It Twice Analysis

### Key Design Decisions Worth Revisiting

#### Decision 1: Channel Architecture

**Current**: Separate implementations for Telegram and Feishu
- `channels/telegram/telegram_bot.c` (564 lines)
- `channels/feishu/feishu_bot.c` (992 lines)

**Alternative**: Unified Channel Abstraction
```c
// Single interface for all channels
channel_t *channel_create(const char *type, const channel_config_t *cfg);
channel_send_message(channel_t *ch, const char *msg);
channel_poll_messages(channel_t *ch, char *buf, size_t len);
```

**Trade-offs**:
- Current: Each channel optimized for its protocol
- Alternative: Reduced code duplication, easier to add new channels

**Assessment**: The current design suffers from significant duplication (deduplication logic, retry logic, JSON formatting). A unified abstraction would be deeper.

---

#### Decision 2: Tool Parameter Passing

**Current**: Tools receive JSON string, parse internally
```c
esp_err_t tool_gpio_write_execute(const char *input_json, char *output, size_t output_size);
```

**Alternative**: Structured parameters
```c
typedef struct {
    int pin;
    int state;
} gpio_write_params_t;

esp_err_t tool_gpio_write_execute(const gpio_write_params_t *params, char *output, size_t output_size);
```

**Trade-offs**:
- Current: Flexible, LLM-friendly (outputs JSON directly)
- Alternative: Type safety, compile-time checking, less parsing overhead

**Assessment**: Current design is appropriate given the LLM integration requirement, but the JSON parsing could be centralized.

---

#### Decision 3: Proxy Integration

**Current**: Tools check `http_proxy_is_enabled()` and branch
```c
if (http_proxy_is_enabled()) {
    err = brave_search_via_proxy(query_copy, &sb);
} else {
    err = brave_search_direct(query_copy, &sb);
}
```

**Alternative**: Transparent proxy (preferred approach)
```c
// http_client automatically uses proxy when configured
esp_err_t http_request(const char *url, const char *method, ...);
```

**Trade-offs**:
- Current: Explicit control, visible in code
- Alternative: Zero proxy-awareness in tools, cleaner code

**Assessment**: The alternative would be significantly deeper. The current approach creates shallow modules that must understand proxy existence.

---

## VIII. Overall Assessment

### Strengths

1. **Message Bus**: Excellent deep module, simple interface hiding queue complexity
2. **HTTP Proxy**: Good abstraction for network tunneling
3. **Tool Registry**: Clean discovery mechanism with JSON caching
4. **WiFi Manager**: Proper error masking with exponential backoff
5. **Memory Management**: Consistent use of PSRAM for large buffers

### Areas for Improvement

1. **Channel Abstraction**: Consider unifying Telegram/Feishu behind common interface
2. **Agent Loop**: Could hide more of the ReAct temporal structure
3. **Error Handling**: More "define errors out of existence" patterns
4. **Code Deduplication**: Extract common HTTP logic from tools
5. **Naming**: Several vague variable names (`tmp`, `sb`, `nvs`)

### Architecture Health Score

| Category | Score | Notes |
|----------|-------|-------|
| Module Depth | 7/10 | Core modules deep, channels shallow |
| Information Hiding | 7/10 | Generally good, some leakage in GPIO |
| Complexity Direction | 6/10 | Some complexity pushed upward |
| Error Handling | 7/10 | Good masking in WiFi, could improve elsewhere |
| Naming | 6/10 | Mixed quality |
| Comments | 7/10 | Good high-level comments, some gaps |
| **Overall** | **6.7/10** | Solid embedded design with room for refinement |

---

## IX. Priority Recommendations

### High Priority

1. **Refactor channel implementations** to share common deduplication and retry logic
2. **Extract HTTP request helper** that handles both direct and proxy modes transparently
3. **Improve variable naming** in `tool_web_search.c` (avoid `sb`, `tmp`)

### Medium Priority

4. **Hide GPIO policy checks** inside tool execution, not exposed to callers
5. **Consider tool self-registration** to remove explicit registry knowledge of each tool
6. **Add more "define errors out"** semantics (e.g., cron_remove succeeds if job doesn't exist)

### Low Priority

7. Unify file path handling (auto-prepend `/spiffs/` base)
8. Extract common protobuf utilities (Feishu varint parser could be general utility)
9. Document temporal assumptions in agent loop (how long should a turn take?)

---

> *Analysis completed based on mimiclaw codebase review*
> *Evaluated against principles from "A Philosophy of Software Design" by John Ousterhout*
