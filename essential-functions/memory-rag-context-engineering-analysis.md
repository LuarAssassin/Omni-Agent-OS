# Memory Systems, RAG & Context Engineering Analysis

## Executive Summary

This analysis examines memory systems, Retrieval-Augmented Generation (RAG) implementations, and context engineering patterns across **15 AI agent projects** (8 Personal Assistants + 7 Programming Agents). Only **3 projects** implement full hybrid RAG with vector + BM25/FTS fusion: **CoPaw**, **IronClaw**, and **OpenClaw**. Most programming agents rely on simpler context management strategies.

| Category | Projects | RAG Maturity |
|----------|----------|--------------|
| **Personal Assistants** | CoPaw, IronClaw, NanoClaw, OpenClaw, ZeroClaw, PicoClaw, MimicClaw, Nanobot | High (3 with full RAG) |
| **Programming Agents** | Aider, Codex, Qwen-Code, Gemini-CLI, Kimi-CLI, Opencode, Pi-Mono | Low-Medium (no full RAG) |

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Three-Layer Memory Architecture](#three-layer-memory-architecture)
3. [Personal Assistants RAG Analysis](#personal-assistants-rag-analysis)
4. [Programming Agents Context Management](#programming-agents-context-management)
5. [Chunking Strategies](#chunking-strategies)
6. [Embedding Providers](#embedding-providers)
7. [Retrieval Mechanisms](#retrieval-mechanisms)
8. [Context Engineering Patterns](#context-engineering-patterns)
9. [Temporal/Recency Handling](#temporalrecency-handling)
10. [Comparison Matrices](#comparison-matrices)
11. [Recommended Unified Architecture](#recommended-unified-architecture)

---

## Core Concepts

### The Three-Layer Memory Model

Advanced AI agents implement a tiered memory architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Three-Layer Memory Model                     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Working Memory (Session Context)                       │
│          ├── Current conversation                               │
│          ├── Tool calls and results                             │
│          └── Immediate state (ephemeral)                        │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Short-Term Memory (Recent History)                     │
│          ├── Daily logs (memory/YYYY-MM-DD.md)                  │
│          ├── Session summaries                                  │
│          └── Recent facts (24-48 hours)                         │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Long-Term Memory (Persistent Knowledge)                │
│          ├── Curated facts (MEMORY.md)                          │
│          ├── User preferences (USER.md)                         │
│          └── Agent identity (SOUL.md, IDENTITY.md)              │
└─────────────────────────────────────────────────────────────────┘
```

### The "Soul" Definition Pattern

Personal assistants use Markdown files to define persistent identity:

| File | Purpose | Projects Using |
|------|---------|----------------|
| **SOUL.md** | Core identity, values, personality | CoPaw, IronClaw, OpenClaw, NanoClaw |
| **AGENTS.md** | Operational manual, workflows | CoPaw, IronClaw, OpenClaw |
| **USER.md** | User preferences, pronouns, timezone | IronClaw, OpenClaw |
| **IDENTITY.md** | Agent name, emoji, avatar | OpenClaw, IronClaw |
| **MEMORY.md** | Long-term curated facts | All personal assistants |
| **HEARTBEAT.md** | Periodic patrol tasks | CoPaw, IronClaw, NanoClaw |

---

## Three-Layer Memory Architecture

### Layer 1: Working Memory (In-Memory)

**CoPaw (Python - AgentScope):**
```python
# src/copaw/agents/memory/memory_manager.py
class MemoryManager(ReMeLight):
    """
    Three-layer memory:
    - Layer 1: In-Memory (session-level conversation)
    - Layer 2: Daily Notes (memory/*.md short-term)
    - Layer 3: MEMORY.md (long-term persistent)
    """
    def __init__(self):
        self.session_context = []  # Working memory
        self.max_input_length = 128 * 1024  # 128K tokens
```

**IronClaw (Rust - Dual Backend):**
```rust
// src/workspace/workspace.rs
pub struct Workspace {
    pub identity: IdentityFiles,  // SOUL.md, AGENTS.md, USER.md
    pub memory: MemoryStore,      // SQLite/PostgreSQL + Vector
    pub sessions: SessionStore,   // JSONL conversation history
}

pub struct WorkingMemory {
    pub current_conversation: Vec<Message>,
    pub tool_outputs: Vec<ToolResult>,
    pub context_budget: TokenBudget,
}
```

### Layer 2: Short-Term Memory (Daily Logs)

**File Organization Pattern:**
```
~/.agent/
├── MEMORY.md                    # Layer 3: Long-term
└── memory/
    ├── 2026-01-15.md            # Layer 2: Daily logs
    ├── 2026-01-16.md
    └── 2026-01-17.md
```

**ZeroClaw (Rust - Categorized):**
```rust
// src/memory/memory_store.rs
pub enum MemoryCategory {
    Core,           // Critical system memory
    Daily,          // Day-to-day interactions
    Conversation,   // Per-conversation context
    Custom(String), // User-defined categories
}

pub struct MemoryEntry {
    pub id: Uuid,
    pub content: String,
    pub category: MemoryCategory,
    pub tags: Vec<String>,
    pub created_at: DateTime<Utc>,
}
```

### Layer 3: Long-Term Memory (Persistent)

**OpenClaw (TypeScript - Hybrid Search):**
```typescript
// src/agents/memory.ts
interface LongTermMemory {
  // Markdown files
  identity: string;      // IDENTITY.md content
  user: string;          // USER.md content
  memory: string;        // MEMORY.md content

  // Vector index
  vectorStore: LanceDBIndex;
  ftsIndex: SQLiteFTS5;
}
```

---

## Personal Assistants RAG Analysis

### 1. CoPaw - ReMeLight Hybrid RAG

**Architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│                     CoPaw RAG Pipeline                          │
├─────────────────────────────────────────────────────────────────┤
│ Document Ingestion                                              │
│   ├── Markdown files (MEMORY.md, memory/*.md)                   │
│   └── Chunking: 400 tokens, 80 overlap                          │
├─────────────────────────────────────────────────────────────────┤
│ Embeddings                                                      │
│   └── text-embedding-v3 (DashScope) or OpenAI                   │
├─────────────────────────────────────────────────────────────────┤
│ Storage                                                         │
│   ├── Vector: Chroma (macOS) / SQLite (Linux) / Local (Windows) │
│   └── BM25: In-memory index                                     │
├─────────────────────────────────────────────────────────────────┤
│ Retrieval                                                       │
│   ├── Vector Search (weight: 0.7)                               │
│   ├── BM25 Search (weight: 0.3)                                 │
│   ├── Candidate pool: 3x topK (max 200)                         │
│   └── Deduplication by chunk ID                                 │
├─────────────────────────────────────────────────────────────────┤
│ Fusion                                                          │
│   └── Weighted score fusion                                     │
└─────────────────────────────────────────────────────────────────┘
```

**Configuration:**
```python
# Memory search configuration
embedding_config = {
    "api_key": "...",
    "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1",
    "model_name": "text-embedding-v3",
    "dimensions": 1024,
}

search_config = {
    "vector_weight": 0.7,
    "bm25_weight": 0.3,
    "candidate_multiplier": 3,
    "max_candidates": 200,
}
```

**Memory Search Tool:**
```python
# src/copaw/agents/tools/memory_search.py
async def memory_search(
    query: str,
    max_results: int = 5,
    min_score: float = 0.1,
) -> ToolResponse:
    """
    Search MEMORY.md and memory/*.md files semantically.
    Uses hybrid search (vector + BM25) with weighted fusion.
    """
    results = await reme.search(
        query=query,
        top_k=max_results,
        min_score=min_score,
    )
    return ToolResponse(results=results)
```

### 2. IronClaw - Dual-Backend Hybrid Search

**Architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│                  IronClaw RAG Pipeline                          │
├─────────────────────────────────────────────────────────────────┤
│ Backends                                                        │
│   ├── PostgreSQL + pgvector (production)                        │
│   └── libSQL/Turso (embedded/edge)                              │
├─────────────────────────────────────────────────────────────────┤
│ Hybrid Search (RRF Fusion)                                      │
│   ├── Vector Search (cosine similarity)                         │
│   ├── FTS5 Full-Text Search                                     │
│   └── Reciprocal Rank Fusion (RRF)                              │
├─────────────────────────────────────────────────────────────────┤
│ Reranking                                                       │
│   ├── MMR (Maximal Marginal Relevance)                          │
│   │   └── lambda: 0.5-0.7 (diversity vs relevance)              │
│   └── Temporal decay (30-day half-life)                         │
└─────────────────────────────────────────────────────────────────┘
```

**Search Implementation:**
```rust
// src/workspace/search.rs
pub struct HybridSearchConfig {
    pub vector_weight: f64,     // 0.6 default
    pub fts_weight: f64,        // 0.4 default
    pub use_rrf: bool,          // RRF vs weighted fusion
    pub mmr_lambda: f64,        // 0.5-0.7 for diversity
    pub temporal_decay: TemporalDecayConfig,
}

pub async fn hybrid_search(
    &self,
    query: &str,
    limit: usize,
    config: &HybridSearchConfig,
) -> Result<Vec<SearchResult>> {
    // Parallel vector + FTS search
    let (vector_results, fts_results) = tokio::join!(
        self.vector_search(query, limit * 2),
        self.fts_search(query, limit * 2),
    );

    // Fuse results
    let fused = if config.use_rrf {
        rrf_fusion(vector_results?, fts_results?, config.vector_weight, config.fts_weight)
    } else {
        weighted_fusion(vector_results?, fts_results?, config.vector_weight, config.fts_weight)
    };

    // Rerank with MMR
    if config.mmr_lambda > 0.0 {
        mmr_rerank(fused, config.mmr_lambda, limit)
    } else {
        Ok(fused.into_iter().take(limit).collect())
    }
}
```

**Four Memory Tools:**
```rust
// src/tools/builtin/memory.rs
pub enum MemoryTool {
    Search { query: String, filters: SearchFilters },
    Write { content: String, category: String, tags: Vec<String> },
    Read { id: String },
    Tree { path: Option<String> },  // Browse memory hierarchy
}
```

**Protected Identity Files:**
```rust
// Identity files cannot be auto-modified
const PROTECTED_FILES: &[&str] = &[
    "SOUL.md",
    "AGENTS.md",
    "USER.md",
    "IDENTITY.md",
];

impl IdentityFile {
    pub fn is_protected(&self) -> bool {
        matches!(self, Self::Soul | Self::Agents | Self::User | Self::Identity)
    }
}
```

### 3. OpenClaw - Most Sophisticated RAG

**Configuration:**
```typescript
// src/agents/memory-search.ts
export type ResolvedMemorySearchConfig = {
  enabled: boolean;
  sources: Array<"memory" | "sessions">;
  provider: "openai" | "local" | "gemini" | "voyage" | "mistral" | "ollama" | "auto";
  model: string;

  // Chunking
  chunkTokens: number;           // 400 default
  chunkOverlapTokens: number;    // 80 default (20%)

  // Hybrid search
  vectorWeight: number;          // 0.7
  textWeight: number;            // 0.3
  candidateMultiplier: number;   // 4x

  // MMR Reranking
  mmr: {
    enabled: boolean;
    lambda: number;              // 0.7 diversity vs relevance
    diversity: number;           // 3 candidates
  };

  // Temporal decay
  temporalDecay: {
    enabled: boolean;
    halfLifeDays: number;        // 30 days
  };
};
```

**Search Pipeline:**
```
Query → Vector Search (weight 0.7) ─┐
      → BM25/FTS5 (weight 0.3) ────┤→ Fusion → MMR Rerank → Temporal Decay → Results
```

**Auto Provider Selection:**
```typescript
function resolveEmbeddingProvider(config: Config): EmbeddingProvider {
  if (config.provider === "auto") {
    // Priority: OpenAI → Gemini → Ollama → Local
    if (process.env.OPENAI_API_KEY) return "openai";
    if (process.env.GEMINI_API_KEY) return "gemini";
    if (process.env.OLLAMA_HOST) return "ollama";
    return "local";
  }
  return config.provider;
}
```

### 4. NanoClaw - Grep-Based Memory

**Simple File-Based Retrieval:**
```python
# nanobot/agent/memory.py
class MemoryManager:
    """Simple grep-based memory retrieval."""

    def search(self, query: str) -> List[str]:
        """Keyword search over MEMORY.md."""
        # No vector search - relies on LLM for semantic understanding
        lines = grep_search(MEMORY_FILE, query.split())
        return rank_by_recency(lines, half_life_days=7)
```

### 5. Other Personal Assistants

**ZeroClaw (SQLite + Categories):**
```rust
// src/tools/builtin/memory_store.rs
pub struct StoreMemoryParams {
    pub content: String,
    pub category: Option<String>,  // Core, Daily, Conversation, Custom
    pub tags: Option<Vec<String>>,
}
```

**PicoClaw (File-Based):**
```go
// pkg/memory/memory.go
func (m *Manager) WriteEntry(entry string) error {
    // Atomic file writes
    // Auto-organizes by date: ~/.picoclaw/2026/03/20260318.md
}
```

**Nanobot (Consolidation-Based):**
```python
# Automatic memory consolidation
# When threshold reached, summarizes old memories
# No vector search, just keyword + recency
```

---

## Programming Agents Context Management

### 1. Aider - RepoMap Context

**No Traditional RAG** - uses codebase structure awareness:

```python
# aider/coders/base_coder.py
class Coder:
    """
    Context management via:
    1. Chat history (recent messages)
    2. Chat summary (compressed older messages)
    3. RepoMap (file structure awareness)
    4. Read-only files (context injection)
    """

# RepoMap: Tree-sitter based code structure extraction
class RepoMap:
    def get_ranked_tags_map(self, chat_files, mentioned_files=None):
        """
        Uses grep_ast to extract function/class tags.
        Builds tree structure with binary search for token limit.
        """
        tags = self.get_tags(chat_files, mentioned_files)
        return self.to_tree(tags, self.max_tokens)
```

**Chat Summarization:**
```python
# Recursive summarization with depth limit
# Summarizes older messages to fit context window
# Preserves recent messages in full
```

### 2. Codex - Memory Traces

**Session-Based Traces:**
```rust
// codex-rs/core/src/memory_trace.rs
pub struct MemoryTrace {
    pub events: Vec<MemoryEvent>,
}

pub enum MemoryEvent {
    ToolCall { name: String, args: Value },
    ToolResult { name: String, output: String },
    UserMessage { content: String },
    AssistantMessage { content: String },
}

// JSONL-based trace files
// Enables session replay and analysis
```

### 3. Opencode - SQLite + Compaction

**Database Schema:**
```typescript
// packages/opencode/src/session/session.sql.ts
export const SessionTable = sqliteTable("session", {
  id: text().$type<SessionID>().primaryKey(),
  project_id: text().$type<ProjectID>().notNull(),
  parent_id: text().$type<SessionID>(),  // Fork support
  title: text().notNull(),
  summary_additions: integer(),
  summary_deletions: integer(),
  summary_files: integer(),
  revert: text({ mode: "json" }).$type<RevertState>(),
});

export const PartTable = sqliteTable("part", {
  id: text().$type<PartID>().primaryKey(),
  message_id: text().$type<MessageID>().notNull(),
  session_id: text().$type<SessionID>().notNull(),
  data: text({ mode: "json" }).$type<PartData>(),
});

// Part types: TextPart, ToolCallPart, ToolResultPart,
//             FilePart, ImagePart, CompactionPart
```

**Overflow Detection:**
```typescript
// packages/opencode/src/session/compaction.ts
const COMPACTION_BUFFER = 20_000;
const PRUNE_MINIMUM = 20_000;
const PRUNE_PROTECT = 40_000;

export async function isOverflow(input) {
  const count = input.tokens.total ||
    input.tokens.input + input.tokens.output +
    input.tokens.cache.read + input.tokens.cache.write;

  const reserved = config.compaction?.reserved ??
    Math.min(COMPACTION_BUFFER, maxOutputTokens(input.model));

  return count >= usable;
}

// Replace old tool outputs with "[Old tool result content cleared]"
```

### 4. Kimi-CLI - Checkpoint-Based Context

**D-Mail Checkpoint System:**
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
```

### 5. Gemini-CLI - Hierarchical Memory

**JIT Context Discovery:**
```typescript
// packages/core/src/services/contextManager.ts
interface MemoryHierarchy {
  global: string;           // ~/.gemini/memory.json
  extension: string;        // Extension-specific
  project: string;          // ./.gemini/memory.json
}

// Discovers relevant context based on accessed paths
// Loads memory just-in-time for relevant files
// Hierarchical override (project > extension > global)
```

### 6. Qwen-Code - ACP Protocol Sessions

**Session Store:**
```typescript
// packages/webui/src/types/chat.ts
interface Session {
  id: string;
  messages: Message[];
  createdAt: number;
  updatedAt: number;
}

// Session grouping by date
// ACP (Agent Communication Protocol) for state sync
```

### 7. Pi-Mono - Two-File Context

**Dual-File Approach:**
```typescript
// packages/mom/src/store.ts
interface ContextStore {
  // context.jsonl - Structured API messages
  // log.jsonl - Human-readable conversation log
}

class SessionManager {
  async sync(): Promise<void> {
    // Sync between dual files
    // Machine-readable + human-readable
  }
}
```

---

## Chunking Strategies

### Token-Based Chunking

**OpenClaw Configuration:**
```typescript
// Default chunking configuration
const DEFAULT_CHUNK_CONFIG = {
  chunkTokens: 400,           // Target chunk size
  chunkOverlapTokens: 80,     // 20% overlap
  boundaryDetection: "paragraph", // paragraph | sentence | token
  codeAware: true,            // Respect function boundaries
};
```

### Code-Aware Chunking

**Aider's Tree-sitter Approach:**
```python
# Uses grep_ast with tree-sitter for code-aware chunking
# Respects function and class boundaries

def get_tags(self, fname: str) -> List[Tag]:
    """
    Extracts function/class definitions using tree-sitter.
    Each tag represents a code entity (function, class, method).
    """
    lang = get_language(fname)
    query_scm = self.TAGS_CACHE.get(lang)
    # Tree-sitter query for definitions
    # Returns structured tags with line numbers and types
```

**CoPaw's ReMe Integration:**
```python
# ReMeLight handles chunking with language awareness
from reme import DocumentChunker

chunker = DocumentChunker(
    chunk_size=400,
    chunk_overlap=80,
    respect_boundaries=True,  # Respect paragraphs/code blocks
    language_detection=True,   # Auto-detect programming language
)
```

### Chunking Configuration Comparison

| Project | Chunk Size | Overlap | Boundary Detection | Code-Aware |
|---------|-----------|---------|-------------------|------------|
| **OpenClaw** | 400 tokens | 80 (20%) | paragraph/sentence | Yes |
| **CoPaw** | 400 tokens | 80 (20%) | paragraph | Yes (ReMe) |
| **IronClaw** | 512 tokens | 128 (25%) | sentence | Yes |
| **NanoClaw** | N/A | N/A | File boundaries | No |
| **Aider** | N/A | N/A | Function/class | Yes |
| **Codex** | Session-based | N/A | Message boundaries | N/A |

---

## Embedding Providers

### Multi-Provider Support

**OpenClaw (7 Providers):**
```typescript
type EmbeddingProvider =
  | "openai"      // text-embedding-3-small/large
  | "local"       // Local transformer models
  | "gemini"      // Google embedding models
  | "voyage"      // Voyage AI embeddings
  | "mistral"     // Mistral embeddings
  | "ollama"      // Self-hosted via Ollama
  | "auto";       // Auto-select based on availability
```

**IronClaw (Rust):**
```rust
pub enum EmbeddingProvider {
    OpenAi { api_key: String, model: String },
    Ollama { host: String, model: String },
    Local { model_path: PathBuf },
}

impl EmbeddingProvider {
    pub async fn embed(&self, texts: &[String]) -> Result<Vec<Vec<f32>>> {
        match self {
            Self::OpenAi { api_key, model } => {
                // Call OpenAI embedding API
            }
            Self::Ollama { host, model } => {
                // Call local Ollama instance
            }
            Self::Local { model_path } => {
                // Run onnxruntime embedding
            }
        }
    }
}
```

**CoPaw (Python):**
```python
# ReMeLight embedding configuration
EMBEDDING_PROVIDERS = {
    "openai": {
        "model": "text-embedding-3-small",
        "dimensions": 1536,
    },
    "ollama": {
        "model": "nomic-embed-text",
        "dimensions": 768,
    },
    "huggingface": {
        "model": "sentence-transformers/all-MiniLM-L6-v2",
        "dimensions": 384,
    }
}
```

### Embedding Provider Comparison

| Provider | Dimensions | Default Model | Local/Cloud |
|----------|-----------|---------------|-------------|
| OpenAI | 1536 | text-embedding-3-small | Cloud |
| OpenAI | 3072 | text-embedding-3-large | Cloud |
| Ollama | 768 | nomic-embed-text | Local |
| HuggingFace | 384 | all-MiniLM-L6-v2 | Local |
| Gemini | 768 | embedding-001 | Cloud |
| Voyage | 1024 | voyage-2 | Cloud |
| Mistral | 1024 | mistral-embed | Cloud |

---

## Retrieval Mechanisms

### Hybrid Search Fusion Algorithms

#### Weighted Score Fusion (CoPaw/OpenClaw)
```python
def weighted_fusion(vector_results, bm25_results, vector_weight=0.7, bm25_weight=0.3):
    """
    Weighted fusion using normalized scores.
    Most common approach across projects.
    """
    from collections import defaultdict
    scores = defaultdict(float)

    # Normalize vector scores (cosine similarity)
    max_vector = max(r.score for r in vector_results)
    for result in vector_results:
        scores[result.id] += vector_weight * (result.score / max_vector)

    # Normalize BM25 scores
    max_bm25 = max(r.score for r in bm25_results)
    for result in bm25_results:
        scores[result.id] += bm25_weight * (result.score / max_bm25)

    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

#### Reciprocal Rank Fusion (IronClaw)
```python
def rrf_fusion(vector_results, bm25_results, k=60):
    """
    Reciprocal Rank Fusion combines results from multiple sources.
    k: constant that dampens rank impact (typically 60).
    Better for combining ranked lists of different scales.
    """
    from collections import defaultdict
    scores = defaultdict(float)

    # Score vector results by rank
    for rank, result in enumerate(vector_results):
        scores[result.id] += 1.0 / (k + rank + 1)

    # Score BM25 results by rank
    for rank, result in enumerate(bm25_results):
        scores[result.id] += 1.0 / (k + rank + 1)

    # Sort by combined RRF score
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

### MMR Reranking

**Maximal Marginal Relevance:**
```typescript
// Balances relevance vs diversity
function mmr_rerank(
  results: SearchResult[],
  lambda: number,      // 0.7 = 70% relevance, 30% diversity
  k: number,           // Top K to return
  diversityCandidates: number = 3
): SearchResult[] {
  const selected: SearchResult[] = [];
  const candidates = [...results];

  while (selected.length < k && candidates.length > 0) {
    // Score each candidate
    const scored = candidates.map(c => {
      const relevance = c.score;
      const diversity = selected.length === 0 ? 0 :
        Math.max(...selected.map(s => similarity(c.embedding, s.embedding)));
      return {
        result: c,
        mmrScore: lambda * relevance - (1 - lambda) * diversity
      };
    });

    // Pick highest MMR score
    const best = scored.reduce((a, b) => a.mmrScore > b.mmrScore ? a : b);
    selected.push(best.result);
    candidates.splice(candidates.indexOf(best.result), 1);
  }

  return selected;
}
```

### Memory Tool Interfaces

**OpenClaw:**
```typescript
// Memory retrieval tools exposed to agent
interface MemoryTools {
  searchMemory: (query: string, options: {
    topK: number;
    recencyBias: number;
    sourceFilter?: ("memory" | "sessions")[];
  }) => Promise<MemoryResult[]>;

  getMemory: (id: string) => Promise<MemoryEntry>;

  searchSessions: (query: string, options: {
    timeRange?: [Date, Date];
    channelFilter?: string[];
  }) => Promise<SessionResult[]>;
}

interface MemoryResult {
  id: string;
  content: string;
  source: "memory" | "session";
  timestamp: Date;
  relevanceScore: number;
  citation: string;  // For attribution
}
```

---

## Context Engineering Patterns

### Three-Zone Context Model

**CoPaw Implementation:**
```
┌─────────────────────────────────────────────────────────────┐
│              Context Window Management                       │
│             (max_input_length: 128K tokens)                  │
├─────────────────────────────────────────────────────────────┤
│ System Prompt (fixed)                                       │
│ - Core identity and capabilities                            │
│ - ~500-1000 tokens                                          │
├─────────────────────────────────────────────────────────────┤
│ Compacted Summary Zone                                      │
│ - Summarized older conversation                             │
│ - Generated when compact_ratio (0.75) exceeded              │
│ - ~10-20% of window                                         │
├─────────────────────────────────────────────────────────────┤
│ Compactable Zone                                            │
│ - Active conversation history                               │
│ - Tool outputs                                              │
│ - File contents                                             │
│ - Compacted when full                                       │
├─────────────────────────────────────────────────────────────┤
│ Reserved Zone (reserve_ratio: 0.1)                          │
│ - Space for response generation                             │
│ - 10% of context window                                     │
└─────────────────────────────────────────────────────────────┘
```

**Configuration:**
```python
max_input_length: int = 128 * 1024        # 128K tokens
memory_compact_ratio: float = 0.75        # 75% trigger
memory_reserve_ratio: float = 0.1         # 10% reserved
```

### Post-Compaction Reinjection

**OpenClaw Pattern:**
```typescript
// src/auto-reply/reply/post-compaction-context.ts
const DEFAULT_POST_COMPACTION_SECTIONS = [
  "Session Startup",
  "Red Lines"
];

// After compaction, re-inject critical sections from AGENTS.md
// Ensures agent doesn't forget critical instructions

async function compactAndReinject(context: Context): Promise<Context> {
  const summary = await summarizeContext(context.messages);

  // Reinject critical context from AGENTS.md
  const criticalContext = await loadCriticalContext();

  return {
    systemPrompt: context.systemPrompt,
    summary: summary,
    recentMessages: context.recentMessages.slice(-10),
    reinjected: criticalContext,  // Ensures identity preserved
  };
}
```

### Tool Output Pruning

**Opencode:**
```typescript
// Prune old tool outputs to save tokens
const PRUNE_MINIMUM = 20_000;   // Minimum threshold before pruning
const PRUNE_PROTECT = 40_000;   // Protect most recent tokens

function pruneToolOutputs(context: Context): Context {
  const toolOutputs = context.messages.filter(m => m.role === "tool");
  const protected_outputs = toolOutputs.slice(-5);  // Keep last 5
  const prunable = toolOutputs.slice(0, -5);

  // Summarize or replace old tool outputs
  const pruned = prunable.map(output => ({
    ...output,
    content: "[Old tool result content cleared]",
  }));

  return {
    ...context,
    messages: [
      ...context.messages.filter(m => m.role !== "tool"),
      ...pruned,
      ...protected_outputs,
    ],
  };
}
```

---

## Temporal/Recency Handling

### Exponential Time Decay

**OpenClaw Implementation:**
```typescript
interface TemporalDecayConfig {
  enabled: boolean;
  halfLifeDays: number;  // Default: 30 days
  minScore: number;      // Floor for decayed scores (0.1)
}

function applyTemporalDecay(
  results: SearchResult[],
  config: TemporalDecayConfig
): SearchResult[] {
  const now = Date.now();
  const halfLifeMs = config.halfLifeDays * 24 * 60 * 60 * 1000;

  return results.map(result => {
    const age = now - result.timestamp.getTime();
    const decayFactor = Math.pow(0.5, age / halfLifeMs);

    return {
      ...result,
      score: Math.max(
        result.score * decayFactor,
        result.score * config.minScore
      ),
    };
  });
}
```

**CoPaw Recency-Weighted Retrieval:**
```python
def calculate_recency_score(timestamp: datetime, half_life_days: int = 30) -> float:
    """
    Exponential decay based on age.
    score = e^(-λ * age) where λ = ln(2) / half_life
    """
    import math
    age_days = (datetime.now() - timestamp).days
    decay_constant = math.log(2) / half_life_days
    return math.exp(-decay_constant * age_days)

# Combined relevance + recency score
def combined_score(
    relevance: float,
    timestamp: datetime,
    recency_weight: float = 0.3
) -> float:
    recency = calculate_recency_score(timestamp)
    return (1 - recency_weight) * relevance + recency_weight * recency
```

**IronClaw Hybrid Scoring:**
```rust
pub fn score_with_recency(
    base_score: f32,
    timestamp: DateTime<Utc>,
    config: &RecencyConfig,
) -> f32 {
    let age = Utc::now().signed_duration_since(timestamp).num_days() as f32;
    let half_life = config.half_life_days as f32;

    // Exponential decay: score * 2^(-age/half_life)
    let decay = 0.5_f32.powf(age / half_life);
    let recency_bonus = 1.0 + (config.max_boost - 1.0) * decay;

    base_score * recency_bonus
}
```

### Temporal Decay Configuration Summary

| Project | Decay Function | Half-Life | Min Score | Configurable |
|---------|---------------|-----------|-----------|--------------|
| **OpenClaw** | Exponential | 30 days | 0.1 | Yes |
| **CoPaw** | Exponential | 30 days | 0.0 | Yes |
| **IronClaw** | Exponential with boost | 30 days | N/A | Yes |
| **NanoClaw** | Linear age penalty | 7 days | 0.0 | No |
| **Aider** | None (uses repo structure) | N/A | N/A | N/A |
| **Codex** | Session recency | Session-based | N/A | No |

---

## Comparison Matrices

### RAG Implementation Matrix

| Feature | CoPaw | IronClaw | OpenClaw | NanoClaw | Others |
|---------|-------|----------|----------|----------|--------|
| **Vector Search** | ✓ | ✓ | ✓ | ✗ | ✗ |
| **BM25/FTS** | ✓ | ✓ (FTS5) | ✓ (FTS5) | ✗ | ✗ |
| **Hybrid Search** | ✓ | ✓ | ✓ | ✗ | ✗ |
| **RRF Fusion** | ✗ | ✓ | ✗ | ✗ | ✗ |
| **MMR Reranking** | ✓ | ✓ | ✓ | ✗ | ✗ |
| **Temporal Decay** | ✓ | ✓ | ✓ | ✓ | ✗ |
| **Citation Support** | ✗ | ✓ | ✓ | ✗ | ✗ |
| **Multi-source** | ✗ | ✓ | ✓ | ✗ | ✗ |

### RAG Configuration Comparison

| Parameter | CoPaw | IronClaw | OpenClaw |
|-----------|-------|----------|----------|
| **Chunk Size** | 400 tokens | 512 tokens | 400 tokens |
| **Chunk Overlap** | 80 tokens (20%) | 128 tokens (25%) | 80 tokens (20%) |
| **Vector Weight** | 0.7 | 0.6 | 0.7 |
| **Text Weight** | 0.3 | 0.4 | 0.3 |
| **Candidate Multiplier** | 3x | 4x | 4x |
| **MMR Lambda** | 0.5 | 0.5-0.7 | 0.7 |
| **Decay Half-Life** | 30 days | 30 days | 30 days |
| **Fusion Method** | Weighted | RRF/Weighted | Weighted |

### Memory Tools Availability

| Project | Search | Write | Read | Tree/List | Protected Files |
|---------|--------|-------|------|-----------|-----------------|
| **CoPaw** | ✓ (hybrid) | ✓ | ✓ | ✗ | ✗ |
| **IronClaw** | ✓ (hybrid) | ✓ | ✓ | ✓ | ✓ |
| **OpenClaw** | ✓ (hybrid) | ✓ | ✓ | ✗ | ✓ |
| **NanoClaw** | ✓ (keyword) | ✓ | ✓ | ✗ | ✗ |
| **ZeroClaw** | ✓ (SQL) | ✓ | ✓ | ✓ | ✗ |
| **PicoClaw** | ✓ (file) | ✓ | ✓ | ✓ | ✗ |
| **Nanobot** | ✓ (grep) | ✓ | ✓ | ✗ | ✗ |
| **Aider** | RepoMap | ✗ | ✗ | ✗ | ✗ |
| **Codex** | Traces | ✗ | ✗ | ✗ | ✗ |
| **Opencode** | ✗ | ✗ | ✗ | ✗ | ✗ |

### Context Compression Strategies

| Project | Trigger | Method | Post-Compaction | Checkpoint |
|---------|---------|--------|-----------------|------------|
| **CoPaw** | 75% full | Summarize | Append summary | ✗ |
| **IronClaw** | Configurable | Summarize | Reinject identity | ✗ |
| **OpenClaw** | Configurable | Summarize | Reinject AGENTS.md | ✗ |
| **Aider** | Window full | Summarize | Keep summary | ✗ |
| **Gemini-cli** | 70% full | Compress | JIT rediscovery | ✗ |
| **Kimi-cli** | Manual | Checkpoint | N/A | ✓ |
| **Opencode** | Buffer-based | Compact | CompactionPart | ✓ |

---

## Recommended Unified Architecture

### Fusion Memory Architecture (4-Layer)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Unified Memory Architecture                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Soul Definition (Human-Readable, Protected)            │
│   ├── SOUL.md          - Core identity, values                  │
│   ├── AGENTS.md        - Operational manual                     │
│   ├── USER.md          - User profile, preferences              │
│   └── IDENTITY.md      - Agent name, avatar                     │
│   [Protected: cannot be auto-modified by agent]                 │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Long-Term Memory (Agent-Managed, Searchable)           │
│   ├── MEMORY.md        - Curated facts, decisions               │
│   ├── memory/*.md      - Daily logs (YYYY-MM-DD.md)             │
│   └── Vector Index     - Semantic search layer                  │
│   [Hybrid: Vector + BM25/FTS with RRF fusion]                   │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Session Storage (Structured, Queryable)                │
│   ├── SQLite/PostgreSQL                                       │
│   │   ├── sessions table   - Session metadata                   │
│   │   ├── messages table   - Message history                    │
│   │   └── parts table      - Tool calls, files, images          │
│   └── JSONL backup       - Portability, human readability       │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Working Memory (In-Memory, Ephemeral)                  │
│   ├── Current conversation                                    │
│   ├── Tool call results                                       │
│   └── Context window management (Three-Zone Model)            │
└─────────────────────────────────────────────────────────────────┘
```

### Unified RAG Pipeline

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
│    └── Hybrid Fusion (Weighted or RRF)                          │
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

### Unified Context Management

```
┌─────────────────────────────────────────────────────────────────┐
│                  Unified Context Management                     │
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
│ Post-Compaction Actions                                         │
│   ├── Reinject SOUL.md key principles                           │
│   ├── Reinject AGENTS.md red lines                              │
│   └── Update compaction summary                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Soul Files** | SOUL.md + AGENTS.md + USER.md | CoPaw/IronClaw pattern, human-readable |
| **Storage** | SQLite + Markdown files | IronClaw/OpenClaw dual approach |
| **Vector DB** | Chroma/pgvector/LanceDB | Proven solutions across projects |
| **Embeddings** | text-embedding-3-small | OpenAI standard, good quality/cost |
| **Context** | Three-zone model | CoPaw proven approach |
| **Compaction** | Summary + reinjection | OpenClaw preserves critical context |
| **Session** | SQLite + JSONL dual | Opencode/Pi-mono portability |
| **Tools** | search/write/read/tree | IronClaw comprehensive pattern |

---

## Key Insights

### RAG Maturity Spectrum

```
High ─┬─ OpenClaw (Full hybrid + MMR + temporal + 7 providers)
      ├─ IronClaw (Dual backend + RRF + WASM isolation)
      ├─ CoPaw (ReMe integration + multi-layer memory)
      │
Medium ─┼─ NanoClaw (File-based + grep + recency)
        ├─ Aider (RepoMap + tree-sitter)
        │
Low ────┴─ Codex (Session traces only)
        ├─ Opencode (SQLite sessions)
        ├─ Qwen-Code (Basic context)
        └─ Gemini-CLI (Minimal retrieval)
```

### Best Practices Identified

1. **Hybrid Search**: Vector (0.7) + BM25 (0.3) weighting provides optimal relevance
2. **RRF vs Weighted**: RRF better for combining different scales; Weighted simpler
3. **MMR Reranking**: Lambda 0.5-0.7 balances relevance vs diversity
4. **Chunking**: 400-512 tokens with 20% overlap is standard
5. **Context Zones**: System (fixed) + Compacted + Active + Reserved (10%)
6. **Protected Identity**: Mark SOUL/IDENTITY files as protected from auto-modification
7. **Temporal Decay**: 30-day half-life with exponential decay is common
8. **Provider Fallback**: Auto-select embedding provider based on availability

### Project Strengths

| Project | Memory Strength | RAG Strength | Context Strength |
|---------|----------------|--------------|------------------|
| **CoPaw** | File layering, ReMe | Vector+BM25 hybrid | Three-zone model |
| **IronClaw** | Protected files, dual-backend | FTS5+Vector RRF | Four memory tools |
| **OpenClaw** | Multi-agent isolation | MMR+temporal decay | Post-compaction reinject |
| **Aider** | Git integration | RepoMap | Code-aware context |
| **Opencode** | SQLite structured | - | Compaction parts |

---

## Summary

This comprehensive analysis of 15 AI agent projects reveals a clear divide in RAG maturity:

**Personal Assistants** (8 projects):
- 3 implement full hybrid RAG (CoPaw, IronClaw, OpenClaw)
- Emphasize file-based soul definitions and long-term memory
- Complex retrieval with vector + BM25 + MMR + temporal decay

**Programming Agents** (7 projects):
- None implement full RAG
- Focus on context window management and session storage
- Rely on RepoMap (Aider), session traces (Codex), or compaction (Opencode)

The recommended unified architecture synthesizes best practices:
- **4-layer memory** (Soul → Long-term → Session → Working)
- **Hybrid RAG** (Vector + BM25/FTS with MMR reranking)
- **Three-zone context** (System + Compacted + Active)
- **Protected identity files** (IronClaw pattern)
- **Dual storage** (SQLite + JSONL for portability)

This architecture provides a robust foundation for production-grade AI agent memory systems.
