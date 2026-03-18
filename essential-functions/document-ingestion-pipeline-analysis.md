# Document Ingestion Pipeline Analysis

## 目录
1. [执行摘要](#执行摘要)
2. [Personal Assistants 分析](#personal-assistants-分析)
3. [Programming Agents 分析](#programming-agents-分析)
4. [核心组件深度对比](#核心组件深度对比)
5. [推荐统一架构](#推荐统一架构)
6. [实现参考代码](#实现参考代码)

---

## 执行摘要

本文档对 Omni-Agent-OS 生态中 **15 个 AI Agent 项目**的文档摄取流程进行了全面分析。

### 项目分类

| 类别 | 数量 | 项目列表 |
|------|------|----------|
| **Personal Assistants** | 8 | CoPaw, IronClaw, NanoClaw, PicoClaw, ZeroClaw, MimicClaw, OpenClaw, Nanobot |
| **Programming Agents** | 7 | Aider, Codex, Qwen-Code, Gemini-CLI, Kimi-CLI, OpenCode, Pi-Mono |

### 关键发现

1. **架构多样性显著**：Python 单体 (Aider, CoPaw) → TypeScript/Node.js 微服务 (Qwen Code, Gemini CLI) → Rust 系统 (Codex, IronClaw, ZeroClaw) → Go 轻量方案 (PicoClaw)

2. **向量存储光谱**：从简单文件存储 (MEMORY.md) → SQLite + 扩展 (sqlite-vec, FTS5) → 完整向量数据库 (Chroma, LanceDB, PostgreSQL/pgvector)

3. **三种上下文管理哲学**：
   - **全代码库上下文** (Aider RepoMap, Codex 文件索引)
   - **选择性检索** (OpenClaw LanceDB, IronClaw 混合搜索)
   - **简单文件记忆** (NanoClaw CLAUDE.md, Nanobot MEMORY.md)

---

## Personal Assistants 分析

### 1. CoPaw - ReMe 外部库集成

**架构定位**

CoPaw 使用外部 **ReMe (ReMeLight)** 库处理文档摄取，通过环境变量配置。

```
┌─────────────────────────────────────────────────────────────┐
│                    CoPaw Document Pipeline                  │
│                                                             │
│  ┌─────────────────┐      ┌──────────────────────┐        │
│  │   MemoryManager │─────▶│   ReMe Library       │        │
│  │                 │      │   (External)          │        │
│  │ - search()      │      │                      │        │
│  │ - add_memory()  │      │ - Vector Store        │        │
│  │ - compact()     │      │ - FTS Search          │        │
│  └─────────────────┘      │ - Embedding           │        │
│                           └──────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

**配置驱动架构**

```python
# CoPaw 文档摄取配置（环境变量）
EMBEDDING_API_KEY          # API key for embedding service
EMBEDDING_BASE_URL         # Default: dashscope (阿里)
EMBEDDING_MODEL_NAME       # 模型名称
EMBEDDING_DIMENSIONS       # Default: 1024
EMBEDDING_CACHE_ENABLED    # Default: true
EMBEDDING_MAX_CACHE_SIZE   # Default: 2000
EMBEDDING_MAX_INPUT_LENGTH # Default: 8192
EMBEDDING_MAX_BATCH_SIZE   # Default: 10
FTS_ENABLED                # 全文搜索 (default: true)
MEMORY_STORE_BACKEND       # auto/local/chroma
```

**Memory Manager 实现**

```python
# src/copaw/agents/memory/memory_manager.py
class MemoryManager:
    """Memory management wrapper around ReMe library."""

    def __init__(self):
        self.vector_enabled = self._check_vector_config()
        self.memory_backend = self._select_backend()
        self.embedding_cache = {}

    def _check_vector_config(self) -> bool:
        """Check if vector search is configured."""
        return bool(
            os.getenv("EMBEDDING_API_KEY") and
            os.getenv("EMBEDDING_MODEL_NAME")
        )

    def _select_backend(self) -> str:
        """Select storage backend based on platform."""
        if platform.system() == "Windows":
            return "local"  # Windows 使用本地文件存储
        return "chroma"   # 其他使用 ChromaDB

    async def add_memory(
        self,
        content: str,
        metadata: Dict[str, Any] = None,
    ) -> bool:
        """Add content to memory with optional embedding."""
        return await self._reme.add(
            content=content,
            metadata=metadata or {},
            vectorize=self.vector_enabled,
        )

    async def search(
        self,
        query: str,
        top_k: int = 5,
        use_vector: bool = True,
    ) -> List[MemoryResult]:
        """Search memory with hybrid approach."""
        if use_vector and self.vector_enabled:
            # 向量 + 全文混合搜索
            return await self._reme.hybrid_search(
                query=query,
                top_k=top_k,
                vector_weight=0.7,
                text_weight=0.3,
            )
        else:
            # 仅全文搜索
            return await self._reme.fts_search(
                query=query,
                top_k=top_k,
            )
```

**核心特点**
- Token-based 分块 (~400 tokens/块, 20% 重叠)
- ChromaDB 向量存储 + 混合搜索
- 多模态输入支持 (通过 AgentScope)
- Windows 降级为本地文件存储

---

### 2. NanoClaw - 极简文件记忆

**架构定位**

NanoClaw 采用极简主义方案，使用文件系统直接存储，无向量数据库。

```
┌─────────────────────────────────────────────────────────────┐
│                    NanoClaw Memory System                   │
│                                                             │
│  ┌─────────────────┐      ┌──────────────────────┐        │
│  │   CLAUDE.md     │      │   better-sqlite3     │        │
│  │   (Memory File) │      │   (Messages)          │        │
│  │                 │      │                      │        │
│  │ - System prompt │      │ - Chat history       │        │
│  │ - User context  │      │ - Media metadata     │        │
│  │ - Instructions  │      │ - Timestamps         │        │
│  └─────────────────┘      └──────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

**分块策略**

```typescript
// NanoClaw 使用会话级分块
// groups/<groupJid>/CLAUDE.md - 按会话组织
interface MemorySession {
  sessionId: string;
  claudeMdPath: string;  // 会话专属记忆文件
  messages: Message[];   // SQLite 存储
}

// 无复杂分块，直接追加到 CLAUDE.md
async function appendToMemory(content: string): Promise<void> {
  const memoryPath = getSessionMemoryPath();
  await fs.appendFile(memoryPath, `\n${content}`);
}
```

**核心特点**
- 无向量存储，纯文件系统
- CLAUDE.md 热重载机制
- SQLite 仅用于消息持久化
- 适合个人轻量级使用

---

### 3. IronClaw - 企业级完整流程

**架构定位**

IronClaw 拥有最完整的文档摄取流程，PostgreSQL + pgvector + FTS，支持多 embedding 提供商。

```
┌─────────────────────────────────────────────────────────────┐
│                    IronClaw Ingestion Pipeline              │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Parse     │  │    Chunk    │  │     Embed           │ │
│  │             │  │             │  │                     │ │
│  │ - Markdown  │  │ - Semantic  │  │ - OpenAI            │ │
│  │ - PDF       │  │ - Token-based│  │ - Anthropic         │ │
│  │ - Images    │  │ - Code-aware│  │ - FastEmbed         │ │
│  └──────┬──────┘  └──────┬──────┘  │ - Ollama            │ │
│         │                │         └─────────────────────┘ │
│         └────────────────┼─────────────────────────────────┘
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Index (PostgreSQL + pgvector)            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │   │
│  │  │ documents   │  │ vectors     │  │ fts_index    │ │   │
│  │  │ (metadata)  │  │ (pgvector)  │  │ (GIN/GiST)   │ │   │
│  │  └─────────────┘  └─────────────┘  └──────────────┘ │   │
│  │                                                      │   │
│  │  Hybrid Search: Vector + FTS + RRF Fusion         │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Workspace 记忆系统**

```rust
// IronClaw 的 workspace/memory.rs
pub struct WorkspaceMemory {
    db: Arc<Database>,
    embedding_provider: Arc<dyn EmbeddingProvider>,
    config: MemoryConfig,
}

impl WorkspaceMemory {
    /// 添加文档到记忆
    pub async fn ingest_document(
        &self,
        path: &Path,
        content: &str,
    ) -> Result<(), MemoryError> {
        // 1. 分块
        let chunks = self.chunker.chunk(content, &self.config.chunking);

        // 2. 生成 embeddings
        let embeddings = self.embedding_provider
            .embed_batch(chunks.iter().map(|c| c.text.clone()).collect())
            .await?;

        // 3. 存储到 PostgreSQL
        for (chunk, embedding) in chunks.iter().zip(embeddings.iter()) {
            self.db.execute(
                "INSERT INTO memory_chunks (path, text, embedding, metadata)
                 VALUES ($1, $2, $3, $4)
                 ON CONFLICT (chunk_hash) DO UPDATE SET updated_at = NOW()",
                &[&path.to_string_lossy(), &chunk.text, embedding, &chunk.metadata],
            ).await?;
        }

        Ok(())
    }

    /// 混合搜索
    pub async fn hybrid_search(
        &self,
        query: &str,
        top_k: usize,
    ) -> Result<Vec<SearchResult>, MemoryError> {
        // 1. 向量搜索
        let query_embedding = self.embedding_provider.embed(query).await?;
        let vector_results = self.vector_search(&query_embedding, top_k * 2).await?;

        // 2. 全文搜索
        let fts_results = self.fts_search(query, top_k * 2).await?;

        // 3. RRF 融合
        let fused = self.reciprocal_rank_fusion(&[vector_results, fts_results]);

        Ok(fused.into_iter().take(top_k).collect())
    }
}
```

**混合搜索实现 (RRF)**

```rust
// Reciprocal Rank Fusion
fn reciprocal_rank_fusion(
    &self,
    result_sets: &[Vec<SearchResult>],
) -> Vec<SearchResult> {
    const K: f64 = 60.0;  // RRF 常数
    let mut scores: HashMap<String, f64> = HashMap::new();

    for results in result_sets {
        for (rank, result) in results.iter().enumerate() {
            let score = 1.0 / (K + rank as f64 + 1.0);
            scores.entry(result.id.clone())
                .and_modify(|s| *s += score)
                .or_insert(score);
        }
    }

    // 按融合分数排序
    let mut fused: Vec<_> = scores.into_iter().collect();
    fused.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

    fused.into_iter()
        .map(|(id, _)| self.get_result_by_id(&id))
        .collect()
}
```

**Embedding 提供商抽象**

```rust
pub trait EmbeddingProvider: Send + Sync {
    async fn embed(&self, text: &str) -> Result<Vec<f32>, EmbeddingError>;
    async fn embed_batch(&self, texts: Vec<String>) -> Result<Vec<Vec<f32>>, EmbeddingError>;
    fn dimensions(&self) -> usize;
}

// 实现示例：FastEmbed (本地)
pub struct FastEmbedProvider {
    model: FastEmbedModel,
}

impl EmbeddingProvider for FastEmbedProvider {
    async fn embed(&self, text: &str) -> Result<Vec<f32>, EmbeddingError> {
        self.model.embed(text)
    }
    // ...
}

// 实现示例：OpenAI
pub struct OpenAIEmbedProvider {
    client: OpenAIClient,
    model: String,
}

impl EmbeddingProvider for OpenAIEmbedProvider {
    async fn embed(&self, text: &str) -> Result<Vec<f32>, EmbeddingError> {
        self.client.embeddings().create(
            CreateEmbeddingRequest {
                model: self.model.clone(),
                input: text.into(),
                ..Default::default()
            }
        ).await.map(|r| r.data[0].embedding.clone())
    }
}
```

**核心特点**
- PostgreSQL + pgvector 企业级存储
- 真正混合搜索：Vector + FTS + RRF 融合
- FastEmbed 本地 embedding 支持
- 增量同步 + 文件监听

---

### 4. PicoClaw - Go 轻量实现

**架构定位**

PicoClaw 采用 Go 语言实现，JSONL 格式存储，追求极致轻量。

```go
// PicoClaw 的记忆存储结构
package memory

type MemoryEntry struct {
    ID        string    `json:"id"`
    Content   string    `json:"content"`
    Category  string    `json:"category"`
    Timestamp time.Time `json:"timestamp"`
    Tokens    int       `json:"tokens"`  // CJK-aware 估算
}

// CJK-aware token 估算 (与 context-management 中相同)
func EstimateTokens(text string) int {
    cjkCount := 0
    totalChars := len([]rune(text))

    for _, r := range text {
        if isCJK(r) {
            cjkCount++
        }
    }

    // CJK 字符计 1 token，其他按 1/4 估算
    return cjkCount + (totalChars - cjkCount) / 4
}

func isCJK(r rune) bool {
    return (r >= '\u4e00' && r <= '\u9fff') ||  // CJK Unified
           (r >= '\u3040' && r <= '\u309f') ||  // Hiragana
           (r >= '\u30a0' && r <= '\u30ff')    // Katakana
}
```

**存储实现**

```go
// JSONL 追加写入
func (m *MemoryStore) Append(entry MemoryEntry) error {
    line, err := json.Marshal(entry)
    if err != nil {
        return err
    }

    f, err := os.OpenFile(m.filePath, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer f.Close()

    _, err = f.WriteString(string(line) + "\n")
    return err
}

// 简单全文搜索 (无索引)
func (m *MemoryStore) Search(query string) ([]MemoryEntry, error) {
    file, err := os.Open(m.filePath)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    var results []MemoryEntry
    scanner := bufio.NewScanner(file)

    for scanner.Scan() {
        var entry MemoryEntry
        if err := json.Unmarshal(scanner.Bytes(), &entry); err != nil {
            continue
        }
        // 简单字符串匹配
        if strings.Contains(strings.ToLower(entry.Content), strings.ToLower(query)) {
            results = append(results, entry)
        }
    }

    return results, scanner.Err()
}
```

**核心特点**
- JSONL 顺序存储，无数据库依赖
- CJK-aware token 估算
- 简单字符串搜索 (无向量)
- 适合边缘部署

---

### 5. ZeroClaw - Rust 模块化架构

**架构定位**

ZeroClaw 使用 Rust trait 驱动的模块化架构，支持多种存储后端。

```rust
// src/memory/traits.rs
pub trait Memory: Send + Sync {
    async fn store(&self, category: &str, content: &str, metadata: Metadata) -> Result<MemoryId>;
    async fn search(&self, query: &str, category: Option<&str>, limit: usize) -> Result<Vec<MemoryEntry>>;
    async fn compact(&self, strategy: CompactionStrategy) -> Result<()>;
    async fn health_check(&self) -> HealthStatus;
}

pub trait EmbeddingProvider: Send + Sync {
    async fn embed(&self, texts: &[&str]) -> Result<Vec<Embedding>>;
    fn dimensions(&self) -> usize;
}

// 存储后端枚举
pub enum MemoryBackend {
    InMemory(InMemoryStore),    // 内存 (测试用)
    Sqlite(SqliteStore),        // SQLite
    External(Arc<dyn VectorStore>), // 外部向量存储
}
```

**分类记忆系统**

```rust
// ZeroClaw 的记忆分类
pub enum MemoryCategory {
    Core,          // 系统核心记忆
    Daily,         // 每日摘要
    Conversation,  // 对话历史
    Custom(String), // 用户自定义
}

impl Memory for ZeroClawMemory {
    async fn store(
        &self,
        category: &str,
        content: &str,
        metadata: Metadata,
    ) -> Result<MemoryId> {
        // 按分类存储，支持不同压缩策略
        let strategy = match category {
            "core" => RetentionStrategy::Permanent,
            "daily" => RetentionStrategy::ExpireAfter(Duration::days(30)),
            "conversation" => RetentionStrategy::CompactAfter(100),
            _ => RetentionStrategy::Default,
        };

        self.backend.store(category, content, metadata, strategy).await
    }
}
```

**核心特点**
- Trait 驱动的可扩展架构
- 多存储后端支持 (内存/SQLite/外部)
- 分类记忆 + 生命周期管理
- 适合需要灵活性的场景

---

### 6. MimicClaw - 嵌入式 ESP32 方案

**架构定位**

MimicClaw 针对 ESP32-S3 嵌入式设备设计，极度资源受限环境下的方案。

```cpp
// MimicClaw 的 SPIFFS 存储 (embedded C)
#include "memory.h"
#include <SPIFFS.h>

#define MAX_CHUNK_SIZE 512      // 最大块 512 bytes
#define MAX_MEMORY_ENTRIES 100  // 最多 100 条记忆

struct MemoryEntry {
    char id[32];
    char content[MAX_CHUNK_SIZE];
    uint32_t timestamp;
    uint8_t category;  // 0=SOUL, 1=USER, 2=MEMORY
};

class EmbeddedMemory {
private:
    const char* soulPath = "/spiffs/SOUL.md";
    const char* userPath = "/spiffs/USER.md";
    const char* memoryPath = "/spiffs/MEMORY.md";

public:
    bool init() {
        if (!SPIFFS.begin(true)) {
            return false;
        }
        // 确保文件存在
        ensureFile(soulPath);
        ensureFile(userPath);
        ensureFile(memoryPath);
        return true;
    }

    bool appendMemory(const char* content) {
        File f = SPIFFS.open(memoryPath, FILE_APPEND);
        if (!f) return false;

        // 截断到最大大小
        size_t len = strlen(content);
        if (len > MAX_CHUNK_SIZE - 1) {
            len = MAX_CHUNK_SIZE - 1;
        }

        // 添加时间戳
        char entry[MAX_CHUNK_SIZE + 64];
        snprintf(entry, sizeof(entry), "[%lu] %.*s\n",
                 millis(), (int)len, content);

        bool ok = f.print(entry) > 0;
        f.close();
        return ok;
    }

    // 简单搜索 (线性扫描)
    void search(const char* query, char* result, size_t resultSize) {
        File f = SPIFFS.open(memoryPath, FILE_READ);
        if (!f) {
            strncpy(result, "Error opening memory", resultSize);
            return;
        }

        // 简单的字符串匹配
        char line[MAX_CHUNK_SIZE];
        while (f.available() && strlen(result) < resultSize - MAX_CHUNK_SIZE) {
            size_t len = f.readBytesUntil('\n', line, sizeof(line));
            line[len] = '\0';

            if (strstr(line, query) != NULL) {
                strcat(result, line);
                strcat(result, "\n");
            }
        }
        f.close();
    }
};
```

**核心特点**
- SPIFFS 文件系统 (Flash 存储)
- 512 bytes 固定块大小
- 无线性搜索，无向量
- WiFi 连接云端获取 embeddings

---

### 7. OpenClaw - TypeScript 完整实现

**架构定位**

OpenClaw 拥有 Personal Assistants 中最完整的 TypeScript 实现，LanceDB + sqlite-vec。

```typescript
// OpenClaw 的文档摄取流程 (已在上文展示，此处补充关键实现)
// src/memory/internal.ts

// 文件发现
export async function listMemoryFiles(
  workspaceDir: string,
  extraPaths?: string[],
  multimodal?: MemoryMultimodalSettings,
): Promise<string[]> {
  const files: string[] = [];

  // 1. 扫描 memory/ 目录
  const memoryDir = path.join(workspaceDir, "memory");
  if (await fs.pathExists(memoryDir)) {
    const mdFiles = await glob("**/*.md", { cwd: memoryDir });
    files.push(...mdFiles.map((f) => path.join(memoryDir, f)));
  }

  // 2. 扫描 MEMORY.md
  const memoryFile = path.join(workspaceDir, "MEMORY.md");
  if (await fs.pathExists(memoryFile)) {
    files.push(memoryFile);
  }

  // 3. 多模态文件支持
  if (multimodal?.enabled) {
    const imageExtensions = ["png", "jpg", "jpeg", "gif", "webp"];
    for (const ext of imageExtensions) {
      const images = await glob(`**/*.${ext}`, { cwd: workspaceDir });
      files.push(...images.map((f) => path.join(workspaceDir, f)));
    }
  }

  return files;
}
```

**增量同步策略**

```typescript
// src/memory/manager-sync-ops.ts
export class MemorySyncManager {
  async sync(options?: { reason?: string; force?: boolean }): Promise<void> {
    if (this.isSyncing && !options?.force) {
      return;
    }

    this.isSyncing = true;
    try {
      // 1. 扫描文件变化
      const fileChanges = await this.detectFileChanges();

      // 2. 处理删除
      for (const deleted of fileChanges.deleted) {
        await this.removeFileFromIndex(deleted);
      }

      // 3. 处理新增/修改
      for (const file of [...fileChanges.added, ...fileChanges.modified]) {
        await this.indexFile(file);
      }
    } finally {
      this.isSyncing = false;
    }
  }

  private async detectFileChanges(): Promise<FileChanges> {
    const currentFiles = await listMemoryFiles(this.workspaceDir);
    const indexedFiles = await this.db.query<MemoryFile>("SELECT * FROM files");

    // 检测新增、修改、删除
    // ...
  }
}
```

**核心特点**
- LanceDB 本地向量存储
- SQLite FTS5 全文搜索
- chokidar 文件监听
- 多模态支持 (图片)

---

### 8. Nanobot - Python 双级记忆

**架构定位**

Nanobot 采用双级记忆系统：MEMORY.md (长期) + HISTORY.md (归档)。

```python
# Nanobot 的记忆系统
class DualTierMemory:
    """
    两级记忆系统：
    - MEMORY.md: 进入系统 prompt，常驻
    - HISTORY.md: 归档历史，需检索
    """

    def __init__(self, workspace_dir: str):
        self.memory_path = Path(workspace_dir) / "MEMORY.md"
        self.history_path = Path(workspace_dir) / "HISTORY.md"
        self.memory_content = self._load_memory()

    def _load_memory(self) -> str:
        """加载 MEMORY.md 内容"""
        if self.memory_path.exists():
            return self.memory_path.read_text()
        return ""

    def add_to_memory(self, content: str, tier: str = "memory"):
        """添加到记忆"""
        if tier == "memory":
            # 直接追加到 MEMORY.md
            with open(self.memory_path, "a") as f:
                f.write(f"\n## {datetime.now().isoformat()}\n")
                f.write(content)
            # 重新加载
            self.memory_content = self._load_memory()
        else:
            # 归档到 HISTORY.md
            with open(self.history_path, "a") as f:
                f.write(f"\n## {datetime.now().isoformat()}\n")
                f.write(content)

    def search_history(self, query: str, top_k: int = 5) -> List[str]:
        """搜索历史 (简单字符串匹配)"""
        if not self.history_path.exists():
            return []

        content = self.history_path.read_text()
        entries = content.split("\n## ")

        # 简单相关性排序
        results = []
        for entry in entries:
            score = self._relevance_score(entry, query)
            results.append((score, entry))

        results.sort(reverse=True)
        return [r[1] for r in results[:top_k]]

    def _relevance_score(self, entry: str, query: str) -> float:
        """简单 TF-IDF 风格评分"""
        query_words = set(query.lower().split())
        entry_words = entry.lower().split()

        if not entry_words:
            return 0.0

        matches = sum(1 for w in entry_words if w in query_words)
        return matches / len(entry_words)
```

**核心特点**
- 双级记忆设计
- MEMORY.md 直接进入系统 prompt
- 简单 TF-IDF 历史检索
- 无向量数据库

---

## Programming Agents 分析

### 9. Aider - RepoMap 智能索引

**架构定位**

Aider 开创性地使用 **RepoMap** 技术，通过 Tree-sitter 提取代码结构，构建代码库地图。

```
┌─────────────────────────────────────────────────────────────┐
│                    Aider RepoMap System                     │
│                                                             │
│  ┌─────────────────┐      ┌──────────────────────┐        │
│  │   Tree-sitter   │─────▶│   Tag Extraction     │        │
│  │   Parsing       │      │   (AST traversal)     │        │
│  │                 │      │                      │        │
│  │ - Functions     │      │ - Function names      │        │
│  │ - Classes       │      │ - Class names         │        │
│  │ - Comments      │      │ - Comment headers     │        │
│  └─────────────────┘      └──────────────────────┘        │
│            │                          │                     │
│            ▼                          ▼                     │
│  ┌──────────────────────────────────────────────┐          │
│  │              RepoMap Builder                  │          │
│  │  (Binary search to fit token limit)           │          │
│  │                                               │          │
│  │  ┌─────────────┐  ┌─────────────┐            │          │
│  │  │ Tags Cache  │  │ Tree String │            │          │
│  │  │ (SQLite)    │  │ (Hierarchical)│           │          │
│  │  └─────────────┘  └─────────────┘            │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

**RepoMap 实现**

```python
# aider/repomap.py
from grep_ast import TreeContext  # Tree-sitter wrapper

class RepoMap:
    """
    代码库地图：提取代码结构，构建可放入 context window 的树状表示
    """

    def __init__(self, root_dir: str, token_limit: int = 8000):
        self.root_dir = root_dir
        self.token_limit = token_limit
        self.cache = SQLiteCache()  # SQLite 缓存 tags

    def get_repo_map(self, chat_files: List[str], other_files: List[str]) -> str:
        """
        生成代码库地图，优先包含 chat_files，尽可能包含 other_files
        """
        # 1. 收集所有 tags
        all_tags = []
        for file in chat_files + other_files:
            tags = self.get_tags(file)
            all_tags.extend(tags)

        # 2. 按重要性排序
        all_tags.sort(key=lambda t: t.importance, reverse=True)

        # 3. 二分搜索找到最佳 token 限制
        best_map = self._binary_search_tree(all_tags, self.token_limit)

        return best_map

    def get_tags(self, filepath: str) -> List[Tag]:
        """从文件提取 tags"""
        # 检查缓存
        cached = self.cache.get(filepath)
        if cached:
            return cached

        # Tree-sitter 解析
        content = Path(filepath).read_text()
        tree = self.parser.parse(content.encode())

        tags = []
        for node in tree.root_node.children:
            if node.type == 'function_definition':
                name = self._get_node_text(node, content)
                tags.append(Tag(
                    name=name,
                    line=node.start_point[0],
                    type='function',
                    importance=self._calc_importance(node)
                ))
            elif node.type == 'class_definition':
                name = self._get_node_text(node, content)
                tags.append(Tag(
                    name=name,
                    line=node.start_point[0],
                    type='class',
                    importance=self._calc_importance(node) * 1.5  # 类更重要
                ))

        # 缓存结果
        self.cache.set(filepath, tags)
        return tags

    def _binary_search_tree(self, tags: List[Tag], max_tokens: int) -> str:
        """
        二分搜索：找到能放入 max_tokens 的最完整的树状表示
        """
        low, high = 0, len(tags)
        best_tree = ""

        while low <= high:
            mid = (low + high) // 2
            selected = tags[:mid]
            tree = self._build_tree_string(selected)
            tokens = self._count_tokens(tree)

            if tokens <= max_tokens:
                best_tree = tree
                low = mid + 1  # 尝试更多
            else:
                high = mid - 1  # 减少

        return best_tree

    def _build_tree_string(self, tags: List[Tag]) -> str:
        """构建树状字符串表示"""
        lines = []
        current_file = None

        for tag in sorted(tags, key=lambda t: (t.file, t.line)):
            if tag.file != current_file:
                lines.append(f"\n{tag.file}:")
                current_file = tag.file
            indent = "  " * tag.depth
            lines.append(f"{indent}{tag.type} {tag.name} ({tag.line})")

        return "\n".join(lines)
```

**Grep 工具集成**

```python
# aider/tools/grep.py
import subprocess

class GrepTool:
    """Ripgrep 集成，用于快速代码搜索"""

    def search(self, pattern: str, paths: List[str] = None) -> List[SearchResult]:
        """使用 ripgrep 搜索代码"""
        args = [
            "rg",
            "--line-number",
            "--with-filename",
            "--color=never",
            "-i",  # 不区分大小写
            pattern,
        ]
        if paths:
            args.extend(paths)
        else:
            args.append(".")

        result = subprocess.run(args, capture_output=True, text=True)

        # 解析结果
        matches = []
        for line in result.stdout.split("\n"):
            if ":" in line:
                parts = line.split(":", 2)
                if len(parts) == 3:
                    matches.append(SearchResult(
                        file=parts[0],
                        line=int(parts[1]),
                        content=parts[2]
                    ))

        return matches
```

**文件监听 (watch.py)**

```python
# aider/watch.py
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class FileWatcher(FileSystemEventHandler):
    """IDE 集成的文件监听"""

    def __init__(self, callback: Callable):
        self.callback = callback
        self.observer = Observer()

    def start(self, paths: List[str]):
        for path in paths:
            self.observer.schedule(self, path, recursive=True)
        self.observer.start()

    def on_modified(self, event):
        if not event.is_directory:
            self.callback(event.src_path, "modified")

    def on_created(self, event):
        if not event.is_directory:
            self.callback(event.src_path, "created")
```

**核心特点**
- **RepoMap**: 智能代码结构提取
- Tree-sitter 多语言解析
- SQLite 缓存 tags
- 二分搜索优化 token 使用
- 文件监听 + Git 集成

---

### 10. Codex - Rust 多 crate 架构

**架构定位**

Codex CLI 使用 Rust 70+ crates 的复杂架构，沙箱 + 多模支持。

```rust
// codex-rs 的文件处理
// crates/codex/src/protocol.rs

/// 文件操作协议
pub enum FileOperation {
    Read {
        path: PathBuf,
        offset: Option<usize>,
        limit: Option<usize>,
    },
    Write {
        path: PathBuf,
        content: String,
    },
    Search {
        pattern: String,
        paths: Vec<PathBuf>,
    },
}

/// 上下文管理
pub struct ContextManager {
    /// 已加载的文件内容
    loaded_files: HashMap<PathBuf, String>,
    /// 沙箱路径限制
    sandbox_paths: Vec<PathBuf>,
}

impl ContextManager {
    pub async fn read_file(&self, path: &Path, offset: usize, limit: usize) -> Result<String> {
        // 沙箱安全检查
        if !self.is_path_allowed(path) {
            return Err(Error::SandboxViolation);
        }

        let content = tokio::fs::read_to_string(path).await?;
        let lines: Vec<_> = content.lines().collect();

        // 应用 offset/limit
        let start = offset.min(lines.len());
        let end = (offset + limit).min(lines.len());

        Ok(lines[start..end].join("\n"))
    }
}
```

**核心特点**
- Rust 70+ crates 模块化架构
- 沙箱文件访问控制
- Session-based 上下文管理
- 无持久向量存储

---

### 11. Qwen-Code - MCP 上下文引擎

**架构定位**

Qwen-Code 通过 **Model Context Protocol (MCP)** 注入代码上下文。

```typescript
// Qwen-Code MCP 上下文处理
// src/core/context/mcp-context.ts

interface MCPContextPayload {
  files: Array<{
    path: string;
    content: string;
    language?: string;
  }>;
  selection?: {
    path: string;
    startLine: number;
    endLine: number;
    content: string;
  };
  diagnostics?: Array<{
    file: string;
    line: number;
    message: string;
    severity: 'error' | 'warning';
  }>;
}

class MCPContextEngine {
  /// 构建 MCP 上下文 payload
  async buildContext(
    filePaths: string[],
    cursorPosition?: Position,
  ): Promise<MCPContextPayload> {
    const files: MCPContextPayload['files'] = [];

    for (const path of filePaths) {
      const content = await fs.readFile(path, 'utf-8');
      const language = this.detectLanguage(path);

      // 智能截断大文件
      const truncated = this.truncateFile(content, MAX_FILE_SIZE);

      files.push({ path, content: truncated, language });
    }

    // 获取当前选区
    const selection = cursorPosition
      ? await this.getSelection(cursorPosition)
      : undefined;

    // 获取 LSP 诊断信息
    const diagnostics = await this.lspClient.getDiagnostics();

    return { files, selection, diagnostics };
  }

  /// 智能截断：保留函数签名和结构
  private truncateFile(content: string, maxSize: number): string {
    if (content.length <= maxSize) return content;

    // 提取文件头部 (imports, 全局声明)
    const headerMatch = content.match(/^(.*?)(?=\n(?:function|class|const|let))/s);
    const header = headerMatch ? headerMatch[1] : '';

    // 提取函数/类签名
    const signatures = content.match(/(?:function|class|interface)\s+\w+[^;{]*[;{]/g) || [];

    return [
      header.slice(0, maxSize * 0.2),
      '\n... [truncated] ...\n',
      signatures.join('\n').slice(0, maxSize * 0.7),
      '\n... [truncated] ...\n',
    ].join('');
  }
}
```

**Ripgrep 搜索工具**

```typescript
// src/tools/search.ts
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

export class SearchTool {
  async grep(
    pattern: string,
    options: {
      paths?: string[];
      include?: string[];
      exclude?: string[];
    } = {},
  ): Promise<SearchResult[]> {
    const args = [
      'rg',
      '--line-number',
      '--with-filename',
      '--color=never',
      '-i', // 不区分大小写
      pattern,
      ...(options.paths || ['.']),
    ];

    if (options.include) {
      for (const glob of options.include) {
        args.push('--include', glob);
      }
    }

    if (options.exclude) {
      for (const glob of options.exclude) {
        args.push('--exclude', glob);
      }
    }

    const { stdout } = await execAsync(args.join(' '));

    return stdout
      .split('\n')
      .filter(Boolean)
      .map((line) => {
        const match = line.match(/^([^:]+):(\d+):(.*)$/);
        if (!match) return null;
        return {
          file: match[1],
          line: parseInt(match[2], 10),
          content: match[3],
        };
      })
      .filter(Boolean) as SearchResult[];
  }
}
```

**核心特点**
- MCP 协议上下文注入
- 智能文件截断 (保留签名)
- Ripgrep 快速搜索
- LSP 诊断信息集成

---

### 12. Gemini-CLI - 上下文缓存与多模态

**架构定位**

Gemini CLI 利用 Gemini API 的**上下文缓存 (Context Caching)** 和原生多模态支持。

```typescript
// Gemini CLI 上下文缓存
// src/context/cache.ts

import { GoogleGenAI } from '@google/genai';

interface ContextCache {
  cacheId: string;
  content: Content[];
  expiresAt: Date;
  tokenCount: number;
}

class GeminiContextCache {
  private genAI: GoogleGenAI;
  private activeCaches: Map<string, ContextCache> = new Map();

  /// 创建上下文缓存
  async createCache(
    files: Array<{ path: string; content: string }>,
    ttlMinutes: number = 60,
  ): Promise<string> {
    const contents: Content[] = [];

    for (const file of files) {
      // 检测文件类型
      if (this.isImage(file.path)) {
        // 多模态：图片内容
        contents.push({
          role: 'user',
          parts: [{
            fileData: {
              mimeType: this.getMimeType(file.path),
              fileUri: await this.uploadFile(file.path),
            },
          }],
        });
      } else {
        // 文本内容
        contents.push({
          role: 'user',
          parts: [{ text: `File: ${file.path}\n${file.content}` }],
        });
      }
    }

    // 创建 Gemini 缓存
    const cache = await this.genAI.caches.create({
      model: 'gemini-1.5-pro',
      contents,
      ttl: { seconds: ttlMinutes * 60 },
    });

    this.activeCaches.set(cache.name, {
      cacheId: cache.name,
      content: contents,
      expiresAt: new Date(Date.now() + ttlMinutes * 60 * 1000),
      tokenCount: cache.usageMetadata?.totalTokenCount || 0,
    });

    return cache.name;
  }

  /// 使用缓存进行对话
  async chatWithCache(
    cacheId: string,
    prompt: string,
  ): Promise<string> {
    const cache = this.activeCaches.get(cacheId);
    if (!cache) {
      throw new Error('Cache not found');
    }

    // 使用缓存引用
    const response = await this.genAI.models.generateContent({
      model: 'gemini-1.5-pro',
      contents: [{ role: 'user', parts: [{ text: prompt }] }],
      config: {
        cachedContent: cacheId,
      },
    });

    return response.text || '';
  }
}
```

**GEMINI.md 支持**

```typescript
// GEMINI.md 上下文文件
// src/context/gemini-md.ts

/// 自动读取 GEMINI.md 作为系统上下文
async function loadGeminiMd(workspaceDir: string): Promise<string | null> {
  const geminiMdPath = path.join(workspaceDir, 'GEMINI.md');

  try {
    const content = await fs.readFile(geminiMdPath, 'utf-8');
    return content;
  } catch {
    return null;
  }
}

/// 构建 prompts 时注入
export async function buildSystemPrompt(workspaceDir: string): Promise<string> {
  const basePrompt = `You are Gemini CLI...`;
  const geminiMd = await loadGeminiMd(workspaceDir);

  if (geminiMd) {
    return `${basePrompt}\n\n## Project Context\n${geminiMd}`;
  }

  return basePrompt;
}
```

**核心特点**
- Gemini 原生上下文缓存 (节省 tokens)
- 多模态支持 (图片、PDF)
- GEMINI.md 项目上下文
- 缓存 TTL 管理

---

### 13. Kimi-CLI - 基础会话状态

**架构定位**

Kimi-CLI 采用简单的会话状态持久化，无复杂文档摄取流程。

```python
# kimi_cli/session/state.py
import json
from pathlib import Path
from datetime import datetime
from typing import List, Optional

class SessionState:
    """简单的 JSON 文件会话存储"""

    def __init__(self, session_file: Path):
        self.session_file = session_file
        self.messages: List[Message] = []
        self.metadata: dict = {}
        self.load()

    def load(self) -> None:
        """从 JSON 加载会话"""
        if self.session_file.exists():
            data = json.loads(self.session_file.read_text())
            self.messages = [Message(**m) for m in data.get('messages', [])]
            self.metadata = data.get('metadata', {})

    def save(self) -> None:
        """保存到 JSON"""
        data = {
            'messages': [m.to_dict() for m in self.messages],
            'metadata': {
                **self.metadata,
                'last_saved': datetime.now().isoformat(),
            },
        }
        self.session_file.write_text(json.dumps(data, indent=2))

    def add_message(self, role: str, content: str) -> None:
        """添加消息并自动保存"""
        self.messages.append(Message(role=role, content=content))
        self.save()
```

**文件读取工具**

```python
# kimi_cli/tools/file.py
import aiofiles

class FileReadTool:
    """基础文件读取，无分块"""

    async def execute(
        self,
        path: str,
        offset: int = 0,
        limit: int = 100,
    ) -> str:
        """Read file content directly."""
        async with aiofiles.open(path, "r") as f:
            if offset > 0:
                await f.seek(offset)
            content = await f.read(limit * 100)  # 近似字符数

        return content

    async def search_in_file(self, path: str, pattern: str) -> List[Match]:
        """Simple grep-like search."""
        matches = []
        async with aiofiles.open(path, "r") as f:
            async for line_num, line in enumerate(f, 1):
                if pattern in line:
                    matches.append(Match(
                        line=line_num,
                        content=line.strip(),
                    ))
        return matches
```

**核心特点**
- JSON 文件会话存储
- 基础文件读取 (无分块)
- 简单 grep 搜索
- 无向量/embedding 支持

---

### 14. OpenCode - 代码中心方法

**架构定位**

OpenCode 专注于代码文件直接读取，LSP 支持，Drizzle ORM 管理。

```typescript
// OpenCode 文件读取限制
// packages/opencode/src/tool/read.ts

const DEFAULT_READ_LIMIT = 2000;   // 最大行数
const MAX_LINE_LENGTH = 2000;      // 单行截断
const MAX_BYTES = 50 * 1024;       // 50KB 上限

export async function readFile(
  filePath: string,
  options?: {
    offset?: number;
    limit?: number;
  },
): Promise<ReadResult> {
  const content = await fs.readFile(filePath, "utf-8");

  // 二进制文件检测
  if (isBinary(content)) {
    return {
      error: `File appears to be binary: ${filePath}`,
    };
  }

  const lines = content.split("\n");

  // 应用限制
  const offset = options?.offset || 0;
  const limit = Math.min(options?.limit || DEFAULT_READ_LIMIT, DEFAULT_READ_LIMIT);

  const sliced = lines.slice(offset, offset + limit);

  // 截断超长行
  const truncated = sliced.map((line) =>
    line.length > MAX_LINE_LENGTH
      ? line.slice(0, MAX_LINE_LENGTH) + "... [truncated]"
      : line,
  );

  return {
    content: truncated.join("\n"),
    totalLines: lines.length,
    returnedLines: truncated.length,
    hasMore: offset + limit < lines.length,
  };
}
```

**Grep 搜索**

```typescript
// packages/opencode/src/tool/grep.ts
export async function grep(params: {
  pattern: string;
  paths?: string[];
  include?: string[];
  exclude?: string[];
  caseSensitive?: boolean;
}): Promise<GrepResult[]> {
  const { pattern, paths, include, exclude, caseSensitive } = params;

  // 使用 ripgrep
  const args = [
    "--line-number",
    "--with-filename",
    "--color=never",
    caseSensitive ? "" : "-i",
    pattern,
    ...(paths || ["."]),
  ];

  if (include) {
    for (const glob of include) {
      args.push("--include", glob);
    }
  }

  if (exclude) {
    for (const glob of exclude) {
      args.push("--exclude", glob);
    }
  }

  const { stdout } = await execAsync(`rg ${args.join(" ")}`);

  // 解析结果
  return stdout
    .split("\n")
    .filter(Boolean)
    .map((line) => {
      const match = line.match(/^([^:]+):(\d+):(.*)$/);
      if (match) {
        return {
          file: match[1],
          line: parseInt(match[2], 10),
          content: match[3],
        };
      }
      return null;
    })
    .filter(Boolean) as GrepResult[];
}
```

**数据库存储**

```typescript
// OpenCode 使用 Drizzle ORM + SQLite
// packages/opencode/src/db/schema.ts

import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';

export const conversations = sqliteTable('conversations', {
  id: integer('id').primaryKey(),
  sessionId: text('session_id').notNull(),
  role: text('role').notNull(), // 'user' | 'assistant' | 'system'
  content: text('content').notNull(),
  toolCalls: text('tool_calls', { mode: 'json' }), // JSON
  timestamp: integer('timestamp', { mode: 'timestamp' }).notNull(),
});

export const toolExecutions = sqliteTable('tool_executions', {
  id: integer('id').primaryKey(),
  toolName: text('tool_name').notNull(),
  input: text('input').notNull(),
  output: text('output'),
  error: text('error'),
  durationMs: integer('duration_ms'),
  timestamp: integer('timestamp', { mode: 'timestamp' }).notNull(),
});
```

**核心特点**
- 代码中心文件读取 (2000 行限制)
- LSP 语言服务器支持
- Drizzle ORM + SQLite
- Ripgrep 搜索
- chokidar 文件监听

---

### 15. Pi-Mono - 图结构代码分析

**架构定位**

Pi-Mono 使用图结构分析代码依赖关系，构建代码图谱。

```typescript
// Pi-Mono 的图结构代码分析
// src/analysis/graph.ts

interface CodeNode {
  id: string;
  type: 'file' | 'function' | 'class' | 'variable';
  name: string;
  file: string;
  line: number;
  column: number;
}

interface CodeEdge {
  source: string;  // node id
  target: string;  // node id
  type: 'calls' | 'imports' | 'extends' | 'references';
}

class CodeGraph {
  private nodes: Map<string, CodeNode> = new Map();
  private edges: CodeEdge[] = [];
  private adjacencyList: Map<string, string[]> = new Map();

  /// 添加节点
  addNode(node: CodeNode): void {
    this.nodes.set(node.id, node);
    if (!this.adjacencyList.has(node.id)) {
      this.adjacencyList.set(node.id, []);
    }
  }

  /// 添加边
  addEdge(edge: CodeEdge): void {
    this.edges.push(edge);
    this.adjacencyList.get(edge.source)?.push(edge.target);
  }

  /// 获取调用图
  getCallGraph(functionName: string, depth: number = 3): CodeNode[] {
    const startNode = this.findNodeByName(functionName);
    if (!startNode) return [];

    const visited = new Set<string>();
    const result: CodeNode[] = [];
    const queue: Array<{ nodeId: string; level: number }> = [
      { nodeId: startNode.id, level: 0 },
    ];

    while (queue.length > 0) {
      const { nodeId, level } = queue.shift()!;
      if (visited.has(nodeId) || level > depth) continue;

      visited.add(nodeId);
      const node = this.nodes.get(nodeId);
      if (node) result.push(node);

      // 找到所有调用的函数
      const neighbors = this.adjacencyList.get(nodeId) || [];
      for (const neighbor of neighbors) {
        if (!visited.has(neighbor)) {
          queue.push({ nodeId: neighbor, level: level + 1 });
        }
      }
    }

    return result;
  }

  /// 查找相关文件 (基于图距离)
  findRelatedFiles(filePath: string, maxDistance: number = 2): string[] {
    const fileNode = this.nodes.values().find(n => n.file === filePath && n.type === 'file');
    if (!fileNode) return [];

    // BFS 查找相关文件
    const visited = new Set<string>();
    const related = new Set<string>();
    const queue: Array<{ nodeId: string; distance: number }> = [
      { nodeId: fileNode.id, distance: 0 },
    ];

    while (queue.length > 0) {
      const { nodeId, distance } = queue.shift()!;
      if (visited.has(nodeId) || distance > maxDistance) continue;

      visited.add(nodeId);
      const node = this.nodes.get(nodeId);
      if (node && node.file !== filePath) {
        related.add(node.file);
      }

      const neighbors = this.adjacencyList.get(nodeId) || [];
      for (const neighbor of neighbors) {
        queue.push({ nodeId: neighbor, distance: distance + 1 });
      }
    }

    return Array.from(related);
  }
}
```

**核心特点**
- 图结构代码分析
- 调用图提取
- 相关文件推荐 (基于图距离)
- 无向量存储

---

## 核心组件深度对比

### 文件解析策略对比

| 项目 | Tree-sitter | Markdown | PDF | Images | Code-Aware | Multi-modal |
|------|-------------|----------|-----|--------|------------|-------------|
| **CoPaw** | Partial | ✅ | ❌ | ❌ | ❌ | Limited |
| **IronClaw** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **OpenClaw** | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Aider** | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Qwen Code** | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Gemini CLI** | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ |
| **Codex** | Partial | ✅ | ❌ | ❌ | ✅ | ❌ |
| **OpenCode** | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Kimi CLI** | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |

### 分块策略对比

| 项目 | Fixed-Size | Semantic | Token-Based | Line-Based | Code-Aware | Hierarchical |
|------|------------|----------|-------------|------------|------------|--------------|
| **CoPaw** | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **IronClaw** | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **OpenClaw** | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **Aider** | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **Qwen Code** | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Gemini CLI** | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Codex** | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ |
| **OpenCode** | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ |
| **Kimi CLI** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

### Embedding 提供商对比

| 项目 | OpenAI | Anthropic | Local | Ollama | FastEmbed | Multi-Provider |
|------|--------|-----------|-------|--------|-----------|----------------|
| **CoPaw** | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ |
| **IronClaw** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **OpenClaw** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Aider** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| **Codex** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| **Qwen Code** | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| **Gemini CLI** | ❌ | ❌ | ❌ | ❌ | ❌ | Google Only |
| **Kimi CLI** | ❌ | ❌ | ❌ | ❌ | ❌ | Moonshot Only |

### 向量存储对比

| 项目 | ChromaDB | SQLite | PostgreSQL/pgvector | LanceDB | In-Memory | File-Based |
|------|----------|--------|---------------------|---------|-----------|------------|
| **CoPaw** | ✅ | ❌ | ❌ | ❌ | Partial | ❌ |
| **IronClaw** | ❌ | ❌ | ✅ | ❌ | Partial | ❌ |
| **OpenClaw** | ❌ | ✅ | ❌ | ✅ | Partial | ❌ |
| **Aider** | ❌ | ✅ | ❌ | ❌ | Partial | ❌ |
| **Codex** | ❌ | ❌ | ❌ | ❌ | Partial | ❌ |
| **NanoClaw** | ❌ | ✅ | ❌ | ❌ | Partial | ✅ |
| **PicoClaw** | ❌ | ✅ | ❌ | ❌ | Partial | ✅ |

### 全文搜索对比

| 项目 | SQLite FTS5 | PostgreSQL FTS | Ripgrep | Custom | String Search |
|------|-------------|----------------|---------|--------|---------------|
| **CoPaw** | ❌ | ❌ | ❌ | Partial | ✅ |
| **IronClaw** | ❌ | ✅ | ❌ | ❌ | ✅ |
| **OpenClaw** | ✅ | ❌ | ❌ | ❌ | ✅ |
| **Aider** | ❌ | ❌ | ✅ | ❌ | ✅ |
| **Qwen Code** | ❌ | ❌ | ✅ | ❌ | ✅ |
| **NanoClaw** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **PicoClaw** | Partial | ❌ | ❌ | ❌ | ✅ |

### 混合搜索实现

| 项目 | Vector+FTS | Vector+Keyword | RRF Fusion | Weighted Fusion | 未实现 |
|------|------------|----------------|------------|-----------------|--------|
| **IronClaw** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **OpenClaw** | ✅ | Partial | ❌ | Partial | ❌ |
| **CoPaw** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Aider** | Partial | Partial | ❌ | ❌ | Partial |
| **其余 11 项目** | ❌ | ❌ | ❌ | ❌ | ✅ |

### 增量同步/文件监听

| 项目 | File Watcher | IDE Integration | Git Hooks | Real-time |
|------|--------------|-----------------|-----------|-----------|
| **IronClaw** | ✅ | ✅ | Partial | ✅ |
| **OpenClaw** | ✅ | ✅ | ❌ | ✅ |
| **Aider** | ✅ | ✅ | ✅ | ✅ |
| **Qwen Code** | ✅ | ✅ | ❌ | ✅ |
| **Codex** | ✅ | ❌ | ❌ | ✅ |
| **NanoClaw** | ✅ | ❌ | ❌ | ✅ |
| **Kimi CLI** | ❌ | ❌ | ❌ | ❌ |
| **Gemini CLI** | ❌ | ❌ | ❌ | ❌ |

---

## 推荐统一架构

基于对 15 个项目的分析，以下是推荐的统一文档摄取流程架构：

### 高层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Document Ingestion Pipeline                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Input     │───▶│   Parsing   │───▶│  Chunking   │───▶│  Embedding  │
│  Sources    │    │   Engine    │    │   Engine    │    │   Engine    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
      │                   │                   │                   │
      ▼                   ▼                   ▼                   ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ - Files     │    │ - Tree-     │    │ - Semantic  │    │ - OpenAI    │
│ - URLs      │    │   sitter    │    │ - Token-    │    │ - Anthropic │
│ - Git repos │    │ - Markitdown│    │   based     │    │ - FastEmbed │
│ - Databases │    │ - OCR       │    │ - Code-     │    │ - Ollama    │
│ - APIs      │    │ - PDF       │    │   aware     │    │ - Cache     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      Storage Layer (Unified)                     │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Vector     │  │  Full-Text   │  │   Metadata   │          │
│  │   Store      │  │    Search    │  │    Store     │          │
│  │  (LanceDB/   │  │   (SQLite    │  │  (SQLite/    │          │
│  │   pgvector)  │  │    FTS5)     │  │   PostgreSQL)│          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                    ┌──────────────┐                             │
│                    │ Hybrid Search│                             │
│                    │  (RRF Fusion)│                             │
│                    └──────────────┘                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     Retrieval & Query Layer                      │
├─────────────────────────────────────────────────────────────────┤
│  - Semantic Search    - Keyword Search    - Hybrid Search       │
│  - Reranking          - Caching           - MMR Diversity       │
│  - Temporal Decay     - Filters           - Incremental Sync    │
└─────────────────────────────────────────────────────────────────┘
```

### 推荐实现模式

#### 1. 文件解析策略
**推荐**：多引擎解析器，自动检测
- **Tree-sitter** (Aider/OpenClaw)：代码文件的 AST 感知解析
- **Markitdown** (IronClaw)：PDF、Word、Excel 文档处理
- **OCR** (Gemini CLI)：图片文字提取
- **自定义解析器**：JSON、XML、YAML 结构化数据

#### 2. 分块策略
**推荐**：分层语义分块 (Aider/IronClaw)
- **代码文件**：函数/类级别 AST 分块
- **文档**：段落级语义边界
- **可配置限制**：基于 token 和字符的限制
- **元数据保留**：源位置、文件类型、时间戳

```rust
pub struct ChunkingConfig {
    pub strategy: ChunkingStrategy,  // Semantic, Fixed, Hierarchical
    pub chunk_size: usize,           // 512 tokens default
    pub chunk_overlap: usize,        // 50 tokens
    pub code_aware: bool,            // Use AST for code
    pub preserve_hierarchy: bool,    // Keep document structure
}
```

#### 3. Embedding 生成
**推荐**：多提供商 + 本地降级 (IronClaw/OpenClaw)
- **云端提供商**：OpenAI、Anthropic (质量最佳)
- **本地选项**：Ollama、FastEmbed (隐私/成本)
- **自动降级**：云端优先，失败时本地
- **缓存**：避免重复计算

```rust
pub trait EmbeddingProvider: Send + Sync {
    async fn embed(&self, text: &str) -> Result<Vec<f32>>;
    async fn embed_batch(&self, texts: Vec<String>) -> Result<Vec<Vec<f32>>>;
    fn dimensions(&self) -> usize;
}

pub struct TieredEmbeddingProvider {
    primary: Box<dyn EmbeddingProvider>,   // Cloud
    fallback: Box<dyn EmbeddingProvider>,  // Local
    cache: EmbeddingCache,
}
```

#### 4. 向量存储
**推荐**：LanceDB (OpenClaw) 或 PostgreSQL/pgvector (IronClaw)
- **LanceDB**：本地优先，查询快，零依赖
- **PostgreSQL**：企业级，可扩展
- **SQLite 降级**：超轻量级部署

#### 5. 全文搜索
**推荐**：SQLite FTS5 (OpenClaw) / PostgreSQL FTS (IronClaw)
- **FTS5**：本地部署，快速，零外部依赖
- **PostgreSQL FTS**：企业级扩展
- **Ripgrep**：代码搜索集成 (Aider/Qwen Code)

#### 6. 混合搜索
**推荐**：真混合 + RRF (IronClaw)
- **向量搜索**：语义相似度
- **关键词搜索**：FTS 精确匹配
- **RRF 融合**：Reciprocal Rank Fusion
- **可配置权重**：根据用例调整

```rust
pub struct HybridSearcher {
    pub vector_weight: f64,  // e.g., 0.7
    pub text_weight: f64,    // e.g., 0.3
}

impl HybridSearcher {
    pub async fn search(&self, query: &str, top_k: usize) -> Vec<SearchResult> {
        let query_embedding = self.embed(query).await;

        // 并行执行两种搜索
        let (vector_results, text_results) = tokio::join!(
            self.vector_search(&query_embedding, top_k * 2),
            self.keyword_search(query, top_k * 2),
        );

        // RRF 融合
        self.reciprocal_rank_fusion(&[vector_results, text_results])
            .into_iter()
            .take(top_k)
            .collect()
    }
}
```

#### 7. 增量同步
**推荐**：事件驱动 + 防抖 (Aider/IronClaw)
- **文件监听**：chokidar/notify/watchdog
- **Git 集成**：代码库操作钩子
- **防抖**：批量快速变更
- **增量更新**：仅重新索引变更块

```rust
pub struct IncrementalSync {
    watcher: Box<dyn FileWatcher>,
    debounce_ms: u64,
}

impl IncrementalSync {
    pub async fn watch(&self, paths: &[PathBuf]) -> Result<()> {
        // 设置文件监听
        // 变更时：确定受影响块
        // 仅重新解析和嵌入变更内容
        // 增量更新向量存储
    }
}
```

#### 8. 缓存策略
**推荐**：多级缓存 (IronClaw/OpenClaw)
- **L1**：内存 LRU 热数据
- **L2**：磁盘 embedding 和查询结果缓存
- **L3**：数据库向量存储缓存

### 技术栈推荐

| 组件 | 首选 | 备选 | 说明 |
|------|------|------|------|
| **语言** | Rust/TypeScript | Python | Rust 性能，TS 生态 |
| **解析** | Tree-sitter + Markitdown | 自定义 | Tree-sitter 代码，Markitdown 文档 |
| **分块** | Semantic (tiktoken) | Fixed | Token 感知 |
| **Embedding** | OpenAI + FastEmbed | Ollama | 云端 + 本地 |
| **向量存储** | LanceDB | PostgreSQL/pgvector | 本地优先 |
| **FTS** | SQLite FTS5 | PostgreSQL FTS | 零依赖 |
| **文件监听** | notify/chokidar | watchdog | 跨平台 |
| **缓存** | LRU + Disk | Redis | 多级 |

---

## 实现参考代码

### 完整 Rust 实现示例

```rust
// unified_document_pipeline.rs

use std::path::Path;
use tokio::fs;
use async_trait::async_trait;
use serde::{Deserialize, Serialize};

// ============ 1. 核心类型定义 ============

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Document {
    pub path: String,
    pub content: String,
    pub content_type: ContentType,
    pub metadata: DocumentMetadata,
}

#[derive(Debug, Clone)]
pub enum ContentType {
    Markdown,
    Code(String), // language
    Pdf,
    Image,
    Text,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DocumentMetadata {
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub modified_at: chrono::DateTime<chrono::Utc>,
    pub size: usize,
}

#[derive(Debug, Clone)]
pub struct Chunk {
    pub id: String,
    pub document_path: String,
    pub text: String,
    pub start_pos: usize,
    pub end_pos: usize,
    pub token_count: usize,
}

pub type Embedding = Vec<f32>;

// ============ 2. Parser Trait & Implementations ============

#[async_trait]
pub trait DocumentParser: Send + Sync {
    async fn parse(&self, path: &Path) -> anyhow::Result<Document>;
    fn supported_extensions(&self) -> Vec<&'static str>;
}

pub struct MarkdownParser;
#[async_trait]
impl DocumentParser for MarkdownParser {
    async fn parse(&self, path: &Path) -> anyhow::Result<Document> {
        let content = fs::read_to_string(path).await?;
        let metadata = fs::metadata(path).await?;

        Ok(Document {
            path: path.to_string_lossy().to_string(),
            content,
            content_type: ContentType::Markdown,
            metadata: DocumentMetadata {
                created_at: chrono::DateTime::from(metadata.created()?),
                modified_at: chrono::DateTime::from(metadata.modified()?),
                size: metadata.len() as usize,
            },
        })
    }

    fn supported_extensions(&self) -> Vec<&'static str> {
        vec!["md", "markdown"]
    }
}

// ============ 3. Chunker Trait & Implementations ============

pub struct ChunkingConfig {
    pub target_chunk_size: usize,  // tokens
    pub chunk_overlap: usize,      // tokens
}

pub trait Chunker: Send + Sync {
    fn chunk(&self, document: &Document, config: &ChunkingConfig) -> Vec<Chunk>;
}

pub struct SemanticChunker;

impl Chunker for SemanticChunker {
    fn chunk(&self, document: &Document, config: &ChunkingConfig) -> Vec<Chunk> {
        let lines: Vec<&str> = document.content.lines().collect();
        let mut chunks = Vec::new();
        let mut current_chunk = Vec::new();
        let mut current_tokens = 0;
        let mut start_pos = 0;

        // ~4 chars per token
        let target_chars = config.target_chunk_size * 4;
        let overlap_chars = config.chunk_overlap * 4;

        for (idx, line) in lines.iter().enumerate() {
            let line_chars = line.len();

            if current_tokens + line_chars > target_chars && !current_chunk.is_empty() {
                // Save chunk
                let text = current_chunk.join("\n");
                chunks.push(Chunk {
                    id: format!("{}-{}", document.path, chunks.len()),
                    document_path: document.path.clone(),
                    text,
                    start_pos,
                    end_pos: idx,
                    token_count: current_tokens / 4,
                });

                // Handle overlap
                if overlap_chars > 0 {
                    let mut overlap_lines = Vec::new();
                    let mut overlap_count = 0;
                    for line in current_chunk.iter().rev() {
                        if overlap_count + line.len() > overlap_chars {
                            break;
                        }
                        overlap_lines.insert(0, *line);
                        overlap_count += line.len() + 1;
                    }
                    current_chunk = overlap_lines;
                    current_tokens = overlap_count;
                    start_pos = idx - current_chunk.len();
                } else {
                    current_chunk.clear();
                    current_tokens = 0;
                    start_pos = idx;
                }
            }

            current_chunk.push(*line);
            current_tokens += line_chars + 1; // +1 for newline
        }

        // Handle last chunk
        if !current_chunk.is_empty() {
            let text = current_chunk.join("\n");
            chunks.push(Chunk {
                id: format!("{}-{}", document.path, chunks.len()),
                document_path: document.path.clone(),
                text,
                start_pos,
                end_pos: lines.len(),
                token_count: current_tokens / 4,
            });
        }

        chunks
    }
}

// ============ 4. Embedding Provider ============

#[async_trait]
pub trait EmbeddingProvider: Send + Sync {
    async fn embed(&self, text: &str) -> anyhow::Result<Embedding>;
    async fn embed_batch(&self, texts: &[&str]) -> anyhow::Result<Vec<Embedding>>;
    fn dimensions(&self) -> usize;
}

pub struct OpenAIEmbeddingProvider {
    client: reqwest::Client,
    api_key: String,
    model: String,
}

#[async_trait]
impl EmbeddingProvider for OpenAIEmbeddingProvider {
    async fn embed(&self, text: &str) -> anyhow::Result<Embedding> {
        let response = self.client
            .post("https://api.openai.com/v1/embeddings")
            .header("Authorization", format!("Bearer {}", self.api_key))
            .json(&serde_json::json!({
                "input": text,
                "model": self.model,
            }))
            .send()
            .await?;

        let result: serde_json::Value = response.json().await?;
        let embedding = result["data"][0]["embedding"]
            .as_array()
            .unwrap()
            .iter()
            .map(|v| v.as_f64().unwrap() as f32)
            .collect();

        Ok(embedding)
    }

    async fn embed_batch(&self, texts: &[&str]) -> anyhow::Result<Vec<Embedding>> {
        // Batch API implementation
        let mut embeddings = Vec::new();
        for text in texts {
            embeddings.push(self.embed(text).await?);
        }
        Ok(embeddings)
    }

    fn dimensions(&self) -> usize {
        1536 // text-embedding-3-small
    }
}

// ============ 5. Vector Store ============

#[async_trait]
pub trait VectorStore: Send + Sync {
    async fn upsert(&self, chunks: &[Chunk], embeddings: &[Embedding]) -> anyhow::Result<()>;
    async fn search(&self, query_embedding: &Embedding, top_k: usize) -> anyhow::Result<Vec<SearchResult>>;
    async fn delete(&self, chunk_ids: &[&str]) -> anyhow::Result<()>;
}

#[derive(Debug, Clone)]
pub struct SearchResult {
    pub chunk: Chunk,
    pub score: f32,
}

// ============ 6. Hybrid Searcher ============

pub struct HybridSearcher {
    vector_store: Arc<dyn VectorStore>,
    fts: Arc<dyn FullTextSearch>,
    vector_weight: f64,
    text_weight: f64,
    embedding_provider: Arc<dyn EmbeddingProvider>,
}

impl HybridSearcher {
    pub async fn search(&self, query: &str, top_k: usize) -> anyhow::Result<Vec<SearchResult>> {
        // Parallel search
        let query_embedding = self.embedding_provider.embed(query).await?;

        let (vector_results, text_results) = tokio::join!(
            self.vector_store.search(&query_embedding, top_k * 2),
            self.fts.search(query, top_k * 2),
        );

        let vector_results = vector_results?;
        let text_results = text_results?;

        // RRF fusion
        let fused = self.reciprocal_rank_fusion(&vector_results, &text_results);

        Ok(fused.into_iter().take(top_k).collect())
    }

    fn reciprocal_rank_fusion(
        &self,
        vector_results: &[SearchResult],
        text_results: &[SearchResult],
    ) -> Vec<SearchResult> {
        const K: f64 = 60.0;
        let mut scores: std::collections::HashMap<String, f64> = std::collections::HashMap::new();

        // Score vector results
        for (rank, result) in vector_results.iter().enumerate() {
            let score = self.vector_weight * (1.0 / (K + rank as f64 + 1.0));
            scores.entry(result.chunk.id.clone())
                .and_modify(|s| *s += score)
                .or_insert(score);
        }

        // Score text results
        for (rank, result) in text_results.iter().enumerate() {
            let score = self.text_weight * (1.0 / (K + rank as f64 + 1.0));
            scores.entry(result.chunk.id.clone())
                .and_modify(|s| *s += score)
                .or_insert(score);
        }

        // Sort by fused score
        let mut fused: Vec<_> = scores.into_iter().collect();
        fused.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

        // Map back to SearchResult
        fused.iter()
            .filter_map(|(id, _)| {
                vector_results.iter()
                    .chain(text_results.iter())
                    .find(|r| r.chunk.id == *id)
                    .cloned()
            })
            .collect()
    }
}

// ============ 7. Ingestion Pipeline ============

pub struct IngestionPipeline {
    parsers: Vec<Box<dyn DocumentParser>>,
    chunker: Box<dyn Chunker>,
    embedding_provider: Arc<dyn EmbeddingProvider>,
    vector_store: Arc<dyn VectorStore>,
}

impl IngestionPipeline {
    pub async fn ingest(&self, path: &Path) -> anyhow::Result<()> {
        // 1. Parse document
        let parser = self.find_parser(path)?;
        let document = parser.parse(path).await?;

        // 2. Chunk
        let chunks = self.chunker.chunk(&document, &ChunkingConfig {
            target_chunk_size: 400,
            chunk_overlap: 80,
        });

        // 3. Embed
        let texts: Vec<&str> = chunks.iter().map(|c| c.text.as_str()).collect();
        let embeddings = self.embedding_provider.embed_batch(&texts).await?;

        // 4. Store
        self.vector_store.upsert(&chunks, &embeddings).await?;

        Ok(())
    }

    fn find_parser(&self, path: &Path) -> anyhow::Result<&dyn DocumentParser> {
        let ext = path.extension()
            .and_then(|e| e.to_str())
            .unwrap_or("");

        for parser in &self.parsers {
            if parser.supported_extensions().contains(&ext) {
                return Ok(parser.as_ref());
            }
        }

        anyhow::bail!("No parser found for extension: {}", ext)
    }
}

// Placeholder for FullTextSearch trait
#[async_trait]
pub trait FullTextSearch: Send + Sync {
    async fn search(&self, query: &str, top_k: usize) -> anyhow::Result<Vec<SearchResult>>;
}
```

---

## 附录：关键文件索引

### Personal Assistants

| 项目 | 关键文件 | 说明 |
|------|----------|------|
| **CoPaw** | `agents/memory/memory_manager.py` | Memory manager wrapper |
| **CoPaw** | `agents/tools/memory_search.py` | Memory search tool |
| **IronClaw** | `src/workspace/memory.rs` | Hybrid search implementation |
| **IronClaw** | `src/workspace/embedding.rs` | FastEmbed integration |
| **OpenClaw** | `memory/internal.ts` | File discovery & chunking |
| **OpenClaw** | `memory/embeddings.ts` | Multi-provider embeddings |
| **OpenClaw** | `memory/hybrid.ts` | Hybrid search + MMR |
| **NanoClaw** | `src/memory.ts` | CLAUDE.md management |
| **PicoClaw** | `memory/memory.go` | JSONL storage |
| **ZeroClaw** | `src/memory/traits.rs` | Memory trait definition |
| **Nanobot** | `memory/dual_tier.py` | Two-tier memory system |

### Programming Agents

| 项目 | 关键文件 | 说明 |
|------|----------|------|
| **Aider** | `aider/repomap.py` | RepoMap implementation |
| **Aider** | `aider/watch.py` | File watcher |
| **Aider** | `aider/grep_ast.py` | Tree-sitter integration |
| **Qwen Code** | `src/core/context/mcp-context.ts` | MCP context engine |
| **Qwen Code** | `src/tools/search.ts` | Ripgrep integration |
| **Gemini CLI** | `src/context/cache.ts` | Context caching |
| **Codex** | `crates/codex/src/protocol.rs` | File operations |
| **OpenCode** | `src/tool/read.ts` | File reading limits |
| **OpenCode** | `src/db/schema.ts` | Drizzle ORM schema |
| **Pi-Mono** | `src/analysis/graph.ts` | Code graph analysis |
| **Kimi CLI** | `session/state.py` | JSON session storage |

---

*报告生成时间: 2025-03-18*
*分析项目: 15 个 AI Agent 项目 (8 Personal Assistants + 7 Programming Agents)*
