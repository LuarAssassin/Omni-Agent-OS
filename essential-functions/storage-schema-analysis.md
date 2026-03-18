# Storage Schema 与核心对象模型分析

## 目录

1. [执行摘要](#执行摘要)
2. [存储架构总览](#存储架构总览)
3. [Personal Assistant 项目](#personal-assistant-项目)
4. [Programming Agent 项目](#programming-agent-项目)
5. [Browser Automation 项目](#browser-automation-项目)
6. [存储系统对比矩阵](#存储系统对比矩阵)
7. [对象模型模式](#对象模型模式)
8. [迁移策略对比](#迁移策略对比)
9. [内存管理架构](#内存管理架构)
10. [推荐统一存储架构](#推荐统一存储架构)

---

## 执行摘要

本分析覆盖15个AI Agent项目的存储架构，从嵌入式ESP32的SPIFFS到企业级PostgreSQL集群，呈现完整的存储技术光谱：

| 成熟度级别 | 代表项目 | 存储技术栈 |
|-----------|---------|-----------|
| **Level 4** | IronClaw, edict | PostgreSQL + libSQL双后端 + 向量存储 |
| **Level 3** | NanoClaw, OpenClaw | SQLite + JSONL + 外键约束 |
| **Level 2** | Codex, Gemini CLI, Qwen Code | 内存缓存 + 文件持久化 + 事件流 |
| **Level 1** | Nanobot, Kimi CLI, ZeroClaw | JSONL/Markdown + 简单索引 |
| **Level 0** | Mimiclaw, PicoClaw | SPIFFS/纯内存 + 极简存储 |

**关键洞察**：
- **事件流架构**成为现代Agent的标准（Codex、Gemini CLI、Pi Mono、Qwen Code）
- **双层内存模式**（Nanobot的MEMORY.md+HISTORY.md）在轻量级项目中流行
- **PostgreSQL JSONB**在需要复杂查询的企业场景占主导
- **加密存储**仅在agent-browser和IronClaw中实现完整

---

## 存储架构总览

### 分层存储模型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Storage Architecture                            │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │   Runtime    │  │   Session    │  │  Persistent  │  │   Archive   │ │
│  │   Memory     │  │   Store      │  │   Store      │  │   Store     │ │
│  │              │  │              │  │              │  │             │ │
│  │ - Context    │  │ - JSONL      │  │ - SQLite     │  │ - S3        │ │
│  │ - Cache      │  │ - Markdown   │  │ - PostgreSQL │  │ - Glacier   │ │
│  │ - Hot data   │  │ - TOML       │  │ - Vector DB  │  │ - Cold tape │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘ │
│         │                 │                 │                  │        │
│         └─────────────────┼─────────────────┼──────────────────┘        │
│                           ▼                 ▼                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Object Model Layer                           │   │
│  │                                                                  │   │
│  │  User ──► Workspace ──► Session ──► Turn ──► Message ──► Part   │   │
│  │                    │                  │                         │   │
│  │                    ▼                  ▼                         │   │
│  │              Agent ──► Tool ──► Skill ──► Memory                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Serialization: JSON / MessagePack / Binary / Encrypted / Compressed   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Personal Assistant 项目

### IronClaw (Rust) — Level 4 企业级存储

**File**: `ironclaw/src/db/CLAUDE.md`, `ironclaw/src/workspace/README.md`

IronClaw实现了双后端存储架构（PostgreSQL + libSQL/Turso），支持向量搜索和全文检索：

```rust
// 双后端数据库 trait 抽象
#[async_trait]
pub trait Database: Send + Sync {
    async fn init(&self) -> Result<()>;
    async fn health_check(&self) -> Result<HealthStatus>;
    async fn begin_transaction(&self) -> Result<Transaction>;
}

// PostgreSQL 实现
pub struct PostgresDb {
    pool: PgPool,
    config: PostgresConfig,
}

// libSQL/Turso 实现
pub struct LibsqlDb {
    conn: libsql::Connection,
    config: LibsqlConfig,
}
```

**Workspace Memory Schema**:
```rust
pub struct Workspace {
    pub id: Uuid,
    pub name: String,
    pub path: PathBuf,
    pub identities: Vec<IdentityFile>,  // AGENTS.md, SOUL.md, USER.md
    pub memory_backend: MemoryBackend,
}

pub enum MemoryBackend {
    Markdown,  // Simple files
    Sqlite,    // FTS + vector search
    Hybrid,    // FTS + vector via RRF
}
```

**向量搜索实现**:
```rust
// Hybrid search with RRF (Reciprocal Rank Fusion)
pub async fn hybrid_search(
    &self,
    query: &str,
    embedding: Vec<f32>,
    limit: usize,
) -> Result<Vec<MemoryEntry>> {
    // 1. Full-text search
    let fts_results = self.fts_search(query, limit * 2).await?;

    // 2. Vector similarity search
    let vector_results = self.vector_search(embedding, limit * 2).await?;

    // 3. RRF fusion
    let fused = reciprocal_rank_fusion(fts_results, vector_results);

    Ok(fused.into_iter().take(limit).collect())
}
```

**Secrets 加密存储**:
```rust
pub struct SecretStore {
    backend: SecretBackend,
    master_key: Protected<Key>,  // Zeroize on drop
}

pub enum SecretBackend {
    File(Aes256GcmStore),       // AES-256-GCM encrypted file
    Keyring(OsKeyring),          // OS native keychain
    Hardware(SecurityKey),       // YubiKey etc.
}
```

---

### NanoClaw (TypeScript) — Level 3 结构化存储

**File**: `nanoclaw/src/db.ts`

使用 better-sqlite3 实现多频道聊天机器人的完整CRUD：

```typescript
// Schema with foreign key constraints
database.exec(`
  CREATE TABLE IF NOT EXISTS chats (
    jid TEXT PRIMARY KEY,
    name TEXT,
    last_message_time TEXT,
    channel TEXT,
    is_group INTEGER DEFAULT 0
  );

  CREATE TABLE IF NOT EXISTS messages (
    id TEXT,
    chat_jid TEXT,
    sender TEXT,
    content TEXT,
    timestamp TEXT,
    is_from_me INTEGER,
    is_bot_message INTEGER DEFAULT 0,
    PRIMARY KEY (id, chat_jid),
    FOREIGN KEY (chat_jid) REFERENCES chats(jid)
  );

  CREATE TABLE IF NOT EXISTS scheduled_tasks (
    id TEXT PRIMARY KEY,
    group_folder TEXT NOT NULL,
    chat_jid TEXT NOT NULL,
    prompt TEXT NOT NULL,
    schedule_type TEXT NOT NULL,  -- cron/interval/once
    schedule_value TEXT NOT NULL,
    next_run TEXT,
    status TEXT DEFAULT 'active',
    created_at TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS sessions (
    group_folder TEXT PRIMARY KEY,
    session_id TEXT NOT NULL
  );
`);
```

**Schema Migration**:
```typescript
// Runtime ALTER TABLE migration
const result = database.prepare(
  "SELECT COUNT(*) as count FROM pragma_table_info('messages') WHERE name='is_bot_message'"
).get() as { count: number };

if (result.count === 0) {
  database.exec('ALTER TABLE messages ADD COLUMN is_bot_message INTEGER DEFAULT 0');
  database.exec('UPDATE messages SET is_bot_message = 0');
}
```

---

### Nanobot (Python) — Level 1 文件驱动

**File**: `nanobot/nanobot/agent/memory.py`, `nanobot/nanobot/session/manager.py`

双层内存架构（事实+历史）：

```python
class MemoryStore:
    """Two-layer memory: MEMORY.md (facts) + HISTORY.md (logs)"""

    def __init__(self, workspace: Path):
        self.memory_dir = ensure_dir(workspace / "memory")
        self.memory_file = self.memory_dir / "MEMORY.md"      # Long-term facts
        self.history_file = self.memory_dir / "HISTORY.md"    # Searchable logs

    async def add_memory(self, fact: str, source: str = "tool"):
        """Append fact to MEMORY.md with timestamp."""
        timestamp = datetime.now().isoformat()
        entry = f"\n## [{timestamp}] {source}\n{fact}\n"
        async with aiofiles.open(self.memory_file, "a") as f:
            await f.write(entry)

    async def log_interaction(self, role: str, content: str):
        """Append to HISTORY.md for grep-based search."""
        timestamp = datetime.now().isoformat()
        entry = f"[{timestamp}] {role}: {content[:200]}...\n"
        async with aiofiles.open(self.history_file, "a") as f:
            await f.write(entry)
```

**Session Manager with JSONL**:
```python
@dataclass
class Session:
    key: str  # channel:chat_id
    messages: list[dict[str, Any]] = field(default_factory=list)
    last_consolidated: int = 0

class SessionManager:
    def _get_session_path(self, key: str) -> Path:
        safe_key = safe_filename(key.replace(":", "_"))
        return self.sessions_dir / f"{safe_key}.jsonl"

    async def append_message(self, session_key: str, message: dict):
        path = self._get_session_path(session_key)
        async with aiofiles.open(path, "a") as f:
            await f.write(json.dumps(message, default=str) + "\n")

        # Auto-consolidate when threshold exceeded
        session = self._sessions.get(session_key)
        if len(session.messages) > self.consolidation_threshold:
            await self._consolidate_session(session_key)
```

---

### Mimiclaw (C/ESP32) — Level 0 嵌入式存储

**File**: `mimiclaw/main/memory/session_mgr.c`

ESP32微控制器上的极简存储：

```c
// JSON Lines format on SPIFFS flash
static void session_path(const char *chat_id, char *buf, size_t size)
{
    snprintf(buf, size, "%s/tg_%s.jsonl", MIMI_SPIFFS_SESSION_DIR, chat_id);
}

esp_err_t session_append(const char *chat_id, const char *role, const char *content)
{
    char path[64];
    session_path(chat_id, path, sizeof(path));
    FILE *f = fopen(path, "a");

    cJSON *obj = cJSON_CreateObject();
    cJSON_AddStringToObject(obj, "role", role);
    cJSON_AddStringToObject(obj, "content", content);
    cJSON_AddNumberToObject(obj, "ts", (double)time(NULL));

    char *line = cJSON_PrintUnformatted(obj);
    fprintf(f, "%s\n", line);  // JSONL append-only

    cJSON_Delete(obj);
    free(line);
    fclose(f);
    return ESP_OK;
}

// Ring buffer for memory-constrained devices
#define MAX_SESSION_MSGS 20

void session_gc(const char *chat_id)
{
    // Keep only last N messages, archive older to daily markdown
    char daily_path[64];
    time_t now = time(NULL);
    struct tm *tm = localtime(&now);
    strftime(daily_path, sizeof(daily_path), "%s/%04d-%02d-%02d.md",
             MIMI_SPIFFS_DAILY_DIR, tm->tm_year + 1900, tm->tm_mon + 1, tm->tm_mday);
}
```

---

### ZeroClaw (Rust) — Level 1 Trait驱动

**File**: `zeroclaw/src/memory/traits.rs`

Trait抽象支持多种后端：

```rust
#[async_trait]
pub trait Memory: Send + Sync {
    async fn get(&self, key: &str) -> Result<Option<String>>;
    async fn set(&self, key: &str, value: &str) -> Result<()>;
    async fn delete(&self, key: &str) -> Result<()>;
    async fn search(&self, query: &str) -> Result<Vec<MemoryEntry>>;
}

// Implementations
pub struct FileMemory { /* JSON files */ }
pub struct SqliteMemory { /* SQLite FTS */ }
pub struct VectorMemory { /* Embedding-based */ }
```

---

### CoPaw (Python) — Level 2 Pydantic + JSON

**File**: `copaw/src/copaw/app/runner/models.py`

```python
class ChatSpec(BaseModel):
    """Chat session with metadata."""
    id: str = Field(default_factory=lambda: str(uuid4()))
    name: str = Field(default="New Chat")
    session_id: str = Field(...)  # format: channel:user_id
    user_id: str = Field(...)
    channel: str = Field(default=DEFAULT_CHANNEL)
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    meta: Dict[str, Any] = Field(default_factory=dict)

class ChatsFile(BaseModel):
    """Root container with schema versioning."""
    version: int = 1  # Schema version for migrations
    chats: list[ChatSpec] = Field(default_factory=list)
```

**Atomic Write Pattern**:
```python
class JSONChatRepository:
    async def _atomic_save(self, jobs_file: ChatsFile) -> None:
        async with self._lock:
            # Serialize
            payload = jobs_file.model_dump(mode="json")

            # Write temp
            tmp_path = self._path.with_suffix(self._path.suffix + ".tmp")
            tmp_path.write_text(
                json.dumps(payload, ensure_ascii=False, indent=2, sort_keys=True),
                encoding="utf-8",
            )

            # Atomic replace
            tmp_path.replace(self._path)
```

---

## Programming Agent 项目

### Codex (Rust/TypeScript) — Level 2 事件流

**File**: `codex/codex-rs/app-server-protocol/schema/typescript/v2/Thread.ts`

```typescript
export type Thread = {
    id: string,
    preview: string,           // First user message
    ephemeral: boolean,        // Don't materialize on disk
    modelProvider: string,
    createdAt: number,         // Unix timestamp
    updatedAt: number,
    status: ThreadStatus,
    path: string | null,       // Path on disk
    cwd: string,               // Working directory
    cliVersion: string,
    source: SessionSource,     // CLI, VSCode, etc.
    agentNickname: string | null,
    gitInfo: GitInfo | null,
    turns: Array<Turn>,
};

export type Turn = {
    id: string,
    items: Array<ThreadItem>,
    status: TurnStatus,
    error: TurnError | null,
};

export type ThreadItem =
    | { type: "message"; role: "user" | "assistant"; content: ContentBlock[] }
    | { type: "tool_call"; tool_call: ToolCall }
    | { type: "tool_result"; tool_result: ToolResult }
    | { type: "input_guardrail"; result: GuardrailResult }
    | { type: "output_guardrail"; result: GuardrailResult };
```

**Streaming Event Model**:
```typescript
export type StreamEvent =
    | { type: "turn.start"; turnId: string }
    | { type: "item.start"; turnId: string; itemId: string }
    | { type: "item.delta"; turnId: string; itemId: string; delta: ContentDelta }
    | { type: "item.completed"; turnId: string; item: ThreadItem }
    | { type: "turn.completed"; turnId: string; turn: Turn; usage: Usage }
    | { type: "turn.error"; turnId: string; error: TurnError }
    | { type: "turn.incomplete"; turnId: string; reason: string };
```

---

### Gemini CLI (TypeScript) — Level 2 技能驱动

**File**: `gemini-cli/packages/sdk/src/session.ts`

```typescript
export class GeminiCliSession {
    private readonly config: Config;
    private readonly tools: Array<Tool<any>>;
    private readonly skillRefs: SkillReference[];
    private client: GeminiClient | undefined;

    constructor(
        options: GeminiCliAgentOptions,
        private readonly sessionId: string,
        private readonly agent: GeminiCliAgent,
        private readonly resumedData?: ResumedSessionData,
    ) {
        const configParams: ConfigParameters = {
            sessionId: this.sessionId,
            targetDir: cwd,
            cwd,
            debugMode: options.debug ?? false,
            model: options.model || PREVIEW_GEMINI_MODEL_AUTO,
            userMemory: initialMemory,      // Persistent memory
            skillsSupport: true,            // Skill registry
            adminSkillsEnabled: true,
            policyEngineConfig: {
                defaultDecision: PolicyDecision.ALLOW,
            },
        };
    }

    async *sendStream(
        prompt: string,
        signal?: AbortSignal,
    ): AsyncGenerator<ServerGeminiStreamEvent> {
        // Streaming with tool call scheduling loop
        while (true) {
            const stream = client.sendMessageStream(request, abortSignal, sessionId);
            const toolCallsToSchedule: ToolCallRequestInfo[] = [];

            for await (const event of stream) {
                yield event;
                if (event.type === GeminiEventType.ToolCallRequest) {
                    toolCallsToSchedule.push(event.toolCall);
                }
            }

            if (toolCallsToSchedule.length === 0) break;

            // Execute and continue conversation
            const completed = await scheduleAgentTools(
                this.config, toolCallsToSchedule, { sessionId, toolRegistry }
            );
        }
    }
}
```

---

### Qwen Code (TypeScript) — Level 2 模块化发射器

**File**: `qwen-code/packages/cli/src/acp-integration/session/Session.ts`

```typescript
export class Session implements SessionContext {
    private pendingPrompt: AbortController | null = null;
    private turn: number = 0;

    // Modular emitters for event separation
    private readonly historyReplayer: HistoryReplayer;
    private readonly toolCallEmitter: ToolCallEmitter;
    private readonly planEmitter: PlanEmitter;
    private readonly messageEmitter: MessageEmitter;

    readonly sessionId: string;

    async prompt(params: PromptRequest): Promise<PromptResponse> {
        // Sequential handling with race prevention
        this.pendingPrompt?.abort();
        const pendingSend = new AbortController();
        this.pendingPrompt = pendingSend;

        // Wait for previous prompt for history consistency
        if (this.pendingPromptCompletion) {
            await this.pendingPromptCompletion;
        }

        // Turn-based ID generation
        this.turn += 1;
        const promptId = this.sessionId + '########' + this.turn;

        // ... prompt handling
    }
}
```

---

### Kimi CLI (TypeScript) — Level 1 文件基础

**File**: `kimi-cli/web/src/lib/api/models/Session.ts`

```typescript
export interface Session {
    sessionId: string;      // Unique ID
    title: string;          // Derived from history
    lastUpdated: Date;
    isRunning?: boolean;
    status?: SessionStatus | null;
    workDir?: string | null;      // Working directory
    sessionDir?: string | null;   // Session storage path
    archived?: boolean;           // Archival status
}

// OpenAPI-generated serialization
export function SessionFromJSON(json: any): Session {
    return {
        'sessionId': json['session_id'],
        'title': json['title'],
        'lastUpdated': new Date(json['last_updated']),
        'isRunning': json['is_running'],
        'status': json['status'],
        'workDir': json['work_dir'],
        'sessionDir': json['session_dir'],
        'archived': json['archived'],
    };
}
```

---

### OpenCode (TypeScript) — Level 2 智能缓存

**File**: `opencode/packages/app/src/context/global-sync/session-cache.ts`

```typescript
export const SESSION_CACHE_LIMIT = 40;

type SessionCache = {
    session_status: Record<string, SessionStatus | undefined>
    session_diff: Record<string, FileDiff[] | undefined>
    todo: Record<string, Todo[] | undefined>
    message: Record<string, Message[] | undefined>
    part: Record<string, Part[] | undefined>
    permission: Record<string, PermissionRequest[] | undefined>
}

export function pickSessionCacheEvictions(input: {
    seen: Set<string>
    keep: string
    limit: number
    preserve?: Iterable<string>
}) {
    // LRU eviction when cache exceeds limit
    const stale: string[] = [];
    const keep = new Set([input.keep, ...Array.from(input.preserve ?? [])]);

    for (const id of input.seen) {
        if (input.seen.size - stale.length <= input.limit) break;
        if (keep.has(id)) continue;
        stale.push(id);
    }
    return stale;
}
```

**TTL + Pinning Eviction**:
```typescript
export function pickDirectoriesToEvict(input: EvictPlan) {
    const overflow = Math.max(0, input.stores.length - input.max);

    // Sort by last access, evict oldest first
    const sorted = input.stores
        .filter((dir) => !input.pins.has(dir))
        .sort((a, b) => (input.state.get(a)?.lastAccessAt ?? 0)
                      - (input.state.get(b)?.lastAccessAt ?? 0));

    for (const dir of sorted) {
        const last = input.state.get(dir)?.lastAccessAt ?? 0;
        const idle = input.now - last >= input.ttl;
        if (!idle && overflow <= 0) continue;
        output.push(dir);
    }
    return output;
}
```

---

### Pi Mono (TypeScript) — Level 2 事件流

**File**: `pi-mono/packages/agent/src/agent-loop.ts`

```typescript
export function agentLoop(
    prompts: AgentMessage[],
    context: AgentContext,
    config: AgentLoopConfig,
    signal?: AbortSignal,
    streamFn?: StreamFn,
): EventStream<AgentEvent, AgentMessage[]> {
    const stream = createAgentStream();

    void runAgentLoop(prompts, context, config, async (event) => {
        stream.push(event);
    }, signal, streamFn).then((messages) => {
        stream.stream.end(messages);
    });

    return stream;
}

async function runLoop(
    currentContext: AgentContext,
    newMessages: AgentMessage[],
    config: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink,
): Promise<void> {
    // Outer loop for follow-up messages
    while (true) {
        let hasMoreToolCalls = true;

        // Inner loop for tool execution
        while (hasMoreToolCalls || pendingMessages.length > 0) {
            // Transform context before sending
            let messages = context.messages;
            if (config.transformContext) {
                messages = await config.transformContext(messages, signal);
            }

            // Stream assistant response
            const message = await streamAssistantResponse(
                currentContext, config, signal, emit, streamFn
            );

            // Check for tool calls
            const toolCalls = message.content.filter((c) => c.type === "toolCall");
            hasMoreToolCalls = toolCalls.length > 0;
        }
    }
}
```

---

## Browser Automation 项目

### agent-browser (Rust/TypeScript) — Level 2 加密状态

**File**: `agent-browser/cli/src/native/state.rs`

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct StorageState {
    pub cookies: Vec<Cookie>,
    pub origins: Vec<OriginStorage>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct OriginStorage {
    pub origin: String,
    pub local_storage: Vec<StorageEntry>,
    pub session_storage: Vec<StorageEntry>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Cookie {
    pub name: String,
    pub value: String,
    pub domain: String,
    pub path: String,
    pub expires: f64,
    pub size: i64,
    pub http_only: bool,
    pub secure: bool,
    pub session: bool,
    pub same_site: Option<String>,
}
```

**AES-256-GCM 加密**:
```rust
use aes_gcm::{
    aead::{Aead, KeyInit},
    Aes256Gcm, Key, Nonce,
};
use sha2::{Sha256, Digest};

pub fn encrypt_data(data: &[u8], key_str: &str) -> Result<Vec<u8>, String> {
    // SHA256 key derivation
    let mut hasher = Sha256::new();
    hasher.update(key_str.as_bytes());
    let key_bytes = hasher.finalize();

    // AES-256-GCM
    let cipher = Aes256Gcm::new_from_slice(&key_bytes)
        .map_err(|e| e.to_string())?;

    // Random 12-byte nonce
    let mut nonce = [0u8; 12];
    getrandom::getrandom(&mut nonce)
        .map_err(|e| e.to_string())?;

    let ciphertext = cipher
        .encrypt(Nonce::from_slice(&nonce), data)
        .map_err(|e| e.to_string())?;

    // nonce || ciphertext
    let mut result = Vec::with_capacity(12 + ciphertext.len());
    result.extend_from_slice(&nonce);
    result.extend_from_slice(&ciphertext);

    Ok(result)
}
```

---

### pinchtab (Go) — Level 1 快照

**File**: `pinchtab/internal/bridge/state.go`

```go
type TabState struct {
    ID    string `json:"id"`
    URL   string `json:"url"`
    Title string `json:"title"`
}

type SessionState struct {
    Tabs    []TabState `json:"tabs"`
    SavedAt string     `json:"savedAt"`
}

// Concurrent restoration with semaphores
func (b *Bridge) RestoreSession() error {
    data, err := os.ReadFile("sessions.json")
    if err != nil {
        return err
    }

    var state SessionState
    if err := json.Unmarshal(data, &state); err != nil {
        return err
    }

    // Concurrency control
    const maxConcurrentTabs = 3
    const maxConcurrentNavs = 2

    tabSem := make(chan struct{}, maxConcurrentTabs)
    navSem := make(chan struct{}, maxConcurrentNavs)

    for _, tab := range state.Tabs {
        tabSem <- struct{}{}

        go func(tabCtx context.Context, url string) {
            defer func() { <-tabSem }()

            // Create tab
            tabID, ctx, cancel, err := b.CreateTab("")
            if err != nil {
                return
            }
            defer cancel()

            // Navigate with semaphore
            navSem <- struct{}{}
            defer func() { <-navSem }()

            b.Navigate(tabID, url)
        }(ctx, tab.URL)
    }

    return nil
}
```

---

## 存储系统对比矩阵

### 后端数据库对比

| 项目 | 主存储 | 辅助存储 | ORM/抽象 | 迁移方案 |
|------|--------|---------|----------|---------|
| **IronClaw** | PostgreSQL + libSQL | ChromaDB/向量 | sqlx + 自定义trait | sqlx migrate |
| **edict** | PostgreSQL | Redis | SQLAlchemy Async | Alembic |
| **NanoClaw** | SQLite | JSON backup | better-sqlite3 | Runtime ALTER |
| **Nanobot** | JSONL + Markdown | - | Pydantic + dataclasses | 文件版本 |
| **Mimiclaw** | SPIFFS/Flash | Daily Markdown | cJSON | N/A |
| **ZeroClaw** | TOML + JSON | SQLite可选 | Serde + 自定义trait | 手动 |
| **CoPaw** | JSON + SQLite | ChromaDB | Pydantic | 版本字段 |
| **Codex** | 内存 + JSON文件 | - | TypeScript接口 | 协议版本 |
| **Gemini CLI** | Config + Memory | - | TypeScript类 | 无 |
| **Qwen Code** | ChatRecordingService | - | TypeScript接口 | 无 |
| **Kimi CLI** | File-based | - | OpenAPI生成 | 无 |
| **OpenCode** | In-memory LRU | File系统 | TypeScript | 缓存驱逐 |
| **Pi Mono** | EventStream | - | TypeScript | 无 |
| **agent-browser** | Encrypted JSON | CDP State | Serde | 无 |
| **pinchtab** | JSON Snapshot | - | Go structs | 无 |

### 存储格式对比

| 格式 | 使用项目 | 适用场景 | 优缺点 |
|------|---------|---------|--------|
| **PostgreSQL JSONB** | IronClaw, edict | 复杂查询、灵活schema | 强索引、事务，但重量级 |
| **SQLite** | NanoClaw, CoPaw | 单机、嵌入式 | 零配置、事务，但并发有限 |
| **JSONL** | Nanobot, Mimiclaw | 追加写、日志 | 简单、可grep，但查询弱 |
| **Markdown** | Nanobot, Mimiclaw | 人类可读记忆 | LLM友好，但结构化差 |
| **TOML** | ZeroClaw, IronClaw | 配置存储 | 人类可读，但不适合大数据 |
| **MessagePack** | Codex | 二进制序列化 | 紧凑高效，但不可读 |
| **AES-256-GCM** | agent-browser, IronClaw | 加密敏感数据 | 安全，但有性能开销 |

---

## 对象模型模式

### 统一对象关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                      Core Object Model                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────┐    ┌─────────────┐    ┌─────────────────────────┐│
│   │  User   │◄──►│   Identity  │    │      Session            ││
│   │         │    │             │    │  ┌─────────────────────┐││
│   └────┬────┘    └─────────────┘    │  │  ┌───────┐ ┌─────┐ │││
│        │                             │  │  │ Turn1 │ │ ... │ │││
│        ▼                             │  │  └───┬───┘ └─────┘ │││
│   ┌─────────┐    ┌─────────────┐    │  │      │             │││
│   │Workspace│◄──►│    Agent    │◄──►│◄─┘      ▼             │││
│   │         │    │             │    │   ┌──────────────┐    │││
│   │ AGENTS  │    │  SOUL.md    │    │   │    Items     │    │││
│   │ SOUL.md │    │  USER.md    │    │   │ ┌──┐┌──┐┌──┐│    │││
│   └─────────┘    └──────┬──────┘    │   │ │M ││T ││M ││    │││
│                         │           │   │ │s ││C ││s ││    │││
│        ┌────────────────┼────────┐  │   │ └──┘└──┘└──┘│    │││
│        │                │        │  │   └──────────────┘    │││
│        ▼                ▼        ▼  │                       │││
│   ┌─────────┐    ┌──────────┐ ┌────┴─┐                     │││
│   │  Tool   │    │  Memory  │ │ Skill│                     │││
│   │         │    │          │ │      │                     │││
│   │memory_* │    │MEMORY.md │ │.md   │                     │││
│   │shell_*  │    │HISTORY.md│ │      │                     │││
│   │file_*   │    └──────────┘ └──────┘                     │││
│   └─────────┘                                               │││
│                                                             │││
└─────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘

Legend:
  M = Message (user/assistant)
  T = ToolCall
  C = ToolResult
```

### 各项目对象模型对比

| 项目 | 核心抽象 | 层级结构 | 持久化粒度 |
|------|---------|---------|-----------|
| **IronClaw** | Session → Turn → Message | User → Workspace → Agent → Session | Turn级别 |
| **Codex** | Thread → Turn → Item | Thread (扁平) | Item级别 |
| **Gemini CLI** | Session → Chat | User → Session | Message级别 |
| **Qwen Code** | Session → Prompt → Turn | Session → Chat | Turn级别 |
| **NanoClaw** | Chat → Message | JID → Chat → Message | Message级别 |
| **Nanobot** | Session → Message | Workspace → Session | 自动合并 |
| **edict** | Task → Event → Thought | Org → Task → Event | Event级别 |

---

## 迁移策略对比

### 数据库迁移方案

| 策略 | 代表项目 | 实现方式 | 适用场景 |
|------|---------|---------|---------|
| **Schema Version字段** | CoPaw, Nanobot | JSON中的`version`字段 | 简单JSON文件 |
| **Runtime ALTER TABLE** | NanoClaw | `ALTER TABLE ADD COLUMN` | SQLite轻量迁移 |
| **Alembic** | edict | Python migration脚本 | SQLAlchemy项目 |
| **sqlx migrate** | IronClaw | SQL migration文件 | Rust sqlx项目 |
| **协议版本** | Codex | TypeScript接口版本 | API兼容性 |
| **无迁移** | Mimiclaw, pinchtab | 直接覆盖/重建 | 嵌入式/开发工具 |

### NanoClaw 运行时迁移示例

```typescript
// Check and apply schema migrations at runtime
function migrateSchema(db: Database) {
    // Migration 1: Add is_bot_message column
    const result = db.prepare(
        "SELECT COUNT(*) as count FROM pragma_table_info('messages') WHERE name='is_bot_message'"
    ).get() as { count: number };

    if (result.count === 0) {
        db.exec('ALTER TABLE messages ADD COLUMN is_bot_message INTEGER DEFAULT 0');
        db.exec('UPDATE messages SET is_bot_message = 0');
        console.log('Migration applied: messages.is_bot_message');
    }

    // Migration 2: Add scheduled_tasks table
    db.exec(`
        CREATE TABLE IF NOT EXISTS scheduled_tasks (
            id TEXT PRIMARY KEY,
            group_folder TEXT NOT NULL,
            chat_jid TEXT NOT NULL,
            prompt TEXT NOT NULL,
            schedule_type TEXT NOT NULL,
            schedule_value TEXT NOT NULL,
            next_run TEXT,
            last_run TEXT,
            last_result TEXT,
            status TEXT DEFAULT 'active',
            created_at TEXT NOT NULL
        )
    `);
}
```

---

## 内存管理架构

### 双层内存模式 (Nanobot)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Two-Layer Memory                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: MEMORY.md (Long-term Facts)                            │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ ## [2026-03-18T10:30:00] tool                              ││
│  │ User prefers Python over JavaScript for data processing    ││
│  │                                                            ││
│  │ ## [2026-03-18T11:00:00] user                              ││
│  │ Project deadline is March 25th                             ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                  │
│  Layer 2: HISTORY.md (Interaction Log)                          │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ [2026-03-18T10:30:00] user: Analyze the sales data...      ││
│  │ [2026-03-18T10:30:05] assistant: I'll analyze the data...  ││
│  │ [2026-03-18T10:35:00] tool: Found 3 anomalies...           ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                  │
│  Usage:                                                         │
│  - MEMORY.md: Injected into system prompt                       │
│  - HISTORY.md: grep for context retrieval                       │
└─────────────────────────────────────────────────────────────────┘
```

### Token窗口管理 (Aider)

```python
class ChatSummary:
    def __init__(self, models=None, max_tokens=1024):
        self.models = models if isinstance(models, list) else [models]
        self.max_tokens = max_tokens
        self.token_count = self.models[0].token_count

    def summarize(self, messages, depth=0):
        sized = self.tokenize(messages)
        total = sum(tokens for tokens, _msg in sized)

        if total <= self.max_tokens and depth == 0:
            return messages

        # Recursive summarization
        if len(messages) <= min_split or depth > 3:
            return self.summarize_all(messages)

        # Split and summarize halves
        mid = len(messages) // 2
        left = self.summarize(messages[:mid], depth + 1)
        right = self.summarize(messages[mid:], depth + 1)
        return left + right
```

### LRU缓存驱逐 (OpenCode)

```typescript
export function pickSessionCacheEvictions(input: {
    seen: Set<string>      // All seen session IDs
    keep: string           // Current session (always keep)
    limit: number          // Max cache size
    preserve?: Iterable<string>  // Pinned sessions
}) {
    const stale: string[] = [];
    const keepSet = new Set([input.keep, ...Array.from(input.preserve ?? [])]);

    for (const id of input.seen) {
        // Stop when under limit
        if (input.seen.size - stale.length <= input.limit) break;

        // Skip pinned
        if (keepSet.has(id)) continue;

        stale.push(id);
    }
    return stale;
}
```

### Ring Buffer (Mimiclaw)

```c
#define MAX_SESSION_MSGS 20

void session_append(const char *chat_id, const char *role, const char *content)
{
    // Check current size
    int current = session_count(chat_id);

    if (current >= MAX_SESSION_MSGS) {
        // Archive oldest to daily file
        session_archive_oldest(chat_id, 5);  // Archive 5 oldest
    }

    // Append new message
    FILE *f = fopen(session_path, "a");
    fprintf(f, "%s\n", json_line);
    fclose(f);
}
```

---

## 推荐统一存储架构

### 分层混合架构

```yaml
# storage-config.yaml
storage:
  # Layer 1: Runtime Memory (Hot)
  runtime:
    type: memory
    max_size: 100mb
    eviction: lru

  # Layer 2: Local Cache (Warm)
  local:
    type: sqlite
    path: "~/.agent/cache.db"
    encryption: aes-256-gcm

  # Layer 3: Persistent Store (Cold)
  persistent:
    type: postgresql
    url: "${DATABASE_URL}"
    pool_size: 10
    migrations: alembic

  # Layer 4: Vector Store (Semantic)
  vector:
    type: chromadb  # or pinecone, qdrant
    embedding_model: text-embedding-3-small

  # Layer 5: Archive (Frozen)
  archive:
    type: s3
    bucket: "agent-archive"
    lifecycle: 90d

# Memory Configuration
memory:
  two_layer:
    enabled: true
    facts_file: "MEMORY.md"
    history_file: "HISTORY.md"

  vector:
    enabled: true
    dimensions: 1536
    distance: cosine

  context_management:
    max_tokens: 128000
    summarization_threshold: 0.8

# Object Model
objects:
  session:
    id: uuid
    user_id: string
    workspace_id: string
    created_at: datetime
    expires_at: datetime

  turn:
    id: uuid
    session_id: reference
    sequence: integer
    status: enum[pending, active, completed, failed]

  message:
    id: uuid
    turn_id: reference
    role: enum[user, assistant, system, tool]
    content: jsonb
    tokens: integer

  memory_entry:
    id: uuid
    type: enum[fact, interaction, vector]
    content: text
    embedding: vector[1536]
    metadata: jsonb
```

### 推荐技术栈矩阵

| 场景 | 推荐方案 | 备选方案 |
|------|---------|---------|
| **单机/嵌入式** | SQLite + JSONL | LevelDB, RocksDB |
| **企业部署** | PostgreSQL + pgvector | MySQL + Pinecone |
| **Serverless** | libSQL/Turso | Supabase, PlanetScale |
| **向量搜索** | ChromaDB (本地) | Pinecone, Qdrant (云端) |
| **配置存储** | TOML/YAML | JSON, environment |
| **会话缓存** | In-memory LRU | Redis, Valkey |
| **加密敏感数据** | AES-256-GCM | age, libsodium |
| **迁移工具** | sqlx migrate (Rust) | Alembic (Python) |

### 关键设计原则

1. **分层存储**：热/温/冷三级，自动分层
2. **事件溯源**：关键操作记录不可变事件日志
3. **向量混合**：RRF融合FTS和向量搜索结果
4. **加密默认**：敏感数据始终加密存储
5. **Schema版本**：所有持久化格式带版本字段
6. **原子写入**：tmp+rename模式保证数据完整性
7. **LRU驱逐**：缓存始终有上限和驱逐策略
8. **双写安全**：关键数据同时写入本地和远程

---

## 附录：关键文件索引

| 项目 | 关键文件 | 说明 |
|------|----------|------|
| **IronClaw** | `src/db/CLAUDE.md` | 数据库双后端架构 |
| **IronClaw** | `src/workspace/README.md` | Memory系统 |
| **NanoClaw** | `src/db.ts` | SQLite schema |
| **Nanobot** | `nanobot/agent/memory.py` | 双层内存 |
| **Nanobot** | `nanobot/session/manager.py` | JSONL sessions |
| **Mimiclaw** | `main/memory/session_mgr.c` | ESP32 JSONL |
| **ZeroClaw** | `src/memory/traits.rs` | Memory trait |
| **CoPaw** | `app/runner/models.py` | Pydantic models |
| **edict** | `backend/app/models/task.py` | SQLAlchemy models |
| **edict** | `backend/app/db.py` | Async PostgreSQL |
| **Codex** | `schema/typescript/v2/Thread.ts` | Thread模型 |
| **Gemini CLI** | `sdk/src/session.ts` | Session管理 |
| **Qwen Code** | `session/Session.ts` | 模块化发射器 |
| **OpenCode** | `session-cache.ts` | LRU缓存 |
| **agent-browser** | `cli/src/native/state.rs` | 加密存储 |
| **pinchtab** | `internal/bridge/state.go` | 会话快照 |
| **Aider** | `aider/history.py` | 上下文摘要 |

---

*报告更新时间: 2026-03-18*
*分析项目: 全部15个AI Agent项目*
*文档版本: 2.0*
