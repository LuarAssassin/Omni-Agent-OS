# Nanobot Architecture Analysis

## Executive Summary

**Nanobot** is a Python-based personal AI assistant framework with multi-channel support (Slack, Discord, Telegram, QQ, DingTalk, Feishu, Matrix, WebSocket) and multi-provider LLM integration. The architecture demonstrates **solid modular design** with clear separation of concerns, though several areas exhibit **shallow module** anti-patterns and **information leakage** that increase cognitive load.

**Overall Assessment**: The codebase shows intentional design with strategic use of abstractions, though some complexity has accumulated in the provider matching logic and channel implementations.

---

## I. Architectural Overview

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           USER INTERFACE LAYER                          │
├─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────────┤
│  Slack  │ Discord │Telegram │   QQ    │DingTalk │ Feishu  │  WebSocket  │
│ Channel │ Channel │ Channel │ Channel │ Channel │ Channel │   Channel   │
└────┬────┴────┬────┴────┬────┴────┬────┴────┬────┴────┬────┴──────┬──────┘
     │         │         │         │         │         │           │
     └─────────┴─────────┴─────────┴────┬────┴─────────┴───────────┘
                                        │
                          ┌─────────────┴─────────────┐
                          │      MessageBus (Pub/Sub)  │
                          └─────────────┬─────────────┘
                                        │
     ┌──────────────────────────────────┼──────────────────────────────────┐
     │                                  │                                  │
┌────┴────┐                    ┌───────┴────────┐               ┌────────┴───────┐
│ Agent   │                    │  Conversation  │               │    Session     │
│  Loop   │◄──────────────────►│    Manager     │◄─────────────►│    Store       │
│         │                    │                │               │  (JSONL-based) │
└────┬────┘                    └────────────────┘               └────────────────┘
     │
     │    ┌──────────────────────────────────────────────────────────────┐
     │    │                     TOOL REGISTRY                            │
     │    ├──────────┬──────────┬──────────┬──────────┬──────────────────┤
     │    │  shell   │  file    │   web    │  skill   │  MCP Servers     │
     │    │  exec    │  ops     │  search  │  exec    │  (external)      │
     │    └──────────┴──────────┴──────────┴──────────┴──────────────────┘
     │
     │    ┌──────────────────────────────────────────────────────────────┐
     │    │                   LLM PROVIDER LAYER                         │
     │    ├──────────┬──────────┬──────────┬──────────┬──────────────────┤
     │    │Anthropic │  OpenAI  │DeepSeek  │OpenRouter│  15+ more...     │
     │    │(claude)  │  (gpt)   │          │(gateway) │                  │
     │    └──────────┴──────────┴──────────┴──────────┴──────────────────┘
     │
┌────┴──────────────────────────────────────────────────────────────────┐
│                        SUBAGENT SYSTEM                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│  │  Spawn      │  │   Cancel    │  │   Status    │                   │
│  │  Tasks      │  │   Tasks     │  │   Monitor   │                   │
│  └─────────────┘  └─────────────┘  └─────────────┘                   │
└───────────────────────────────────────────────────────────────────────┘
```

---

## II. Deep Modules Analysis

### 2.1 Deep Modules (Well-Designed)

#### MessageBus (`nanobot/bus/queue.py`)

**Interface Complexity**: LOW - Simple `subscribe()`/`publish()` API
**Implementation Complexity**: MEDIUM - Async queue management, typed events

```python
class MessageBus:
    """Simple pub/sub event bus with typed consumers."""

    def __init__(self):
        self._consumers: dict[type, list[Callable]] = {}

    def subscribe[T](self, event_type: type[T], handler: Callable[[T], Any]) -> None:
        self._consumers.setdefault(event_type, []).append(handler)

    def publish[T](self, event: T) -> None:
        for handler in self._consumers.get(type(event), []):
            handler(event)
```

**Assessment**: **DEEP MODULE** ✅
- Simple interface hides complex async event handling
- Type-safe generic subscriptions
- Zero cognitive load for users - just subscribe and publish

#### BaseChannel (`nanobot/channels/base.py`)

**Interface Complexity**: LOW - Abstract `start()`/`stop()`/`send()` methods
**Implementation Complexity**: HIGH - WebSocket/Socket Mode handling, retry logic, rate limiting

**Assessment**: **DEEP MODULE** ✅
- Clean abstraction for 8+ channel implementations
- Common message handling in base class
- Channel-specific complexity hidden in subclasses

#### ProviderSpec (`nanobot/providers/registry.py`)

**Interface Complexity**: LOW - Dataclass with clear fields
**Implementation Complexity**: MEDIUM - Registry pattern, 20+ provider configurations

```python
@dataclass(frozen=True)
class ProviderSpec:
    name: str
    keywords: tuple[str, ...]
    env_key: str
    is_gateway: bool = False
    is_local: bool = False
    # ... 15+ more fields
```

**Assessment**: **DEEP MODULE** ✅
- Single source of truth for provider metadata
- Frozen dataclass ensures immutability
- Extensible for new providers via simple tuple addition

---

### 2.2 Shallow Modules (Needs Improvement)

#### Config._match_provider (`nanobot/config/schema.py:167-227`)

**Interface Complexity**: MEDIUM - Returns tuple with optional values
**Implementation Complexity**: VERY HIGH - 60+ lines of nested conditions

```python
def _match_provider(self, model: str | None = None) -> tuple["ProviderConfig | None", str | None]:
    """Match provider config and its registry name."""
    # 1. Check explicit provider override
    forced = self.agents.defaults.provider
    if forced != "auto":
        ...

    # 2. Normalize model name
    model_lower = (model or self.agents.defaults.model).lower()
    model_normalized = model_lower.replace("-", "_")
    ...

    # 3. Explicit provider prefix matching
    for spec in PROVIDERS:
        ...

    # 4. Keyword matching
    for spec in PROVIDERS:
        ...

    # 5. Local provider fallback
    local_fallback: tuple[ProviderConfig, str] | None = None
    for spec in PROVIDERS:
        ...

    # 6. Final fallback to any provider with API key
    for spec in PROVIDERS:
        ...
```

**Assessment**: **SHALLOW MODULE** ⚠️
- **Red Flag**: Interface is nearly as complex as implementation
- **Cognitive Load**: 6 different matching strategies must be understood
- **Change Amplification**: Adding a provider requires understanding all matching logic
- **Recommendation**: Extract matching strategies into separate classes with single responsibility

#### SlackChannel._on_socket_request (`nanobot/channels/slack.py:149-241`)

**Interface Complexity**: MEDIUM - SocketModeRequest handler
**Implementation Complexity**: VERY HIGH - 90+ lines of event processing

```python
async def _on_socket_request(self, client: SocketModeClient, req: SocketModeRequest) -> None:
    # 1. Acknowledge request
    # 2. Filter event types
    # 3. Extract sender/chat info
    # 4. Check bot/system messages
    # 5. Avoid double-processing
    # 6. Check permissions
    # 7. Strip mentions
    # 8. Handle threading
    # 9. Add reactions
    # 10. Call message handler
```

**Assessment**: **SHALLOW MODULE** ⚠️
- **Information Leakage**: Knows about event types, permissions, threading, reactions
- **Red Flag**: Single method doing 10 different things
- **Cognitive Load**: Hard to understand what this method actually does
- **Recommendation**: Decompose into smaller private methods or use a request pipeline pattern

---

## III. Information Hiding Assessment

### 3.1 Well-Hidden Information

| Information | Hidden In | Exposed Through |
|-------------|-----------|-----------------|
| LLM provider API keys | `ProviderConfig` | `get_api_key()` - no key details |
| Session storage format | `SessionStore` | `save()`/`load()` methods |
| Channel connection state | `BaseChannel` subclasses | `is_connected` property |
| Tool JSON schemas | `Tool` base class | `get_schema()` method |
| OAuth token refresh | OAuth providers | Transparent to callers |

### 3.2 Information Leakage

#### Session Key Construction

```python
# nanobot/channels/slack.py:224
session_key = f"slack:{chat_id}:{thread_ts}" if thread_ts and channel_type != "im" else None
```

**Leakage**: Session key format exposed to channel implementations.
**Impact**: All channels must know the colon-delimited format.
**Fix**: Move to `SessionManager.create_key(channel, chat_id, thread_id)`

#### Provider Detection Logic

```python
# nanobot/config/schema.py:183-185
def _kw_matches(kw: str) -> bool:
    kw = kw.lower()
    return kw in model_lower or kw.replace("-", "_") in model_normalized
```

**Leakage**: Normalization rules (hyphen to underscore) scattered across methods.
**Impact**: Must update all locations when changing normalization.
**Fix**: Centralize in `ModelName` value object.

---

## IV. Complexity Hotspots

### 4.1 Provider Matching (Cognitive Load: HIGH)

The provider matching system uses **6 different strategies** in priority order:

1. Explicit `provider` config field
2. Model name prefix matching (e.g., `anthropic/claude-3`)
3. Keyword matching in model name
4. Local provider detection by `api_base`
5. Gateway provider fallback with API key
6. Any provider with API key fallback

**Problem**: Unknown Unknowns - Users can't predict which provider will be selected.

**Current Code**:
```python
def _match_provider(self, model: str | None = None) -> tuple["ProviderConfig | None", str | None]:
    """60+ lines of complex matching logic"""
```

**Recommendation**: Replace with explicit `ProviderResolver` class:

```python
class ProviderResolver:
    """Determines which provider should handle a model request."""

    def __init__(self, config: ProvidersConfig):
        self._strategies = [
            ExplicitProviderStrategy(),
            PrefixMatchStrategy(),
            KeywordMatchStrategy(),
            LocalProviderStrategy(),
            GatewayFallbackStrategy(),
        ]

    def resolve(self, model: str) -> tuple[ProviderConfig, str]:
        for strategy in self._strategies:
            if result := strategy.match(model):
                return result
        raise NoProviderError(model)
```

### 4.2 Channel Permission Checking (Cognitive Load: MEDIUM)

Each channel implements its own permission logic:

```python
# SlackChannel._is_allowed() - 12 lines
# QQChannel - no explicit filtering, relies on SDK
# DiscordChannel._is_allowed() - different logic
```

**Problem**: Change Amplification - Policy changes require updating all channels.

**Recommendation**: Extract `PermissionPolicy` abstraction:

```python
class PermissionPolicy(ABC):
    @abstractmethod
    def is_allowed(self, sender: str, chat: str, channel_type: str) -> bool: ...

class AllowlistPolicy(PermissionPolicy): ...
class OpenPolicy(PermissionPolicy): ...
class MentionPolicy(PermissionPolicy): ...
```

### 4.3 Tool Registration (Cognitive Load: LOW)

Tool registration is **well-designed** with clear separation:

```python
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    async def execute(self, **kwargs) -> ToolResult: ...
```

**Assessment**: Deep module with simple interface hiding complex execution logic.

---

## V. Error Handling Analysis

### 5.1 "Define Errors Out of Existence" (Well Done)

#### Session Loading

```python
# nanobot/session/store.py
def load(self, session_id: str) -> Session:
    """Returns empty session if file doesn't exist - no exception."""
    path = self._path(session_id)
    if not path.exists():
        return Session(id=session_id)  # Empty session
    ...
```

**Good**: Missing session is valid state, not error.

#### Tool Execution

```python
# nanobot/tools/base.py
class ToolResult:
    content: str
    is_error: bool = False  # Semantic error, not exception
```

**Good**: Tool failures are return values, not exceptions.

### 5.2 Exception Propagation Issues

#### Channel Message Handling

```python
# nanobot/channels/slack.py:240-241
try:
    await self._handle_message(...)
except Exception:
    logger.exception("Error handling Slack message from {}", sender_id)
```

**Problem**: Bare `except Exception` catches everything including `KeyboardInterrupt`.

**Fix**:
```python
except ChannelError as e:
    logger.error("Channel error: {}", e)
except Exception as e:
    logger.exception("Unexpected error handling message")
    raise  # Let unexpected errors propagate
```

---

## VI. Interface Design Assessment

### 6.1 Good Interfaces

#### MessageBus.subscribe

```python
def subscribe[T](self, event_type: type[T], handler: Callable[[T], Any]) -> None:
```

- Generic type ensures type safety
- Simple two-parameter signature
- No configuration options needed

#### BaseChannel.default_config

```python
@classmethod
def default_config(cls) -> dict[str, Any]:
```

- Class method allows subclass customization
- Returns dict for easy serialization
- Consistent across all channels

### 6.2 Problematic Interfaces

#### MCPServerConfig

```python
class MCPServerConfig(Base):
    type: Literal["stdio", "sse", "streamableHttp"] | None = None
    command: str = ""  # Stdio: command to run
    args: list[str] = Field(default_factory=list)
    env: dict[str, str] = Field(default_factory=dict)
    url: str = ""  # HTTP/SSE: endpoint URL
    headers: dict[str, str] = Field(default_factory=dict)
    tool_timeout: int = 30
    enabled_tools: list[str] = Field(default_factory=lambda: ["*"])
```

**Problems**:
- **Overexposure**: Common case (stdio) requires setting 3+ fields
- **Red Flag**: Optional fields with empty defaults suggest poor cohesion
- **Cognitive Load**: User must understand all transport types

**Better Design**:
```python
class MCPServerConfig(Base):
    """MCP server configuration - use MCPServerConfig.stdio() or .sse() factory methods."""

    @classmethod
    def stdio(cls, command: str, args: list[str] | None = None) -> Self: ...

    @classmethod
    def sse(cls, url: str, headers: dict[str, str] | None = None) -> Self: ...
```

---

## VII. Strategic vs Tactical Programming Evidence

### 7.1 Strategic Programming Examples

#### Two-Layer Memory System

The memory system demonstrates **strategic investment**:

```python
# nanobot/memory/two_layer.py
class TwoLayerMemory:
    """
    MEMORY.md: Stable facts, consolidated by LLM
    HISTORY.md: Searchable raw logs
    """

    async def consolidate(self) -> None:
        """Periodically summarize HISTORY into MEMORY."""
        ...
```

**Investment**: Extra complexity for better organization pays off in long-term maintainability.

#### Provider Registry Pattern

```python
# nanobot/providers/registry.py
PROVIDERS: tuple[ProviderSpec, ...] = (
    ProviderSpec(name="anthropic", ...),
    ProviderSpec(name="openai", ...),
    # 20+ providers
)
```

**Investment**: Single registry enables auto-discovery, status display, and routing.

### 7.2 Tactical Programming Examples

#### Duplicate Message Deduplication

```python
# nanobot/channels/qq.py:80
self._processed_ids: deque = deque(maxlen=1000)

# nanobot/channels/discord.py (similar pattern)
self._processed_ids: set = set()
```

**Problem**: Different implementations in each channel instead of shared `MessageDeduplicator`.

#### Slack Markdown Conversion

```python
# nanobot/channels/slack.py:294-343
_TABLE_RE = re.compile(...)
_CODE_FENCE_RE = re.compile(...)
# 5 regex patterns + 50 lines of conversion logic
```

**Problem**: Complex inline conversion instead of dedicated `MarkdownConverter` class.

---

## VIII. Naming Quality Assessment

### 8.1 Good Names

| Name | Assessment |
|------|------------|
| `AgentLoop` | Clear - represents the main execution loop |
| `OutboundMessage` | Precise - message going out to user |
| `ProviderSpec` | Clear - specification for a provider |
| `should_warn_deprecated_memory_window` | Self-documenting boolean |
| `reply_in_thread` | Clear intent for Slack threading |

### 8.2 Vague Names (Red Flags)

| Name | Problem | Suggestion |
|------|---------|------------|
| `Base` (config schema) | Too generic | `ConfigModel` or `BaseConfig` |
| `_fixup_mrkdwn` | Colloquial/slang | `_sanitize_slack_markup` |
| `_kw_matches` | Abbreviation unclear | `_keyword_matches` |
| `msg` | Ambiguous abbreviation | `message` |
| `p` (provider var) | Single letter | `provider` |

---

## IX. Red Flags Summary

| Red Flag | Location | Severity |
|----------|----------|----------|
| Shallow Module | `_match_provider()` - 60+ lines of matching logic | HIGH |
| Information Leakage | Session key format in channel implementations | MEDIUM |
| Pass-Through Variable | `metadata` dict passed through multiple layers | MEDIUM |
| Comment Repeats Code | "# Handle app mentions" above `if event_type == "app_mention"` | LOW |
| Vague Name | `Base` class in config/schema.py | MEDIUM |
| Overexposure | `MCPServerConfig` with 7 optional fields | MEDIUM |
| Repetition | Duplicate deduplication in each channel | MEDIUM |
| Conjoined Methods | `_on_socket_request` requires understanding `_is_allowed` | MEDIUM |

---

## X. Recommendations by Priority

### High Priority

1. **Refactor Provider Matching** (`_match_provider`)
   - Extract into `ProviderResolver` with strategy pattern
   - Each strategy in its own class with single responsibility
   - Add explicit logging for which strategy matched

2. **Create Shared MessageDeduplicator**
   - Single implementation used by all channels
   - Configurable TTL and max size
   - Clear interface: `is_duplicate(message_id: str) -> bool`

### Medium Priority

3. **Extract PermissionPolicy Abstraction**
   - Separate policy from channel implementations
   - Enable policy composition (Allowlist + Mention)

4. **Create MarkdownConverter for Slack**
   - Dedicated class for markdown → mrkdwn conversion
   - Testable independently

5. **Rename Vague Identifiers**
   - `Base` → `ConfigModel`
   - `_fixup_mrkdwn` → `_sanitize_slack_markup`

### Low Priority

6. **Add Factory Methods to MCPServerConfig**
   - `MCPServerConfig.stdio(command, args)`
   - `MCPServerConfig.sse(url)`

7. **Centralize Model Name Normalization**
   - Create `ModelName` value object
   - Encapsulate hyphen/underscore handling

---

## XI. Architecture Strengths

1. **Clear Module Boundaries** - Channels, providers, tools are well-separated
2. **Type Safety** - Extensive use of Pydantic and generics
3. **Plugin Architecture** - New channels/providers/tools easy to add
4. **Configuration Flexibility** - Environment variables, config files, defaults
5. **Async Throughout** - Proper async/await patterns
6. **Testability** - Abstract base classes enable mocking

## XII. Overall Verdict

**Score: 7.5/10** - Well-designed with intentional complexity in the right places.

The nanobot codebase demonstrates **strategic programming** with good separation of concerns and thoughtful abstractions. The main issues are **shallow modules** in provider matching and channel message handling, which could be improved by applying the **"pull complexity downward"** principle. Information hiding is generally well-done, though some **leakage** exists in session key construction.

The architecture successfully manages complexity through **deep modules** like `MessageBus` and `ProviderSpec`, while the **two-layer memory system** shows excellent long-term thinking.
