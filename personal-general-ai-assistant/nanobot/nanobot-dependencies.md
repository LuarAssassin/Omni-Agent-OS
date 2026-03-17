# Nanobot Dependencies Analysis

## Overview

Nanobot has **50+ direct dependencies** organized into functional groups. This document analyzes the dependency structure, version constraints, and architectural patterns.

---

## Dependency Summary

| Category | Count | Purpose |
|----------|-------|---------|
| Core Framework | 5 | CLI, LLM abstraction, configuration |
| Web & Network | 6 | HTTP clients, WebSocket, async networking |
| Channel SDKs | 8 | Slack, Telegram, DingTalk, Feishu, QQ, Discord |
| Data Processing | 8 | Parsing, encoding, serialization |
| MCP & Integration | 2 | Model Context Protocol |
| Authentication | 2 | OAuth flows, token management |
| Utilities | 12 | Logging, prompts, QR codes, scheduling |
| Development | 10 | Testing, linting, type checking |

---

## Core Framework Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `typer` | >=0.20.0,<1.0.0 | Modern CLI framework with type hints |
| `litellm` | >=1.82.1,<2.0.0 | Unified LLM provider abstraction |
| `pydantic` | >=2.12.0,<3.0.0 | Data validation and settings management |
| `pydantic-settings` | >=2.0.0,<3.0.0 | Environment variable integration |
| `python-dotenv` | >=1.0.0,<2.0.0 | .env file loading |

**Design Pattern**: Pydantic v2 provides runtime validation with type safety. The `env_nested_delimiter="__"` enables nested config via env vars like `NANOBOT_PROVIDERS__OPENAI__API_KEY`.

---

## Web & Network Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `httpx` | >=0.28.0,<1.0.0 | Modern async HTTP client |
| `websockets` | >=16.0,<17.0.0 | Native WebSocket implementation |
| `websocket-client` | >=1.8.0,<2.0.0 | Synchronous WebSocket fallback |
| `aiohttp` | >=3.9.0,<4.0.0 | Async HTTP server/client |
| `requests` | >=2.32.0,<3.0.0 | Synchronous HTTP (legacy support) |
| `urllib3` | >=2.0.0,<3.0.0 | HTTP client underlying requests |

**Note**: Dual WebSocket libraries needed for different channel SDK requirements. `websockets` is preferred for native async; `websocket-client` for SDKs requiring sync interfaces.

---

## Channel SDK Dependencies

| Package | Version | Channel | Notes |
|---------|---------|---------|-------|
| `slack-sdk` | >=3.39.0,<4.0.0 | Slack | Socket Mode + Web API |
| `python-telegram-bot` | >=22.0,<23.0.0 | Telegram | Async support via `telegram.ext` |
| `dingtalk-stream` | >=0.21.0,<1.0.0 | DingTalk | Stream SDK for events |
| `lark-oapi` | >=1.4.0,<2.0.0 | Feishu/Lark | Open API SDK |
| `qq-botpy` | >=1.2.0,<2.0.0 | QQ | Official bot SDK |
| `discord.py` | >=2.4.0,<3.0.0 | Discord | Async gateway client |
| `matrix-nio` | >=0.25.0,<1.0.0 | Matrix | E2E encryption support |
| `wechat-enterprise-sdk` | >=0.2.0,<1.0.0 | WeCom | Enterprise WeChat |

**Architecture Pattern**: Each channel implements `BaseChannel` interface but brings its own SDK with different async models. The Channel Manager abstracts these differences.

---

## Data Processing Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `readability-lxml` | >=0.8.0,<1.0.0 | HTML article extraction |
| `msgpack` | >=1.1.0,<2.0.0 | Binary JSON alternative |
| `json-repair` | >=0.39.0,<1.0.0 | Fix malformed LLM JSON output |
| `chardet` | >=5.2.0,<6.0.0 | Character encoding detection |
| `pyyaml` | >=6.0.0,<7.0.0 | YAML parsing for config/skills |
| `toml` | >=0.10.2,<1.0.0 | TOML config support |
| `python-dateutil` | >=2.9.0,<3.0.0 | Date parsing utilities |
| `pytz` | >=2024.1 | Timezone handling |

**Insight**: `json-repair` is a pragmatic addition - LLMs occasionally output malformed JSON, and this library defines the error out of existence by auto-fixing common issues.

---

## MCP & Integration Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `mcp` | >=1.26.0,<2.0.0 | Model Context Protocol SDK |
| `langsmith` | >=0.1.0,<1.0.0 (optional) | LangChain tracing/monitoring |

**MCP Architecture**: The `mcp` package provides stdio, SSE, and HTTP transports for connecting to external tool servers without code changes.

---

## Authentication Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `oauth-cli-kit` | >=0.4.0,<1.0.0 | OAuth flow for CLI tools |
| `keyring` | >=25.0.0,<26.0.0 | Secure credential storage |

**Security Pattern**: OAuth providers (OpenAI Codex, GitHub Copilot) use `oauth-cli-kit` for device flow authentication. API keys stored in system keyring when available.

---

## Utility Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `loguru` | >=0.7.0,<1.0.0 | Modern structured logging |
| `rich` | >=13.9.0,<14.0.0 | Terminal formatting/console UI |
| `prompt-toolkit` | >=3.0.0,<4.0.0 | Interactive CLI prompts |
| `click` | >=8.1.0,<9.0.0 | CLI utilities (via Typer) |
| `tenacity` | >=8.5.0,<9.0.0 | Retry logic with backoff |
| `croniter` | >=6.0.0,<7.0.0 | Cron schedule parsing |
| `qrcode` | >=7.4.0,<8.0.0 | QR code generation (WeChat) |
| `pillow` | >=11.0.0,<12.0.0 | Image processing |
| `psutil` | >=6.0.0,<7.0.0 | System process monitoring |
| `watchdog` | >=6.0.0,<7.0.0 | File system event monitoring |
| `jinja2` | >=3.1.0,<4.0.0 | Template rendering |
| `markdown` | >=3.7.0,<4.0.0 | Markdown parsing |

---

## Web Search Dependencies

| Package | Version | Provider |
|---------|---------|----------|
| `duckduckgo-search` | >=7.5.0,<8.0.0 | DuckDuckGo |
| `tavily-python` | >=0.5.0,<1.0.0 | Tavily |

**Design**: Web search tools use provider-specific SDKs. The `WebSearchConfig` abstraction allows switching providers via config without code changes.

---

## Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `pytest` | >=8.0.0,<9.0.0 | Test framework |
| `pytest-asyncio` | >=0.24.0,<1.0.0 | Async test support |
| `pytest-cov` | >=6.0.0,<7.0.0 | Coverage reporting |
| `ruff` | >=0.9.0,<1.0.0 | Fast Python linter |
| `mypy` | >=1.11.0,<2.0.0 | Static type checking |
| `openai` | >=1.35.0,<2.0.0 | OpenAI SDK (testing) |
| `tiktoken` | >=0.8.0,<1.0.0 | Token counting (testing) |
| `respx` | >=0.22.0,<1.0.0 | HTTPX mocking |
| `pytest-mock` | >=3.14.0,<4.0.0 | Pytest mocking utilities |
| `types-*` | various | Type stubs |

---

## Dependency Architecture Patterns

### 1. Abstraction Layer Pattern

```python
# litellm abstracts 20+ providers
from litellm import acompletion

# Single interface, multiple providers
response = await acompletion(
    model="anthropic/claude-3-opus",
    messages=messages,
    api_key=config.api_key,
)
```

### 2. Optional Dependency Groups

```toml
[project.optional-dependencies]
wecom = ["wechat-enterprise-sdk>=0.2.0,<1.0.0"]
matrix = ["matrix-nio>=0.25.0,<1.0.0"]
langsmith = ["langsmith>=0.1.0,<1.0.0"]
dev = ["pytest>=8.0.0", "ruff>=0.9.0", ...]
```

Channels with heavy dependencies are optional to keep base install lightweight.

### 3. Version Constraint Strategy

| Pattern | Example | Rationale |
|---------|---------|-----------|
| Upper bound | `<2.0.0` | Prevent breaking changes |
| Lower bound | `>=1.26.0` | Require bug fixes/features |
| Tilde | `~1.26.0` | (Not used) Prefer explicit bounds |

---

## Dependency Risk Assessment

| Risk Level | Dependencies | Mitigation |
|------------|--------------|------------|
| **Low** | pydantic, typer, httpx | Well-maintained, stable APIs |
| **Medium** | litellm, mcp | Rapidly evolving, but well-tested |
| **Medium** | Channel SDKs | Vendor-controlled, breaking changes possible |
| **High** | oauth-cli-kit, json-repair | Smaller projects, monitor for maintenance |

---

## Total Dependency Tree

```
nanobot
├── Direct: ~50 packages
├── With dev: ~70 packages
├── Total transitive: ~200+ packages
└── Install size: ~150MB (base), ~250MB (all extras)
```

---

## Key Insights

1. **LiteLLM reduces complexity**: Single dependency replaces 20+ provider SDKs
2. **Channel SDKs dominate**: 8 of 50 dependencies are channel-specific
3. **Optional groups keep base small**: WeCom, Matrix, LangSmith are optional
4. **Pydantic v2 is foundational**: Configuration, validation, serialization all use Pydantic
5. **Async-first design**: All network I/O uses async libraries (httpx, websockets, aiohttp)

---

## Recommendations

1. **Monitor litellm updates**: Core abstraction layer, test thoroughly on upgrades
2. **Pin channel SDKs**: Vendor APIs change frequently, use exact versions
3. **Consider uv**: For faster installs with 200+ transitive dependencies
4. **Audit json-repair**: Evaluate if still needed as LLM outputs improve

