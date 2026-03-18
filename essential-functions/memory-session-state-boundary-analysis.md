# Memory Systems, Session State & Boundary Management Analysis
## Across 15 AI Agent Projects in Omni-Agent-OS

**Scope:** 8 Personal Assistants + 7 Programming Agents
**Analysis Dimensions:** Memory architecture, Session lifecycle, State boundaries, Synchronization patterns

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Memory Systems Architecture](#memory-systems-architecture)
3. [Session State Management](#session-state-management)
4. [Boundary Management Patterns](#boundary-management-patterns)
5. [Project-by-Project Analysis](#project-by-project-analysis)
6. [Comparative Matrices](#comparative-matrices)
7. [Recommended Unified Architecture](#recommended-unified-architecture)

---

## Executive Summary

This analysis examines memory persistence strategies and session-state boundaries across 15 AI agent implementations. Two distinct architectural philosophies emerge:

| Category | Primary Focus | Storage Pattern | RAG Strategy |
|----------|---------------|-----------------|--------------|
| **Personal Assistants** (8) | Long-term memory, user context continuity | SQLite + Markdown + Vector DB | Hybrid (Vector + BM25/FTS) |
| **Programming Agents** (7) | Context window efficiency, session persistence | JSONL + SQLite | Minimal/None (focus on RepoMap/context) |

### Key Architectural Patterns Discovered

**Memory Write Strategies:**
1. **Token Threshold-Based** (CoPaw) - Compaction triggers at 75% capacity
2. **Real-Time Append-Only** (kimi-cli, Pi-Mono) - Immediate JSONL persistence
3. **Debounced Multi-Trigger** (OpenClaw) - Watcher + timer + lifecycle events
4. **Transactional Event-Driven** (opencode, IronClaw) - ACID with pub/sub

**Session-State Boundary Models:**
1. **Clear Layer Separation** (opencode) - Session ↔ Message ↔ Part tables
2. **File-Based Separation** (kimi-cli) - context.jsonl / wire.jsonl / state.json
3. **Delta Synchronization** (OpenClaw) - SessionState ↔ MemoryIndexManager
4. **Protected Identity Files** (IronClaw) - SOUL.md/AGENTS.md/USER.md are immutable

---

## Memory Systems Architecture

### The Three-Layer Memory Model

All sophisticated agent systems implement layered memory:

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: Working Memory (Session Context)                       │
│          └── Current conversation, tool calls, immediate state  │
│          └── Lives in RAM, ephemeral, highest bandwidth         │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Short-Term Memory (Recent History)                     │
│          └── Daily logs, session summaries, recent facts        │
│          └── JSONL files or daily Markdown (YYYY-MM-DD.md)      │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Long-Term Memory (Persistent Knowledge)                │
│          └── Curated facts, user preferences, decisions         │
│          └── MEMORY.md + vector embeddings + FTS index          │
└─────────────────────────────────────────────────────────────────┘
```

### Memory Write Strategy Patterns

#### Pattern 1: Token Threshold-Based (CoPaw)

**Source:** `personal-general-ai-assistant/copaw/copaw-adr.md`

```python
# src/copaw/agents/hooks/memory_compaction.py
class MemoryCompactionHook:
    async def __call__(self, agent: ReActAgent, kwargs: dict):
        token_count = agent.memory.get_token_count()
        threshold = agent.max_input_length * MEMORY_COMPACT_RATIO  # 0.75

        if token_count > threshold:
            messages = agent.memory.get_messages()
            reserved_count = max(3, int(len(messages) * MEMORY_RESERVE_RATIO))
            to_compact = messages[:-reserved_count]

            summary = await agent.memory_manager.compact_memory(to_compact)
            await agent.memory.update_compressed_summary(summary)
```

| Dimension | Implementation |
|-----------|----------------|
| **Trigger** | Token threshold at 75% of max context |
| **Content** | Message summaries, vector embeddings |
| **Storage** | SQLite (ReMeLight) + Chroma (vectors) |
| **Sync** | Hook-based async processing |

#### Pattern 2: Real-Time Append-Only (kimi-cli)

**Source:** `forge-code/kimi-cli/kimi-cli-adr.md`

```python
# src/kimi_cli/wire/file.py
async def append_record(self, record: WireMessageRecord) -> None:
    self.path.parent.mkdir(parents=True, exist_ok=True)
    needs_header = not self.path.exists() or self.path.stat().st_size == 0
    async with aiofiles.open(self.path, mode="a", encoding="utf-8") as f:
        if needs_header:
            metadata = WireFileMetadata(protocol_version=self.protocol_version)
            await f.write(_dump_line(metadata))
        await f.write(_dump_line(record))  # Real-time append
```

**Session Directory Structure:**
```
~/.kimi/sessions/{workdir_md5}/
└── {session_id}/
    ├── context.jsonl      # Conversation history (append-only)
    ├── wire.jsonl         # Protocol events for D-Mail/replay
    └── state.json         # Session configuration
```

| File | Content | Purpose |
|------|---------|---------|
| `context.jsonl` | Dialogue messages | LLM context building |
| `wire.jsonl` | Runtime events | Debugging, visualization, D-Mail |
| `state.json` | Session config | State recovery |

#### Pattern 3: Debounced Multi-Trigger (OpenClaw)

**Source:** `personal-general-ai-assistant/openclaw/openclaw-adr.md`

```typescript
// src/memory/manager.ts
class MemoryManager {
  private syncTimer?: NodeJS.Timeout;
  private watcher: FSWatcher;

  constructor() {
    // Trigger 1: File Watcher
    this.watcher = chokidar.watch(dirs).on('change', () => {
      this.debouncedSync({ reason: "file-change" });
    });

    // Trigger 2: Timer
    this.ensureIntervalSync();
  }

  // Trigger 3: Session lifecycle
  async warmSession(sessionKey: string) {
    await this.sync({ reason: "session-start" });
  }

  private debouncedSync(params: SyncParams) {
    clearTimeout(this.syncTimer);
    this.syncTimer = setTimeout(() => this.sync(params), 1000);
  }
}
```

| Dimension | Implementation |
|-----------|----------------|
| **Trigger** | File watcher + timer + session lifecycle |
| **Content** | Vector embeddings, FTS index, file chunks |
| **Storage** | SQLite + sqlite-vec extension |
| **Sync** | Debounced batch with readonly recovery |

#### Pattern 4: Transactional Event-Driven (opencode)

**Source:** `forge-code/opencode/opencode-adr.md`

```typescript
// src/session/index.ts
export const updateMessage = fn(MessageV2.Info, async (msg) => {
  const { id, sessionID, ...data } = msg;
  Database.use((db) => {
    db.insert(MessageTable)
      .values({ id, session_id: sessionID, data })
      .onConflictDoUpdate({ target: MessageTable.id, set: { data } })
      .run();
    // Publish event after transaction
    Database.effect(() =>
      Bus.publish(MessageV2.Event.Updated, { info: msg }),
    );
  });
  return msg;
});
```

**Event Bus Pattern:**
```typescript
class Database {
  static effects: (() => void)[] = [];

  static use<T>(fn: (db: DrizzleDB) => T): T {
    const result = fn(this.db);
    this.flushEffects();  // Execute after transaction
    return result;
  }

  static effect(fn: () => void): void {
    this.effects.push(fn);
  }
}
```

| Dimension | Implementation |
|-----------|----------------|
| **Trigger** | Per-message/part real-time |
| **Content** | Messages, Parts, metadata, summaries |
| **Storage** | SQLite (Drizzle ORM) |
| **Sync** | Event Bus (pub/sub) with transaction boundary |

---

## Session State Management

### Session Lifecycle Patterns

#### Pattern A: Three-Phase Lifecycle (IronClaw)

**Source:** `personal-general-ai-assistant/ironclaw/ironclaw-adr.md`

```rust
// Session Lifecycle: Init → Active → Cleanup
pub struct SessionManager {
    sessions: HashMap<SessionId, Session>,
    persistence: Arc<dyn SessionPersistence>,
}

impl SessionManager {
    pub async fn create_session(&self, config: SessionConfig) -> Session {
        // 1. Initialize session with unique ID
        // 2. Load user context from MEMORY.md
        // 3. Setup vector search index
    }

    pub async fn archive_session(&self, id: SessionId) {
        // 1. Flush pending writes to persistence
        // 2. Update session metadata
        // 3. Archive to long-term storage
    }
}
```

#### Pattern B: D-Mail Checkpoint System (kimi-cli)

**Source:** `forge-code/kimi-cli/kimi-cli-adr.md`

```python
# src/kimi_cli/soul/context.py
class Context:
    def create_checkpoint(self) -> int:
        """Create a checkpoint for time travel (D-Mail)."""
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

**D-Mail Time Travel Flow:**
```
User creates checkpoint ──► Checkpoint N saved
                               │
User continues working ──► Messages accumulate
                               │
User sends D-Mail ───────► Message delivered to Checkpoint N
                               │
Agent at Checkpoint N ──► Receives "message from future"
```

#### Pattern C: Fork/Branch Support (opencode)

**Source:** `forge-code/opencode/opencode-adr.md`

```typescript
// Database Schema with Fork Support
export const SessionTable = sqliteTable("session", {
  id: text().$type<SessionID>().primaryKey(),
  project_id: text().$type<ProjectID>().notNull(),
  parent_id: text().$type<SessionID>(),  // Parent for forks
  title: text().notNull(),
  summary_additions: integer(),
  summary_deletions: integer(),
  time_compacting: integer(),
  revert: text({ mode: "json" }).$type<{ messageID: MessageID }>(),
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
    const messages = await Message.list({ session_id: parentId });
    for (const msg of messages) {
      await Message.create({ session_id: newSession.id, data: msg.data });
    }
  }
  return newSession;
}
```

### Context Window Management Patterns

#### Three-Zone Model (CoPaw)

```
┌─────────────────────────────────────────┐
│ System Prompt (Fixed)                   │ ← Always retained
├─────────────────────────────────────────┤
│ Compacted Summary (Dynamic)             │ ← Generated from compaction
├─────────────────────────────────────────┤
│ Compactable Zone (Compressible)         │ ← Summarized when threshold hit
├─────────────────────────────────────────┤
│ Reserved Zone (Recent N messages)       │ ← Always retained (10%)
└─────────────────────────────────────────┘
         ↑ 75% threshold triggers compaction
```

**Configuration:**
```python
max_input_length: int = 128 * 1024        # 128K tokens
memory_compact_ratio: float = 0.75        # 75% trigger
memory_reserve_ratio: float = 0.1         # 10% reserved
```

#### Reverse Token Budget (Gemini-CLI)

**Source:** `forge-code/gemini-cli/gemini-cli-adr.md`

```typescript
// ADR-011: Reverse Token Budget Strategy
const tokenBudget = {
  systemInstruction: 0.1,  // 10% system prompt
  recentMessages: 0.5,     // 50% recent messages (guaranteed)
  olderMessages: 0.3,      // 30% historical (may be compressed)
  functionResponses: 0.1,  // 10% tool responses
};
```

#### Compaction with Part Preservation (opencode)

```typescript
// Part types include CompactionPart
export const PartTable = sqliteTable("part", {
  id: text().$type<PartID>().primaryKey(),
  message_id: text().$type<MessageID>().notNull(),
  session_id: text().$type<SessionID>().notNull(),
  data: text({ mode: "json" }).$type<PartData>(),
  // Types: TextPart, ToolCallPart, ToolResultPart,
  //        FilePart, ImagePart, CompactionPart
});

// Compaction Detection with Reserved Buffer
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

## Boundary Management Patterns

### Session-State Boundary Architectures

#### Clear Layer Separation (opencode)

```
┌────────────────────────────────────────────────────────────┐
│                    Session Runtime                         │
│  ┌─────────────────┐      ┌────────────────────────────┐  │
│  │   Session Object│◀────▶│   Event Bus (Pub/Sub)      │  │
│  │   (In-Memory)   │      │                            │  │
│  └─────────────────┘      └────────────────────────────┘  │
│           │                           │                   │
│           │ Read/Write               │ Event Notify      │
│           ▼                           ▼                   │
│  ┌─────────────────┐      ┌────────────────────────────┐  │
│  │   Database Layer│◀────▶│   Effect System            │  │
│  │  (Drizzle ORM)  │      │   (Side Effect Mgmt)       │  │
│  └─────────────────┘      └────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│                    Persistent Storage                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │SessionTable │  │MessageTable │  │  PartTable  │        │
│  │  (Metadata) │  │  (Messages) │  │  (Fragments)│        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└────────────────────────────────────────────────────────────┘
```

#### Delta Synchronization (OpenClaw)

```
┌────────────────────────────────────────────────────────────┐
│                    Session Runtime                         │
│  ┌─────────────────┐      ┌────────────────────────────┐  │
│  │   SessionState  │─────▶│   MemoryIndexManager       │  │
│  │   (Temp Cache)  │      │   (SQLite + sqlite-vec)    │  │
│  └─────────────────┘      └────────────────────────────┘  │
│           │                           │                   │
│           │ Query Hit Cache           │ Batch Sync        │
│           ▼                           ▼                   │
│  ┌─────────────────┐      ┌────────────────────────────┐  │
│  │   sessionDeltas │◀─────│   File Watcher (chokidar)  │  │
│  │   (Delta Track) │      │   File Change Trigger      │  │
│  └─────────────────┘      └────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                    │
                    ▼ Persistent Write
┌────────────────────────────────────────────────────────────┐
│                    Persistent Storage                      │
│  ┌─────────────────┐      ┌────────────────────────────┐  │
│  │  SQLite Index   │      │   Session JSONL Files      │  │
│  │  (chunks + vec) │      │   (Per-Session Logs)       │  │
│  └─────────────────┘      └────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Synchronization Strategies Comparison

| Strategy | Projects | Pros | Cons |
|----------|----------|------|------|
| **Write-Through** | opencode, ZeroClaw | Immediate consistency | Higher latency |
| **Write-Back (Debounced)** | OpenClaw | Batch efficiency | Potential data loss on crash |
| **Append-Only** | kimi-cli, Pi-Mono | Simplicity, durability | No random access, grows unbounded |
| **Trigger-Based** | CoPaw | Efficient, event-driven | Complex trigger management |
| **Transaction-Based** | IronClaw | ACID guarantees | Higher overhead |

### Recovery Mechanisms

#### Layered Recovery (Recommended Pattern)

```
Recovery Flow:
┌─────────────────────────────────────────┐
│ 1. Session Metadata from SQLite         │ ← Fast (< 10ms)
│    (SELECT * FROM sessions WHERE id=?)  │
├─────────────────────────────────────────┤
│ 2. Message History from JSONL           │ ← Streaming
│    (Pagination: 100 messages/page)      │
├─────────────────────────────────────────┤
│ 3. Vector Index Rebuild                 │ ← Background
│    (Async, non-blocking)                │
└─────────────────────────────────────────┘
```

#### opencode Recovery Implementation

```typescript
export async function resumeSession(sessionId: SessionID): Promise<Session> {
  // Layer 1: Session metadata
  const session = await Session.get(sessionId);

  // Layer 2: Message list
  const messages = await Message.list({
    session_id: sessionId,
    orderBy: "time_created",
  });

  // Layer 3: Parts (streaming fragments)
  for (const msg of messages) {
    msg.parts = await Part.list({ message_id: msg.id });
  }

  // Rebuild memory state
  return new Session({ ...session, messages });
}
```

---

## Project-by-Project Analysis

### Personal Assistants (8)

#### CoPaw

**Memory Architecture:**
- **Storage:** SQLite (ReMeLight) + Chroma (vectors) + Markdown files
- **Pattern:** Three-layer memory with hybrid RAG
- **RAG:** Vector 0.7 + BM25 0.3 with temporal decay
- **Compaction:** Token threshold at 75%

**Session-State Boundary:**
```
InMemoryMemory (session) ──► MemoryManager ──► ReMeLightStorage (persistent)
        │                           │                    │
        │ Token threshold           │ Async hooks        │ SQLite + Chroma
        │ triggers compaction       │                    │
```

#### IronClaw

**Memory Architecture:**
- **Storage:** Dual-backend (PostgreSQL + libSQL/Turso)
- **Pattern:** Protected identity files
- **RAG:** Hybrid RRF (FTS5 + Vector) with MMR reranking
- **Transaction:** Two-phase commit (ADR-016)

**Protected Files:**
```rust
// SOUL.md, AGENTS.md, USER.md, IDENTITY.md are PROTECTED
// Cannot be auto-modified without user confirmation
const PROTECTED_FILES: &[&str] = &["SOUL.md", "AGENTS.md", "USER.md", "IDENTITY.md"];
```

#### NanoClaw

**Memory Architecture:**
- **Storage:** SQLite + Markdown files
- **Pattern:** File System IPC, dual cursor message tracking
- **RAG:** Keyword search only
- **Special:** Container isolation for skills

#### OpenClaw

**Memory Architecture:**
- **Storage:** SQLite + sqlite-vec + Markdown files
- **Pattern:** Multi-agent workspace isolation
- **RAG:** Hybrid with MMR and temporal decay
- **Sync:** Multi-trigger (watcher + timer + lifecycle)

**Post-Compaction Reinjection:**
```typescript
// Re-inject critical sections after compaction
const DEFAULT_POST_COMPACTION_SECTIONS = [
  "Session Startup",
  "Red Lines"  // From AGENTS.md
];
```

#### PicoClaw

**Memory Architecture:**
- **Storage:** Markdown files only
- **Pattern:** Date-organized (YYYY/MM/YYYYMMDD.md)
- **RAG:** File-based search
- **Session:** JSONL storage

#### ZeroClaw

**Memory Architecture:**
- **Storage:** SQLite (categorized tables)
- **Pattern:** Trait-driven architecture
- **Categories:** Core, Daily, Conversation, Custom

#### Nanobot

**Memory Architecture:**
- **Storage:** Markdown files (MEMORY.md + HISTORY.md)
- **Pattern:** Two-layer memory with JSONL sessions
- **RAG:** Grep-based search
- **Sync:** MessageBus pub/sub

```python
# Two-layer memory pattern
HISTORY.md   # Raw conversation logs (append-only)
MEMORY.md    # Curated facts, decisions (agent-managed)
```

#### MimiClaw

**Memory Architecture:**
- **Storage:** SPIFFS (ESP32 flash filesystem)
- **Pattern:** Embedded resource-constrained
- **RAG:** None (hardware limited)

### Programming Agents (7)

#### Aider

**Context Architecture:**
- **Storage:** Git-based + In-memory
- **Pattern:** RepoMap for codebase awareness
- **Context:** Chat history + summaries + repository map
- **No RAG:** Relies on RepoMap and file inclusion

**RepoMap Context Injection:**
```python
# RepoMap provides file outline without full content
# Uses tree-sitter for structure extraction
def get_repo_map(filenames, chat_files):
    # Returns file outlines for context
    # Full content only for chat_files
```

#### Codex

**Memory Architecture:**
- **Storage:** JSONL traces + SQLite
- **Pattern:** Memory traces with session telemetry
- **Protocol:** SQ/EQ (State Query/Event Queue)

#### Qwen-Code

**Session Architecture:**
- **Storage:** JSONL
- **Pattern:** ACP (Agent Communication Protocol)
- **Monorepo:** npm workspaces with 6 packages

#### Gemini-CLI

**Memory Architecture:**
- **Storage:** JSON (hierarchical)
- **Pattern:** Hierarchical memory (global/extension/project)
- **Context:** JIT discovery based on accessed paths

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

#### Opencode

**Memory Architecture:**
- **Storage:** SQLite (Drizzle ORM)
- **Pattern:** Structured tables (Session/Message/Part)
- **Compaction:** CompactionPart type for overflow
- **Fork:** Parent-child session relationships

#### Pi-Mono

**Session Architecture:**
- **Storage:** JSONL dual files
- **Pattern:** Two-file approach (context.jsonl + log.jsonl)
- **Monorepo:** 7 packages with npm workspaces

---

## Comparative Matrices

### Memory Systems Comparison

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

### Session Storage Formats

| Project | Primary Format | Secondary Format | Fork Support | Checkpoint Support |
|---------|----------------|------------------|--------------|-------------------|
| **CoPaw** | SQLite | Markdown | No | No |
| **IronClaw** | SQLite/PostgreSQL | Markdown | No | No |
| **NanoClaw** | SQLite | Markdown | No | No |
| **OpenClaw** | SQLite | JSONL | No | No |
| **PicoClaw** | Markdown | JSONL | No | No |
| **ZeroClaw** | SQLite | - | No | No |
| **Nanobot** | Markdown | JSONL | No | No |
| **Aider** | Git | In-memory | No | No |
| **Codex** | JSONL | SQLite | No | No |
| **Qwen-code** | JSONL | - | No | No |
| **Gemini-cli** | JSON | - | No | No |
| **Kimi-cli** | JSONL | JSON | Yes (Checkpoint) | Yes |
| **Opencode** | SQLite | - | Yes | No |
| **Pi-Mono** | JSONL | - | No | No |

### Write Strategy Comparison

| Project | Strategy | Trigger | Granularity | Safety Mechanism |
|---------|----------|---------|-------------|------------------|
| **CoPaw** | Condition-based | Token threshold 75% | Message batch | Async hooks |
| **IronClaw** | Transactional | Session lifecycle | Record | 2PC |
| **NanoClaw** | Real-time | Message | Message | Direct SQL |
| **OpenClaw** | Multi-trigger | Watcher + timer | Chunk | Debounced + WAL |
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

### Boundary Management Comparison

| Project | Session-Persistent Boundary | Recovery Strategy | Concurrency |
|---------|----------------------------|-------------------|-------------|
| **CoPaw** | InMemoryMemory ↔ ReMeLight | Summary + reserved | Async |
| **IronClaw** | Workspace ↔ Database | Full restore | Transaction |
| **NanoClaw** | Container ↔ Files | JSONL replay | Isolated |
| **OpenClaw** | SessionState ↔ Index | Delta replay | Debounced |
| **ZeroClaw** | Trait ↔ SQLite | Query rebuild | Pool |
| **Nanobot** | MemoryBus ↔ Files | Full scan | Pub/sub |
| **Kimi-cli** | Session ↔ JSONL | Stream load | Async |
| **Opencode** | Session ↔ Drizzle | Layered | Event Bus |
| **Gemini-cli** | Scheduler ↔ State | State machine | EventEmitter |

---

## Recommended Unified Architecture

### Fusion Memory Architecture

Based on synthesis of all 15 projects:

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
│   └── Context window management (Three-zone model)              │
└─────────────────────────────────────────────────────────────────┘
```

### Unified RAG Architecture

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

### Unified Session State Architecture

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
│ Context Zones                                                   │
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

### Component Recommendations

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

## Summary

### Key Insights

1. **Personal assistants favor hybrid RAG** (vector + BM25) with file-based soul definitions for long-term user context
2. **Programming agents prioritize context window management** over retrieval, using RepoMap or JIT discovery
3. **Protected identity files** (IronClaw pattern) prevent agent self-corruption of core values
4. **Post-compaction reinjection** (OpenClaw pattern) maintains context coherence after summarization
5. **Dual storage** (SQLite + JSONL) balances query performance with portability and human readability

### Architectural Strengths by Project

| Project | Memory Strength | Session Strength | Boundary Strength |
|---------|-----------------|------------------|-------------------|
| **CoPaw** | Three-layer, hybrid RAG | Token-threshold compaction | Hook-based async |
| **IronClaw** | Protected files, dual-backend | Transaction safety | 2PC commit |
| **OpenClaw** | Multi-agent isolation | Debounced multi-trigger | Delta sync |
| **kimi-cli** | Checkpoint system | D-Mail time travel | Immutable logs |
| **opencode** | Structured SQLite | Fork support | Event Bus |
| **Gemini-cli** | Hierarchical memory | State machine scheduler | Event-driven |
| **Nanobot** | Two-layer (HISTORY/MEMORY) | JSONL session | MessageBus pub/sub |

### Final Recommended Architecture

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

*Document Version: 2.0 - Rewritten for 15-project coverage*
*Generated: 2026-03-18*
*Projects Analyzed: 15 (8 Personal Assistants + 7 Programming Agents)*
