# Comprehensive Memory, Session State & Boundary Management Analysis
## Across 15 AI Agent Projects in Omni-Agent-OS

**Generated:** 2026-03-18
**Scope:** 8 Personal Assistants + 7 Programming Agents
**Analysis Depth:** Architecture patterns, code snippets, file paths

---

## Executive Summary

This document provides exhaustive analysis of memory systems, session state management, and boundary management patterns across 15 AI agent projects. The analysis reveals distinct architectural approaches between **Personal Assistants** (emphasizing long-term memory and RAG) and **Programming Agents** (emphasizing context window management and session persistence).

### Key Findings

| Category | Primary Focus | Storage Pattern | RAG Support |
|----------|---------------|-----------------|-------------|
| **Personal Assistants** | Long-term memory, user context | SQLite + Markdown + Vector DB | Hybrid (Vector + BM25) |
| **Programming Agents** | Context window, session persistence | JSONL + SQLite | Minimal/None |

---

## Table of Contents

1. [Memory Systems Architecture](#1-memory-systems-architecture)
2. [Session State Management](#2-session-state-management)
3. [Boundary Management Patterns](#3-boundary-management-patterns)
4. [Project-by-Project Analysis](#4-project-by-project-analysis)
5. [Comparative Matrices](#5-comparative-matrices)
6. [Recommended Unified Architecture](#6-recommended-unified-architecture)

---

## 1. Memory Systems Architecture

### 1.1 The Three-Layer Memory Model

All advanced AI agent systems implement a layered memory architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Three-Layer Memory Model                     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Working Memory (Session Context)                       │
│          └── Current conversation, tool calls, immediate state  │
│          └── Lives in RAM, ephemeral                            │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Short-Term Memory (Recent History)                     │
│          └── Daily logs, session summaries, recent facts        │
│          └── Typically JSONL or daily Markdown files            │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Long-Term Memory (Persistent Knowledge)                │
│          └── Curated facts, user preferences, decisions         │
│          └── Stored in MEMORY.md + vector embeddings            │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Memory Write Strategies

#### Strategy 1: Token Threshold-Based (CoPaw Pattern)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/copaw/copaw-adr.md`

```python
# src/copaw/agents/hooks/memory_compaction.py:76-90
config = load_config()
memory_compact_threshold = config.agents.running.memory_compact_threshold
left_compact_threshold = memory_compact_threshold - str_token_count

if left_compact_threshold <= 0:
    logger.warning("The memory_compact_threshold is set too low...")
    return None

# 触发异步摘要任务
trigger_summary = True
self.memory_manager.add_async_summary_task(messages=messages_to_compact)
compact_content = await self.memory_manager.compact_memory(...)
await agent.memory.update_compressed_summary(compact_content)
```

| Dimension | Implementation |
|-----------|----------------|
| **Write Timing** | Condition-triggered (Token threshold at 75%) |
| **Write Content** | Message summaries, vector embeddings, metadata |
| **Storage Medium** | SQLite (ReMeLight), Chroma (vectors), Local files |
| **Sync Mechanism** | Hook-based async processing |

#### Strategy 2: Real-Time Append-Only (kimi-cli Pattern)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/kimi-cli/kimi-cli-adr.md`

```python
# src/kimi_cli/wire/file.py:117-131
async def append_message(self, msg: WireMessage, *, timestamp: float | None = None) -> None:
    record = WireMessageRecord.from_wire_message(
        msg,
        timestamp=time.time() if timestamp is None else timestamp,
    )
    await self.append_record(record)

async def append_record(self, record: WireMessageRecord) -> None:
    self.path.parent.mkdir(parents=True, exist_ok=True)
    needs_header = not self.path.exists() or self.path.stat().st_size == 0
    async with aiofiles.open(self.path, mode="a", encoding="utf-8") as f:
        if needs_header:
            metadata = WireFileMetadata(protocol_version=self.protocol_version)
            await f.write(_dump_line(metadata))
        await f.write(_dump_line(record))  # Real-time append
```

| Dimension | Implementation |
|-----------|----------------|
| **Write Timing** | Real-time append-only |
| **Write Content** | Messages (context.jsonl), events (wire.jsonl), state (state.json) |
| **Storage Medium** | JSONL files |
| **Sync Mechanism** | Direct file I/O |

#### Strategy 3: Debounced Multi-Trigger (OpenClaw Pattern)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/openclaw/openclaw-adr.md`

```typescript
// src/memory/manager.ts:241-255
async warmSession(sessionKey?: string): Promise<void> {
  if (!this.settings.sync.onSessionStart) {
    return;
  }
  // Session start triggers sync
  void this.sync({ reason: "session-start" }).catch((err) => {
    log.warn(`memory sync failed (session-start): ${String(err)}`);
  });
  if (key) {
    this.sessionWarm.add(key);
  }
}

// Multi-trigger synchronization
async sync(params?: { reason?: string; force?: boolean }): Promise<void> {
  if (this.closed || this.syncing) {
    return this.syncing ?? Promise.resolve();
  }
  this.syncing = this.runSyncWithReadonlyRecovery(params).finally(() => {
    this.syncing = null;
  });
  return this.syncing;
}
```

| Dimension | Implementation |
|-----------|----------------|
| **Write Timing** | Real-time + debounced batch + lifecycle triggers |
| **Write Content** | Vector embeddings, FTS index, file chunks, session deltas |
| **Storage Medium** | SQLite + sqlite-vec |
| **Sync Mechanism** | File watcher + timer + session lifecycle |

#### Strategy 4: Transactional Event-Driven (opencode Pattern)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/opencode/opencode-adr.md`

```typescript
// src/session/index.ts:685-705
export const updateMessage = fn(MessageV2.Info, async (msg) => {
  const { id, sessionID, ...data } = msg;
  Database.use((db) => {
    db.insert(MessageTable)
      .values({ id, session_id: sessionID, data })
      .onConflictDoUpdate({ target: MessageTable.id, set: { data } })
      .run();
    // Event publication
    Database.effect(() =>
      Bus.publish(MessageV2.Event.Updated, { info: msg }),
    );
  });
  return msg;
});
```

| Dimension | Implementation |
|-----------|----------------|
| **Write Timing** | Real-time per message/part |
| **Write Content** | Messages, Parts, metadata, summaries, diffs |
| **Storage Medium** | SQLite (Drizzle ORM) |
| **Sync Mechanism** | Event Bus (pub/sub) |

---

## 2. Session State Management

### 2.1 Session Lifecycle Patterns

#### Pattern A: Three-Phase Lifecycle (IronClaw)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/ironclaw/ironclaw-adr.md`

```rust
// Session Lifecycle: Init → Active → Cleanup
pub struct SessionManager {
    sessions: HashMap<SessionId, Session>,
    persistence: Arc<dyn SessionPersistence>,
}

impl SessionManager {
    pub async fn create_session(&self, config: SessionConfig) -> Session {
        // 1. Initialize session
        // 2. Load user context from MEMORY.md
        // 3. Setup vector search index
    }

    pub async fn archive_session(&self, id: SessionId) {
        // 1. Flush pending writes
        // 2. Update metadata
        // 3. Archive to long-term storage
    }
}
```

#### Pattern B: D-Mail Checkpoint System (kimi-cli)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/kimi-cli/kimi-cli-adr.md`

```python
# src/kimi_cli/soul/context.py
class Context:
    def create_checkpoint(self) -> int:
        """Create a checkpoint for time travel."""
        checkpoint_id = self.next_checkpoint_id
        self.checkpoints[checkpoint_id] = Checkpoint(
            messages=self.messages.copy(),
            state=self.state.copy(),
            timestamp=time.time()
        )
        return checkpoint_id

    def revert_to(self, checkpoint_id: int) -> None:
        """Roll back to checkpoint state."""
        checkpoint = self.checkpoints.get(checkpoint_id)
        if checkpoint:
            self.messages = checkpoint.messages
            self.state = checkpoint.state

class DMail(BaseModel):
    """Send message to past checkpoints."""
    message: str
    checkpoint_id: int
```

**Session Directory Structure:**
```
~/.kimi/sessions/{workdir_md5}/
└── {session_id}/
    ├── context.jsonl      # Conversation history
    ├── wire.jsonl         # Protocol events (D-Mail)
    └── state.json         # Session configuration
```

#### Pattern C: Fork/Branch Support (opencode)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/opencode/opencode-adr.md`

```typescript
// Database Schema with Fork Support
export const SessionTable = sqliteTable("session", {
  id: text().$type<SessionID>().primaryKey(),
  project_id: text().$type<ProjectID>().notNull(),
  parent_id: text().$type<SessionID>(),  // Parent session for forks
  title: text().notNull(),
  summary_additions: integer(),
  summary_deletions: integer(),
  time_compacting: integer(),  // Last compaction time
  time_archived: integer(),    // Archive time
});

// Fork Implementation
export async function forkSession(
  parentId: SessionID,
  options?: ForkOptions
): Promise<Session> {
  const parent = await Session.get(parentId);
  const newSession = await Session.create({
    parent_id: parentId,
    project_id: parent.project_id,
    title: `${parent.title} (fork)`,
  });

  if (options?.includeHistory) {
    // Copy message history
    const messages = await Message.list({ session_id: parentId });
    for (const msg of messages) {
      await Message.create({ session_id: newSession.id, data: msg.data });
    }
  }
  return newSession;
}
```

### 2.2 Context Window Management

#### Three-Zone Context Model (CoPaw Pattern)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/copaw/copaw-adr.md`

```python
# Three-Zone Model Configuration
max_input_length: int = 128 * 1024        # 128K tokens
memory_compact_ratio: float = 0.75        # 75% trigger
memory_reserve_ratio: float = 0.1         # 10% reserved

# Zone Layout:
# ┌─────────────────────────────────────────┐
# │ System Prompt (Fixed)                   │ ← Always retained
# ├─────────────────────────────────────────┤
# │ Compacted Summary (Optional)            │ ← Generated
# ├─────────────────────────────────────────┤
# │ Compactable Zone                        │ ← Compressed
# ├─────────────────────────────────────────┤
# │ Reserved Zone (Recent N messages)       │ ← Always retained
# └─────────────────────────────────────────┘
```

#### Reverse Token Budget (Gemini-CLI Pattern)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/gemini-cli/gemini-cli-adr.md`

```typescript
// ADR-011: Reverse Token Budget Strategy
const tokenBudget = {
  systemInstruction: 0.1,  // 10% system prompt
  recentMessages: 0.5,     // 50% recent messages
  olderMessages: 0.3,      // 30% historical (may be compressed)
  functionResponses: 0.1,  // 10% tool responses
};
```

#### Compaction with Part Preservation (opencode Pattern)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/opencode/opencode-adr.md`

```typescript
// Part types include CompactionPart
export const PartTable = sqliteTable("part", {
  id: text().$type<PartID>().primaryKey(),
  message_id: text().$type<MessageID>().notNull(),
  session_id: text().$type<SessionID>().notNull(),
  data: text({ mode: "json" }).$type<PartData>(),
  // Part types: TextPart, ToolCallPart, ToolResultPart,
  //             FilePart, ImagePart, CompactionPart
});

// Compaction Detection
const COMPACTION_BUFFER = 20_000;
export async function isOverflow(input) {
  const count = input.tokens.total || input.tokens.input + input.tokens.output;
  const reserved = config.compaction?.reserved ??
    Math.min(COMPACTION_BUFFER, maxOutputTokens(input.model));
  const usable = input.model.limit.input
    ? input.model.limit.input - reserved
    : input.model.limit.context - maxOutputTokens(input.model);
  return count >= usable;
}
```

---

## 3. Boundary Management Patterns

### 3.1 Session-State Boundary Architecture

The boundary between ephemeral session state and persistent memory is handled differently across projects:

#### Clear Separation (opencode)

```
┌────────────────────────────────────────────────────────────┐
│                    Session 运行时                          │
│  ┌─────────────────┐      ┌────────────────────────────┐  │
│  │   Session 对象   │◀────▶│   Event Bus  (发布-订阅)    │  │
│  │   (内存状态)     │      │                            │  │
│  └─────────────────┘      └────────────────────────────┘  │
│           │                           │                   │
│           │ 读写                     │ 事件通知          │
│           ▼                           ▼                   │
│  ┌─────────────────┐      ┌────────────────────────────┐  │
│  │   Database 层    │◀────▶│   Effect 系统              │  │
│  │  (Drizzle ORM)   │      │   (副作用管理)              │  │
│  └─────────────────┘      └────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│                      持久化存储层                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │SessionTable │  │MessageTable │  │  PartTable  │        │
│  │  (会话元数据) │  │  (消息数据)  │  │  (片段数据)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└────────────────────────────────────────────────────────────┘
```

#### File-Based Separation (kimi-cli)

```
~/.kimi/sessions/{workdir_md5}/
└── {session_id}/
    ├── context.jsonl      # Message history (append-only)
    ├── wire.jsonl         # Event log (for replay/D-Mail)
    └── state.json         # Session configuration
```

| File | Content | Purpose |
|------|---------|---------|
| `context.jsonl` | Dialogue messages | LLM context building |
| `wire.jsonl` | Runtime events | Debugging, visualization, D-Mail |
| `state.json` | Session config | State recovery |

### 3.2 Synchronization Strategies

#### Write-Through vs Write-Back Comparison

| Strategy | Projects | Pros | Cons |
|----------|----------|------|------|
| **Write-Through** | opencode, ZeroClaw | Immediate consistency | Higher latency |
| **Write-Back (Debounced)** | OpenClaw | Batch efficiency | Potential data loss |
| **Append-Only** | kimi-cli, Pi-Mono | Simplicity, durability | No random access |
| **Trigger-Based** | CoPaw | Efficient, event-driven | Complex trigger management |

#### Transaction Safety (IronClaw)

**File:** `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/ironclaw/ironclaw-adr.md`

```rust
// ADR-016: Transaction Safety
pub struct TransactionManager {
    db: DatabaseConnection,
    pending: Vec<Operation>,
}

impl TransactionManager {
    pub async fn commit(&self) -> Result<(), Error> {
        // Two-phase commit
        // 1. Write to WAL
        // 2. Update main database
        // 3. Cleanup WAL
    }

    pub async fn rollback(&self) -> Result<(), Error> {
        // Discard pending operations
    }
}
```

### 3.3 Recovery Mechanisms

#### Layered Recovery (Recommended Pattern)

```
Recovery Flow:
1. Session Metadata from SQLite (fast load)
2. Message History stream-loaded from JSONL (pagination)
3. Vector Index rebuilt asynchronously (background thread)
```

#### opencode Recovery

```typescript
// From SQLite
export async function resumeSession(sessionId: SessionID): Promise<Session> {
  // 1. Session metadata
  const session = await Session.get(sessionId);

  // 2. Message list
  const messages = await Message.list({
    session_id: sessionId,
    orderBy: "time_created",
  });

  // 3. Parts (streaming fragments)
  for (const msg of messages) {
    msg.parts = await Part.list({ message_id: msg.id });
  }

  // 4. Rebuild memory state
  return new Session({ ...session, messages });
}
```

---

## 4. Project-by-Project Analysis

### 4.1 Personal Assistants

#### CoPaw

**Memory Architecture:**
- **Storage:** SQLite (ReMeLight) + Chroma (vectors) + Markdown files
- **Pattern:** Three-layer memory (In-Memory → Daily Notes → MEMORY.md)
- **RAG:** Hybrid search (Vector 0.7 + BM25 0.3)
- **Compaction:** Token threshold at 75%

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/copaw/copaw-adr.md`
- Code: `src/copaw/agents/hooks/memory_compaction.py`
- Code: `src/copaw/memory/memory_manager.py`

**Session-State Boundary:**
```
InMemoryMemory (session) → MemoryManager → ReMeLightStorage (persistent)
```

#### IronClaw

**Memory Architecture:**
- **Storage:** Dual-backend (PostgreSQL + libSQL/Turso)
- **Pattern:** Protected identity files (SOUL.md, AGENTS.md, USER.md)
- **RAG:** Hybrid RRF search (FTS5 + Vector) with MMR reranking
- **Transaction:** ADR-016 Transaction Safety

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/ironclaw/ironclaw-adr.md`

**Protected Files Mechanism:**
```rust
// AGENTS.md, SOUL.md, USER.md, IDENTITY.md are PROTECTED
// Cannot be auto-modified without user confirmation
```

#### NanoClaw

**Memory Architecture:**
- **Storage:** SQLite + Markdown files
- **Pattern:** File System IPC, Dual cursor message tracking
- **RAG:** Keyword search only
- **Special:** Container isolation for skills

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/nanoclaw/nanoclaw-adr.md`

#### OpenClaw

**Memory Architecture:**
- **Storage:** SQLite + sqlite-vec + Markdown files
- **Pattern:** Multi-agent workspace isolation
- **RAG:** Hybrid with MMR and temporal decay
- **Sync:** Multi-trigger (watcher + timer + lifecycle)

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/openclaw/openclaw-adr.md`

**Post-Compaction Reinjection:**
```typescript
// Re-inject critical sections after compaction
const DEFAULT_POST_COMPACTION_SECTIONS = [
  "Session Startup",
  "Red Lines"
];
```

#### PicoClaw

**Memory Architecture:**
- **Storage:** Markdown files only
- **Pattern:** Date-organized (YYYY/MM/YYYYMMDD.md)
- **RAG:** File-based search
- **Session:** JSONL storage

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/picoclaw/picoclaw-adr.md`

#### ZeroClaw

**Memory Architecture:**
- **Storage:** SQLite (categorized)
- **Pattern:** Trait-driven architecture
- **Categories:** Core, Daily, Conversation, Custom

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/zeroclaw/zeroclaw-adr.md`

#### Nanobot

**Memory Architecture:**
- **Storage:** Markdown files (MEMORY.md + HISTORY.md)
- **Pattern:** Two-layer memory with JSONL sessions
- **RAG:** Grep-based search
- **Sync:** MessageBus pub/sub

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/nanobot/nanobot-adr.md`

#### MimiClaw

**Memory Architecture:**
- **Storage:** SPIFFS (ESP32 flash filesystem)
- **Pattern:** Embedded resource-constrained
- **RAG:** None (hardware limited)

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/personal-general-ai-assistant/mimiclaw/mimiclaw-adr.md`

### 4.2 Programming Agents

#### Aider

**Context Architecture:**
- **Storage:** Git-based + In-memory
- **Pattern:** RepoMap for codebase awareness
- **Context:** Chat history + summaries + repository map
- **No RAG:** Relies on RepoMap and file inclusion

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/aider/aider-adr.md`

#### Codex

**Memory Architecture:**
- **Storage:** JSONL traces + SQLite
- **Pattern:** Memory traces with session telemetry
- **Protocol:** SQ/EQ (State Query/Event Queue)

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/codex/codex-adr.md`

#### Qwen-Code

**Session Architecture:**
- **Storage:** JSONL
- **Pattern:** ACP (Agent Communication Protocol)
- **Monorepo:** npm workspaces with 6 packages

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/qwen-code/qwen-code-adr.md`

#### Gemini-CLI

**Memory Architecture:**
- **Storage:** JSON (hierarchical)
- **Pattern:** Hierarchical memory (global/extension/project)
- **Context:** JIT discovery based on accessed paths

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/gemini-cli/gemini-cli-adr.md`

**State Machine Scheduler:**
```typescript
// Tool call lifecycle: Validating → Scheduled → Executing → Success/Error
class Scheduler {
  async schedule(request: ToolCallRequestInfo[]): Promise<CompletedToolCall[]>
}
```

#### Kimi-CLI

**Session Architecture:**
- **Storage:** JSONL (context.jsonl, wire.jsonl)
- **Pattern:** D-Mail checkpoint system
- **Special:** Time-travel via checkpoints

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/kimi-cli/kimi-cli-adr.md`

#### Opencode

**Memory Architecture:**
- **Storage:** SQLite (Drizzle ORM)
- **Pattern:** Structured tables (Session/Message/Part)
- **Compaction:** CompactionPart type for overflow
- **Fork:** Parent-child session relationships

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/opencode/opencode-adr.md`

#### Pi-Mono

**Session Architecture:**
- **Storage:** JSONL dual files
- **Pattern:** Two-file approach (context.jsonl + log.jsonl)
- **Monorepo:** 7 packages with npm workspaces

**Key Files:**
- `/Users/luarassassin/reference-projects-me/Omni-Agent-OS/forge-code/pi-mono/pi-mono-adr.md`

---

## 5. Comparative Matrices

### 5.1 Memory Systems Comparison

| Project | Layer 1 (Working) | Layer 2 (Short-Term) | Layer 3 (Long-Term) | RAG | Compaction |
|---------|-------------------|----------------------|---------------------|-----|------------|
| **CoPaw** | InMemoryMemory | memory/*.md daily | MEMORY.md | Hybrid V+BM25 | Token threshold |
| **IronClaw** | Session struct | Session JSONL | MEMORY.md + vectors | Hybrid RRF | Configurable |
| **NanoClaw** | In-memory | JSONL | MEMORY.md | Keyword | Manual |
| **OpenClaw** | SessionState | sessionDeltas | Vector index + FTS | Hybrid + MMR | Debounced |
| **PicoClaw** | In-memory | YYYY/MM/*.md | MEMORY.md | File search | Manual |
| **ZeroClaw** | MemoryStore | SQLite | SQLite | SQL search | Manual |
| **Nanobot** | In-memory | JSONL | MEMORY.md | Grep | Auto |
| **MimiClaw** | C structs | SPIFFS | Limited | None | Manual |
| **Aider** | Coder class | Git history | None | None | Summarize |
| **Codex** | MemoryTrace | JSONL traces | SQLite | None | Traces |
| **Qwen-code** | Session | JSONL | - | None | Manual |
| **Gemini-cli** | In-memory | JSON | Hierarchical | None | JIT discovery |
| **Kimi-cli** | Session | context.jsonl | - | None | Checkpoint |
| **Opencode** | Session | SQLite | - | None | CompactionPart |
| **Pi-Mono** | Agent state | Dual JSONL | - | None | Manual |

### 5.2 Session Storage Formats

| Project | Primary Format | Secondary Format | Schema | Fork Support |
|---------|----------------|------------------|--------|--------------|
| **CoPaw** | SQLite | Markdown | Flexible | No |
| **IronClaw** | SQLite/PostgreSQL | Markdown | Structured | No |
| **NanoClaw** | SQLite | Markdown | Fixed | No |
| **OpenClaw** | SQLite | JSONL | Dynamic | No |
| **PicoClaw** | Markdown | JSONL | Simple | No |
| **ZeroClaw** | SQLite | - | Categorized | No |
| **Nanobot** | Markdown | JSONL | Simple | No |
| **Aider** | Git | In-memory | Git-based | No |
| **Codex** | JSONL | SQLite | Event-based | No |
| **Qwen-code** | JSONL | - | ACP protocol | No |
| **Gemini-cli** | JSON | - | Hierarchical | No |
| **Kimi-cli** | JSONL | JSON | Wire protocol | Yes (Checkpoint) |
| **Opencode** | SQLite | - | Drizzle ORM | Yes |
| **Pi-Mono** | JSONL | - | Dual files | No |

### 5.3 Write Strategy Comparison

| Project | Strategy | Trigger | Granularity | Sync Mechanism |
|---------|----------|---------|-------------|----------------|
| **CoPaw** | Condition-based | Token threshold 75% | Message batch | Async hooks |
| **IronClaw** | Transactional | Session lifecycle | Record | 2PC |
| **NanoClaw** | Real-time | Message | Message | Direct SQL |
| **OpenClaw** | Multi-trigger | Watcher + timer | Chunk | Debounced |
| **PicoClaw** | Append-only | Message | Entry | File I/O |
| **ZeroClaw** | Transactional | Operation | Record | Write-through |
| **Nanobot** | Event-driven | MessageBus | Message | Pub/sub |
| **Aider** | Git-based | Commit | File | Git |
| **Codex** | Append-only | Event | Event | File I/O |
| **Qwen-code** | Real-time | Message | Message | Direct |
| **Gemini-cli** | State machine | Tool lifecycle | Tool call | Event-driven |
| **Kimi-cli** | Append-only | Message | Message | Async file |
| **Opencode** | Transactional | Message/Part | Part | Event Bus |
| **Pi-Mono** | Dual-write | Turn | Message | Sync |

### 5.4 Boundary Management Comparison

| Project | Session-Persistent Boundary | Recovery Strategy | Concurrency | Safety |
|---------|----------------------------|-------------------|-------------|--------|
| **CoPaw** | InMemoryMemory ↔ ReMeLight | Summary + reserved | Async | Hook-based |
| **IronClaw** | Workspace ↔ Database | Full restore | Transaction | 2PC |
| **NanoClaw** | Container ↔ Files | JSONL replay | Isolated | File locks |
| **OpenClaw** | SessionState ↔ Index | Delta replay | Debounced | WAL |
| **ZeroClaw** | Trait ↔ SQLite | Query rebuild | Pool | Transactions |
| **Nanobot** | MemoryBus ↔ Files | Full scan | Pub/sub | Append-only |
| **Kimi-cli** | Session ↔ JSONL | Stream load | Async | Immutable |
| **Opencode** | Session ↔ Drizzle | Layered | Event Bus | FK constraints |
| **Gemini-cli** | Scheduler ↔ State | State machine | EventEmitter | Immutable |

---

## 6. Recommended Unified Architecture

### 6.1 Fusion Memory Architecture

Based on analysis of all 15 projects:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Unified Memory Architecture                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Soul Definition (Human-Readable, Protected)            │
│   ├── SOUL.md          - Core identity, values                  │
│   ├── AGENTS.md        - Operational manual                     │
│   ├── USER.md          - User profile, preferences              │
│   └── IDENTITY.md      - Agent name, avatar                     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Long-Term Memory (Agent-Managed, Searchable)           │
│   ├── MEMORY.md        - Curated facts, decisions               │
│   ├── memory/*.md      - Daily logs (YYYY-MM-DD.md)             │
│   └── Vector Index     - Semantic search layer                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Session Storage (Structured, Queryable)                │
│   ├── SQLite/PostgreSQL                                         │
│   │   ├── sessions table   - Session metadata                   │
│   │   ├── messages table   - Message history                    │
│   │   └── parts table      - Tool calls, files, images          │
│   └── JSONL backup       - Portability, human readability       │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Working Memory (In-Memory, Ephemeral)                  │
│   ├── Current conversation                                      │
│   ├── Tool call results                                         │
│   └── Context window management                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Unified RAG Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Unified RAG Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│ 1. Document Processing                                          │
│    ├── Chunk Size: 400 tokens                                   │
│    ├── Overlap: 80 tokens (20%)                                 │
│    └── Boundary: Paragraph/sentence-aware                       │
├─────────────────────────────────────────────────────────────────┤
│ 2. Multi-Route Retrieval                                        │
│    ├── Vector Search (semantic similarity)                      │
│    │   └── Embedding: text-embedding-3-small/large              │
│    ├── BM25/FTS5 (keyword matching)                             │
│    │   └── Tokenization: Unicode-aware                          │
│    └── Hybrid Fusion (RRF)                                      │
│        └── Weights: Vector 0.7, BM25 0.3                        │
├─────────────────────────────────────────────────────────────────┤
│ 3. Result Processing                                            │
│    ├── MMR Reranking (diversity = 3, lambda = 0.7)              │
│    ├── Temporal Decay (half-life = 30 days)                     │
│    └── Citation Generation (file path + line/paragraph)         │
├─────────────────────────────────────────────────────────────────┤
│ 4. Memory Tools                                                 │
│    ├── memory_search(query, filters, k)                         │
│    ├── memory_write(content, category, tags)                    │
│    ├── memory_read(id/query)                                    │
│    └── memory_tree(path)                                        │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Unified Session State Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Unified Session Architecture                 │
├─────────────────────────────────────────────────────────────────┤
│ Configuration                                                   │
│   ├── max_context_tokens: 128000                                │
│   ├── compact_trigger_ratio: 0.75                               │
│   ├── reserve_ratio: 0.1                                        │
│   └── compaction_buffer: 20000                                  │
├─────────────────────────────────────────────────────────────────┤
│ Zones                                                           │
│   ├── System Prompt (fixed, protected)                          │
│   ├── Compacted Summary (dynamic)                               │
│   ├── Active Conversation (sliding window)                      │
│   └── Reserved (recent N messages)                              │
├─────────────────────────────────────────────────────────────────┤
│ Write Strategy                                                  │
│   ├── Real-time: Session metadata, message headers              │
│   ├── Debounced: Message content (1s batch)                     │
│   ├── Async: Vector embeddings                                  │
│   └── Triggered: Compaction, archive                            │
├─────────────────────────────────────────────────────────────────┤
│ Post-Compaction Actions                                         │
│   ├── Reinject SOUL.md key principles                           │
│   ├── Reinject AGENTS.md red lines                              │
│   └── Update compaction summary                                 │
├─────────────────────────────────────────────────────────────────┤
│ Fork/Checkpoint Support                                         │
│   ├── Parent session reference                                  │
│   ├── Optional history copy                                     │
│   └── Branch metadata                                           │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4 Implementation Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Soul Files** | SOUL.md + AGENTS.md + USER.md | CoPaw/IronClaw pattern, human-readable |
| **Storage** | SQLite + Markdown files | IronClaw/OpenClaw dual approach |
| **Vector DB** | Chroma/pgvector | CoPaw/IronClaw proven solutions |
| **Embeddings** | text-embedding-3-small | OpenAI standard, good quality/cost |
| **Context** | Three-zone model | CoPaw proven approach |
| **Compaction** | Summary + reinjection | OpenClaw preserves critical context |
| **Session** | SQLite + JSONL dual | Opencode/Pi-mono portability |
| **Tools** | memory_search/write/read/tree | IronClaw complete toolset |
| **Sync** | Event Bus + Debounced | OpenClaw flexibility + performance |
| **Safety** | WAL + Transactions | IronClaw transaction safety |

---

## 7. Key Code Snippets by Project

### 7.1 CoPaw Memory Compaction

```python
# src/copaw/agents/hooks/memory_compaction.py
class MemoryCompactionHook:
    async def __call__(self, agent: ReActAgent, kwargs: dict):
        token_count = agent.memory.get_token_count()
        threshold = agent.max_input_length * MEMORY_COMPACT_RATIO

        if token_count > threshold:
            messages = agent.memory.get_messages()
            reserved_count = max(3, int(len(messages) * MEMORY_RESERVE_RATIO))
            to_compact = messages[:-reserved_count]

            summary = await agent.memory_manager.compact_memory(to_compact)
            await agent.memory.update_compressed_summary(summary)
```

### 7.2 IronClaw Hybrid Search

```rust
// src/workspace/search.rs
pub struct HybridSearchConfig {
    pub vector_weight: f64,     // 0.7 default
    pub fts_weight: f64,        // 0.3 default
    pub mmr_lambda: f64,        // 0.5 for diversity
}

// Reciprocal Rank Fusion
fn rrf_fusion(vector_results: Vec<SearchResult>,
              fts_results: Vec<SearchResult>) -> Vec<SearchResult> {
    // RRF score = sum(1.0 / (k + rank)) for each list
}
```

### 7.3 OpenClaw Debounced Sync

```typescript
// src/memory/manager.ts
class MemoryManager {
  private syncTimer?: NodeJS.Timeout;

  constructor() {
    // File Watcher trigger
    this.watcher = chokidar.watch(dirs).on('change', () => {
      this.debouncedSync({ reason: "file-change" });
    });

    // Timer trigger
    this.ensureIntervalSync();
  }

  private debouncedSync(params: SyncParams) {
    clearTimeout(this.syncTimer);
    this.syncTimer = setTimeout(() => this.sync(params), 1000);
  }
}
```

### 7.4 opencode Fork Session

```typescript
// src/session/fork.ts
export async function forkSession(
  parentId: SessionID,
  options?: ForkOptions
): Promise<Session> {
  const parent = await Session.get(parentId);

  const newSession = await Session.create({
    parent_id: parentId,
    project_id: parent.project_id,
    title: `${parent.title} (fork)`,
  });

  if (options?.includeHistory) {
    const messages = await Message.list({ session_id: parentId });
    for (const msg of messages) {
      await Message.create({
        session_id: newSession.id,
        data: msg.data,
      });
    }
  }

  return newSession;
}
```

### 7.5 kimi-cli D-Mail

```python
# src/kimi_cli/soul/dmail.py
class DMail(BaseModel):
    """Send message to past checkpoints."""
    message: str
    checkpoint_id: int

class DenwaRenji:
    def send_dmail(self, dmail: DMail):
        # D-Mail as Wire event
        self.wire_file.append_message(dmail.to_wire_message())
        self._pending_dmail = dmail
```

### 7.6 Gemini-CLI State Machine

```typescript
// packages/core/src/services/scheduler.ts
class Scheduler {
  private state: Map<string, ToolCallState> = new Map();

  async schedule(requests: ToolCallRequestInfo[]): Promise<CompletedToolCall[]> {
    for (const req of requests) {
      this.state.set(req.id, ToolCallState.VALIDATING);
      await this.validate(req);

      this.state.set(req.id, ToolCallState.SCHEDULED);
      await this.scheduleExecution(req);
    }
  }

  // State transitions: Validating → Scheduled → Executing → Success/Error
}
```

---

## 8. Summary

### 8.1 Key Insights

1. **Personal assistants** favor hybrid RAG (vector + BM25) with file-based soul definitions
2. **Programming agents** prioritize context window management over retrieval (minimal RAG)
3. **Protected identity files** (IronClaw pattern) prevent agent self-corruption
4. **Post-compaction reinjection** (OpenClaw pattern) maintains context coherence
5. **Dual storage** (SQLite + JSONL) balances query performance with portability

### 8.2 Architectural Strengths by Project

| Project | Memory Strength | Session Strength | Boundary Strength |
|---------|-----------------|------------------|-------------------|
| **CoPaw** | Three-layer, hybrid RAG | Token-threshold compaction | Hook-based async |
| **IronClaw** | Protected files, dual-backend | Transaction safety | 2PC commit |
| **OpenClaw** | Multi-agent isolation | Debounced multi-trigger | Delta sync |
| **kimi-cli** | Checkpoint system | D-Mail time travel | Immutable logs |
| **opencode** | Structured SQLite | Fork support | Event Bus |
| **Gemini-cli** | Hierarchical memory | State machine scheduler | Event-driven |

### 8.3 Final Recommendation

The optimal architecture synthesizes:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Final Recommended Architecture               │
├─────────────────────────────────────────────────────────────────┤
│ 1. Soul: SOUL.md (identity) + AGENTS.md (manual) + USER.md     │
│ 2. Storage: Markdown files + SQLite/PostgreSQL dual-backend    │
│ 3. RAG: Vector 0.7 + BM25 0.3 + MMR reranking + time decay     │
│ 4. Context: Three-zone model + post-compaction reinjection     │
│ 5. Session: JSONL for portability + SQLite for querying        │
│ 6. Sync: Event Bus + Debounced + Transaction safety            │
│ 7. Tools: memory_search/write/read/tree with citations         │
│ 8. Fork: Parent-child relationships with optional history      │
└─────────────────────────────────────────────────────────────────┘
```

---

*Document Version: 1.0*
*Generated: 2026-03-18*
*Projects Analyzed: 15 (8 Personal Assistants + 7 Programming Agents)*
