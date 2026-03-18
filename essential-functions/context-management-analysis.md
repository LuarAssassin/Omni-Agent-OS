# Context Management & Conversation State Analysis

## Executive Summary

This document provides a comprehensive technical analysis of context management, token counting, conversation state, and memory systems across 15 AI agent projects (8 personal assistants + 7 programming agents). The analysis covers context window management, token estimation strategies, compression/summarization approaches, checkpoint mechanisms, and storage architectures.

---

## Projects Analyzed

### Personal Assistants (8)
| Project | Language | Key Context Feature |
|---------|----------|---------------------|
| CoPaw | Python | ReMe-based memory with compression ratios |
| IronClaw | Rust | Compaction strategies (Summarize/Truncate/MoveToWorkspace) |
| NanoClaw | TypeScript | SQLite persistence with pagination |
| OpenClaw | TypeScript | Multi-stage compaction with adaptive chunking |
| PicoClaw | Go | CJK-aware token estimation, context caching |
| ZeroClaw | Rust | Trait-driven memory with Markdown/SQLite backends |
| MimicClaw | C | Ring buffer session management (ESP32) |
| Nanobot | Python | Two-layer memory (MEMORY.md + HISTORY.md) |

### Programming Agents (7)
| Project | Language | Key Context Feature |
|---------|----------|---------------------|
| Aider | Python | Recursive summarization with model fallback |
| Codex | Rust | Byte-based truncation with middle-preservation |
| Gemini-CLI | TypeScript | Two-phase compression with probe verification |
| Kimi-CLI | Python | D-Mail checkpoint system (Steins;Gate inspired) |
| OpenCode | TypeScript | Tool output pruning with protected tools |
| Pi-Mono | TypeScript | Tree-based sessions with overflow detection |
| Qwen-Code | TypeScript | Multi-modal token calculation |

---

## 1. Token Counting Strategies

### 1.1 Exact Tokenizers

**HuggingFace Tokenizers**
```python
# CoPaw: Qwen tokenizer with fallback
# File: src/copaw/agents/utils/token_counting.py
def safe_count_str_tokens(text: str) -> int:
    try:
        token_ids = token_counter.tokenizer.encode(text)
        return len(token_ids)
    except Exception:
        return len(text.encode("utf-8")) // 4  # Fallback
```

**tiktoken (OpenAI)**
```python
# Aider: Model-specific with fallback chain
# File: aider/history.py
def token_count(self, text):
    # Try primary model tokenizer
    # Fall back through multiple models
    # Final fallback: approximate
```

### 1.2 Character-Based Estimation

**Standard Heuristic (4 chars/token)**
```typescript
// Codex: Byte-based estimation
// File: codex-rs/core/src/truncate.rs
pub const APPROX_BYTES_PER_TOKEN: usize = 4;

// Qwen-Code: Simple fallback
// File: requestTokenizer.ts
return Math.ceil(content.length / 4);
```

**CJK-Aware Estimation (PicoClaw)**
```go
// File: picoclaw/pkg/routing/features.go
func estimateTokens(msg string) int {
    total := utf8.RuneCountInString(msg)
    cjk := countCJKRunes(msg)
    // CJK characters: 1 token each
    // Non-CJK: 0.25 tokens each (4 chars/token)
    return cjk + (total-cjk)/4
}

// Alternative: 2.5 chars per token
func (al *AgentLoop) estimateTokens(messages []providers.Message) int {
    return totalChars * 2 / 5  // 2.5 chars/token
}
```

**Word-Based Estimation (IronClaw)**
```rust
// File: src/agent/context_monitor.rs
const TOKENS_PER_WORD: f64 = 1.3;

fn estimate_tokens(text: &str) -> usize {
    let word_count = text.split_whitespace().count();
    (word_count as f64 * TOKENS_PER_WORD + 4.0) as usize  // +4 overhead
}
```

### 1.3 Multi-Modal Token Calculation

```typescript
// Qwen-Code: Handles text, image, audio
// File: requestTokenizer.ts
class RequestTokenizer {
    async calculateTokens(content: ContentItem[]): Promise<number> {
        let total = 0;
        for (const item of content) {
            switch (item.type) {
                case 'text': total += this.countText(item.text); break;
                case 'image': total += this.countImage(item.image); break;
                case 'audio': total += this.countAudio(item.audio); break;
            }
        }
        return total;
    }
}
```

---

## 2. Context Window Management

### 2.1 Configuration Patterns

| Project | Default Limit | Trigger Threshold | Reserved Space |
|---------|---------------|-------------------|----------------|
| CoPaw | 128K chars | 0.7 (70%) | 10% |
| IronClaw | 100K tokens | 0.8 (80%) | Configurable |
| Kimi-CLI | Model-specific | 0.85 (85%) | Configurable |
| OpenClaw | Model-specific | Adaptive | 20% buffer |
| OpenCode | Model input limit | Overflow detection | 20K tokens |
| Gemini-CLI | 1M tokens | 0.5 (50%) | 0.3 (30%) preserve |
| MimicClaw | 16KB buffer | Fixed 20 msgs | N/A |

### 2.2 Trigger Mechanisms

**Dual Condition (Kimi-CLI)**
```python
# File: src/kimi_cli/soul/context.py
class SimpleCompaction:
    def should_compact(self, token_count: int) -> bool:
        max_tokens = self.max_context_size
        reserved = self.reserved_context_size

        # Condition 1: Ratio trigger
        if token_count >= max_tokens * 0.85:
            return True

        # Condition 2: Reserved space trigger
        if token_count + reserved >= max_tokens:
            return True

        return False
```

**Overflow Detection (Pi-Mono)**
```typescript
// File: packages/ai/src/utils/overflow.ts
// Detects overflow from 15+ providers
const OVERFLOW_PATTERNS = {
    anthropic: /context.*window.*exceeded/i,
    openai: /maximum context length/i,
    google: /token limit exceeded/i,
    // ... 12 more providers
};

export function detectOverflow(error: Error, provider: string): boolean {
    const pattern = OVERFLOW_PATTERNS[provider];
    return pattern?.test(error.message) ?? false;
}
```

**Silent Detection (Pi-Mono)**
```typescript
// Detect overflow via usage metrics
if (usage.input > contextWindow * 0.95) {
    return { type: 'silent_overflow', confidence: 'high' };
}
```

---

## 3. Compression & Summarization Strategies

### 3.1 Recursive Summarization (Aider)

```python
# File: aider/history.py
class ChatSummary:
    def __init__(self, max_tokens: int = 1024):
        self.max_tokens = max_tokens

    def summarize(self, messages: list) -> str:
        # Split into head (to summarize) and tail (preserve)
        head, tail = self.split_messages(messages)

        # Recursively summarize head
        head_summary = self.summarize_recursive(head)

        # Combine summary with tail
        return f"[Summary]: {head_summary}\n\n[Recent]: {tail}"
```

### 3.2 Multi-Stage Compaction (OpenClaw)

```typescript
// File: src/agents/compaction.ts
class CompactionService {
    BASE_CHUNK_RATIO = 0.4;
    MIN_CHUNK_RATIO = 0.15;
    SAFETY_MARGIN = 1.2;

    async compact(messages: Message[]): Promise<CompactionResult> {
        // Stage 1: Adaptive chunk sizing
        const chunkSize = this.calculateChunkSize(messages);

        // Stage 2: Progressive summarization
        const chunks = this.createChunks(messages, chunkSize);
        const summaries = await Promise.all(
            chunks.map(c => this.summarize(c))
        );

        // Stage 3: Merge summaries
        return this.mergeSummaries(summaries);
    }
}
```

### 3.3 Two-Phase Compression with Probe (Gemini-CLI)

```typescript
// File: chatCompressionService.ts
class ChatCompressionService {
    async compress(chat: Chat[]): Promise<CompressionResult> {
        // Phase 1: Generate summary
        const summary = await this.generateSummary(chat);

        // Phase 2: Probe verification
        const probeResult = await this.probe(summary);

        if (!probeResult.sufficient) {
            // Retry with more context
            return this.compressWithMoreContext(chat);
        }

        return { summary, state: this.captureState() };
    }

    private async probe(summary: string): Promise<ProbeResult> {
        // Ask model if summary is sufficient to continue
        const response = await this.model.ask(
            `Can you continue the conversation with this summary? ${summary}`
        );
        return { sufficient: response.yes };
    }
}
```

### 3.4 Compaction Strategies (IronClaw)

```rust
// File: src/agent/context_monitor.rs
pub enum CompactionStrategy {
    Summarize { keep_recent: usize },  // LLM-based summary
    Truncate { keep_recent: usize },   // Simple truncation
    MoveToWorkspace,                    // Move to persistent memory
}

impl ContextMonitor {
    pub async fn compact(&mut self, strategy: CompactionStrategy) {
        match strategy {
            Summarize { keep_recent } => {
                let to_summarize = &self.messages[..self.messages.len() - keep_recent];
                let summary = self.llm.summarize(to_summarize).await;
                self.compressed_summary = Some(summary);
                self.messages.truncate(keep_recent);
            }
            Truncate { keep_recent } => {
                self.messages.truncate(keep_recent);
            }
            MoveToWorkspace => {
                self.workspace.store_messages(&self.messages).await;
                self.messages.clear();
            }
        }
    }
}
```

### 3.5 Tool Output Pruning (OpenCode)

```typescript
// File: packages/opencode/src/session/compaction.ts
const PRUNE_MINIMUM = 20_000;    // Minimum threshold
const PRUNE_PROTECT = 40_000;    // Protect recent 40K tokens
const PRUNE_PROTECTED_TOOLS = ["skill"];  // Never prune these

async function pruneToolOutputs(session: Session): Promise<void> {
    const messages = await session.getMessages();
    let turns = 0;

    for (let i = messages.length - 1; i >= 0; i--) {
        const msg = messages[i];

        // Protect recent 2 turns
        if (msg.role === "user" && ++turns < 2) continue;

        // Protect specific tools
        if (PRUNE_PROTECTED_TOOLS.includes(msg.toolName)) continue;

        // Mark as compacted
        if (msg.role === "assistant" && msg.hasToolCalls()) {
            msg.state.compacted = Date.now();
            msg.content = "[Old tool result content cleared]";
        }
    }
}
```

---

## 4. Checkpoint & Recovery Mechanisms

### 4.1 D-Mail System (Kimi-CLI)

```python
# File: src/kimi_cli/soul/denwarenji.py
class DMail(BaseModel):
    """Steins;Gate inspired checkpoint messaging system"""
    message: str           # Message to past self
    checkpoint_id: int     # Target checkpoint ID

class Context:
    def __init__(self):
        self._history: list[Message] = []
        self._next_checkpoint_id: int = 0
        self._checkpoints: dict[int, int] = {}  # id -> message index

    def checkpoint(self) -> int:
        """Create a new checkpoint"""
        checkpoint_id = self._next_checkpoint_id
        self._checkpoints[checkpoint_id] = len(self._history)
        self._next_checkpoint_id += 1
        return checkpoint_id

    def revert_to(self, checkpoint_id: int) -> list[Message]:
        """Revert to checkpoint state"""
        if checkpoint_id not in self._checkpoints:
            raise ValueError(f"Checkpoint {checkpoint_id} not found")

        index = self._checkpoints[checkpoint_id]
        self._history = self._history[:index]
        return self._history

    def send_dmail(self, dmail: DMail) -> None:
        """Send message to past checkpoint"""
        self._pending_dmail = dmail
```

### 4.2 Session State Snapshots (Gemini-CLI)

```typescript
// File: chatCompressionService.ts
interface StateSnapshot {
    checkpointId: string;
    timestamp: number;
    compressedContext: string;
    workingDirectory: string;
    fileStates: FileState[];
    toolResults: ToolResult[];
}

class SessionStateManager {
    async createSnapshot(): Promise<StateSnapshot> {
        return {
            checkpointId: generateId(),
            timestamp: Date.now(),
            compressedContext: await this.compressContext(),
            workingDirectory: this.cwd,
            fileStates: await this.captureFileStates(),
            toolResults: this.recentToolResults,
        };
    }

    async restoreSnapshot(snapshot: StateSnapshot): Promise<void> {
        this.cwd = snapshot.workingDirectory;
        await this.restoreFileStates(snapshot.fileStates);
        this.context = await this.decompress(snapshot.compressedContext);
    }
}
```

---

## 5. Storage Architectures

### 5.1 JSONL (Line-delimited JSON)

```python
# Kimi-CLI: Session storage
# File: src/kimi_cli/soul/context.py
class Context:
    def __init__(self, file_backend: Path):
        self._file = file_backend  # context.jsonl
        self._wire_file = file_backend.parent / "wire.jsonl"
        self._state_file = file_backend.parent / "state.json"

    def append(self, message: Message) -> None:
        with open(self._file, 'a') as f:
            f.write(json.dumps(message.to_dict()) + '\n')

    def load(self) -> list[Message]:
        messages = []
        with open(self._file, 'r') as f:
            for line in f:
                if line.strip():
                    messages.append(Message.from_dict(json.loads(line)))
        return messages
```

### 5.2 SQLite (Structured)

```typescript
// NanoClaw: SQLite schema
// File: src/db.ts
interface MessageRow {
    id: number;
    session_id: string;
    timestamp: number;
    from_me: boolean;
    sender_jid: string;
    content: string;
    context_mode: 'group' | 'isolated';
}

class Database {
    async getNewMessages(
        sessionId: string,
        afterTimestamp: number,
        limit: number = 200
    ): Promise<Message[]> {
        return this.db.all(
            `SELECT * FROM messages
             WHERE session_id = ? AND timestamp > ?
             ORDER BY timestamp ASC
             LIMIT ?`,
            [sessionId, afterTimestamp, limit]
        );
    }
}
```

### 5.3 Markdown Memory (Human-readable)

```markdown
<!-- Nanobot: MEMORY.md structure -->
# Memory

## Key Facts
- User prefers Python over JavaScript
- Working on Project X

## Current Tasks
- [ ] Implement feature Y
- [x] Completed feature Z

## File References
- src/main.py: Entry point
- config.yaml: Configuration
```

### 5.4 Ring Buffer (Embedded/Constrained)

```c
// MimicClaw: Fixed-size ring buffer for ESP32
// File: main/memory/session_mgr.c
#define MIMI_SESSION_MAX_MSGS 20

typedef struct {
    Message msgs[MIMI_SESSION_MAX_MSGS];
    uint8_t head;  // Write position
    uint8_t tail;  // Read position
    uint8_t count; // Current count
} MessageRingBuffer;

void session_add_message(Session *s, const Message *msg) {
    RingBuffer *rb = &s->buffer;

    // Overwrite oldest if full
    rb->msgs[rb->head] = *msg;
    rb->head = (rb->head + 1) % MIMI_SESSION_MAX_MSGS;

    if (rb->count < MIMI_SESSION_MAX_MSGS) {
        rb->count++;
    } else {
        rb->tail = (rb->tail + 1) % MIMI_SESSION_MAX_MSGS;
    }
}
```

---

## 6. Memory Systems

### 6.1 Two-Layer Memory (Nanobot)

```python
# File: nanobot/agent/memory.py
class MemoryStore:
    def __init__(self, workspace: Path):
        self.memory_dir = workspace / ".memory"
        self.memory_file = self.memory_dir / "MEMORY.md"      # Long-term
        self.history_file = self.memory_dir / "HISTORY.md"    # Searchable

    async def consolidate(self) -> None:
        """Consolidate history into memory using LLM"""
        history = self.history_file.read_text()

        # Call LLM to extract key facts
        consolidation_prompt = f"""
        Extract key facts from this conversation history:
        {history}

        Update the MEMORY.md with these facts.
        """

        new_memory = await self.llm.complete(consolidation_prompt)
        self._merge_into_memory(new_memory)
```

### 6.2 Daily Notes (PicoClaw)

```go
// File: pkg/agent/memory.go
type MemoryStore struct {
    baseDir string  // ~/.picoclaw/memory/
}

func (m *MemoryStore) GetRecentDailyNotes(days int) ([]DailyNote, error) {
    var notes []DailyNote
    now := time.Now()

    for i := 0; i < days; i++ {
        date := now.AddDate(0, 0, -i)
        path := filepath.Join(
            m.baseDir,
            date.Format("200601"),  // YYYYMM
            date.Format("20060102") + ".md",  // YYYYMMDD.md
        )

        if content, err := os.ReadFile(path); err == nil {
            notes = append(notes, DailyNote{
                Date: date,
                Content: string(content),
            })
        }
    }

    return notes, nil
}

func (m *MemoryStore) WriteDailyNote(content string) error {
    today := time.Now()
    monthDir := filepath.Join(m.baseDir, today.Format("200601"))
    os.MkdirAll(monthDir, 0755)

    path := filepath.Join(monthDir, today.Format("20060102") + ".md")
    return os.WriteFile(path, []byte(content), 0644)
}
```

### 6.3 Context Caching (PicoClaw)

```go
// File: pkg/agent/context.go
type ContextBuilder struct {
    staticCache    *CachedPrompt    // System prompt + SOUL.md
    dynamicCache   *DynamicPrompt   // Per-request context
    cacheMtime     time.Time        // For invalidation
}

func (cb *ContextBuilder) BuildContext(session *Session) ([]Message, error) {
    // 1. Get static context (cached)
    static, err := cb.getStaticContext()
    if err != nil {
        return nil, err
    }

    // 2. Get dynamic context (recent messages)
    dynamic := cb.getDynamicContext(session)

    // 3. Combine with cache control for LLM KV cache
    return append(static, dynamic...), nil
}

func (cb *ContextBuilder) getStaticContext() ([]Message, error) {
    // Check if cache is valid
    mtime := cb.getSourceMtime()
    if mtime.After(cb.cacheMtime) {
        // Rebuild cache
        cb.staticCache = cb.rebuildStaticCache()
        cb.cacheMtime = mtime
    }
    return cb.staticCache.Messages, nil
}
```

---

## 7. Session Structure Comparison

### 7.1 Tree-Based Sessions (Pi-Mono)

```typescript
// File: packages/coding-agent/src/core/session-manager.ts
interface SessionEntry {
    id: string;
    parentId: string | null;
    type: 'message' | 'compaction' | 'branch_summary' | 'custom' | 'label';
    timestamp: number;
    content: unknown;
    metadata: SessionMetadata;
}

class SessionManager {
    CURRENT_SESSION_VERSION = 3;

    async createBranch(parentId: string): Promise<string> {
        const branchId = generateId();
        await this.db.insert({
            id: branchId,
            parentId,
            type: 'branch_summary',
            timestamp: Date.now(),
            content: await this.summarizeBranch(parentId),
        });
        return branchId;
    }

    async getLineage(entryId: string): Promise<SessionEntry[]> {
        // Traverse parent pointers to build full context
        const lineage = [];
        let current = await this.getEntry(entryId);

        while (current) {
            lineage.unshift(current);
            if (current.parentId) {
                current = await this.getEntry(current.parentId);
            } else {
                break;
            }
        }

        return lineage;
    }
}
```

### 7.2 Flat with Checkpoints (Kimi-CLI)

```
Session Structure:
├── context.jsonl       # Messages
├── wire.jsonl          # Protocol messages
└── state.json          # Session state
    {
        "current_checkpoint_id": 5,
        "checkpoints": {
            "0": 0,     # checkpoint_id -> message_index
            "1": 10,
            "2": 25,
            ...
        }
    }
```

---

## 8. Comprehensive Comparison Matrix

### 8.1 Token Counting

| Project | Method | CJK-Aware | Multi-Modal | Fallback |
|---------|--------|-----------|-------------|----------|
| CoPaw | HF Tokenizer | No | No | chars//4 |
| Kimi-CLI | Estimate + API | No | No | chars//4 |
| OpenClaw | chars//4 | No | Yes | N/A |
| OpenCode | chars//4 | No | No | N/A |
| Codex | Bytes//4 | No | Yes (base64) | N/A |
| IronClaw | Words×1.3 | No | No | N/A |
| PicoClaw | CJK=1, other//4 | Yes | No | chars//2.5 |
| MimicClaw | Fixed limit | No | No | N/A |
| Gemini-CLI | Model API | No | Yes | chars//4 |
| Pi-Mono | chars//4 | No | No | N/A |
| Qwen-Code | Multi-modal calc | No | Yes | chars//4 |
| Aider | tiktoken | No | No | Model chain |

### 8.2 Storage Backends

| Project | Primary | Secondary | Format |
|---------|---------|-----------|--------|
| CoPaw | In-Memory | Markdown | ReMe |
| Kimi-CLI | JSONL | - | Line JSON |
| OpenClaw | JSONL | - | Line JSON |
| OpenCode | SQLite | - | SQL |
| Codex | In-Memory | - | Rust structs |
| IronClaw | In-Memory | SQLite | SQL |
| NanoClaw | SQLite | - | SQL |
| PicoClaw | In-Memory | Markdown | MD |
| ZeroClaw | SQLite | Markdown | SQL+MD |
| MimicClaw | Ring Buffer | SPIFFS | JSONL |
| Nanobot | Markdown | - | MD |
| Gemini-CLI | In-Memory | State Snapshots | JSON |
| Pi-Mono | Tree Structure | - | Custom |
| Aider | In-Memory | Git | Text |
| Qwen-Code | In-Memory | - | TS structs |

### 8.3 Compression Strategies

| Project | Strategy | Trigger | Keep Recent | Special Features |
|---------|----------|---------|-------------|------------------|
| CoPaw | LLM Summary | 70% | 10% / 3 msgs | Async, tool integrity |
| Kimi-CLI | LLM Summary | 85% | 2 msgs | D-Mail checkpoints |
| OpenClaw | Multi-stage | Adaptive | 3 assistants | Adaptive chunking |
| OpenCode | Pruning | Overflow | 2 turns | Protected tools |
| Codex | Truncation | Bytes | Configurable | Middle preservation |
| IronClaw | Configurable | 80% | Configurable | 3 strategies |
| Gemini-CLI | Two-phase | 50% | 30% | Probe verification |
| Pi-Mono | File tracking | Config | Config | Extension hooks |
| Aider | Recursive | Token limit | Config | Model fallback |
| MimicClaw | Ring buffer | Fixed 20 | N/A | Fixed size |
| Nanobot | Consolidation | Manual | All | Two-layer memory |

### 8.4 Checkpoint/Recovery

| Project | Checkpoints | Recovery | Persistence |
|---------|-------------|----------|-------------|
| Kimi-CLI | D-Mail system | Full revert | JSONL rotation |
| Gemini-CLI | State snapshots | Full restore | JSON files |
| Pi-Mono | Tree branches | Branch switch | SQL entries |
| OpenClaw | Compaction markers | Partial | In-memory |
| Others | None | N/A | - |

---

## 9. Recommended Unified Architecture

Based on analysis of all 15 projects, here is a recommended unified context management architecture:

### 9.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Unified Context Manager                       │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: Token Estimation                                       │
│  ├─ Exact: tiktoken/HF for supported models                     │
│  ├─ CJK-Aware: 1 token/CJK, 0.25 others                         │
│  └─ Fallback: bytes // 4                                        │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Storage Abstraction                                    │
│  ├─ Hot: In-memory ring buffer (last N messages)                │
│  ├─ Warm: SQLite (recent sessions)                              │
│  └─ Cold: Markdown files (long-term memory)                     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: Trigger Detection                                      │
│  ├─ Ratio trigger: usage >= threshold (default 80%)             │
│  ├─ Space trigger: usage + reserved >= limit                    │
│  └─ Overflow detection: Provider-specific patterns              │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: Protection Zone                                        │
│  ├─ System prompt (always)                                      │
│  ├─ Compressed summary                                          │
│  ├─ Recent N turns (configurable, default 4)                    │
│  ├─ Incomplete tool chains                                      │
│  └─ Protected tools (skill, critical ops)                       │
├─────────────────────────────────────────────────────────────────┤
│  Layer 5: Pruning & Trimming                                     │
│  ├─ Soft trim: Large tool outputs (head + tail)                 │
│  ├─ Hard clear: Old tool results -> placeholder                 │
│  └─ Media strip: Replace images with descriptions               │
├─────────────────────────────────────────────────────────────────┤
│  Layer 6: Compaction                                             │
│  ├─ Two-phase: Generate + Verify (probe)                        │
│  ├─ Structured: Goal/Instructions/Discoveries/Accomplished      │
│  ├─ Incremental: Merge with existing summary                    │
│  └─ Post-injection: Re-add critical context                     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 7: Checkpointing                                          │
│  ├─ Automatic: Every N messages or tool boundary                │
│  ├─ Manual: User-initiated (/checkpoint)                        │
│  ├─ Named: Labeled checkpoints                                  │
│  └─ Branch: D-Mail style time travel                            │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 Reference Implementation (Rust)

```rust
// Unified context management system
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use std::collections::VecDeque;

/// Token estimation strategies
pub enum TokenEstimator {
    Exact(Box<dyn Fn(&str) -> usize>),  // tiktoken, HF tokenizer
    CjkAware,                           // 1 token/CJK, 0.25 others
    Heuristic { chars_per_token: usize }, // Simple division
}

impl TokenEstimator {
    pub fn estimate(&self, text: &str) -> usize {
        match self {
            TokenEstimator::Exact(f) => f(text),
            TokenEstimator::CjkAware => {
                let cjk_count = text.chars()
                    .filter(|c| is_cjk(*c))
                    .count();
                let other_count = text.len() - cjk_count;
                cjk_count + other_count / 4
            }
            TokenEstimator::Heuristic { chars_per_token } => {
                text.len() / chars_per_token
            }
        }
    }
}

/// Message in conversation
#[derive(Clone, Serialize, Deserialize)]
pub struct Message {
    pub id: String,
    pub role: Role,
    pub content: Content,
    pub metadata: MessageMetadata,
    pub timestamp: u64,
}

#[derive(Clone, Serialize, Deserialize)]
pub enum Role {
    System,
    User,
    Assistant,
    Tool { call_id: String },
}

#[derive(Clone, Serialize, Deserialize)]
pub enum Content {
    Text(String),
    Image { url: String, description: Option<String> },
    ToolCall { id: String, name: String, arguments: String },
    ToolResult { call_id: String, output: String },
}

/// Configuration for context management
#[derive(Clone)]
pub struct ContextConfig {
    pub max_context_tokens: usize,
    pub trigger_threshold: f64,      // 0.0-1.0 (default 0.8)
    pub reserved_tokens: usize,      // Safety margin
    pub keep_recent_turns: usize,    // Recent messages to protect
    pub protected_tools: Vec<String>, // Tools to never prune
    pub enable_checkpoints: bool,
    pub compaction_model: String,    // Model for summarization
}

impl Default for ContextConfig {
    fn default() -> Self {
        Self {
            max_context_tokens: 128_000,
            trigger_threshold: 0.8,
            reserved_tokens: 20_000,
            keep_recent_turns: 4,
            protected_tools: vec!["skill".into(), "read_file".into()],
            enable_checkpoints: true,
            compaction_model: "gpt-3.5-turbo".into(),
        }
    }
}

/// Checkpoint for recovery
#[derive(Clone, Serialize, Deserialize)]
pub struct Checkpoint {
    pub id: String,
    pub message_index: usize,
    pub timestamp: u64,
    pub label: Option<String>,
    pub context_hash: String,
}

/// Main context manager
pub struct ContextManager {
    config: ContextConfig,
    estimator: TokenEstimator,

    // Three-zone architecture
    system_prompt: Option<String>,
    compressed_summary: String,
    messages: VecDeque<Message>,

    // Checkpoints
    checkpoints: Vec<Checkpoint>,

    // Storage backends
    hot_storage: Box<dyn HotStorage>,
    warm_storage: Box<dyn WarmStorage>,
    cold_storage: Box<dyn ColdStorage>,
}

#[async_trait]
pub trait HotStorage: Send + Sync {
    fn get_recent(&self, n: usize) -> Vec<Message>;
    fn append(&mut self, message: Message);
    fn truncate(&mut self, keep: usize);
}

#[async_trait]
pub trait WarmStorage: Send + Sync {
    async fn save_session(&self, session: &Session) -> Result<(), Error>;
    async fn load_session(&self, id: &str) -> Result<Option<Session>, Error>;
    async fn list_sessions(&self) -> Result<Vec<SessionSummary>, Error>;
}

#[async_trait]
pub trait ColdStorage: Send + Sync {
    async fn write_memory(&self, content: &str) -> Result<(), Error>;
    async fn read_memory(&self) -> Result<String, Error>;
    async fn get_daily_notes(&self, days: usize) -> Result<Vec<String>, Error>;
}

impl ContextManager {
    pub fn new(config: ContextConfig, estimator: TokenEstimator) -> Self {
        Self {
            config,
            estimator,
            system_prompt: None,
            compressed_summary: String::new(),
            messages: VecDeque::new(),
            checkpoints: Vec::new(),
            hot_storage: Box::new(InMemoryStorage::new(100)),
            warm_storage: Box::new(SqliteStorage::new("sessions.db")),
            cold_storage: Box::new(MarkdownStorage::new("~/.agent/memory")),
        }
    }

    /// Calculate total token count
    pub fn token_count(&self) -> usize {
        let mut total = 0;

        if let Some(ref system) = self.system_prompt {
            total += self.estimator.estimate(system);
        }

        if !self.compressed_summary.is_empty() {
            total += self.estimator.estimate(&self.compressed_summary);
        }

        for msg in &self.messages {
            total += self.estimate_message(msg);
        }

        total
    }

    fn estimate_message(&self, msg: &Message) -> usize {
        match &msg.content {
            Content::Text(text) => self.estimator.estimate(text),
            Content::Image { description, .. } => {
                // Images count as fixed tokens or description length
                description.as_ref()
                    .map(|d| self.estimator.estimate(d))
                    .unwrap_or(1000)
            }
            Content::ToolCall { arguments, .. } => {
                self.estimator.estimate(arguments)
            }
            Content::ToolResult { output, .. } => {
                self.estimator.estimate(output)
            }
        }
    }

    /// Check if compaction should trigger
    pub fn should_compact(&self) -> bool {
        let tokens = self.token_count();
        let max = self.config.max_context_tokens;

        // Condition 1: Ratio trigger
        if tokens as f64 >= max as f64 * self.config.trigger_threshold {
            return true;
        }

        // Condition 2: Reserved space trigger
        if tokens + self.config.reserved_tokens >= max {
            return true;
        }

        false
    }

    /// Get protected messages that cannot be compacted
    fn get_protected_messages(&self) -> Vec<&Message> {
        let mut protected = Vec::new();
        let mut user_count = 0;

        // Protect recent N turns (from end)
        for msg in self.messages.iter().rev() {
            protected.push(msg);

            if matches!(msg.role, Role::User) {
                user_count += 1;
                if user_count >= self.config.keep_recent_turns {
                    break;
                }
            }
        }

        // Ensure tool call/result pairs are complete
        protected = self.ensure_tool_integrity(protected);

        protected.reverse();
        protected
    }

    fn ensure_tool_integrity(&self, mut messages: Vec<&Message>) -> Vec<&Message> {
        let mut tool_calls: std::collections::HashSet<&str> = std::collections::HashSet::new();
        let mut tool_results: std::collections::HashSet<&str> = std::collections::HashSet::new();

        for msg in &messages {
            match &msg.content {
                Content::ToolCall { id, .. } => { tool_calls.insert(id.as_str()); }
                Content::ToolResult { call_id, .. } => { tool_results.insert(call_id.as_str()); }
                _ => {}
            }
        }

        // If there are unmatched tool calls, extend protection
        // (implementation omitted for brevity)

        messages
    }

    /// Two-stage pruning: soft trim + hard clear
    pub fn prune(&mut self) {
        const SOFT_TRIM_MAX: usize = 4000;
        const SOFT_TRIM_HEAD: usize = 1500;
        const SOFT_TRIM_TAIL: usize = 1500;

        let protected_ids: std::collections::HashSet<_> = self
            .get_protected_messages()
            .iter()
            .map(|m| m.id.clone())
            .collect();

        for msg in self.messages.iter_mut() {
            if protected_ids.contains(&msg.id) {
                continue;
            }

            // Stage 1: Soft trim large tool results
            if let Content::ToolResult { output, .. } = &mut msg.content {
                if output.len() > SOFT_TRIM_MAX {
                    let head = &output[..SOFT_TRIM_HEAD];
                    let tail = &output[output.len()-SOFT_TRIM_TAIL..];
                    let omitted = output.len() - SOFT_TRIM_HEAD - SOFT_TRIM_TAIL;
                    *output = format!("{}\n... ({} chars omitted) ...\n{}",
                        head, omitted, tail);
                    msg.metadata.was_soft_trimmed = true;
                }
            }
        }

        // Stage 2: Hard clear old tool results
        // (implementation omitted for brevity)
    }

    /// Compact conversation into summary
    pub async fn compact(&mut self, llm: &dyn LlmClient) -> Result<String, Error> {
        let protected = self.get_protected_messages();
        let to_compact: Vec<_> = self.messages
            .iter()
            .filter(|m| !protected.iter().any(|p| p.id == m.id))
            .cloned()
            .collect();

        if to_compact.is_empty() {
            return Ok(String::new());
        }

        // Generate structured summary
        let prompt = self.build_compaction_prompt(&to_compact);
        let summary = llm.complete(&prompt).await?;

        // Two-phase: Verify with probe
        let probe = format!(
            "Can you continue this conversation with this summary?\n{}",
            summary
        );
        let verification = llm.complete(&probe).await?;

        if !verification.to_lowercase().contains("yes") {
            // Retry with more context
            return self.compact_with_more_context(llm, &to_compact).await;
        }

        // Merge with existing summary
        self.compressed_summary = self.merge_summaries(&summary);

        // Remove compacted messages
        self.messages.retain(|m| protected.iter().any(|p| p.id == m.id));

        // Re-inject critical context
        self.reinject_critical_context().await?;

        Ok(summary)
    }

    fn build_compaction_prompt(&self, messages: &[Message]) -> String {
        let history = messages.iter()
            .map(|m| format!("{:?}: {:?}", m.role, m.content))
            .collect::<Vec<_>>()
            .join("\n");

        format!(r#"Provide a detailed summary of the conversation below.
Focus on information that would be helpful for continuing the conversation.

Use this template:
---
## Goal
[What goal(s) is the user trying to accomplish?]

## Instructions
- [What important instructions did the user give you]

## Discoveries
[What notable things were learned]

## Accomplished
[What work has been completed, what work is still in progress]

## Relevant files / directories
[Construct a structured list of relevant files]

## Next Steps
[What should be done next]
---

Conversation history:
{}
"#, history)
    }

    fn merge_summaries(&self, new: &str) -> String {
        if self.compressed_summary.is_empty() {
            new.to_string()
        } else {
            format!("{}\n\n## New Summary\n{}", self.compressed_summary, new)
        }
    }

    async fn reinject_critical_context(&mut self) -> Result<(), Error> {
        // Read AGENTS.md or similar identity files
        // Extract critical sections
        // Append to compressed_summary
        Ok(())
    }

    /// Create a checkpoint
    pub fn checkpoint(&mut self, label: Option<String>) -> String {
        let id = generate_id();
        let checkpoint = Checkpoint {
            id: id.clone(),
            message_index: self.messages.len(),
            timestamp: now(),
            label,
            context_hash: hash_context(&self.messages),
        };
        self.checkpoints.push(checkpoint);
        id
    }

    /// Revert to checkpoint
    pub fn revert_to(&mut self, checkpoint_id: &str) -> Result<(), Error> {
        let checkpoint = self.checkpoints
            .iter()
            .find(|c| c.id == checkpoint_id)
            .ok_or(Error::CheckpointNotFound)?;

        // Truncate messages to checkpoint index
        while self.messages.len() > checkpoint.message_index {
            self.messages.pop_back();
        }

        Ok(())
    }

    /// Get context for LLM API call
    pub fn get_context_for_llm(&self) -> Vec<LlmMessage> {
        let mut context = Vec::new();

        // System prompt
        if let Some(ref system) = self.system_prompt {
            context.push(LlmMessage::system(system.clone()));
        }

        // Compressed summary
        if !self.compressed_summary.is_empty() {
            context.push(LlmMessage::system(format!(
                "Previous conversation summary:\n{}",
                self.compressed_summary
            )));
        }

        // Recent messages
        for msg in &self.messages {
            context.push(self.convert_to_llm_message(msg));
        }

        context
    }

    fn convert_to_llm_message(&self, msg: &Message) -> LlmMessage {
        match &msg.content {
            Content::Text(text) => LlmMessage::user(text.clone()),
            Content::ToolCall { id, name, arguments } => {
                LlmMessage::tool_call(id.clone(), name.clone(), arguments.clone())
            }
            Content::ToolResult { call_id, output } => {
                LlmMessage::tool_result(call_id.clone(), output.clone())
            }
            Content::Image { url, .. } => LlmMessage::image(url.clone()),
        }
    }
}

// Helper types
fn is_cjk(c: char) -> bool {
    matches!(c as u32, 0x4E00..=0x9FFF | 0x3400..=0x4DBF | 0x3040..=0x309F | 0x30A0..=0x30FF)
}

fn generate_id() -> String {
    uuid::Uuid::new_v4().to_string()
}

fn now() -> u64 {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs()
}

fn hash_context(messages: &VecDeque<Message>) -> String {
    use std::collections::hash_map::DefaultHasher;
    use std::hash::{Hash, Hasher};

    let mut hasher = DefaultHasher::new();
    messages.hash(&mut hasher);
    format!("{:x}", hasher.finish())
}

#[async_trait]
pub trait LlmClient: Send + Sync {
    async fn complete(&self, prompt: &str) -> Result<String, Error>;
}

pub struct LlmMessage {
    pub role: String,
    pub content: String,
}

impl LlmMessage {
    pub fn system(content: String) -> Self {
        Self { role: "system".into(), content }
    }
    pub fn user(content: String) -> Self {
        Self { role: "user".into(), content }
    }
    pub fn tool_call(id: String, name: String, arguments: String) -> Self {
        Self {
            role: "assistant".into(),
            content: format!("<tool_call {}>{}</tool_call>", name, arguments)
        }
    }
    pub fn tool_result(call_id: String, output: String) -> Self {
        Self {
            role: "tool".into(),
            content: format!("<tool_result {}>{}</tool_result>", call_id, output)
        }
    }
    pub fn image(url: String) -> Self {
        Self { role: "user".into(), content: format!("<image>{}</image>", url) }
    }
}

#[derive(Debug)]
pub enum Error {
    CheckpointNotFound,
    StorageError(String),
    LlmError(String),
}

// Placeholder implementations
pub struct InMemoryStorage { capacity: usize }
impl InMemoryStorage {
    pub fn new(capacity: usize) -> Self { Self { capacity } }
}
impl HotStorage for InMemoryStorage {
    fn get_recent(&self, n: usize) -> Vec<Message> { vec![] }
    fn append(&mut self, message: Message) {}
    fn truncate(&mut self, keep: usize) {}
}

pub struct SqliteStorage { path: String }
impl SqliteStorage {
    pub fn new(path: &str) -> Self { Self { path: path.into() } }
}
#[async_trait]
impl WarmStorage for SqliteStorage {
    async fn save_session(&self, session: &Session) -> Result<(), Error> { Ok(()) }
    async fn load_session(&self, id: &str) -> Result<Option<Session>, Error> { Ok(None) }
    async fn list_sessions(&self) -> Result<Vec<SessionSummary>, Error> { Ok(vec![]) }
}

pub struct MarkdownStorage { path: String }
impl MarkdownStorage {
    pub fn new(path: &str) -> Self { Self { path: path.into() } }
}
#[async_trait]
impl ColdStorage for MarkdownStorage {
    async fn write_memory(&self, content: &str) -> Result<(), Error> { Ok(()) }
    async fn read_memory(&self) -> Result<String, Error> { Ok(String::new()) }
    async fn get_daily_notes(&self, days: usize) -> Result<Vec<String>, Error> { Ok(vec![]) }
}

pub struct Session;
pub struct SessionSummary;

#[derive(Default)]
pub struct MessageMetadata {
    pub was_soft_trimmed: bool,
    pub was_hard_cleared: bool,
    pub tool_name: Option<String>,
}

// Implement Hash for Message
use std::hash::Hash;
impl Hash for Message {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
    }
}
```

### 9.3 Configuration Schema

```json
{
  "context_management": {
    "token_estimation": {
      "strategy": "cjk_aware",
      "exact_tokenizer": null,
      "fallback_chars_per_token": 4
    },
    "limits": {
      "max_context_tokens": 128000,
      "trigger_threshold": 0.8,
      "reserved_tokens": 20000
    },
    "protection": {
      "keep_recent_turns": 4,
      "keep_recent_assistants": 3,
      "protected_tools": ["skill", "read_file", "write_file"],
      "ensure_tool_integrity": true
    },
    "pruning": {
      "soft_trim": {
        "enabled": true,
        "max_chars": 4000,
        "head_chars": 1500,
        "tail_chars": 1500
      },
      "hard_clear": {
        "enabled": true,
        "placeholder": "[Old content cleared]"
      }
    },
    "compaction": {
      "strategy": "summarize",
      "model": "gpt-3.5-turbo",
      "template": "structured",
      "two_phase_verification": true,
      "post_injection_sections": ["Session Startup", "Red Lines"]
    },
    "checkpointing": {
      "enabled": true,
      "auto_checkpoint_interval": 10,
      "max_checkpoints": 50
    },
    "storage": {
      "hot": { "type": "memory", "max_messages": 100 },
      "warm": { "type": "sqlite", "path": "~/.agent/sessions.db" },
      "cold": { "type": "markdown", "path": "~/.agent/memory" }
    }
  }
}
```

---

## 10. Best Practices Summary

### 10.1 Token Estimation
1. **Use exact tokenizers when available** (tiktoken, HF) for accurate counting
2. **Implement CJK-aware heuristics** for Asian language support
3. **Always provide fallbacks** to prevent crashes when tokenizers fail
4. **Cache token counts** on messages to avoid re-computation

### 10.2 Trigger Detection
1. **Use dual conditions**: ratio threshold + reserved space
2. **Implement overflow detection** with provider-specific patterns
3. **Add silent detection** via usage metrics before API errors
4. **Make thresholds configurable** per model and use case

### 10.3 Protection Strategies
1. **Protect system prompt** always
2. **Keep recent N turns** configurable (default 4)
3. **Ensure tool integrity** (complete call/result pairs)
4. **Protect critical tools** (skill, file operations)
5. **Handle incomplete tool chains** by extending protection

### 10.4 Compression
1. **Use two-phase approach**: generate + verify
2. **Structured summaries** (Goal/Instructions/Discoveries/Accomplished)
3. **Incremental merging** with existing summaries
4. **Post-compaction injection** of critical context
5. **Configurable compaction models** (lightweight for cost)

### 10.5 Checkpointing
1. **Automatic checkpoints** at tool boundaries
2. **Manual checkpoints** via user command
3. **Named checkpoints** for important states
4. **Branch support** for D-Mail style navigation
5. **State snapshots** including working directory and files

### 10.6 Storage
1. **Hot storage** (memory) for active context
2. **Warm storage** (SQLite) for recent sessions
3. **Cold storage** (Markdown) for long-term memory
4. **Daily notes** for chronological memory
5. **Ring buffers** for constrained environments

---

## 11. Conclusion

This analysis reveals significant diversity in context management approaches across AI agent projects, with each implementing unique solutions based on their target use cases and constraints:

- **Embedded systems** (MimicClaw) use fixed-size ring buffers
- **Personal assistants** favor multi-layer memory (short + long term)
- **Programming agents** prioritize tool integrity and file tracking
- **Research-oriented** projects (Kimi-CLI) implement unique checkpoint systems

The recommended unified architecture combines the best practices:
- CJK-aware token estimation from PicoClaw
- Three-zone protection from CoPaw
- Two-phase compression from Gemini-CLI
- D-Mail checkpoints from Kimi-CLI
- Multi-layer storage from Nanobot/PicoClaw

This architecture provides a solid foundation for building robust context management in AI agent systems.
