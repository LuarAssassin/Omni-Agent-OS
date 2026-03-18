# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a collection of deep research documentation analyzing AI agent development tools. Each subdirectory contains comprehensive architecture analysis, code navigation indexes, dependency analysis, and architecture decision records (ADRs) for specific AI coding tools.

The repository is organized into two main categories:

| Category | Purpose |
|----------|---------|
| `forge-code/` | AI-powered code generation and editing tools (CLI-based coding agents) |
| `personal-general-ai-assistant/` | Personal AI assistant frameworks with multi-channel messaging support |

---

## forge-code

Documentation analyzing AI coding tools and agentic code editors.

### Projects

| Project | Description | Key Technologies |
|---------|-------------|------------------|
| `aider/` | Terminal-based AI pair programming tool analysis | Python, Tree-sitter, prompt-toolkit |
| `codex/` | OpenAI Codex CLI analysis | Rust (70+ crates), TypeScript, Python SDKs |
| `qwen-code/` | Qwen Code editor CLI analysis | TypeScript, React 19, Ink |
| `gemini-cli/` | Google Gemini CLI analysis | TypeScript, npm workspaces, A2A protocol |
| `pi-mono/` | Pi Mono code analysis tool | Python, Graph-based code analysis |
| `kimi-cli/` | Kimi CLI analysis | TypeScript, React |
| `opencode/` | OpenCode analysis | Python, Open-source code intelligence |

### Document Structure (per project)

Each project contains standardized analysis documents:

```
forge-code/{project}/
├── README.md                    # Project overview and navigation
├── {project}-analysis.md        # Architecture deep dive
├── {project}-code-index.md      # Code navigation index
├── {project}-dependencies.md    # Dependency analysis
└── {project}-adr.md             # Architecture decision records
```

---

## personal-general-ai-assistant

Documentation analyzing personal AI assistant frameworks with multi-channel messaging capabilities.

### Projects

| Project | Description | Key Technologies |
|---------|-------------|------------------|
| `copaw/` | Multi-channel AI assistant (AgentScope-based) | Python, TypeScript, React, FastAPI |
| `picoclaw/` | Ultra-lightweight personal AI assistant | Go, SQLite, JSONL |
| `ironclaw/` | Enterprise-grade AI assistant | Python |
| `nanoclaw/` | Minimal personal AI assistant | Python |
| `zeroclaw/` | Zero-dependency AI assistant | Python |
| `mimiclaw/` | Mimic-based AI assistant | Python |
| `openclaw/` | Open-source personal AI assistant | Python |
| `nanobot/` | Nano-scale bot framework | Python |

### Document Structure (per project)

Each project follows the same documentation pattern:

```
personal-general-ai-assistant/{project}/
├── README.md                    # Project overview and navigation
├── {project}-analysis.md        # Architecture deep dive
├── {project}-code-index.md      # Code navigation index
├── {project}-dependencies.md    # Dependency analysis
└── {project}-adr.md             # Architecture decision records
```

---

## Root-Level Deep Research Files

In addition to project-specific documentation, the repository contains consolidated deep-research files:

**forge-code/**:
- `deep-research-aider.md`
- `deep-research-codex.md`
- `deep-research-qwen-code.md`
- `deep-research-gemini-cli.md`
- `deep-research-pi-mono.md`
- `deep-research-kimi-cli.md`
- `deep-research-opencode.md`

**personal-general-ai-assistant/**:
- `deep-research-copaw.md`
- `deep-research-ironclaw.md`
- `deep-research-nanoclaw.md`
- `deep-research-openclaw.md`
- `deep-research-picoclaw.md`
- `deep-research-zeroclaw.md`
- `deep-research-mimiclaw.md`
- `deep-research-nanobot.md`

---

## Working with This Repository

This is a documentation repository. There are no build commands, test suites, or dependencies to install.

### Common Tasks

**Find documentation for a specific tool:**
```bash
# List available documentation
ls forge-code/
ls personal-general-ai-assistant/

# Read the main analysis
cat forge-code/aider/aider-analysis.md
cat personal-general-ai-assistant/copaw/copaw-analysis.md
```

**Search across all documentation:**
```bash
# Search for specific patterns
grep -r "MCP protocol" forge-code/
grep -r "multi-channel" personal-general-ai-assistant/
```

**Compare tools:**
- Architecture patterns differ between Python-based tools (aider, copaw) and Rust-based tools (codex)
- Personal assistants emphasize channel management (Discord, Telegram, etc.)
- Code editors emphasize file editing strategies and diff handling

---

## Analysis Methodology

Documentation in this repository follows a consistent analytical framework based on:

1. **Architecture Analysis** - Module organization, component relationships, data flow
2. **Code Indexing** - Key file locations organized by functional area
3. **Dependency Mapping** - Direct dependencies and their purposes
4. **Architecture Decision Records** - Key design decisions and trade-offs
5. **Design Philosophy Review** - Assessment against "A Philosophy of Software Design" principles

---

## Key Insights by Category

### Code Generation Tools (forge-code)

Common patterns observed:
- **Strategy Pattern** for edit formats (diff, whole, udiff, architect)
- **RepoMap** for codebase context management
- **Multi-model support** via unified LLM interfaces
- **Sandboxed execution** for security
- **MCP (Model Context Protocol)** integration for tool ecosystems

### Personal Assistants (personal-general-ai-assistant)

Common patterns observed:
- **Multi-channel abstraction** for messaging platforms
- **Workspace-per-agent** architecture
- **Skill system** based on Markdown files
- **Message bus** for event-driven communication
- **Memory management** with vector stores
- **Model routing** for intelligent provider selection
