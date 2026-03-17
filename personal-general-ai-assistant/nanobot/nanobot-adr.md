# Nanobot Architecture Decision Records

This document records significant architectural decisions made during the development of nanobot. Each ADR captures the context, decision, and consequences to preserve design rationale for future maintainers.

---

## ADR-001: Pydantic for Configuration Schema

**Status**: Accepted

**Context**:
Nanobot requires complex configuration spanning 20+ LLM providers, multiple channels, and nested tool settings. Configuration must support environment variable overrides, validation, and serialization.

**Decision**:
Use Pydantic v2 with `BaseSettings` for all configuration.

**Key Design Choices**:
- `alias_generator=to_camel` supports both camelCase and snake_case in config files
- `populate_by_name=True` allows using either format
- `env_nested_delimiter="__"` enables nested env vars like `NANOBOT_PROVIDERS__OPENAI__API_KEY`
- Frozen `ProviderSpec` dataclass for registry entries ensures immutability

**Consequences**:
- ✅ Type-safe configuration with runtime validation
- ✅ Auto-generated documentation from type hints
- ✅ Environment variable integration out of the box
- ✅ Config file format flexibility (YAML, JSON, TOML)
- ⚠️ Pydantic v2 migration required updates across the codebase

---

## ADR-002: JSONL for Session Storage

**Status**: Accepted

**Context**:
Sessions must persist conversation history across restarts. Requirements: append-only writes, human-readable format, easy grep/search, no schema migrations.

**Decision**:
Use JSON Lines (JSONL) format - one JSON object per line.

**Implementation**:
```python
# nanobot/session/store.py
class SessionStore:
    def _path(self, session_id: str) -> Path:
        return self.base_dir / f"{session_id}.jsonl"

    def save(self, session: Session) -> None:
        # Append-only write
        with open(path, "a") as f:
            f.write(json.dumps(message) + "\n")
```

**Consequences**:
- ✅ Append-only writes are O(1) regardless of session size
- ✅ Human-readable and grep-friendly
- ✅ No schema migrations needed - new fields are optional
- ✅ Easy to truncate/rotate for memory management
- ⚠️ Loading requires reading entire file (mitigated by consolidation)

---

## ADR-003: Two-Layer Memory System

**Status**: Accepted

**Context**:
LLM context windows are limited but conversations can be very long. Need to preserve important facts without hitting token limits.

**Decision**:
Implement two-layer memory:
- **MEMORY.md**: LLM-curated stable facts (summarized)
- **HISTORY.md**: Complete searchable raw logs

**Consolidation Strategy**:
```python
# nanobot/memory/two_layer.py
class TwoLayerMemory:
    async def consolidate(self) -> None:
        """Periodically summarize HISTORY into MEMORY."""
        recent = self.history.get_recent(since=self.last_consolidation)
        summary = await self.llm.summarize(recent)
        self.memory.merge(summary)
```

**Consequences**:
- ✅ Long conversations stay within context limits
- ✅ Important facts persist across sessions
- ✅ Full history remains grep-searchable
- ✅ Strategic complexity investment pays off in maintainability
- ⚠️ Requires LLM calls for consolidation (async background task)

---

## ADR-004: LiteLLM for Provider Abstraction

**Status**: Accepted

**Context**:
Supporting 20+ LLM providers (Anthropic, OpenAI, DeepSeek, etc.) with different APIs, authentication, and model formats would require massive code duplication.

**Decision**:
Use LiteLLM as the unified API layer with a ProviderSpec registry for metadata.

**Architecture**:
```python
@dataclass(frozen=True)
class ProviderSpec:
    name: str
    keywords: tuple[str, ...]
    env_key: str
    litellm_prefix: str
    is_gateway: bool
    is_local: bool
    is_oauth: bool
```

**Consequences**:
- ✅ Single implementation for 20+ providers
- ✅ Consistent model prefixing via `litellm_prefix`
- ✅ Automatic environment variable management
- ✅ Gateway detection for fallback routing
- ⚠️ Provider-specific features may be hidden by abstraction
- ⚠️ Complex matching logic in `_match_provider()` (identified as shallow module)

---

## ADR-005: MessageBus Pub/Sub Pattern

**Status**: Accepted

**Context**:
Channels (Slack, Discord, Telegram) need to communicate with the AgentLoop without direct coupling. Multiple consumers may need to react to the same event.

**Decision**:
Implement simple typed pub/sub MessageBus.

**Implementation**:
```python
# nanobot/bus/queue.py
class MessageBus:
    def subscribe[T](self, event_type: type[T], handler: Callable[[T], Any]) -> None:
        self._consumers.setdefault(event_type, []).append(handler)

    def publish[T](self, event: T) -> None:
        for handler in self._consumers.get(type(event), []):
            handler(event)
```

**Consequences**:
- ✅ Decoupled channels from agent logic
- ✅ Type-safe event handling with generics
- ✅ Easy to add new event types
- ✅ Simple to test (mock bus)
- ✅ Deep module - simple interface hides async complexity

---

## ADR-006: BaseChannel Abstraction

**Status**: Accepted

**Context**:
8+ channel implementations (Slack, Discord, Telegram, QQ, DingTalk, Feishu, Matrix, WebSocket) share common patterns but have platform-specific details.

**Decision**:
Abstract base class with template method pattern.

```python
class BaseChannel(ABC):
    @abstractmethod
    async def start(self) -> None: ...

    @abstractmethod
    async def stop(self) -> None: ...

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None: ...

    async def _handle_message(self, ...) -> None:
        # Common message processing
```

**Consequences**:
- ✅ New channels implement 3 methods
- ✅ Common deduplication, permission checking in base
- ✅ Consistent error handling across channels
- ⚠️ Permission logic scattered (each channel implements `_is_allowed`)
- ⚠️ Session key format leaked to subclasses

---

## ADR-007: Tool Registry Pattern

**Status**: Accepted

**Context**:
Tools (shell exec, file ops, web search, skills) need dynamic registration and JSON schema generation for LLM function calling.

**Decision**:
Abstract Tool base class with registry.

```python
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    async def execute(self, **kwargs) -> ToolResult: ...

    def get_schema(self) -> dict: ...
```

**Consequences**:
- ✅ Type-safe tool execution
- ✅ Automatic JSON schema generation
- ✅ Easy to add new tools
- ✅ ToolResult encapsulates success/error without exceptions
- ✅ Deep module design

---

## ADR-008: MCP Integration for External Tools

**Status**: Accepted

**Context**:
Model Context Protocol (MCP) allows external tools/servers to integrate without code changes. Need to support stdio, SSE, and HTTP transports.

**Decision**:
Integrate MCP with transport-agnostic configuration.

```python
class MCPServerConfig(Base):
    type: Literal["stdio", "sse", "streamableHttp"]
    command: str  # For stdio
    args: list[str]
    url: str  # For HTTP/SSE
    enabled_tools: list[str] = ["*"]  # Filter support
```

**Consequences**:
- ✅ Extensible tool ecosystem without code changes
- ✅ Multiple transport support
- ✅ Tool filtering per server
- ⚠️ Configuration is complex (7+ optional fields)
- ⚠️ Overexposure - common case requires setting many fields

---

## ADR-009: Subagent System for Background Tasks

**Status**: Accepted

**Context**:
Long-running tasks (research, file processing) should not block the main agent loop. Need spawn/cancel/monitor lifecycle.

**Decision**:
Implement subagent system with isolated tool registries.

```python
class Subagent:
    def spawn(self, task: str, tool_registry: ToolRegistry) -> str:
        # Runs in background with separate context

    def cancel(self, task_id: str) -> bool:
        # Graceful cancellation

    def status(self, task_id: str) -> TaskStatus:
        # Progress monitoring
```

**Consequences**:
- ✅ Non-blocking long-running operations
- ✅ Isolated tool access for security
- ✅ Progress monitoring support
- ⚠️ Complex lifecycle management
- ⚠️ Resource cleanup on cancellation

---

## ADR-010: Skill System with Markdown + YAML Frontmatter

**Status**: Accepted

**Context**:
Users need to customize agent behavior without code changes. Skills should be version controlled and human-readable.

**Decision**:
Markdown files with YAML frontmatter for metadata.

```markdown
---
name: git-commit
description: Generate conventional commits
tools: [shell, file]
---

# Git Commit Skill

When asked to commit, follow these steps...
```

**Consequences**:
- ✅ Human-readable and version-controllable
- ✅ Rich documentation alongside code
- ✅ YAML metadata for programmatic access
- ✅ Easy to share and reuse
- ⚠️ Parsing overhead vs pure code

---

## ADR-011: Provider Matching Priority Strategy

**Status**: Accepted (with noted issues)

**Context**:
Need to automatically route models to correct providers. Users may specify `provider: auto` or omit provider entirely.

**Decision**:
6-strategy priority matching:
1. Explicit provider override
2. Model name prefix matching (`anthropic/claude-3`)
3. Keyword matching in model name
4. Local provider detection by `api_base`
5. Gateway fallback
6. Any provider with API key fallback

**Implementation**:
```python
def _match_provider(self, model: str | None) -> tuple[ProviderConfig | None, str | None]:
    # 60+ lines of complex matching logic
```

**Consequences**:
- ✅ Automatic provider detection usually works
- ✅ Gateway fallback ensures something works
- ⚠️ **Unknown unknowns** - users can't predict selection
- ⚠️ **Shallow module** - interface nearly as complex as implementation
- ⚠️ High cognitive load for debugging

---

## ADR-012: Async Throughout

**Status**: Accepted

**Context**:
IO-bound operations dominate (LLM calls, channel APIs, web search). Blocking would severely limit throughput.

**Decision**:
Use `async`/`await` for all IO operations.

**Consequences**:
- ✅ High concurrency with single-threaded code
- ✅ Proper backpressure via asyncio queues
- ✅ Clean cancellation via task cancellation
- ⚠️ Requires async-compatible libraries
- ⚠️ Debugging async code is harder

---

## ADR-013: OAuth Providers Without API Keys

**Status**: Accepted

**Context**:
Some providers (OpenAI Codex, GitHub Copilot) use OAuth rather than static API keys.

**Decision**:
Mark providers with `is_oauth=True` and handle separately.

```python
ProviderSpec(
    name="openai_codex",
    is_oauth=True,
    env_key="",  # No API key
)
```

**Consequences**:
- ✅ Supports modern authentication flows
- ✅ Clear separation in matching logic
- ⚠️ OAuth flow complexity
- ⚠️ Token refresh handling

---

## ADR-014: Session Key Format with Channel Prefix

**Status**: Accepted (with noted leakage)

**Context**:
Sessions must be uniquely identifiable across channels with thread support.

**Decision**:
Colon-delimited format: `slack:{chat_id}:{thread_ts}`

**Implementation**:
```python
# nanobot/channels/slack.py:224
session_key = f"slack:{chat_id}:{thread_ts}" if thread_ts and channel_type != "im" else None
```

**Consequences**:
- ✅ Unique keys across channels
- ✅ Thread-scoped conversations
- ⚠️ **Information leakage** - format exposed to all channels
- ⚠️ Must update all channels if format changes

---

## Summary Table

| ADR | Decision | Status | Complexity Impact |
|-----|----------|--------|-------------------|
| 001 | Pydantic for config | Accepted | Reduced |
| 002 | JSONL for sessions | Accepted | Reduced |
| 003 | Two-layer memory | Accepted | Strategic investment |
| 004 | LiteLLM abstraction | Accepted | Reduced (with shallow module issue) |
| 005 | MessageBus pub/sub | Accepted | Reduced |
| 006 | BaseChannel | Accepted | Reduced (with leakage issue) |
| 007 | Tool registry | Accepted | Reduced |
| 008 | MCP integration | Accepted | Acceptable complexity |
| 009 | Subagent system | Accepted | Necessary complexity |
| 010 | Markdown skills | Accepted | Reduced |
| 011 | Provider matching | Accepted | **Needs refactoring** |
| 012 | Async throughout | Accepted | Necessary complexity |
| 013 | OAuth providers | Accepted | Necessary complexity |
| 014 | Session key format | Accepted | **Needs encapsulation** |

---

## Pending Decisions

None currently. All major architectural decisions are recorded above.
