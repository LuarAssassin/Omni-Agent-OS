# Nanobot Code Index

## Overview

Nanobot is organized into functional modules following a layered architecture. This index maps features to their implementation locations.

---

## Core Agent System

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Agent Loop | `nanobot/agent/agent_loop.py` | 250 |
| Agent Context | `nanobot/agent/context.py` | 180 |
| Two-Layer Memory | `nanobot/agent/memory.py` | 220 |
| Skill Runner | `nanobot/agent/skill_runner.py` | 160 |
| Subagent System | `nanobot/agent/subagent.py` | 200 |
| Agent State | `nanobot/agent/state.py` | 120 |

---

## Message Channels

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Base Channel (Abstract) | `nanobot/channels/base.py` | 200 |
| Channel Manager | `nanobot/channels/manager.py` | 150 |
| Channel Registry | `nanobot/channels/registry.py` | 80 |
| Slack Channel | `nanobot/channels/slack.py` | 344 |
| Discord Channel | `nanobot/channels/discord.py` | 280 |
| Telegram Channel | `nanobot/channels/telegram.py` | 260 |
| QQ Channel | `nanobot/channels/qq.py` | 240 |
| DingTalk Channel | `nanobot/channels/dingtalk.py` | 220 |
| Feishu Channel | `nanobot/channels/feishu.py` | 230 |
| Matrix Channel | `nanobot/channels/matrix.py` | 300 |
| WebSocket Channel | `nanobot/channels/websocket.py` | 180 |
| Webhook Channel | `nanobot/channels/webhook.py` | 150 |
| Console Channel | `nanobot/channels/console.py` | 120 |

---

## LLM Provider System

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Provider Registry | `nanobot/providers/registry.py` | 180 |
| LiteLLM Integration | `nanobot/providers/litellm_client.py` | 200 |
| Custom Provider | `nanobot/providers/custom.py` | 120 |
| Azure OpenAI | `nanobot/providers/azure.py` | 100 |
| OpenAI Codex (OAuth) | `nanobot/providers/openai_codex.py` | 150 |
| GitHub Copilot (OAuth) | `nanobot/providers/github_copilot.py` | 140 |

---

## Tool System

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Tool Base (Abstract) | `nanobot/tools/base.py` | 120 |
| Tool Registry | `nanobot/tools/registry.py` | 180 |
| Filesystem Tool | `nanobot/tools/filesystem.py` | 250 |
| Shell Execution Tool | `nanobot/tools/shell.py` | 200 |
| Web Search Tool | `nanobot/tools/web_search.py` | 180 |
| Web Fetch Tool | `nanobot/tools/web_fetch.py` | 150 |
| MCP Tool Wrapper | `nanobot/tools/mcp_tool.py` | 160 |
| Cron Tool | `nanobot/tools/cron.py` | 140 |
| Message Tool | `nanobot/tools/message.py` | 100 |
| Spawn Tool | `nanobot/tools/spawn.py` | 120 |

---

## MCP (Model Context Protocol) System

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| MCP Client Manager | `nanobot/mcp/client.py` | 280 |
| MCP Server Connection | `nanobot/mcp/connection.py` | 200 |
| MCP Tool Discovery | `nanobot/mcp/discovery.py` | 150 |
| MCP Transport (stdio) | `nanobot/mcp/transport/stdio.py` | 120 |
| MCP Transport (SSE) | `nanobot/mcp/transport/sse.py` | 140 |
| MCP Transport (HTTP) | `nanobot/mcp/transport/http.py` | 130 |

---

## Session & Memory

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Session Manager | `nanobot/session/manager.py` | 180 |
| Session Store (JSONL) | `nanobot/session/store.py` | 200 |
| Session Model | `nanobot/session/model.py` | 100 |
| Memory Consolidation | `nanobot/memory/consolidation.py` | 160 |
| MEMORY.md Handler | `nanobot/memory/memory_handler.py` | 140 |
| HISTORY.md Handler | `nanobot/memory/history_handler.py` | 120 |

---

## Configuration System

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Config Schema (Pydantic) | `nanobot/config/schema.py` | 261 |
| Config Loader | `nanobot/config/loader.py` | 150 |
| Config Paths | `nanobot/config/paths.py` | 80 |

---

## Message Bus

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Message Bus Core | `nanobot/bus/queue.py` | 120 |
| Event Types | `nanobot/bus/events.py` | 100 |
| Event Handlers | `nanobot/bus/handlers.py` | 140 |

---

## Gateway & Services

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Gateway Server | `nanobot/gateway/server.py` | 200 |
| Heartbeat Service | `nanobot/heartbeat/service.py` | 120 |
| Health Check | `nanobot/gateway/health.py` | 80 |

---

## Skills System

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Skill Loader | `nanobot/skills/loader.py` | 160 |
| Skill Parser | `nanobot/skills/parser.py` | 120 |
| Skill Registry | `nanobot/skills/registry.py` | 100 |
| Git Commit Skill | `nanobot/skills/built-in/git-commit.md` | 80 |
| Code Review Skill | `nanobot/skills/built-in/code-review.md` | 90 |
| Debug Skill | `nanobot/skills/built-in/debug.md` | 70 |
| Test Skill | `nanobot/skills/built-in/test.md` | 60 |
| Document Skill | `nanobot/skills/built-in/document.md` | 70 |
| Refactor Skill | `nanobot/skills/built-in/refactor.md` | 80 |
| Shell Skill | `nanobot/skills/built-in/shell.md` | 50 |
| Web Skill | `nanobot/skills/built-in/web.md` | 60 |

---

## CLI & Commands

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| CLI Entry | `nanobot/cli/commands.py` | 200 |
| CLI Main | `nanobot/cli/main.py` | 120 |
| Init Command | `nanobot/cli/commands/init.py` | 100 |
| Run Command | `nanobot/cli/commands/run.py` | 150 |
| Config Command | `nanobot/cli/commands/config.py` | 120 |
| Provider Command | `nanobot/cli/commands/provider.py` | 100 |
| Status Command | `nanobot/cli/commands/status.py` | 80 |

---

## Security & Authentication

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| OAuth Handler | `nanobot/security/oauth.py` | 180 |
| Token Manager | `nanobot/security/token_manager.py` | 140 |
| Allowlist Check | `nanobot/security/allowlist.py` | 100 |
| Permission Policy | `nanobot/security/policy.py` | 120 |

---

## Utilities

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Logging Setup | `nanobot/utils/logging.py` | 80 |
| Async Helpers | `nanobot/utils/async_helpers.py` | 100 |
| File Helpers | `nanobot/utils/file_helpers.py` | 120 |
| String Utils | `nanobot/utils/strings.py` | 80 |
| Time Utils | `nanobot/utils/time.py` | 60 |
| Evaluator | `nanobot/utils/evaluator.py` | 140 |

---

## Cron System

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Cron Scheduler | `nanobot/cron/scheduler.py` | 160 |
| Cron Job Model | `nanobot/cron/job.py` | 80 |
| Cron Runner | `nanobot/cron/runner.py` | 120 |

---

## Test Suite

| Feature | File Path | Approx. Lines |
|---------|-----------|---------------|
| Config Tests | `tests/test_config.py` | 200 |
| Provider Tests | `tests/test_providers.py` | 180 |
| Channel Tests | `tests/test_channels.py` | 220 |
| Tool Tests | `tests/test_tools.py` | 250 |
| Session Tests | `tests/test_session.py` | 150 |
| Memory Tests | `tests/test_memory.py` | 180 |
| Skill Tests | `tests/test_skills.py` | 160 |
| Integration Tests | `tests/test_integration.py` | 200 |
| MCP Tests | `tests/test_mcp.py` | 180 |
| Conftest | `tests/conftest.py` | 100 |

---

## File Count Summary

| Category | File Count | Primary Language |
|----------|------------|------------------|
| Core Agent | 6 | Python |
| Channels | 13 | Python |
| Providers | 6 | Python |
| Tools | 10 | Python |
| MCP | 6 | Python |
| Session/Memory | 6 | Python |
| Config | 3 | Python |
| Message Bus | 3 | Python |
| Gateway | 3 | Python |
| Skills | 11 | Markdown/YAML |
| CLI | 7 | Python |
| Security | 4 | Python |
| Utils | 6 | Python |
| Cron | 3 | Python |
| Tests | 10+ | Python |
| **Total** | **~90** | **Python/Markdown** |

---

## Key Entry Points

```
nanobot/
├── __main__.py              # Package entry: python -m nanobot
├── cli/commands.py          # CLI command definitions
├── agent/agent_loop.py      # Main agent execution loop
├── channels/manager.py      # Channel lifecycle management
├── gateway/server.py        # HTTP gateway entry
├── config/schema.py         # Configuration validation
└── providers/registry.py    # LLM provider definitions
```

---

## Dependency Flow

```
CLI Entry
    ↓
Config Loader → Schema Validation
    ↓
Channel Manager → BaseChannel implementations
    ↓
Agent Loop ←→ Tool Registry ←→ MCP Client
    ↓               ↓
Message Bus ← Session Manager
    ↓
LLM Provider (via LiteLLM)
```
