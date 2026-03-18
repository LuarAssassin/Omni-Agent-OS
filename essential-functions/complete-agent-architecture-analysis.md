# AI Agent Complete Architecture Analysis

## Executive Summary

This document provides a comprehensive analysis of complete agent architectures across **15 AI agent projects**, including **8 Personal Assistants** and **7 Programming Agents**. The analysis covers agent loop patterns, tool systems, memory integration, multi-channel support, provider abstractions, state management, extensibility patterns, and error handling strategies.

### Project Categorization

| Category | Count | Projects |
|----------|-------|----------|
| **Personal Assistants** | 8 | CoPaw, IronClaw, NanoClaw, PicoClaw, ZeroClaw, MimicClaw, OpenClaw, Nanobot |
| **Programming Agents** | 7 | Aider, Codex, Qwen-Code, Gemini-CLI, Kimi-CLI, OpenCode, Pi-Mono |

---

## 1. Agent Loop Architecture

### 1.1 Loop Pattern Comparison

| Project | Loop Pattern | Concurrency Model | Key Innovation |
|---------|-------------|-------------------|----------------|
| **CoPaw** | ReAct with ToolGuard | Sequential + Guard | Safety-first tool execution |
| **IronClaw** | Generic Delegate Loop | Async Tokio | Trait-based delegation |
| **ZeroClaw** | Runtime Trait Loop | Async + Peripheral polling | 8-core trait architecture |
| **NanoClaw** | Container-per-Group | Async Node.js + Docker | Group queue isolation |
| **OpenClaw** | Gateway Server Loop | Async TypeScript | 20+ channel gateway |
| **PicoClaw** | AgentInstance Loop | Single-threaded Go | Minimal resource footprint |
| **MimicClaw** | ReAct Pattern (C) | FreeRTOS tasks | Embedded ESP32 support |
| **Nanobot** | Event-Driven Loop | Async Python | MessageBus subscribers |
| **Aider** | Template Method | Synchronous Python | Edit format strategies |
| **Codex** | SQ/EQ Protocol | Tokio + Channels | UI/Engine separation |
| **Qwen-Code** | Session Manager | Async TypeScript | MCP context engine |
| **Gemini-CLI** | Scheduler State Machine | Async + MessageBus | Policy-driven execution |
| **Kimi-CLI** | Wire Message Bus | SPMC Pattern | Two-queue architecture |
| **OpenCode** | Multi-Agent Effect | Async + Effect library | Type-safe composition |
| **Pi-Mono** | Event-Driven | Async + TUI diff | Graph-based analysis |

### 1.2 Detailed Loop Implementations

#### IronClaw - Generic Trait-Based Loop (Deep Module)

```rust
// File: ironclaw/src/agent/mod.rs
/// Generic agent loop with delegation pattern
/// Simple interface hides complex implementations
pub async fn run_agentic_loop<D: LoopDelegate>(delegate: &mut D) -> Result<()> {
    loop {
        // 1. Gather context from all sources
        let context = delegate.gather_context().await?;

        // 2. Query LLM with prepared context
        let response = delegate.query_llm(context).await?;

        // 3. Parse response into actionable items
        let actions = delegate.parse_response(response)?;

        // 4. Execute each action
        for action in actions {
            delegate.execute_action(action).await?;
        }

        // 5. Check completion conditions
        if delegate.should_terminate() {
            break;
        }
    }
    Ok(())
}

/// Delegate trait - implementors provide specific behavior
#[async_trait]
pub trait LoopDelegate: Send + Sync {
    async fn gather_context(&self) -> Result<Context>;
    async fn query_llm(&self, context: Context) -> Result<Response>;
    fn parse_response(&self, response: Response) -> Result<Vec<Action>>;
    async fn execute_action(&self, action: Action) -> Result<()>;
    fn should_terminate(&self) -> bool;
}
```

**Key Design Principles:**
- **Deep Module**: Simple 5-step interface, complex implementations hidden
- **Delegation Pattern**: All operations delegated to `LoopDelegate` trait
- **Extensibility**: New agent types implement trait, no loop modification needed

#### Kimi-CLI - Wire Message Bus Pattern (SPMC)

```python
# File: kimi-cli/src/kimi_soul/soul.py
class KimiSoul:
    """
    SPMC (Single Producer Multi Consumer) message bus
    Two-queue architecture for message processing
    """

    def run_turn(self, wire: Wire, turn_input: TurnInput) -> None:
        """
        Execute one turn of the agent loop.
        Wire provides the message bus abstraction.
        """
        turn = Turn.create(turn_input)

        while not turn.completed:
            # Prepare context from wire (merged queue)
            context = self._prepare_step_context(wire, turn)

            # Generate response using kosong library
            for event in kosong.generate(context, tools=self.tools):
                if event.type == "tool_call":
                    result = self._execute_tool(event.tool)
                    wire.send_tool_result(result)
                elif event.type == "message":
                    wire.send_message(event.content)

            # Check for completion
            if turn.step_count >= self.max_steps:
                turn.complete()

class Wire:
    """
    Two-queue message bus:
    - Raw Queue: Incoming messages (single producer)
    - Merged Queue: Processed messages (multi-consumer)
    """
    def __init__(self):
        self.raw_queue: Queue[Message] = Queue()
        self.merged_queue: Queue[Message] = Queue()

    def send(self, message: Message) -> None:
        """Single producer writes to raw queue"""
        self.raw_queue.put(message)
        self._merge()

    def receive(self) -> Optional[Message]:
        """Multi-consumer reads from merged queue"""
        return self.merged_queue.get_non_blocking()
```

#### Codex - SQ/EQ Protocol (Submission/Event Queue)

```rust
// File: codex/codex-rs/protocol/src/protocol.rs
/// SQ (Submission Queue): UI → Codex requests
/// EQ (Event Queue): Codex → UI events
/// Enables complete UI/Engine separation

#[derive(Debug, Clone, Serialize, Deserialize)]
#[non_exhaustive]
pub enum Op {
    /// Interrupt current operation
    Interrupt,

    /// Start a new user turn
    UserTurn {
        messages: Vec<Message>,
        #[serde(default)]
        prompt: Option<String>,
    },

    /// Approve/deny tool execution
    ExecApproval {
        approved: bool,
        command: String,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[non_exhaustive]
pub enum EventMsg {
    /// Agent message start
    AgentMessage { id: String },

    /// Content delta (streaming)
    AgentMessageContentDelta { id: String, delta: String },

    /// Tool approval request
    ExecApprovalRequest { command: String },

    /// State transitions
    TurnStarted { turn_id: String },
    TurnComplete { turn_id: String },

    /// Errors
    Error { message: String },
    Warning { message: String },
}

// State machine for task lifecycle
pub enum TaskState {
    Submitted,
    Working,
    Completed,
    Cancelled,
    Failed,
}
```

#### Gemini-CLI - Scheduler State Machine

```typescript
// File: gemini-cli/packages/core/src/scheduler/scheduler.ts
export class Scheduler {
    /**
     * State machine for tool execution:
     * Validating → Scheduled → Executing → Success
     *                         ↘ Error
     *                    ↘ Cancelled → AwaitingApproval
     */
    async schedule(
        requests: ToolCallRequestInfo[],
        signal: AbortSignal
    ): Promise<CompletedToolCall[]> {
        const results: CompletedToolCall[] = [];

        for (const request of requests) {
            // Check policy before execution
            const policy = this.policyEngine.check(request);

            switch (policy.action) {
                case PolicyAction.ALLOW:
                    results.push(await this.execute(request, signal));
                    break;

                case PolicyAction.DENY:
                    results.push(this.create_denied_result(request));
                    break;

                case PolicyAction.ASK_USER:
                    const approved = await this.request_approval(request);
                    if (approved) {
                        results.push(await this.execute(request, signal));
                    } else {
                        results.push(this.create_cancelled_result(request));
                    }
                    break;
            }
        }

        return results;
    }
}
```

### 1.3 Loop Architecture Comparison Matrix

| Feature | IronClaw | Kimi-CLI | Codex | Gemini-CLI | Aider |
|---------|----------|----------|-------|------------|-------|
| **Pattern** | Trait Delegation | SPMC Bus | SQ/EQ Protocol | State Machine | Template Method |
| **Concurrency** | Async Tokio | Async Python | Tokio Channels | Async TS | Sync Python |
| **Extensibility** | High | Medium | High | Medium | Medium |
| **UI Separation** | No | Partial | Full | Partial | No |
| **Policy Integration** | Via Delegate | No | Guardian Layer | PolicyEngine | No |
| **Streaming Support** | Yes | Yes | Yes | Yes | Limited |

---

## 2. Tool System Architecture

### 2.1 Tool Registration Patterns

| Project | Registration | Discovery | Key Innovation |
|---------|-------------|-----------|----------------|
| **CoPaw** | AgentScope decorators | Static + Dynamic | ToolGuard safety layer |
| **IronClaw** | Trait objects | Runtime | 3-tier sandbox |
| **ZeroClaw** | Factory pattern | Compile-time | Trait-based tools |
| **NanoClaw** | MCP protocol | Server-based | Container tool server |
| **PicoClaw** | Static registration | Build-time | TTL-based caching |
| **MimicClaw** | Static array | Compile-time | C struct registry |
| **Aider** | Class-based | Import-time | Strategy pattern |
| **Codex** | Dynamic registry | MCP + Built-in | 72-crate workspace |
| **Gemini-CLI** | Three-tier | Priority-based | Built-in/Discovered/MCP |
| **Qwen-Code** | 50+ built-in | Registration | Extensive tool set |
| **Kimi-CLI** | Skill-based | Markdown | SKILL.md discovery |
| **OpenCode** | Ruleset-based | Configuration | Permission integration |
| **Pi-Mono** | Tool handlers | Registration | Graph-aware tools |

### 2.2 Detailed Tool System Implementations

#### Gemini-CLI - Three-Tier Tool Registry

```typescript
// File: gemini-cli/packages/core/src/tools/tool-registry.ts
export class ToolRegistry {
    /**
     * Three-tier priority system:
     * 1. Built-in tools (highest priority)
     * 2. Discovered tools (user-defined)
     * 3. MCP tools (external protocol)
     */

    private builtInTools: Map<string, Tool> = new Map();
    private discoveredTools: Map<string, Tool> = new Map();
    private mcpTools: Map<string, Tool> = new Map();

    registerBuiltIn(tool: Tool): void {
        this.builtInTools.set(tool.name, tool);
    }

    registerDiscovered(tool: Tool): void {
        this.discoveredTools.set(tool.name, tool);
    }

    registerMcp(tool: Tool): void {
        this.mcpTools.set(tool.name, tool);
    }

    getTool(name: string): Tool | undefined {
        // Priority order: Built-in > Discovered > MCP
        return this.builtInTools.get(name)
            ?? this.discoveredTools.get(name)
            ?? this.mcpTools.get(name);
    }

    getAllTools(): Tool[] {
        // Return merged list with priority ordering
        return [
            ...this.builtInTools.values(),
            ...this.discoveredTools.values(),
            ...this.mcpTools.values(),
        ];
    }
}

// Built-in tools include: read-file, edit-file, shell, git, web-search, etc.
const BUILT_IN_TOOLS: Tool[] = [
    {
        name: "read-file",
        description: "Read contents of a file",
        parameters: {
            type: "object",
            properties: {
                path: { type: "string", description: "File path" },
                offset: { type: "number", description: "Start line (optional)" },
                limit: { type: "number", description: "Max lines (optional)" },
            },
            required: ["path"],
        },
        execute: async (params) => {
            // Implementation
        },
    },
    // ... more tools
];
```

#### MimicClaw - C-Based Tool Registry (Embedded)

```c
// File: mimiclaw/tools/tool_registry.c
/**
 * Static tool registry for ESP32 embedded systems
 * Minimal overhead, compile-time registration
 */

typedef esp_err_t (*tool_execute_func_t)(
    const char *input,
    char *output,
    size_t output_size
);

typedef struct {
    const char *name;
    const char *description;
    const char *input_schema;
    tool_execute_func_t execute;
} tool_entry_t;

// Static array of 10 tools (compile-time defined)
static tool_entry_t s_tools[] = {
    {
        .name = "gpio_write",
        .description = "Control GPIO pins on ESP32",
        .input_schema = GPIO_WRITE_SCHEMA,
        .execute = gpio_write_execute,
    },
    {
        .name = "web_search",
        .description = "Search the web",
        .input_schema = WEB_SEARCH_SCHEMA,
        .execute = web_search_execute,
    },
    {
        .name = "sensor_read",
        .description = "Read sensor values",
        .input_schema = SENSOR_READ_SCHEMA,
        .execute = sensor_read_execute,
    },
    // ... 7 more tools
};

// Returns JSON schema for all tools (for LLM context)
const char *tool_registry_get_tools_json(void) {
    static char buffer[2048];
    // Serialize s_tools array to JSON
    return buffer;
}

// Execute tool by name
esp_err_t tool_registry_execute(
    const char *name,
    const char *input,
    char *output,
    size_t output_size
) {
    for (size_t i = 0; i < sizeof(s_tools) / sizeof(s_tools[0]); i++) {
        if (strcmp(s_tools[i].name, name) == 0) {
            return s_tools[i].execute(input, output, output_size);
        }
    }
    return ESP_ERR_NOT_FOUND;
}
```

#### Kimi-CLI - Skill-Based Tool System

```python
# File: kimi-cli/src/skills/skill_loader.py
@dataclass
class Skill:
    """
    Skills are Markdown files with hierarchical discovery.
    Each skill defines tools, agent types, and capabilities.
    """
    name: str
    description: str
    agent_type: AgentType
    tools: List[Tool]
    prompt_template: str

class SkillLoader:
    """Load skills from directory tree with frontmatter parsing"""

    def discover(self, root_path: Path) -> List[Skill]:
        skills = []
        for md_file in root_path.rglob("*.md"):
            skill = self._parse_skill(md_file)
            if skill:
                skills.append(skill)
        return skills

    def _parse_skill(self, path: Path) -> Optional[Skill]:
        content = path.read_text()

        # Parse frontmatter
        frontmatter, body = self._split_frontmatter(content)

        # Extract metadata
        name = frontmatter.get("name", path.stem)
        agent_type = AgentType(frontmatter.get("agent", "general"))

        # Extract tool definitions from markdown
        tools = self._extract_tools(body)

        return Skill(
            name=name,
            description=frontmatter.get("description", ""),
            agent_type=agent_type,
            tools=tools,
            prompt_template=body,
        )

    def _extract_tools(self, markdown: str) -> List[Tool]:
        """Extract tool definitions from markdown code blocks"""
        tools = []
        # Parse ```tool schema blocks
        return tools
```

#### Aider - Strategy Pattern for Edit Formats

```python
# File: aider/coders/base_coder.py
class Coder:
    """
    Template Method pattern for code editing.
    Subclasses implement specific edit formats.
    """

    def run(self, with_message=None):
        """Template method - defines the algorithm skeleton"""
        self.before_run()
        self.send_message(with_message)
        self.after_run()

    def send_message(self, message):
        """Main loop - calls get_edits() for format-specific behavior"""
        # Get edits from subclass
        edits = self.get_edits()

        # Apply edits
        for edit in edits:
            self.apply_edit(edit)

    def get_edits(self) -> List[Edit]:
        """Abstract method - subclasses implement"""
        raise NotImplementedError

    def apply_edit(self, edit: Edit):
        """Abstract method - subclasses implement"""
        raise NotImplementedError


class EditBlockCoder(Coder):
    """SEARCH/REPLACE block format"""

    def get_edits(self) -> List[Edit]:
        # Parse SEARCH/REPLACE blocks from LLM response
        return self.parse_edit_blocks()

    def apply_edit(self, edit: Edit):
        # Find SEARCH text, replace with REPLACE text
        self.fuzzy_replace(edit.search, edit.replace)


class WholeFileCoder(Coder):
    """Full file replacement format"""

    def get_edits(self) -> List[Edit]:
        # Parse complete file contents
        return self.parse_whole_files()

    def apply_edit(self, edit: Edit):
        # Write entire file
        self.write_file(edit.path, edit.content)


class UnifiedDiffCoder(Coder):
    """Unified diff format"""

    def get_edits(self) -> List[Edit]:
        # Parse unified diff
        return self.parse_udiff()

    def apply_edit(self, edit: Edit):
        # Apply diff patches
        self.apply_patch(edit.patch)
```

### 2.3 Tool Execution Patterns

| Pattern | Projects | Description |
|---------|----------|-------------|
| **Pre-execution validation** | CoPaw, IronClaw | ToolGuard, policy checks before execution |
| **Sandboxed execution** | IronClaw, ZeroClaw, Codex | WASM, Docker, Landlock isolation |
| **Approval workflow** | Gemini-CLI, Codex | PolicyEngine, ASK_USER state |
| **Streaming results** | Qwen-Code, Gemini-CLI | Real-time output streaming |
| **Rate limiting** | IronClaw, PicoClaw | Sliding window, TTL-based |

---

## 3. Memory Integration

### 3.1 Memory Architecture Patterns

| Project | Short-Term | Long-Term | Context Management |
|---------|-----------|-----------|-------------------|
| **CoPaw** | AgentScope Memory | Persistent storage | Auto-summarization |
| **IronClaw** | ConversationStore | PostgreSQL + Vector | Sliding window |
| **ZeroClaw** | Runtime state | Persistent | Configurable limit |
| **Nanobot** | HISTORY.md | MEMORY.md | Two-layer merge |
| **Aider** | Chat history | RepoMap cache | Token counting |
| **Codex** | Session state | SQLite | Phase1/Phase2 |
| **Gemini-CLI** | Message history | ContextManager | Compression |
| **Kimi-CLI** | Wire context | Checkpoints | Summarization |
| **OpenCode** | Conversation | Vector store | Context rules |
| **Pi-Mono** | Session | Filesystem | LRU cache |

### 3.2 Detailed Memory Implementations

#### Nanobot - Two-Layer Memory System

```python
# File: nanobot/memory/manager.py
class MemoryManager:
    """
    Two-layer memory architecture:
    - Layer 1 (HISTORY.md): Raw conversation logs
    - Layer 2 (MEMORY.md): Consolidated facts/preferences

    Inspired by human memory formation:
    - Immediate: Write to HISTORY
    - Periodic: Summarize to MEMORY
    """

    def __init__(self, workspace: Path):
        self.history_path = workspace / "HISTORY.md"
        self.memory_path = workspace / "MEMORY.md"
        self.llm = OpenAIClient()

    async def add_message(self, message: Message) -> None:
        """Immediate: Append to HISTORY.md"""
        entry = self._format_history_entry(message)
        async with aiofiles.open(self.history_path, "a") as f:
            await f.write(entry)

    async def consolidate(self) -> None:
        """
        Periodic: Summarize HISTORY into MEMORY.
        Called every N messages or on session end.
        """
        # Read recent history
        history = await self._read_recent_history(limit=100)

        # Extract facts using LLM
        prompt = f"""
        Extract key facts and preferences from this conversation:
        {history}

        Format as:
        - [Fact] User prefers...
        - [Preference] Language is...
        - [Knowledge] Project uses...
        """

        facts = await self.llm.extract_facts(prompt)

        # Read existing memory
        existing_memory = await self._read_memory()

        # Merge facts (deduplicate, update)
        merged = self._merge_facts(existing_memory, facts)

        # Write back
        await self._write_memory(merged)

    async def get_context(self, query: str, max_tokens: int = 2000) -> str:
        """
        Retrieve relevant context for query.
        Searches both HISTORY and MEMORY.
        """
        # Search MEMORY first (consolidated facts)
        memory_results = await self._search_memory(query)

        # Fill remaining budget with recent HISTORY
        remaining_tokens = max_tokens - self._estimate_tokens(memory_results)
        history_results = await self._search_recent_history(query, remaining_tokens)

        return self._format_context(memory_results, history_results)
```

#### Codex - Phase-Based Memory System

```rust
// File: codex/codex-rs/core/src/memories/mod.rs
/**
 * Phase-based memory extraction and consolidation
 *
 * Phase 1: Real-time memory extraction during conversation
 * Phase 2: Background consolidation (dedupe, cluster, summarize)
 */

pub struct MemoryManager {
    phase1: Phase1Extractor,
    phase2: Phase2Consolidator,
    storage: Arc<dyn MemoryStorage>,
}

impl MemoryManager {
    /// Phase 1: Extract memories in real-time during turn
    pub async fn extract(&self, turn: &Turn) -> Vec<Memory> {
        let mut memories = Vec::new();

        // Extract user preferences
        if let Some(pref) = self.phase1.extract_preference(&turn.messages) {
            memories.push(pref);
        }

        // Extract facts
        memories.extend(self.phase1.extract_facts(&turn.messages));

        // Extract code patterns
        memories.extend(self.phase1.extract_patterns(&turn.messages));

        // Store immediately
        for memory in &memories {
            self.storage.store(memory).await.ok();
        }

        memories
    }

    /// Phase 2: Background consolidation
    pub async fn consolidate(&self) -> Result<()> {
        // 1. Fetch unconsolidated memories
        let memories = self.storage.fetch_unconsolidated().await?;

        // 2. Deduplicate
        let deduped = self.phase2.deduplicate(memories);

        // 3. Cluster similar memories
        let clusters = self.phase2.cluster(deduped);

        // 4. Summarize clusters
        let summaries = self.phase2.summarize(clusters).await?;

        // 5. Store consolidated memories
        for summary in summaries {
            self.storage.store_consolidated(summary).await?;
        }

        Ok(())
    }
}

pub struct Memory {
    pub id: String,
    pub content: String,
    pub memory_type: MemoryType,
    pub confidence: f32,
    pub created_at: DateTime<Utc>,
    pub consolidated: bool,
}

pub enum MemoryType {
    Preference,
    Fact,
    CodePattern,
    Relationship,
}
```

#### Aider - RepoMap with Caching

```python
# File: aider/repomap.py
class RepoMap:
    """
    Tree-sitter based codebase context extraction.
    Uses SQLite for caching and binary search for token optimization.
    """

    def __init__(self, root: str, max_map_tokens: int = 1024):
        self.root = Path(root)
        self.max_map_tokens = max_map_tokens
        self.cache = RepoMapCache()

    def get_ranked_tags_map(
        self,
        chat_files: List[str],
        other_files: List[str],
        max_map_tokens: Optional[int] = None
    ) -> str:
        """
        Generate a ranked map of repository structure.

        1. Extract tags from all files using Tree-sitter
        2. Rank tags by reference relationships
        3. Select tags within token budget using binary search
        """
        max_tokens = max_map_tokens or self.max_map_tokens

        # Extract tags from all files
        all_tags = []
        for file_path in chat_files + other_files:
            # Check cache first
            cached = self.cache.get(file_path)
            if cached:
                all_tags.extend(cached)
            else:
                tags = self._extract_tags(file_path)
                self.cache.set(file_path, tags)
                all_tags.extend(tags)

        # Rank tags by importance
        # - Definitions in chat_files rank higher
        # - Referenced definitions rank higher
        ranked = self._rank_tags(all_tags, chat_files)

        # Binary search for optimal token count
        return self._select_tags_binary_search(ranked, max_tokens)

    def _extract_tags(self, file_path: str) -> List[Tag]:
        """Extract code structure using Tree-sitter"""
        content = Path(file_path).read_text()
        language = self._detect_language(file_path)

        parser = tree_sitter.Parser()
        parser.set_language(language)
        tree = parser.parse(content)

        tags = []
        root = tree.root_node

        # Walk tree to find definitions
        for node in self._walk_tree(root):
            if self._is_definition(node):
                tags.append(Tag(
                    name=self._get_name(node),
                    kind=self._get_kind(node),
                    line=node.start_point[0],
                    file=file_path,
                ))

        return tags

    def _select_tags_binary_search(
        self,
        tags: List[Tag],
        max_tokens: int
    ) -> str:
        """
        Use binary search to find optimal number of tags
        that fit within token budget.
        """
        lo, hi = 0, len(tags)
        result = ""

        while lo < hi:
            mid = (lo + hi + 1) // 2
            selected = tags[:mid]
            text = self._format_tags(selected)
            tokens = self._count_tokens(text)

            if tokens <= max_tokens:
                result = text
                lo = mid
            else:
                hi = mid - 1

        return result
```

#### Gemini-CLI - Context Compression

```typescript
// File: gemini-cli/packages/core/src/core/client.ts
/**
 * Reverse Token Budget - prioritize recent messages
 * Allocates token budget by message importance/recency.
 */

const TOKEN_BUDGET = {
    systemInstruction: 0.10,  // 10% for system prompt
    recentMessages: 0.50,     // 50% for recent conversation
    olderMessages: 0.30,      // 30% for older messages (compressed)
    functionResponses: 0.10,  // 10% for tool results
};

export class ContextManager {
    private messages: Message[] = [];
    private maxTokens: number;

    constructor(maxTokens: number = 128000) {
        this.maxTokens = maxTokens;
    }

    addMessage(message: Message): void {
        this.messages.push(message);
        this._compress_if_needed();
    }

    private _compress_if_needed(): void {
        const totalTokens = this._estimateTokens(this.messages);

        if (totalTokens <= this.maxTokens) {
            return;
        }

        // Calculate budget allocations
        const budgets = {
            system: this.maxTokens * TOKEN_BUDGET.systemInstruction,
            recent: this.maxTokens * TOKEN_BUDGET.recentMessages,
            older: this.maxTokens * TOKEN_BUDGET.olderMessages,
            functions: this.maxTokens * TOKEN_BUDGET.functionResponses,
        };

        // Split messages into recent and older
        const recentCutoff = Math.max(0, this.messages.length - 20);
        const recent = this.messages.slice(recentCutoff);
        const older = this.messages.slice(0, recentCutoff);

        // Compress older messages
        const compressedOlder = this._summarize_messages(older, budgets.older);

        // Rebuild message list
        this.messages = [
            ...compressedOlder,
            ...recent,
        ];
    }

    private _summarize_messages(
        messages: Message[],
        budget: number
    ): Message[] {
        // Use LLM to summarize batches of messages
        // Return summary messages that fit within budget
        return summarized;
    }
}
```

---

## 4. Multi-Channel Support

### 4.1 Channel Architecture Comparison

| Project | Channels | Abstraction Pattern | Key Files |
|---------|----------|---------------------|-----------|
| **CoPaw** | 15+ | AgentScope adapters | `channels/` |
| **IronClaw** | 10+ | Unified Channel trait | `channel/mod.rs` |
| **ZeroClaw** | Configurable | `Channel` trait | `channel.rs` |
| **NanoClaw** | 15+ | Self-registration | `channel/registry.ts` |
| **OpenClaw** | 20+ | Gateway adapter | `adapters/` |
| **PicoClaw** | 8+ | Platform structs | `channel/*.go` |
| **MimicClaw** | 2 | Separate implementations | `channels/` |
| **Nanobot** | Multiple | MessageBus | `channels/` |

### 4.2 Detailed Channel Implementations

#### IronClaw - Trait-Based Channel Abstraction

```rust
// File: ironclaw/src/channel/mod.rs
/**
 * Unified Channel trait for all messaging platforms.
 * Deep module: Simple interface, complex implementations hidden.
 */

#[async_trait]
pub trait Channel: Send + Sync {
    /// Send a message to a contact
    async fn send_message(
        &self,
        to: &Contact,
        content: &Content
    ) -> Result<MessageId>;

    /// Receive messages (polling or webhook)
    async fn receive_messages(&self) -> Result<Vec<IncomingMessage>>;

    /// Get list of contacts
    async fn get_contacts(&self) -> Result<Vec<Contact>>;

    /// Get channel capabilities
    fn capabilities(&self) -> ChannelCapabilities;
}

pub struct ChannelCapabilities {
    pub supports_media: bool,
    pub supports_threads: bool,
    pub max_message_length: Option<usize>,
    pub rate_limit: Option<RateLimit>,
}

// Telegram implementation
pub struct TelegramChannel {
    bot_token: String,
    client: reqwest::Client,
}

#[async_trait]
impl Channel for TelegramChannel {
    async fn send_message(
        &self,
        to: &Contact,
        content: &Content
    ) -> Result<MessageId> {
        let url = format!("https://api.telegram.org/bot{}/sendMessage", self.bot_token);

        let response = self.client
            .post(&url)
            .json(&json!({
                "chat_id": to.id,
                "text": content.text,
                "parse_mode": "MarkdownV2",
            }))
            .send()
            .await?;

        let result: TelegramResponse = response.json().await?;
        Ok(MessageId(result.message_id.to_string()))
    }

    async fn receive_messages(&self) -> Result<Vec<IncomingMessage>> {
        // Use long polling or webhook
        self.poll_updates().await
    }

    fn capabilities(&self) -> ChannelCapabilities {
        ChannelCapabilities {
            supports_media: true,
            supports_threads: true,
            max_message_length: Some(4096),
            rate_limit: Some(RateLimit::per_second(30)),
        }
    }
}

// Discord implementation
pub struct DiscordChannel {
    token: String,
    http: Http,
}

#[async_trait]
impl Channel for DiscordChannel {
    async fn send_message(
        &self,
        to: &Contact,
        content: &Content
    ) -> Result<MessageId> {
        // Discord-specific implementation
    }

    // ... other implementations
}

// Channel manager for multi-channel operation
pub struct ChannelManager {
    channels: HashMap<String, Box<dyn Channel>>,
}

impl ChannelManager {
    pub async fn broadcast(
        &self,
        content: &Content
    ) -> Vec<(String, Result<MessageId>)> {
        let mut results = Vec::new();

        for (name, channel) in &self.channels {
            let result = channel.send_message(&content).await;
            results.push((name.clone(), result));
        }

        results
    }
}
```

#### NanoClaw - Self-Registration Pattern

```typescript
// File: nanoclaw/src/channel/registry.ts
/**
 * Self-registration pattern for channels.
 * Each channel type registers itself on import.
 */

// Channel factory interface
interface ChannelFactory {
    create(config: ChannelConfig): Channel;
    validateConfig(config: ChannelConfig): boolean;
}

// Global registry
class ChannelRegistry {
    private factories = new Map<string, ChannelFactory>();

    /**
     * Register a channel factory.
     * Called by each channel module on import.
     */
    registerChannel(name: string, factory: ChannelFactory): void {
        this.factories.set(name, factory);
        console.log(`Registered channel: ${name}`);
    }

    /**
     * Create channel instance from configuration.
     */
    createChannel(config: ChannelConfig): Channel {
        const factory = this.factories.get(config.type);
        if (!factory) {
            throw new Error(`Unknown channel type: ${config.type}`);
        }

        if (!factory.validateConfig(config)) {
            throw new Error(`Invalid configuration for ${config.type}`);
        }

        return factory.create(config);
    }

    getRegisteredTypes(): string[] {
        return Array.from(this.factories.keys());
    }
}

// Global singleton
export const channelRegistry = new ChannelRegistry();

// WhatsApp channel self-registration
// File: nanoclaw/src/channel/whatsapp.ts
import { channelRegistry } from './registry';

class WhatsAppChannelFactory implements ChannelFactory {
    create(config: ChannelConfig): Channel {
        return new WhatsAppChannel(config);
    }

    validateConfig(config: ChannelConfig): boolean {
        return !!(config.credentials?.phoneNumber && config.credentials?.apiKey);
    }
}

// Self-register on module load
channelRegistry.registerChannel('whatsapp', new WhatsAppChannelFactory());

// Telegram channel self-registration
// File: nanoclaw/src/channel/telegram.ts
class TelegramChannelFactory implements ChannelFactory {
    create(config: ChannelConfig): Channel {
        return new TelegramChannel(config);
    }

    validateConfig(config: ChannelConfig): boolean {
        return !!config.credentials?.botToken;
    }
}

channelRegistry.registerChannel('telegram', new TelegramChannelFactory());
```

#### OpenClaw - Gateway Adapter Pattern

```typescript
// File: openclaw/src/adapters/base.ts
/**
 * Gateway adapter pattern with 20+ platform support.
 * Deep module: startGatewayServer() is 1065 lines but exposes simple interface.
 */

// Base adapter interface
interface PlatformAdapter {
    readonly name: string;

    initialize(config: AdapterConfig): Promise<void>;
    send(message: OutgoingMessage): Promise<void>;
    onMessage(handler: (msg: IncomingMessage) => void): void;
    onError(handler: (error: Error) => void): void;
    dispose(): Promise<void>;
}

// Message types
interface IncomingMessage {
    id: string;
    platform: string;
    channel: string;
    author: Author;
    content: string;
    attachments: Attachment[];
    timestamp: Date;
    replyTo?: string;
}

interface OutgoingMessage {
    channel: string;
    content: string;
    attachments?: Attachment[];
    replyTo?: string;
}

// Discord adapter
class DiscordAdapter implements PlatformAdapter {
    name = 'discord';
    private client: Client;
    private messageHandler?: (msg: IncomingMessage) => void;

    async initialize(config: AdapterConfig): Promise<void> {
        this.client = new Client({
            intents: [
                GatewayIntentBits.Guilds,
                GatewayIntentBits.GuildMessages,
                GatewayIntentBits.MessageContent,
            ],
        });

        this.client.on('messageCreate', (msg) => {
            if (this.messageHandler && !msg.author.bot) {
                this.messageHandler(this.toIncomingMessage(msg));
            }
        });

        await this.client.login(config.token);
    }

    async send(message: OutgoingMessage): Promise<void> {
        const channel = await this.client.channels.fetch(message.channel);
        if (channel?.isTextBased()) {
            await channel.send({
                content: message.content,
                reply: message.replyTo ? { messageReference: message.replyTo } : undefined,
            });
        }
    }

    onMessage(handler: (msg: IncomingMessage) => void): void {
        this.messageHandler = handler;
    }

    private toIncomingMessage(msg: Message): IncomingMessage {
        return {
            id: msg.id,
            platform: 'discord',
            channel: msg.channelId,
            author: {
                id: msg.author.id,
                name: msg.author.username,
                isBot: msg.author.bot,
            },
            content: msg.content,
            attachments: msg.attachments.map(a => ({
                name: a.name,
                url: a.url,
                size: a.size,
            })),
            timestamp: msg.createdAt,
            replyTo: msg.reference?.messageId,
        };
    }

    // ... other implementations
}

// Unified gateway server
export async function startGatewayServer(
    config: GatewayConfig
): Promise<GatewayServer> {
    /**
     * Deep module: ~1065 lines
     * - Adapter lifecycle management
     * - Message routing
     * - Rate limiting
     * - Error handling
     * - WebSocket management
     */

    const adapters = new Map<string, PlatformAdapter>();
    const messageBus = new MessageBus();

    // Initialize all configured adapters
    for (const [name, adapterConfig] of Object.entries(config.adapters)) {
        const adapter = createAdapter(adapterConfig.type);
        await adapter.initialize(adapterConfig);

        adapter.onMessage((msg) => {
            messageBus.publish(`platform.${msg.platform}`, msg);
        });

        adapters.set(name, adapter);
    }

    return {
        send: async (platform: string, message: OutgoingMessage) => {
            const adapter = adapters.get(platform);
            if (adapter) {
                await adapter.send(message);
            }
        },
        onMessage: (handler: (msg: IncomingMessage) => void) => {
            messageBus.subscribe('platform.*', handler);
        },
        dispose: async () => {
            for (const adapter of adapters.values()) {
                await adapter.dispose();
            }
        },
    };
}
```

#### MimicClaw - C Implementation (Shallow vs Deep)

```c
// File: mimiclaw/channels/telegram/telegram_bot.c
/**
 * SHALLOW MODULE - exposes WebSocket details
 * This is an anti-pattern per "A Philosophy of Software Design"
 */

// WebSocket event handler exposed at module level
static void websocket_event_handler(
    void *handler_args,
    esp_event_base_t base,
    int32_t event_id,
    void *event_data
) {
    esp_websocket_event_data_t *data = event_data;

    switch (event_id) {
        case WEBSOCKET_EVENT_CONNECTED:
            ESP_LOGI(TAG, "WebSocket connected");
            break;

        case WEBSOCKET_EVENT_DATA:
            // Parse Telegram update
            cJSON *update = cJSON_Parse(data->data_ptr);
            process_telegram_update(update);
            cJSON_Delete(update);
            break;

        case WEBSOCKET_EVENT_DISCONNECTED:
            ESP_LOGI(TAG, "WebSocket disconnected, retrying...");
            // Reconnection logic exposed here
            vTaskDelay(pdMS_TO_TICKS(5000));
            esp_websocket_client_start(client);
            break;
    }
}

void telegram_start(void) {
    // Creates FreeRTOS tasks directly
    xTaskCreate(websocket_task, "telegram_ws", 4096, NULL, 5, NULL);
    xTaskCreate(polling_task, "telegram_poll", 4096, NULL, 5, NULL);
}

// File: mimiclaw/channels/feishu/feishu_bot.c
/**
 * SIMILAR STRUCTURE - Code duplication
 * Violates DRY principle
 */

// RECOMMENDATION: Create unified transport abstraction
// File: mimiclaw/channels/transport.h

typedef struct {
    const char *name;
    esp_err_t (*connect)(const char *url);
    esp_err_t (*send)(const char *data);
    esp_err_t (*receive)(char *buffer, size_t size);
    esp_err_t (*disconnect)(void);
} transport_interface_t;

// Both Telegram and Feishu would use this interface
```

---

## 5. Provider/Model Abstraction

### 5.1 LLM Provider Abstractions

| Project | Abstraction | Providers | Key Innovation |
|---------|-------------|-----------|----------------|
| **CoPaw** | ProviderManager | 15+ | Thread-safe singleton |
| **IronClaw** | Provider trait | Multiple | Capability-based routing |
| **ZeroClaw** | Provider trait | Configurable | Trait objects |
| **PicoClaw** | Model routing | 5+ | Fallback chains |
| **Aider** | LiteLLM | 100+ | Universal API |
| **Codex** | Connectors | Multiple | OpenAI Responses API |
| **Gemini-CLI** | Gemini Client | Gemini family | Native API |
| **Kimi-CLI** | kosong library | Moonshot + | Streaming support |
| **OpenCode** | ModelProvider | Multiple | Effect-based |
| **Pi-Mono** | Model registry | Multiple | Cost optimization |

### 5.2 Detailed Provider Implementations

#### CoPaw - ProviderManager (Singleton + Thread Safety)

```python
# File: copaw/copaw/core/provider/manager.py
import threading
from typing import Dict, Optional

class ProviderManager:
    """
    Singleton pattern with double-checked locking.
    Provides thread-safe global access to configured providers.
    """

    _instance: Optional['ProviderManager'] = None
    _lock = threading.Lock()

    def __new__(cls) -> 'ProviderManager':
        if cls._instance is None:
            with cls._lock:
                # Double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._initialized = False
        return cls._instance

    def __init__(self):
        if self._initialized:
            return

        self._providers: Dict[str, Provider] = {}
        self._default_provider: Optional[str] = None
        self._initialized = True

    def register_provider(self, name: str, provider: Provider) -> None:
        """Register a new provider"""
        with self._lock:
            self._providers[name] = provider

            if self._default_provider is None:
                self._default_provider = name

    def get_provider(self, name: Optional[str] = None) -> Provider:
        """
        Get provider by name or default.
        Thread-safe read access.
        """
        with self._lock:
            provider_name = name or self._default_provider
            if provider_name not in self._providers:
                raise ProviderNotFoundError(f"Provider '{provider_name}' not found")
            return self._providers[provider_name]

    def route_request(self, request: Request) -> Provider:
        """
        Intelligent routing based on:
        - Model capabilities
        - Cost constraints
        - Availability
        """
        requirements = self._extract_requirements(request)

        candidates = []
        for name, provider in self._providers.items():
            score = self._score_provider(provider, requirements)
            if score > 0:
                candidates.append((score, provider))

        if not candidates:
            raise NoSuitableProviderError("No provider matches requirements")

        # Return highest-scoring provider
        candidates.sort(key=lambda x: x[0], reverse=True)
        return candidates[0][1]

    def _extract_requirements(self, request: Request) -> Requirements:
        """Extract model requirements from request"""
        return Requirements(
            vision=request.has_images(),
            json_mode=request.requires_json(),
            max_tokens=request.max_tokens,
            tools=request.tools is not None,
        )

    def _score_provider(
        self,
        provider: Provider,
        requirements: Requirements
    ) -> float:
        """Score provider match (0 = unsuitable, higher = better)"""
        caps = provider.capabilities()

        # Check hard requirements
        if requirements.vision and not caps.supports_vision:
            return 0
        if requirements.tools and not caps.supports_tools:
            return 0
        if requirements.json_mode and not caps.supports_json_mode:
            return 0

        # Score based on cost and performance
        score = 1.0
        score -= provider.cost_per_token * 1000  # Lower cost = higher score

        return score
```

#### IronClaw - Capability-Based Provider Selection

```rust
// File: ironclaw/src/provider/mod.rs
/**
 * Trait-based provider abstraction with capability-based routing.
 * Enables selecting providers based on feature requirements.
 */

#[async_trait]
pub trait Provider: Send + Sync {
    /// Generate completion
    async fn complete(
        &self,
        prompt: &str,
        options: CompletionOptions
    ) -> Result<String>;

    /// Generate streaming completion
    async fn complete_stream(
        &self,
        prompt: &str,
        options: CompletionOptions
    ) -> Result<BoxStream<'static, Result<String>>>;

    /// Get provider capabilities
    fn capabilities(&self) -> Capabilities;

    /// Get estimated cost for request
    fn estimate_cost(&self, tokens: u32) -> Cost;
}

pub struct Capabilities {
    pub max_tokens: u32,
    pub supports_vision: bool,
    pub supports_tools: bool,
    pub supports_json_mode: bool,
    pub supports_streaming: bool,
    pub context_window: u32,
}

pub struct Requirements {
    pub needs_vision: bool,
    pub needs_tools: bool,
    pub needs_json: bool,
    pub needs_streaming: bool,
    pub min_context: u32,
    pub max_cost: Option<Cost>,
}

// OpenAI implementation
pub struct OpenAiProvider {
    client: reqwest::Client,
    api_key: String,
    model: String,
}

#[async_trait]
impl Provider for OpenAiProvider {
    async fn complete(
        &self,
        prompt: &str,
        options: CompletionOptions
    ) -> Result<String> {
        let response = self.client
            .post("https://api.openai.com/v1/chat/completions")
            .bearer_auth(&self.api_key)
            .json(&json!({
                "model": self.model,
                "messages": [{"role": "user", "content": prompt}],
                "temperature": options.temperature,
                "max_tokens": options.max_tokens,
            }))
            .send()
            .await?;

        let result: OpenAIResponse = response.json().await?;
        Ok(result.choices[0].message.content.clone())
    }

    fn capabilities(&self) -> Capabilities {
        Capabilities {
            max_tokens: 4096,
            supports_vision: self.model.starts_with("gpt-4"),
            supports_tools: true,
            supports_json_mode: true,
            supports_streaming: true,
            context_window: if self.model.contains("32k") { 32768 } else { 8192 },
        }
    }

    fn estimate_cost(&self, tokens: u32) -> Cost {
        let rate = match self.model.as_str() {
            "gpt-4" => 0.03,
            "gpt-4-turbo" => 0.01,
            "gpt-3.5-turbo" => 0.0015,
            _ => 0.01,
        };
        Cost::usd(tokens as f64 * rate / 1000.0)
    }
}

// Router for capability-based selection
pub struct ProviderRouter {
    providers: Vec<Box<dyn Provider>>,
}

impl ProviderRouter {
    pub fn select(&self, requirements: Requirements) -> Option<&dyn Provider> {
        self.providers
            .iter()
            .map(|p| p.as_ref())
            .filter(|p| self.meets_requirements(p, &requirements))
            .min_by_key(|p| p.estimate_cost(1000))  // Select cheapest suitable
    }

    fn meets_requirements(
        &self,
        provider: &dyn Provider,
        req: &Requirements
    ) -> bool {
        let caps = provider.capabilities();

        (!req.needs_vision || caps.supports_vision)
            && (!req.needs_tools || caps.supports_tools)
            && (!req.needs_json || caps.supports_json_mode)
            && (!req.needs_streaming || caps.supports_streaming)
            && (req.min_context <= caps.context_window)
    }
}
```

#### PicoClaw - Go Model Routing with Fallback

```go
// File: picoclaw/internal/agent/agent.go
/**
 * Go-based model routing with fallback chains.
 * Minimal footprint for resource-constrained environments.
 */

type ModelRouter struct {
    primary     ModelProvider
    fallbacks   []ModelProvider
    currentIdx  int
}

type ModelProvider interface {
    Generate(ctx context.Context, prompt string) (string, error)
    HealthCheck(ctx context.Context) error
}

func (r *ModelRouter) Generate(ctx context.Context, prompt string) (string, error) {
    // Try primary first
    if r.currentIdx == 0 {
        result, err := r.primary.Generate(ctx, prompt)
        if err == nil {
            return result, nil
        }

        log.Printf("Primary model failed: %v, trying fallbacks", err)
    }

    // Try fallbacks in order
    for i := r.currentIdx; i < len(r.fallbacks); i++ {
        result, err := r.fallbacks[i].Generate(ctx, prompt)
        if err == nil {
            r.currentIdx = i  // Remember working fallback
            return result, nil
        }

        log.Printf("Fallback %d failed: %v", i, err)
    }

    return "", fmt.Errorf("all models exhausted")
}

func (r *ModelRouter) Reset() {
    r.currentIdx = 0
}

// Ollama provider implementation
type OllamaProvider struct {
    endpoint string
    model    string
    client   *http.Client
}

func (o *OllamaProvider) Generate(ctx context.Context, prompt string) (string, error) {
    reqBody := map[string]interface{}{
        "model":  o.model,
        "prompt": prompt,
        "stream": false,
    }

    jsonBody, _ := json.Marshal(reqBody)
    req, _ := http.NewRequestWithContext(
        ctx,
        "POST",
        o.endpoint+"/api/generate",
        bytes.NewBuffer(jsonBody),
    )

    resp, err := o.client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    var result struct {
        Response string `json:"response"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return "", err
    }

    return result.Response, nil
}

func (o *OllamaProvider) HealthCheck(ctx context.Context) error {
    req, _ := http.NewRequestWithContext(ctx, "GET", o.endpoint+"/api/tags", nil)
    resp, err := o.client.Do(req)
    if err != nil {
        return err
    }
    resp.Body.Close()
    return nil
}
```

---

## 6. State Management

### 6.1 State Management Approaches

| Project | State Model | Persistence | Key Files |
|---------|------------|-------------|-----------|
| **CoPaw** | Session-based | File/DB | `session.py` |
| **IronClaw** | Database-backed | PostgreSQL/SQLite | `state/` module |
| **ZeroClaw** | Trait-based | Configurable | `State` trait |
| **NanoClaw** | Group-based | File system | `GroupQueue` |
| **Codex** | Session + Task | SQLite | `state/` crate |
| **Gemini-CLI** | Session | In-memory + persist | `session.ts` |
| **Kimi-CLI** | Turn-based | Checkpoint files | `checkpoint.py` |
| **OpenCode** | Agent state | Multiple backends | `state/` |
| **Pi-Mono** | Event-sourced | File system | `events/` |

### 6.2 Detailed State Patterns

#### Codex - Session/Task/Turn Hierarchy

```rust
// File: codex/codex-rs/protocol/src/protocol.rs
/**
 * Hierarchical state model:
 * Session (configuration)
 *   └── Task (execution unit)
 *         └── Turn (iteration)
 */

pub struct Session {
    pub id: SessionId,
    pub created_at: DateTime<Utc>,
    pub config: Config,
    pub working_dir: PathBuf,
    // Global state across all tasks
}

pub struct Task {
    pub id: TaskId,
    pub session_id: SessionId,
    pub created_at: DateTime<Utc>,
    pub state: TaskState,
    pub description: Option<String>,
}

pub enum TaskState {
    Submitted,
    Working,
    Completed,
    Cancelled,
    Failed,
}

pub struct Turn {
    pub id: TurnId,
    pub task_id: TaskId,
    pub messages: Vec<Message>,
    pub response_id: Option<String>,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}

// State machine transitions
impl Task {
    pub fn transition(&mut self, new_state: TaskState) -> Result<()> {
        use TaskState::*;

        let valid = match (&self.state, &new_state) {
            (Submitted, Working) => true,
            (Working, Completed) => true,
            (Working, Failed) => true,
            (Working, Cancelled) => true,
            (Cancelled, Working) => true,  // Retry
            (Failed, Working) => true,     // Retry
            _ => false,
        };

        if !valid {
            return Err(Error::InvalidStateTransition {
                from: self.state.clone(),
                to: new_state,
            });
        }

        self.state = new_state;
        Ok(())
    }
}
```

#### IronClaw - Database Abstraction with Dual Backend

```rust
// File: ironclaw/src/state/mod.rs
/**
 * Dual-backend state abstraction: PostgreSQL + SQLite
 * All persistence through traits, backend selected at runtime.
 */

pub trait StateStore: Send + Sync {
    async fn save_conversation(&self, conv: &Conversation) -> Result<()>;
    async fn load_conversation(&self, id: &str) -> Result<Option<Conversation>>;
    async fn list_conversations(&self, limit: usize) -> Result<Vec<ConversationSummary>>;

    async fn save_job(&self, job: &Job) -> Result<()>;
    async fn load_job(&self, id: &str) -> Result<Option<Job>>;
    async fn list_pending_jobs(&self) -> Result<Vec<Job>>;

    async fn save_sandbox(&self, sandbox: &Sandbox) -> Result<()>;
    async fn load_sandbox(&self, id: &str) -> Result<Option<Sandbox>>;
}

// PostgreSQL implementation
pub struct PostgresStore {
    pool: PgPool,
}

#[async_trait]
impl StateStore for PostgresStore {
    async fn save_conversation(&self, conv: &Conversation) -> Result<()> {
        sqlx::query(
            r#"
            INSERT INTO conversations (id, title, messages, created_at, updated_at)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (id) DO UPDATE SET
                title = EXCLUDED.title,
                messages = EXCLUDED.messages,
                updated_at = EXCLUDED.updated_at
            "#
        )
        .bind(&conv.id)
        .bind(&conv.title)
        .bind(Json(&conv.messages))
        .bind(conv.created_at)
        .bind(conv.updated_at)
        .execute(&self.pool)
        .await?;

        Ok(())
    }

    // ... other implementations
}

// SQLite implementation
pub struct SqliteStore {
    pool: SqlitePool,
}

#[async_trait]
impl StateStore for SqliteStore {
    async fn save_conversation(&self, conv: &Conversation) -> Result<()> {
        // Similar structure but SQLite-specific queries
        sqlx::query(
            r#"
            INSERT OR REPLACE INTO conversations
            (id, title, messages, created_at, updated_at)
            VALUES (?1, ?2, ?3, ?4, ?5)
            "#
        )
        .bind(&conv.id)
        .bind(&conv.title)
        .bind(&serde_json::to_string(&conv.messages)?)
        .bind(conv.created_at)
        .bind(conv.updated_at)
        .execute(&self.pool)
        .await?;

        Ok(())
    }

    // ... other implementations
}

// Factory for runtime selection
pub async fn create_state_store(config: &Config) -> Result<Box<dyn StateStore>> {
    match config.database.backend {
        DatabaseBackend::Postgres => {
            let pool = PgPool::connect(&config.database.url).await?;
            Ok(Box::new(PostgresStore { pool }))
        }
        DatabaseBackend::Sqlite => {
            let pool = SqlitePool::connect(&config.database.url).await?;
            Ok(Box::new(SqliteStore { pool }))
        }
    }
}
```

#### NanoClaw - GroupQueue State Machine

```typescript
// File: nanoclaw/src/queue/group-queue.ts
/**
 * Container-per-group state machine.
 * Each group of messages runs in isolated Docker container.
 */

enum GroupState {
    PENDING = 'pending',
    RUNNING = 'running',
    SUCCESS = 'success',
    ERROR = 'error',
    RETRYING = 'retrying',
    CANCELLED = 'cancelled',
}

interface GroupConfig {
    maxRetries: number;
    timeoutMs: number;
    containerImage: string;
}

class GroupQueue {
    private state: GroupState = GroupState.PENDING;
    private retryCount = 0;
    private messages: Message[] = [];
    private containerId?: string;

    constructor(
        private id: string,
        private config: GroupConfig,
        private docker: Docker
    ) {}

    async addMessage(message: Message): Promise<void> {
        if (this.state !== GroupState.PENDING) {
            throw new Error('Cannot add messages to active group');
        }
        this.messages.push(message);
    }

    async process(): Promise<GroupResult> {
        this.transition(GroupState.RUNNING);

        try {
            // Create isolated container
            this.containerId = await this.createContainer();

            // Process messages in container
            const result = await this.runInContainer(this.containerId);

            this.transition(GroupState.SUCCESS);
            return { success: true, output: result };

        } catch (error) {
            return this.handleError(error);
        } finally {
            // Cleanup container
            if (this.containerId) {
                await this.docker.removeContainer(this.containerId);
            }
        }
    }

    private async handleError(error: Error): Promise<GroupResult> {
        this.retryCount++;

        if (this.retryCount < this.config.maxRetries) {
            this.transition(GroupState.RETRYING);

            // Exponential backoff
            await delay(Math.pow(2, this.retryCount) * 1000);

            // Retry
            return this.process();
        }

        this.transition(GroupState.ERROR);
        return { success: false, error: error.message };
    }

    private transition(newState: GroupState): void {
        console.log(`Group ${this.id}: ${this.state} -> ${newState}`);
        this.state = newState;
        this.emit('stateChange', { from: this.state, to: newState });
    }

    private async createContainer(): Promise<string> {
        const container = await this.docker.createContainer({
            Image: this.config.containerImage,
            Cmd: ['node', '/app/processor.js'],
            HostConfig: {
                Memory: 512 * 1024 * 1024,  // 512MB limit
                CpuQuota: 50000,             // 50% CPU
                AutoRemove: true,
            },
        });

        return container.id;
    }
}
```

---

## 7. Extensibility Patterns

### 7.1 Extension Architecture Comparison

| Project | Pattern | Extension Points | Key Innovation |
|---------|---------|------------------|----------------|
| **CoPaw** | Mixin composition | ToolGuard, Agents | AgentScope integration |
| **IronClaw** | Traits + Plugins | All components | Deep module design |
| **ZeroClaw** | 8 Core Traits | Everything | Trait-driven architecture |
| **NanoClaw** | Skill system | Tools, channels | Markdown-based skills |
| **Aider** | Strategy Pattern | Coders, Edit formats | Template method |
| **Codex** | Crate-based | 72 crates | Workspace architecture |
| **Gemini-CLI** | Hook system | Lifecycle events | Before/After hooks |
| **Kimi-CLI** | Skill Markdown | Agent types | Hierarchical discovery |
| **OpenCode** | Ruleset + Agents | Tools, permissions | Effect library |
| **Pi-Mono** | Package-based | 7 packages | Graph analysis |

### 7.2 Detailed Extensibility Patterns

#### ZeroClaw - 8 Core Traits Architecture

```rust
// File: zeroclaw/src/traits.rs
/**
 * Trait-driven architecture with 8 core extension points.
 * All major components are trait-based for maximum flexibility.
 */

/// LLM provider abstraction
pub trait Provider: Send + Sync {
    async fn complete(&self, prompt: &str, opts: Options) -> Result<String>;
    fn capabilities(&self) -> Capabilities;
}

/// Communication channel abstraction
pub trait Channel: Send + Sync {
    async fn send(&self, to: &str, content: &str) -> Result<()>;
    async fn receive(&self) -> Result<Option<Message>>;
}

/// Tool execution abstraction
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn schema(&self) -> ToolSchema;
    async fn execute(&self, input: serde_json::Value) -> Result<ToolOutput>;
}

/// Memory storage abstraction
pub trait Memory: Send + Sync {
    async fn store(&self, key: &str, value: &str) -> Result<()>;
    async fn retrieve(&self, key: &str) -> Result<Option<String>>;
    async fn search(&self, query: &str, top_k: usize) -> Result<Vec<SearchResult>>;
}

/// Sandbox isolation abstraction
pub trait Sandbox: Send + Sync {
    async fn execute(&self, command: &str) -> Result<ExecutionResult>;
    fn policy(&self) -> SandboxPolicy;
}

/// Event observation abstraction
pub trait Observer: Send + Sync {
    async fn observe(&self, event: Event) -> Result<()>;
}

/// Runtime execution abstraction
pub trait Runtime: Send + Sync {
    async fn run(&self, agent: &dyn Agent) -> Result<()>;
}

/// Hardware peripheral abstraction
pub trait Peripheral: Send + Sync {
    fn name(&self) -> &str;
    async fn read(&self) -> Result<PeripheralData>;
    async fn write(&self, data: PeripheralData) -> Result<()>;
}

// Factory pattern for component creation
pub mod factories {
    use super::*;

    pub fn create_provider(config: &ProviderConfig) -> Arc<dyn Provider> {
        match config.provider_type {
            ProviderType::OpenAi => Arc::new(OpenAiProvider::new(config)),
            ProviderType::Anthropic => Arc::new(AnthropicProvider::new(config)),
            ProviderType::Ollama => Arc::new(OllamaProvider::new(config)),
        }
    }

    pub fn create_memory(config: &MemoryConfig) -> Arc<dyn Memory> {
        match config.memory_type {
            MemoryType::Sqlite => Arc::new(SqliteMemory::new(config)),
            MemoryType::Postgres => Arc::new(PostgresMemory::new(config)),
            MemoryType::InMemory => Arc::new(InMemoryMemory::new()),
        }
    }

    pub fn create_channel(config: &ChannelConfig) -> Arc<dyn Channel> {
        match config.channel_type {
            ChannelType::Telegram => Arc::new(TelegramChannel::new(config)),
            ChannelType::Discord => Arc::new(DiscordChannel::new(config)),
            ChannelType::Slack => Arc::new(SlackChannel::new(config)),
        }
    }
}
```

#### Gemini-CLI - Hook System

```typescript
// File: gemini-cli/packages/core/src/hooks/hookSystem.ts
/**
 * Comprehensive hook system for lifecycle events.
 * Allows extensions at 8 key points in the agent lifecycle.
 */

export enum HookEvent {
    // Session lifecycle
    SessionStart = 'session:start',
    SessionEnd = 'session:end',

    // Agent execution
    BeforeAgent = 'agent:before',
    AfterAgent = 'agent:after',

    // Model interaction
    BeforeModel = 'model:before',
    AfterModel = 'model:after',

    // Tool execution
    BeforeTool = 'tool:before',
    AfterTool = 'tool:after',
}

export type HookHandler<T = any> = (
    context: HookContext<T>
) => Promise<HookResult | void>;

export interface HookContext<T> {
    event: HookEvent;
    data: T;
    abort: (reason: string) => void;
}

export interface HookResult {
    modified?: boolean;
    data?: any;
}

export class HookSystem {
    private registry: Map<HookEvent, HookHandler[]> = new Map();

    register(event: HookEvent, handler: HookHandler): void {
        const handlers = this.registry.get(event) || [];
        handlers.push(handler);
        this.registry.set(event, handlers);
    }

    async execute<T>(
        event: HookEvent,
        data: T
    ): Promise<T> {
        const handlers = this.registry.get(event) || [];
        let currentData = data;

        for (const handler of handlers) {
            const context: HookContext<T> = {
                event,
                data: currentData,
                abort: (reason) => {
                    throw new HookAbortError(reason);
                },
            };

            const result = await handler(context);

            if (result?.modified && result.data) {
                currentData = result.data;
            }
        }

        return currentData;
    }
}

// Usage example
const hooks = new HookSystem();

// Log all tool calls
hooks.register(HookEvent.BeforeTool, async (context) => {
    console.log(`Executing tool: ${context.data.toolName}`);
    console.log(`Arguments: ${JSON.stringify(context.data.args)}`);
});

// Validate file writes
hooks.register(HookEvent.BeforeTool, async (context) => {
    if (context.data.toolName === 'write-file') {
        const path = context.data.args.path;
        if (path.includes('..')) {
            context.abort('Path traversal detected');
        }
    }
});
```

#### Kimi-CLI - Skill-Based Extension

```python
# File: kimi-cli/src/skills/loader.py
from pathlib import Path
from dataclasses import dataclass
from enum import Enum
import frontmatter
import markdown

class AgentType(Enum):
    GENERAL = "general"
    CODE = "code"
    PLAN = "plan"
    BUILD = "build"
    EXPLORE = "explore"
    REVIEW = "review"

@dataclass
class Tool:
    name: str
    description: str
    parameters: dict
    handler: callable

@dataclass
class Skill:
    """
    Skills are Markdown files with YAML frontmatter.
    Hierarchical discovery: skills/ directory tree.
    """
    name: str
    description: str
    agent_type: AgentType
    tools: list[Tool]
    prompt_template: str
    tags: list[str]
    priority: int  # Higher = more likely to be selected

class SkillLoader:
    """Load and manage skills from filesystem"""

    def discover(self, root_path: Path) -> list[Skill]:
        """
        Recursively discover all skills in directory.
        Directory structure becomes skill hierarchy.
        """
        skills = []

        for md_file in root_path.rglob("*.md"):
            # Skip non-skill markdown files
            if not self._is_skill_file(md_file):
                continue

            skill = self._parse_skill(md_file)
            if skill:
                skills.append(skill)

        # Sort by priority (descending)
        skills.sort(key=lambda s: s.priority, reverse=True)

        return skills

    def _parse_skill(self, path: Path) -> Optional[Skill]:
        """Parse skill from markdown file with frontmatter"""
        content = path.read_text(encoding='utf-8')

        # Parse frontmatter and body
        metadata, body = frontmatter.parse(content)

        # Required fields
        if 'name' not in metadata:
            return None

        # Extract tool definitions from code blocks
        tools = self._extract_tools(body)

        return Skill(
            name=metadata.get('name', path.stem),
            description=metadata.get('description', ''),
            agent_type=AgentType(metadata.get('agent', 'general')),
            tools=tools,
            prompt_template=body,
            tags=metadata.get('tags', []),
            priority=metadata.get('priority', 0),
        )

    def _extract_tools(self, markdown: str) -> list[Tool]:
        """Extract tool schemas from ```tool blocks"""
        tools = []

        # Parse markdown for tool definitions
        # Format:
        # ```tool
        # name: tool_name
        # parameters:
        #   - name: param1
        #     type: string
        # ```

        return tools

    def load_for_context(
        self,
        query: str,
        agent_type: AgentType,
        max_tokens: int = 2000
    ) -> list[Skill]:
        """
        Select relevant skills for current context.
        Based on keyword matching and agent type.
        """
        all_skills = self.discover(Path("skills"))

        # Filter by agent type
        matching = [s for s in all_skills if s.agent_type == agent_type]

        # Score by keyword relevance
        scored = []
        for skill in matching:
            score = self._calculate_relevance(skill, query)
            scored.append((score, skill))

        # Sort by score
        scored.sort(key=lambda x: x[0], reverse=True)

        # Select within token budget
        selected = []
        total_tokens = 0

        for _, skill in scored:
            skill_tokens = self._estimate_tokens(skill.prompt_template)
            if total_tokens + skill_tokens > max_tokens:
                break
            selected.append(skill)
            total_tokens += skill_tokens

        return selected
```

---

## 8. Error Handling & Resilience

### 8.1 Error Handling Patterns

| Project | Strategy | Retry/Circuit Breaker | Key Innovation |
|---------|----------|----------------------|----------------|
| **CoPaw** | Exceptions | AgentScope retries | Automatic recovery |
| **IronClaw** | thiserror + anyhow | Exponential backoff | Layered errors |
| **ZeroClaw** | Result<T, E> | Configurable | Type-safe errors |
| **Codex** | Structured errors | Retry policies | Error taxonomy |
| **Gemini-CLI** | Error aggregation | Per-tool retry | Continue on error |
| **Kimi-CLI** | Exception hierarchy | Checkpoint recovery | Resume from checkpoint |
| **OpenCode** | Effect library | Built-in retry | Type-safe composition |
| **Pi-Mono** | Result types | Manual retry | Rust-style errors |

### 8.2 Detailed Error Handling Patterns

#### IronClaw - Layered Error Handling

```rust
// File: ironclaw/src/error.rs
use thiserror::Error;

/**
 * Layered error types using thiserror.
 * Provides clear error taxonomy and context.
 */

#[derive(Error, Debug)]
pub enum IronClawError {
    #[error("Provider error: {0}")]
    Provider(#[from] ProviderError),

    #[error("Sandbox violation: {0}")]
    Sandbox(#[from] SandboxError),

    #[error("Tool execution failed: {source}")]
    Tool {
        #[source]
        source: ToolError,
        tool_name: String,
    },

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Channel error ({channel}): {message}")]
    Channel {
        channel: String,
        message: String,
    },

    #[error("Configuration error: {0}")]
    Config(String),

    #[error("Validation error: {0}")]
    Validation(String),
}

#[derive(Error, Debug)]
pub enum ProviderError {
    #[error("API request failed: {0}")]
    Request(#[from] reqwest::Error),

    #[error("Rate limited. Retry after {retry_after}s")]
    RateLimited { retry_after: u64 },

    #[error("Model unavailable: {model}")]
    ModelUnavailable { model: String },

    #[error("Invalid API key")]
    InvalidApiKey,

    #[error("Context length exceeded: {tokens}/{max}")]
    ContextLengthExceeded { tokens: u32, max: u32 },
}

#[derive(Error, Debug)]
pub enum SandboxError {
    #[error("Permission denied: {action}")]
    PermissionDenied { action: String },

    #[error("Resource limit exceeded: {resource}")]
    ResourceLimitExceeded { resource: String },

    #[error("Execution timeout after {duration}s")]
    Timeout { duration: u64 },
}

// Application code uses anyhow for flexibility
pub type Result<T> = anyhow::Result<T>;

// Retry configuration
pub struct RetryConfig {
    pub max_attempts: u32,
    pub base_delay: Duration,
    pub max_delay: Duration,
    pub exponential_base: f64,
}

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_attempts: 3,
            base_delay: Duration::from_millis(100),
            max_delay: Duration::from_secs(60),
            exponential_base: 2.0,
        }
    }
}

pub async fn with_retry<F, Fut, T>(
    config: &RetryConfig,
    operation: F,
) -> Result<T>
where
    F: Fn() -> Fut,
    Fut: Future<Output = Result<T>>,
{
    let mut attempt = 0;
    let mut delay = config.base_delay;

    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) => {
                attempt += 1;

                if attempt >= config.max_attempts {
                    return Err(e);
                }

                // Check if error is retryable
                if !is_retryable(&e) {
                    return Err(e);
                }

                // Exponential backoff with jitter
                sleep(delay).await;
                delay = min(
                    Duration::from_millis(
                        (delay.as_millis() as f64 * config.exponential_base) as u64
                    ),
                    config.max_delay,
                );
            }
        }
    }
}

fn is_retryable(error: &anyhow::Error) -> bool {
    // Check if error type indicates retryability
    if let Some(provider_err) = error.downcast_ref::<ProviderError>() {
        matches!(provider_err,
            ProviderError::RateLimited { .. } |
            ProviderError::ModelUnavailable { .. } |
            ProviderError::Request(_)
        )
    } else {
        false
    }
}
```

#### Gemini-CLI - Error Aggregation Pattern

```typescript
// File: gemini-cli/packages/core/src/scheduler/scheduler.ts
export interface ToolError {
    toolName: string;
    error: Error;
    recoverable: boolean;
}

export class AggregatedError extends Error {
    constructor(
        public readonly errors: ToolError[],
        public readonly partialResults: CompletedToolCall[]
    ) {
        super(`Multiple tool failures: ${errors.map(e => e.toolName).join(', ')}`);
    }
}

export class Scheduler {
    /**
     * Error aggregation: collect errors but continue execution
     * when possible. Return partial results.
     */
    async schedule(
        tools: ToolCall[],
        signal: AbortSignal
    ): Promise<CompletedToolCall[]> {
        const results: CompletedToolCall[] = [];
        const errors: ToolError[] = [];

        for (const tool of tools) {
            try {
                const result = await this.execute(tool, signal);
                results.push(result);

            } catch (error) {
                const toolError: ToolError = {
                    toolName: tool.name,
                    error: error as Error,
                    recoverable: this.isRecoverable(error),
                };

                errors.push(toolError);

                // Check if we should continue
                if (!toolError.recoverable || this.policy.stopOnError(tool)) {
                    break;
                }
            }
        }

        // If we have errors, throw aggregated error with partial results
        if (errors.length > 0) {
            throw new AggregatedError(errors, results);
        }

        return results;
    }

    private isRecoverable(error: unknown): boolean {
        if (error instanceof NetworkError) return true;
        if (error instanceof RateLimitError) return true;
        if (error instanceof TimeoutError) return true;
        if (error instanceof PermissionError) return false;
        if (error instanceof ValidationError) return false;
        return false;
    }
}

// Usage with error handling
try {
    const results = await scheduler.schedule(tools, signal);
} catch (error) {
    if (error instanceof AggregatedError) {
        // Handle partial success
        console.log(`Completed: ${error.partialResults.length}`);
        console.log(`Failed: ${error.errors.length}`);

        for (const err of error.errors) {
            if (err.recoverable) {
                // Queue for retry
                await retryQueue.add(err.toolName);
            } else {
                // Log fatal error
                console.error(`Fatal error in ${err.toolName}:`, err.error);
            }
        }
    }
}
```

#### MimicClaw - "Define Errors Out of Existence"

```c
// File: mimiclaw/wifi/wifi_manager.c
/**
 * Excellent example from "A Philosophy of Software Design"
 * Instead of propagating errors, handle them internally with retry.
 * Only expose permanent failures.
 */

static const int MAX_RETRIES = 5;
static int s_retry_count = 0;

static void wifi_event_handler(
    void* arg,
    esp_event_base_t event_base,
    int32_t event_id,
    void* event_data
) {
    if (event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_count < MAX_RETRIES) {
            // Auto-retry with exponential backoff
            // No error propagated to caller
            int delay_ms = (1 << s_retry_count) * 1000;  // 1, 2, 4, 8, 16 seconds

            ESP_LOGI(TAG, "Retry connection in %dms (attempt %d/%d)",
                     delay_ms, s_retry_count + 1, MAX_RETRIES);

            vTaskDelay(pdMS_TO_TICKS(delay_ms));
            esp_wifi_connect();
            s_retry_count++;
        } else {
            // Only permanent failure becomes visible
            ESP_LOGE(TAG, "Failed to connect after %d attempts", MAX_RETRIES);

            // Notify application of permanent failure
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
        }
    } else if (event_id == IP_EVENT_STA_GOT_IP) {
        // Success - reset retry counter
        s_retry_count = 0;

        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "Connected with IP: " IPSTR, IP2STR(&event->ip_info.ip));

        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}
```

---

## 9. Comprehensive Comparison Matrices

### 9.1 Agent Loop Architecture Matrix

| Feature | CoPaw | IronClaw | ZeroClaw | Aider | Codex | Gemini-CLI | Kimi-CLI |
|---------|-------|----------|----------|-------|-------|------------|----------|
| **Async** | ✓ | ✓ (Tokio) | ✓ (Tokio) | ✗ | ✓ (Tokio) | ✓ | ✓ |
| **Trait-based** | ✗ | ✓ | ✓ | ✗ | Partial | ✗ | Partial |
| **UI Separation** | Partial | ✗ | ✗ | ✗ | ✓ (SQ/EQ) | Partial | ✓ (Wire) |
| **Streaming** | Limited | ✓ | ✓ | Limited | ✓ | ✓ | ✓ |
| **Policy Integration** | ✓ (ToolGuard) | ✓ (Delegate) | ✓ | ✗ | ✓ (Guardian) | ✓ | ✗ |
| **State Machine** | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ |

### 9.2 Tool System Matrix

| Feature | CoPaw | IronClaw | Gemini-CLI | Aider | Qwen-Code | Kimi-CLI |
|---------|-------|----------|------------|-------|-----------|----------|
| **Registration** | Decorator | Trait | 3-tier | Class | Built-in | Markdown |
| **Dynamic Loading** | ✓ | ✓ | ✓ | ✗ | ✓ | ✓ |
| **MCP Support** | ✗ | Planned | ✓ | ✗ | ✓ | ✗ |
| **Sandbox** | Limited | ✓ (3-tier) | Platform | ✗ | ✗ | ✗ |
| **Approval Flow** | ✓ (ToolGuard) | ✓ | ✓ | ✗ | ✗ | ✗ |
| **Rate Limiting** | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ |

### 9.3 Memory Integration Matrix

| Feature | CoPaw | IronClaw | Nanobot | Aider | Codex | Gemini-CLI |
|---------|-------|----------|---------|-------|-------|------------|
| **Short-term** | AgentScope Memory | ConversationStore | HISTORY.md | Chat history | Session | Message history |
| **Long-term** | Persistent | PostgreSQL/pgvector | MEMORY.md | RepoMap | SQLite | ContextManager |
| **Hybrid Search** | ✗ | ✓ (RRF) | ✗ | ✗ | ✗ | ✗ |
| **Context Compression** | ✗ | ✓ | ✗ | ✗ | ✗ | ✓ |
| **Token Management** | ✗ | ✓ | ✗ | ✓ | ✗ | ✓ |
| **Consolidation** | ✗ | ✗ | ✓ | ✗ | ✓ | ✗ |

### 9.4 Multi-Channel Support Matrix

| Feature | CoPaw | IronClaw | ZeroClaw | NanoClaw | OpenClaw | PicoClaw |
|---------|-------|----------|----------|----------|----------|----------|
| **Channels** | 15+ | 10+ | Configurable | 15+ | 20+ | 8+ |
| **Abstraction** | AgentScope | Trait | Trait | Self-register | Adapter | Struct |
| **Gateway Pattern** | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ |
| **Trait-based** | ✗ | ✓ | ✓ | ✗ | Partial | ✗ |
| **Registration** | Manual | Factory | Factory | Self-reg | Factory | Static |

### 9.5 Provider Abstraction Matrix

| Feature | CoPaw | IronClaw | PicoClaw | Aider | Codex | Gemini-CLI |
|---------|-------|----------|----------|-------|-------|------------|
| **Providers** | 15+ | Multiple | 5+ | 100+ (LiteLLM) | Multiple | Gemini |
| **Abstraction** | Manager | Trait | Router | LiteLLM | Connector | Client |
| **Capability-based** | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ |
| **Fallback Chain** | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ |
| **Cost Estimation** | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| **Routing Logic** | Score-based | Capability | Failover | Direct | Config | Direct |

### 9.6 State Management Matrix

| Feature | CoPaw | IronClaw | NanoClaw | Codex | Gemini-CLI | Kimi-CLI |
|---------|-------|----------|----------|-------|------------|----------|
| **Persistence** | File/DB | PostgreSQL/SQLite | File | SQLite | File | File |
| **Hierarchy** | Session | Flat | Group | Session/Task/Turn | Session | Turn |
| **State Machine** | ✗ | ✗ | ✓ | ✓ | ✗ | ✗ |
| **Dual Backend** | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ |
| **Migration** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

### 9.7 Extensibility Matrix

| Feature | CoPaw | IronClaw | ZeroClaw | Aider | Codex | Gemini-CLI | Kimi-CLI |
|---------|-------|----------|----------|-------|-------|------------|----------|
| **Pattern** | Mixin | Traits | 8 Traits | Strategy | Crates | Hooks | Markdown |
| **Extension Points** | 5+ | 10+ | 8 | 3+ | 72 crates | 8 hooks | Unlimited |
| **Hot Reload** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Skill System** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

### 9.8 Error Handling Matrix

| Feature | CoPaw | IronClaw | Codex | Gemini-CLI | OpenCode | MimicClaw |
|---------|-------|----------|-------|------------|----------|-----------|
| **Strategy** | Exceptions | thiserror | Structured | Aggregation | Effect | Define-away |
| **Retry** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ (internal) |
| **Circuit Breaker** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| **Error Taxonomy** | ✗ | ✓ | ✓ | ✓ | ✓ | ✗ |
| **Aggregation** | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ |

---

## 10. Recommended Unified Architecture

Based on the analysis of all 15 projects, here is a recommended unified architecture incorporating best practices:

### 10.1 Architecture Principles

1. **Deep Modules**: Simple interfaces hiding complex implementations
2. **Trait-Based Design**: Maximum flexibility through composition
3. **Separation of Concerns**: UI, Logic, and Storage as separate layers
4. **Graceful Degradation**: Continue operating when components fail
5. **Type Safety**: Leverage type systems for correctness

### 10.2 Recommended Component Stack

| Layer | Recommended Implementation | Source Inspiration |
|-------|---------------------------|-------------------|
| **Agent Loop** | Generic Delegate Pattern | IronClaw |
| **Message Bus** | SQ/EQ Protocol | Codex |
| **Tool System** | Three-tier Registry + Traits | Gemini-CLI + IronClaw |
| **Memory** | Two-layer + Hybrid Search | Nanobot + IronClaw |
| **Channels** | Trait-based + Gateway | IronClaw + OpenClaw |
| **Providers** | Capability-based Routing | IronClaw |
| **State** | Hierarchical + Dual Backend | Codex + IronClaw |
| **Extensibility** | Hooks + Skills | Gemini-CLI + Kimi-CLI |
| **Errors** | Layered + Aggregation | IronClaw + Gemini-CLI |

### 10.3 Rust Reference Implementation

```rust
// File: unified_agent/src/lib.rs
/**
 * Recommended unified agent architecture.
 * Combines best practices from all 15 analyzed projects.
 */

use async_trait::async_trait;
use std::sync::Arc;
use thiserror::Error;

// ============================================================================
// LAYER 1: CORE TRAITS (Deep Module Foundation)
// ============================================================================

/// Generic agent loop delegate trait
#[async_trait]
pub trait AgentDelegate: Send + Sync {
    async fn gather_context(&self) -> Result<Context>;
    async fn query_llm(&self, context: Context) -> Result<Response>;
    fn parse_response(&self, response: Response) -> Result<Vec<Action>>;
    async fn execute_action(&self, action: Action) -> Result<ActionResult>;
    fn should_terminate(&self) -> bool;
}

/// Unified LLM provider trait with capability-based routing
#[async_trait]
pub trait Provider: Send + Sync {
    async fn complete(&self, request: CompletionRequest) -> Result<Completion>;
    async fn complete_stream(&self, request: CompletionRequest) -> Result<Stream>;
    fn capabilities(&self) -> Capabilities;
    fn estimate_cost(&self, tokens: u32) -> Cost;
}

/// Unified channel trait for multi-platform messaging
#[async_trait]
pub trait Channel: Send + Sync {
    async fn send(&self, message: OutgoingMessage) -> Result<MessageId>;
    async fn receive(&self) -> Result<Vec<IncomingMessage>>;
    fn capabilities(&self) -> ChannelCapabilities;
}

/// Tool execution trait with sandbox integration
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn schema(&self) -> ToolSchema;
    async fn execute(&self, input: serde_json::Value) -> Result<ToolOutput>;
    fn sandbox_policy(&self) -> SandboxPolicy;
}

/// Memory trait with hybrid search support
#[async_trait]
pub trait Memory: Send + Sync {
    async fn store(&self, entry: MemoryEntry) -> Result<()>;
    async fn search(&self, query: &str, opts: SearchOpts) -> Result<Vec<MemoryEntry>>;
    async fn hybrid_search(&self, query: &str, opts: SearchOpts) -> Result<Vec<MemoryEntry>>;
}

/// State persistence trait with dual backend support
#[async_trait]
pub trait StateStore: Send + Sync {
    async fn save_session(&self, session: &Session) -> Result<()>;
    async fn load_session(&self, id: &str) -> Result<Option<Session>>;
    async fn save_task(&self, task: &Task) -> Result<()>;
    async fn load_task(&self, id: &str) -> Result<Option<Task>>;
}

// ============================================================================
// LAYER 2: MESSAGE BUS (SQ/EQ Protocol)
// ============================================================================

/// Submission Queue - UI to Agent
#[derive(Debug, Clone)]
pub enum Op {
    StartTurn { prompt: String, context: Context },
    ApproveTool { tool_id: String, approved: bool },
    Interrupt,
}

/// Event Queue - Agent to UI
#[derive(Debug, Clone)]
pub enum Event {
    TurnStarted { turn_id: String },
    MessageDelta { content: String },
    ToolCallRequested { tool: String, args: serde_json::Value },
    ToolResult { result: ToolOutput },
    TurnCompleted,
    Error { message: String },
}

pub struct MessageBus {
    submission: async_channel::Sender<Op>,
    events: async_channel::Receiver<Event>,
}

// ============================================================================
// LAYER 3: AGENT LOOP
// ============================================================================

pub struct Agent<D: AgentDelegate> {
    delegate: D,
    bus: MessageBus,
}

impl<D: AgentDelegate> Agent<D> {
    pub async fn run(&mut self) -> Result<()> {
        loop {
            // 1. Gather context
            let context = self.delegate.gather_context().await?;

            // 2. Query LLM
            let response = self.delegate.query_llm(context).await?;

            // 3. Parse into actions
            let actions = self.delegate.parse_response(response)?;

            // 4. Execute actions
            for action in actions {
                let result = self.delegate.execute_action(action).await?;

                // Handle tool approval if needed
                if let ActionResult::NeedsApproval { tool, args } = result {
                    // Request approval through message bus
                }
            }

            // 5. Check termination
            if self.delegate.should_terminate() {
                break;
            }
        }

        Ok(())
    }
}

// ============================================================================
// LAYER 4: TOOL REGISTRY (Three-tier)
// ============================================================================

pub struct ToolRegistry {
    builtin: dashmap::DashMap<String, Arc<dyn Tool>>,
    discovered: dashmap::DashMap<String, Arc<dyn Tool>>,
    mcp: dashmap::DashMap<String, Arc<dyn Tool>>,
}

impl ToolRegistry {
    pub fn get(&self, name: &str) -> Option<Arc<dyn Tool>> {
        self.builtin
            .get(name)
            .map(|t| t.clone())
            .or_else(|| self.discovered.get(name).map(|t| t.clone()))
            .or_else(|| self.mcp.get(name).map(|t| t.clone()))
    }

    pub fn all_tools(&self) -> Vec<Arc<dyn Tool>> {
        let mut tools: Vec<_> = self.builtin.iter().map(|t| t.clone()).collect();
        tools.extend(self.discovered.iter().map(|t| t.clone()));
        tools.extend(self.mcp.iter().map(|t| t.clone()));
        tools
    }
}

// ============================================================================
// LAYER 5: PROVIDER ROUTER (Capability-based)
// ============================================================================

pub struct ProviderRouter {
    providers: Vec<Arc<dyn Provider>>,
}

impl ProviderRouter {
    pub fn select(&self, requirements: Requirements) -> Option<Arc<dyn Provider>> {
        self.providers
            .iter()
            .filter(|p| self.meets_requirements(p, &requirements))
            .min_by_key(|p| p.estimate_cost(1000))
            .cloned()
    }

    fn meets_requirements(&self, p: &dyn Provider, req: &Requirements) -> bool {
        let caps = p.capabilities();
        (!req.needs_vision || caps.supports_vision)
            && (!req.needs_tools || caps.supports_tools)
            && req.min_context <= caps.context_window
    }
}

// ============================================================================
// LAYER 6: MEMORY (Two-layer + Hybrid Search)
// ============================================================================

pub struct HybridMemory<M: Memory, V: VectorStore> {
    /// Short-term: Recent messages
    short_term: M,
    /// Long-term: Vector embeddings
    long_term: V,
    /// Full-text search
    fts: Arc<dyn FullTextSearch>,
}

impl<M: Memory, V: VectorStore> HybridMemory<M, V> {
    pub async fn search(&self, query: &str, opts: SearchOpts) -> Result<Vec<MemoryEntry>> {
        // Parallel search
        let (vector_results, fts_results) = tokio::join!(
            self.long_term.search(query, opts.clone()),
            self.fts.search(query, opts.top_k * 2)
        );

        // Reciprocal Rank Fusion
        let fused = reciprocal_rank_fusion(
            vector_results?,
            fts_results?,
            opts.alpha
        );

        Ok(fused)
    }
}

// RRF algorithm
fn reciprocal_rank_fusion(
    vector: Vec<MemoryEntry>,
    fts: Vec<MemoryEntry>,
    alpha: f32
) -> Vec<MemoryEntry> {
    let mut scores: HashMap<String, f32> = HashMap::new();

    // Score from vector results
    for (rank, entry) in vector.iter().enumerate() {
        let score = 1.0 / (alpha + rank as f32 + 1.0);
        *scores.entry(entry.id.clone()).or_insert(0.0) += score;
    }

    // Score from FTS results
    for (rank, entry) in fts.iter().enumerate() {
        let score = 1.0 / (alpha + rank as f32 + 1.0);
        *scores.entry(entry.id.clone()).or_insert(0.0) += score;
    }

    // Sort by combined score
    let mut results: Vec<_> = scores.into_iter().collect();
    results.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

    // Map back to entries
    results.into_iter()
        .filter_map(|(id, _)| vector.iter().chain(fts.iter()).find(|e| e.id == id))
        .cloned()
        .collect()
}

// ============================================================================
// LAYER 7: STATE MANAGEMENT (Hierarchical)
// ============================================================================

pub struct HierarchicalState<S: StateStore> {
    store: S,
    current_session: Option<Session>,
    current_task: Option<Task>,
}

impl<S: StateStore> HierarchicalState<S> {
    pub async fn start_session(&mut self, config: Config) -> Result<String> {
        let session = Session::new(config);
        let id = session.id.clone();
        self.store.save_session(&session).await?;
        self.current_session = Some(session);
        Ok(id)
    }

    pub async fn start_task(&mut self, description: String) -> Result<String> {
        let task = Task::new(
            self.current_session.as_ref().unwrap().id.clone(),
            description
        );
        let id = task.id.clone();
        self.store.save_task(&task).await?;
        self.current_task = Some(task);
        Ok(id)
    }

    pub async fn transition_task(&mut self, state: TaskState) -> Result<()> {
        if let Some(ref mut task) = self.current_task {
            task.transition(state)?;
            self.store.save_task(task).await?;
        }
        Ok(())
    }
}

// ============================================================================
// LAYER 8: ERROR HANDLING (Layered + Aggregation)
// ============================================================================

#[derive(Error, Debug)]
pub enum AgentError {
    #[error("Provider error: {0}")]
    Provider(#[from] ProviderError),

    #[error("Tool execution failed: {source} (tool: {tool_name})")]
    Tool {
        #[source]
        source: ToolError,
        tool_name: String,
    },

    #[error("Channel error: {0}")]
    Channel(String),

    #[error("Multiple failures occurred")]
    Aggregated { errors: Vec<AgentError> },
}

pub async fn with_retry<T>(
    operation: impl Fn() -> impl Future<Output = Result<T>>,
    config: RetryConfig
) -> Result<T> {
    let mut last_error = None;
    let mut delay = config.base_delay;

    for attempt in 0..config.max_attempts {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) => {
                if !is_retryable(&e) {
                    return Err(e);
                }
                last_error = Some(e);
                sleep(delay).await;
                delay = min(delay * 2, config.max_delay);
            }
        }
    }

    Err(last_error.unwrap())
}

// ============================================================================
// TYPES AND DATA STRUCTURES
// ============================================================================

pub struct Context {
    pub messages: Vec<Message>,
    pub tools: Vec<ToolSchema>,
    pub system_prompt: String,
    pub memory_context: Option<String>,
}

pub struct Response {
    pub content: String,
    pub tool_calls: Vec<ToolCall>,
    pub finish_reason: FinishReason,
}

pub enum Action {
    SendMessage(String),
    CallTool { name: String, args: serde_json::Value },
    SearchMemory(String),
    Terminate,
}

pub enum ActionResult {
    Success(ToolOutput),
    NeedsApproval { tool: String, args: serde_json::Value },
    Error(ToolError),
}

pub struct Capabilities {
    pub supports_vision: bool,
    pub supports_tools: bool,
    pub supports_json_mode: bool,
    pub context_window: u32,
}

pub struct Requirements {
    pub needs_vision: bool,
    pub needs_tools: bool,
    pub needs_json: bool,
    pub min_context: u32,
}

pub struct SearchOpts {
    pub top_k: usize,
    pub alpha: f32,
    pub threshold: f32,
}

pub struct RetryConfig {
    pub max_attempts: u32,
    pub base_delay: Duration,
    pub max_delay: Duration,
}

pub type Result<T> = std::result::Result<T, AgentError>;
```

---

## 11. Summary

This analysis examined complete agent architectures across **15 AI agent projects**, revealing diverse approaches to common problems:

### Key Findings

1. **Agent Loops**: Range from simple synchronous (Aider) to complex trait-based async systems (IronClaw, Codex SQ/EQ)

2. **Tool Systems**: Three-tier registries (Gemini-CLI) and trait-based execution (IronClaw) offer the best flexibility

3. **Memory**: Hybrid search with RRF (IronClaw) and two-layer consolidation (Nanobot, Codex) represent best practices

4. **Multi-Channel**: Trait-based abstraction (IronClaw) with gateway pattern (OpenClaw) scales best

5. **Providers**: Capability-based routing (IronClaw) enables intelligent model selection

6. **State**: Hierarchical models (Codex) with dual backend support (IronClaw) provide flexibility

7. **Extensibility**: Hook systems (Gemini-CLI) combined with skill-based loading (Kimi-CLI) maximize extensibility

8. **Errors**: Layered error types (IronClaw) with aggregation (Gemini-CLI) provide resilience

### Architecture Quality Rankings

| Category | Best Implementation | Why |
|----------|-------------------|-----|
| Deep Modules | IronClaw | Trait delegation hides complexity |
| Error Handling | IronClaw + Gemini-CLI | Layered + aggregation |
| Extensibility | ZeroClaw | 8 core traits |
| Memory | IronClaw | Hybrid search + dual backend |
| Tool System | Gemini-CLI | Three-tier registry |
| State Management | Codex | Hierarchical model |

The recommended unified architecture combines these best practices into a cohesive, type-safe, and extensible system suitable for both personal assistants and programming agents.
