# AI Agent Model Provider 抽象层与多模型路由机制深度调研报告

## 概述

本报告对15个开源 AI Agent 项目（8个个人助手 + 7个编程代理）的 Model Provider 抽象层与多模型路由机制进行深度技术分析，对比各自的 Provider 抽象设计、多模型配置、负载均衡、Fallback 机制、Streaming 实现和成本追踪。

**个人助手项目 (8个):** CoPaw, IronClaw, NanoClaw, ZeroClaw, MimiClaw, PicoClaw, Nanobot, OpenClaw
**编程代理项目 (7个):** Aider, Codex, Qwen-code, Gemini-cli, Pi-mono, Kimi-cli, Opencode

---

## 一、各项目 Provider 架构概览

### 架构对比总表

| 维度 | CoPaw | IronClaw | NanoClaw | ZeroClaw | MimiClaw | PicoClaw | Nanobot | OpenClaw |
|------|-------|----------|----------|----------|----------|----------|---------|----------|
| **抽象层** | AgentScope + ABC | LlmProvider Trait | Container-based | Provider Trait | C Struct | LLMProvider Interface | ABC + LiteLLM | Gateway-based |
| **语言** | Python | Rust | TypeScript | Rust | C | Go | Python | TypeScript |
| **配置格式** | JSON | JSON/Env | TypeScript/Env | TOML/Env | NVS/JSON | JSON/Env | YAML/JSON/Env | JSON/Wizard |
| **内置 Provider** | 10+ | 7+ | Claude SDK | 4+ | 2+ | 20+ | LiteLLM生态 | Extensible |
| **本地模型** | llama.cpp, MLX, Ollama | Ollama | No | Ollama | No | Ollama | Ollama (via LiteLLM) | Ollama |
| **Streaming** | AsyncGenerator | SSE | N/A | SSE | HTTP Chunked | Channels/SSE | Async Generators | SSE/WebSocket |
| **成本追踪** | TokenUsageManager | Decimal precision | No | Basic tokens | No | Basic | LiteLLM | Usage metrics |
| **Fallback** | RetryChatModel | Decorator Chain | Concurrent limit | No | No | FallbackChain | LiteLLM retry | Gateway routing |

| 维度 | Aider | Codex | Qwen-code | Gemini-cli | Pi-mono | Kimi-cli | Opencode |
|------|-------|-------|-----------|------------|---------|----------|----------|
| **抽象层** | LiteLLM Wrapper | Provider Struct | ModelRegistry | Policy Chains | ApiProvider Interface | ChatProvider Protocol | Vercel AI SDK |
| **语言** | Python | Rust | TypeScript | TypeScript | TypeScript | Python | TypeScript |
| **配置格式** | YAML/Env | TOML/Env | JSON/OAuth | TypeScript/Env | JSON/Env | TOML/JSON/Env | JSON/Markdown/Env |
| **内置 Provider** | LiteLLM生态 | OpenAI, Azure | Qwen + OpenAI | Gemini only | 15+ | 7+ + testing | 12+ |
| **本地模型** | Ollama (via LiteLLM) | Custom endpoints | User endpoints | No | Ollama | No | OpenAI-compatible |
| **Streaming** | LiteLLM | WebSocket | SSE | Event-based | SSE/WebSocket events | Async iterators | Vercel SDK |
| **成本追踪** | Model-based | Azure-specific | Basic | API-side | $/M tokens | TokenUsage with cache | models.dev API |
| **Fallback** | LiteLLM retry | RetryConfig | Auth-type switching | Model policy chains | Configurable retry | RetryableChatProvider | Provider error handling |

---

## 二、个人助手项目 Provider 详解

### 1. CoPaw - AgentScope + 自定义 Provider 架构

#### 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     CoPaw Provider Stack                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Application Layer                                        │   │
│  │  - ProviderManager (Singleton)                            │   │
│  │  - TokenUsageManager                                      │   │
│  │  - RoutingChatModel                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Provider Abstraction Layer                         │ │ │
│  │  │  - Provider (ABC)                                   │ │ │
│  │  │  - OpenAIProvider                                   │ │ │
│  │  │  - AnthropicProvider                                │ │ │
│  │  │  - OllamaProvider                                   │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  │                           │                               │ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  ChatModel Layer (AgentScope)                       │ │ │
│  │  │  - OpenAIChatModel                                  │ │ │
│  │  │  - AnthropicChatModel                               │ │ │
│  │  │  - LocalChatModel                                   │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  │                           │                               │ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Wrapper Layer                                      │ │ │
│  │  │  - TokenRecordingModelWrapper                       │ │ │
│  │  │  - RetryChatModel                                   │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Storage Layer                                      │ │ │
│  │  │  - providers/ (builtin + custom)                    │ │ │
│  │  │  - active_model.json                                │ │ │
│  │  │  - token_usage/                                     │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Provider 抽象接口

```python
# src/copaw/providers/provider.py
class Provider(ProviderInfo, ABC):
    """Represents a provider instance with its configuration."""

    @abstractmethod
    async def check_connection(self, timeout: float = 5) -> tuple[bool, str]:
        """Check if the provider is reachable."""

    @abstractmethod
    async def fetch_models(self, timeout: float = 5) -> List[ModelInfo]:
        """Fetch available models from the provider."""

    @abstractmethod
    def get_chat_model_instance(self, model_id: str) -> ChatModelBase:
        """Return a ChatModel instance for this provider and model."""
```

#### Provider 信息模型

```python
class ProviderInfo(BaseModel):
    id: str                          # Provider 标识符
    name: str                        # 人类可读名称
    base_url: str                    # API 基础 URL
    api_key: str                     # API 密钥
    chat_model: str                  # AgentScope ChatModel 类名
    models: List[ModelInfo]          # 预定义模型列表
    extra_models: List[ModelInfo]    # 用户添加的模型
    is_local: bool                   # 是否为本地托管
    require_api_key: bool            # 是否需要 API 密钥
    support_model_discovery: bool    # 是否支持模型发现
```

#### 内置 Provider 列表

| Provider ID | 类型 | Base URL | 特点 |
|-------------|------|----------|------|
| `openai` | OpenAIProvider | https://api.openai.com/v1 | 官方 API |
| `anthropic` | AnthropicProvider | https://api.anthropic.com | Claude 系列 |
| `dashscope` | OpenAIProvider | https://dashscope.aliyuncs.com | 阿里云 |
| `modelscope` | OpenAIProvider | https://api-inference.modelscope.cn | 魔搭社区 |
| `ollama` | OllamaProvider | http://localhost:11434 | 本地模型 |
| `lmstudio` | OpenAIProvider | http://localhost:1234/v1 | LM Studio |
| `llamacpp` | DefaultProvider | - | llama.cpp 本地 |
| `mlx` | DefaultProvider | - | Apple Silicon 本地 |

#### Fallback 与重试机制

```python
# src/copaw/providers/retry_chat_model.py
class RetryChatModel(ChatModelBase):
    """Transparent retry wrapper around any ChatModelBase."""

    async def __call__(self, *args, **kwargs):
        retries = LLM_MAX_RETRIES  # 默认 3 次

        for attempt in range(1, retries + 1):
            try:
                return await self._inner(*args, **kwargs)
            except Exception as exc:
                if not _is_retryable(exc) or attempt >= retries:
                    raise
                delay = _compute_backoff(attempt)  # 指数退避
                await asyncio.sleep(delay)

RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504}
```

#### Token 计费与成本追踪

```python
# src/copaw/token_usage/manager.py
class TokenUsageManager:
    async def record(
        self,
        provider_id: str,
        model_name: str,
        prompt_tokens: int,
        completion_tokens: int,
        at_date: date | None = None,
    ) -> None:
        """Record token usage for a given provider, model and date."""
        composite_key = f"{provider_id}:{model_name}"

        with self._file_lock:
            data = await self._load_data()
            if date_str not in data:
                data[date_str] = {}

            by_key = data[date_str]
            if composite_key not in by_key:
                by_key[composite_key] = {
                    "provider_id": provider_id,
                    "model_name": model_name,
                    "prompt_tokens": 0,
                    "completion_tokens": 0,
                    "call_count": 0,
                }

            entry = by_key[composite_key]
            entry["prompt_tokens"] += prompt_tokens
            entry["completion_tokens"] += completion_tokens
            entry["call_count"] += 1

            await self._save_data(data)
```

---

### 2. IronClaw - Rust Trait + 装饰器模式架构

#### 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                  IronClaw Provider Architecture                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Application Layer                                        │   │
│  │  - LlmProvider Trait                                      │   │
│  │  - ProviderRegistry                                       │   │
│  │  - ProviderManager                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Decorator Chain (Layered Providers)                │ │ │
│  │  │                                                     │ │ │
│  │  │  ┌─────────────────────────────────────────────┐   │ │ │
│  │  │  │ Raw Provider                                │   │ │ │
│  │  │  │ (OpenAI, Anthropic, Ollama, NEAR AI)        │   │ │ │
│  │  │  └─────────────────────────────────────────────┘   │ │ │
│  │  │                          │                          │ │ │
│  │  │  ┌───────────────────────┼───────────────────────┐ │ │ │
│  │  │  │                       ▼                       │ │ │ │
│  │  │  │  ┌─────────────────────────────────────────┐ │ │ │ │
│  │  │  │  │ RetryProvider                           │ │ │ │ │
│  │  │  │  │ - Exponential backoff                   │ │ │ │ │
│  │  │  │  │ - Configurable max retries              │ │ │ │ │
│  │  │  │  └─────────────────────────────────────────┘ │ │ │ │
│  │  │  │                          │                      │ │ │ │
│  │  │  │  ┌───────────────────────┼───────────────────┐ │ │ │ │
│  │  │  │  │                       ▼                   │ │ │ │ │
│  │  │  │  │  ┌─────────────────────────────────────┐ │ │ │ │ │
│  │  │  │  │  │ SmartRoutingProvider                │ │ │ │ │ │
│  │  │  │  │  │ - Intelligent model selection       │ │ │ │ │ │
│  │  │  │  │  │ - Cost-aware routing                │ │ │ │ │ │
│  │  │  │  │  └─────────────────────────────────────┘ │ │ │ │ │
│  │  │  │  │                      │                      │ │ │ │ │
│  │  │  │  │  ┌───────────────────┼─────────────────┐ │ │ │ │ │
│  │  │  │  │  │                   ▼                 │ │ │ │ │ │
│  │  │  │  │  │  ┌───────────────────────────────┐ │ │ │ │ │ │
│  │  │  │  │  │  │ FailoverProvider              │ │ │ │ │ │ │
│  │  │  │  │  │  │ - Multiple backend support    │ │ │ │ │ │ │
│  │  │  │  │  │  │ - Health checking             │ │ │ │ │ │ │
│  │  │  │  │  │  └─────────────────────────────┘ │ │ │ │ │ │ │
│  │  │  │  │  │                  │                  │ │ │ │ │ │
│  │  │  │  │  │  ┌───────────────┼─────────────┐ │ │ │ │ │ │ │
│  │  │  │  │  │  │               ▼             │ │ │ │ │ │ │ │
│  │  │  │  │  │  │  CircuitBreakerProvider     │ │ │ │ │ │ │ │
│  │  │  │  │  │  └───────────────────────────┘ │ │ │ │ │ │ │ │
│  │  │  │  │  │                  │              │ │ │ │ │ │ │ │
│  │  │  │  │  │  ┌───────────────┼───────────┐ │ │ │ │ │ │ │ │
│  │  │  │  │  │  │               ▼           │ │ │ │ │ │ │ │
│  │  │  │  │  │  │  CachedProvider           │ │ │ │ │ │ │ │
│  │  │  │  │  │  └─────────────────────────┘ │ │ │ │ │ │ │ │
│  │  │  │  │  │                  │          │ │ │ │ │ │ │ │
│  │  │  │  │  │  ┌───────────────┼───────┐ │ │ │ │ │ │ │ │ │
│  │  │  │  │  │  │               ▼       │ │ │ │ │ │ │ │ │ │
│  │  │  │  │  │  │  RecordingProvider    │ │ │ │ │ │ │ │ │ │
│  │  │  │  │  │  └───────────────────────┘ │ │ │ │ │ │ │ │ │
│  │  │  │  │  └────────────────────────────┘ │ │ │ │ │ │ │ │
│  │  │  │  └─────────────────────────────────┘ │ │ │ │ │ │ │
│  │  │  └─────────────────────────────────────┘ │ │ │ │ │ │
│  │  │  └───────────────────────────────────────┘ │ │ │ │ │
│  │  │  └─────────────────────────────────────────┘ │ │ │ │
│  │  │  └───────────────────────────────────────────┘ │ │ │
│  │  └───────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────┘ │
│                              │                           │
│  ┌───────────────────────────┼───────────────────────────┤
│  │                           ▼                           │
│  │  ┌─────────────────────────────────────────────────┐ │
│  │  │  Configuration Layer                            │ │
│  │  │  - JSON Provider Registry                       │ │
│  │  │  - Environment Variables                        │ │
│  │  │  - ProviderProtocol enum                        │ │
│  │  └─────────────────────────────────────────────────┘ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

#### LlmProvider Trait 定义

```rust
// src/llm/provider.rs
#[async_trait]
pub trait LlmProvider: Send + Sync {
    /// Complete a chat request
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError>;

    /// Complete with tools
    async fn complete_with_tools(&self, request: ToolCompletionRequest) -> Result<ToolCompletionResponse, LlmError>;

    /// List available models
    async fn list_models(&self) -> Result<Vec<ModelInfo>, LlmError>;

    /// Get model metadata
    async fn model_metadata(&self, model_id: &str) -> Result<ModelMetadata, LlmError>;

    /// Calculate cost for usage
    fn calculate_cost(&self, usage: &TokenUsage, model_id: &str) -> Option<Decimal>;
}

pub struct TokenUsage {
    pub prompt_tokens: u64,
    pub completion_tokens: u64,
    pub total_tokens: u64,
    pub cache_read_tokens: Option<u64>,
    pub cache_write_tokens: Option<u64>,
}
```

#### Provider 注册表

```rust
// src/llm/registry.rs
pub struct ProviderDefinition {
    pub id: String,
    pub aliases: Vec<String>,
    pub protocol: ProviderProtocol,
    pub default_base_url: String,
    pub api_key_env: Option<String>,
    pub setup_hint: Option<SetupHint>,
}

pub enum ProviderProtocol {
    OpenAiCompletions,
    Anthropic,
    Ollama,
    NearAi,
    OpenAiCompatible,
}

// Built-in providers
pub const BUILT_IN_PROVIDERS: &[ProviderDefinition] = &[
    ProviderDefinition {
        id: "openai".to_string(),
        aliases: vec!["gpt".to_string()],
        protocol: ProviderProtocol::OpenAiCompletions,
        default_base_url: "https://api.openai.com/v1".to_string(),
        api_key_env: Some("OPENAI_API_KEY".to_string()),
        setup_hint: Some(SetupHint::VisitUrl("https://platform.openai.com/api-keys".to_string())),
    },
    ProviderDefinition {
        id: "anthropic".to_string(),
        aliases: vec!["claude".to_string()],
        protocol: ProviderProtocol::Anthropic,
        default_base_url: "https://api.anthropic.com".to_string(),
        api_key_env: Some("ANTHROPIC_API_KEY".to_string()),
        setup_hint: Some(SetupHint::VisitUrl("https://console.anthropic.com/".to_string())),
    },
    // ... NEAR AI, Ollama, Bedrock, etc.
];
```

#### 装饰器模式实现

```rust
// Retry Provider
pub struct RetryProvider {
    inner: Box<dyn LlmProvider>,
    max_retries: u32,
    base_delay_ms: u64,
}

#[async_trait]
impl LlmProvider for RetryProvider {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError> {
        let mut last_error = None;

        for attempt in 0..=self.max_retries {
            match self.inner.complete(request.clone()).await {
                Ok(response) => return Ok(response),
                Err(e) if is_retryable(&e) && attempt < self.max_retries => {
                    let delay = self.base_delay_ms * 2_u64.pow(attempt);
                    tokio::time::sleep(Duration::from_millis(delay)).await;
                    last_error = Some(e);
                }
                Err(e) => return Err(e),
            }
        }

        Err(last_error.unwrap_or_else(|| LlmError::MaxRetriesExceeded))
    }
}

// Failover Provider
pub struct FailoverProvider {
    primaries: Vec<Box<dyn LlmProvider>>,
    fallbacks: Vec<Box<dyn LlmProvider>>,
    health_checker: Arc<dyn HealthChecker>,
}

#[async_trait]
impl LlmProvider for FailoverProvider {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError> {
        // Try primaries first
        for provider in &self.primaries {
            if self.health_checker.is_healthy(provider).await {
                match provider.complete(request.clone()).await {
                    Ok(response) => return Ok(response),
                    Err(_) => continue,
                }
            }
        }

        // Fall back to backup providers
        for provider in &self.fallbacks {
            match provider.complete(request.clone()).await {
                Ok(response) => return Ok(response),
                Err(_) => continue,
            }
        }

        Err(LlmError::AllProvidersFailed)
    }
}
```

---

### 3. ZeroClaw - Rust async_trait Provider 架构

#### Provider Trait 定义

```rust
// src/providers/traits.rs
#[async_trait]
pub trait Provider: Send + Sync {
    /// Send a chat message and get response
    async fn chat(&self, messages: Vec<ChatMessage>) -> Result<ChatResponse, ProviderError>;

    /// Chat with system prompt
    async fn chat_with_system(&self, system: &str, messages: Vec<ChatMessage>) -> Result<ChatResponse, ProviderError>;

    /// Chat with conversation history
    async fn chat_with_history(&self, history: Vec<ChatMessage>) -> Result<ChatResponse, ProviderError>;

    /// Chat with tools
    async fn chat_with_tools(&self, messages: Vec<ChatMessage>, tools: Vec<ToolDefinition>) -> Result<ChatResponse, ProviderError>;

    /// Stream chat response
    async fn chat_stream(&self, messages: Vec<ChatMessage>) -> Result<BoxStream<StreamChunk>, ProviderError>;

    /// Get provider capabilities
    fn capabilities(&self) -> ProviderCapabilities;

    /// Get model info
    fn model_info(&self) -> ModelInfo;
}

pub struct ProviderCapabilities {
    pub native_tool_calling: bool,
    pub vision: bool,
    pub prompt_caching: bool,
    pub streaming: bool,
}

pub struct ModelInfo {
    pub id: String,
    pub name: String,
    pub context_window: usize,
    pub max_output_tokens: Option<usize>,
}
```

#### Token Usage 追踪

```rust
pub struct TokenUsage {
    pub input_tokens: u64,
    pub output_tokens: u64,
    pub cache_read_tokens: Option<u64>,
    pub cache_write_tokens: Option<u64>,
    pub total_tokens: u64,
}

pub struct ChatResponse {
    pub content: String,
    pub tool_calls: Option<Vec<ToolCall>>,
    pub usage: TokenUsage,
    pub finish_reason: FinishReason,
}
```

#### 配置示例 (TOML)

```toml
# .zeroclaw/config.toml
[providers.openai]
api_key = "${OPENAI_API_KEY}"
base_url = "https://api.openai.com/v1"
default_model = "gpt-4o"

[providers.anthropic]
api_key = "${ANTHROPIC_API_KEY}"
base_url = "https://api.anthropic.com"
default_model = "claude-3-5-sonnet-20241022"

[providers.ollama]
base_url = "http://localhost:11434"
default_model = "llama3.2"
```

---

### 4. PicoClaw - Go Interface + FallbackChain 架构

#### LLMProvider Interface

```go
// pkg/providers/types.go
type LLMProvider interface {
    // Chat sends messages and returns response
    Chat(ctx context.Context, messages []Message) (*Response, error)

    // GetDefaultModel returns the default model ID
    GetDefaultModel() string

    // GetName returns provider name
    GetName() string
}

// StatefulProvider extends with lifecycle management
type StatefulProvider interface {
    LLMProvider
    Close() error
}

// FailoverReason enum
type FailoverReason int

const (
    FailoverAuth FailoverReason = iota
    FailoverRateLimit
    FailoverBilling
    FailoverTimeout
    FailoverFormat
    FailoverOverloaded
    FailoverUnknown
)
```

#### FallbackChain 实现

```go
// pkg/providers/fallback.go
type FallbackChain struct {
    providers []ProviderCandidate
    cooldowns map[string]time.Time
    cooldownDuration time.Duration
}

type ProviderCandidate struct {
    Provider LLMProvider
    ModelID  string
    Priority int
}

func (fc *FallbackChain) ResolveCandidates(primary string) []ProviderCandidate {
    var candidates []ProviderCandidate

    // Filter out providers in cooldown
    for _, pc := range fc.providers {
        if cooldownEnd, ok := fc.cooldowns[pc.Provider.GetName()]; ok {
            if time.Now().Before(cooldownEnd) {
                continue // Skip this provider
            }
        }
        candidates = append(candidates, pc)
    }

    // Sort by priority
    sort.Slice(candidates, func(i, j int) bool {
        return candidates[i].Priority < candidates[j].Priority
    })

    return candidates
}

func (fc *FallbackChain) MarkFailed(providerName string, reason FailoverReason) {
    // Set cooldown based on failure reason
    duration := fc.cooldownDuration
    switch reason {
    case FailoverRateLimit:
        duration = time.Minute * 5
    case FailoverOverloaded:
        duration = time.Minute * 2
    case FailoverAuth:
        duration = time.Hour // Longer cooldown for auth failures
    }
    fc.cooldowns[providerName] = time.Now().Add(duration)
}
```

#### Provider Factory

```go
// pkg/providers/factory.go
func CreateProvider(providerType string, config ProviderConfig) (LLMProvider, error) {
    switch providerType {
    case "openai":
        return NewOpenAIProvider(config)
    case "anthropic":
        return NewAnthropicProvider(config)
    case "ollama":
        return NewOllamaProvider(config)
    case "openrouter":
        return NewOpenRouterProvider(config)
    case "groq":
        return NewGroqProvider(config)
    case "gemini":
        return NewGeminiProvider(config)
    case "bedrock":
        return NewBedrockProvider(config)
    // ... 20+ providers
    default:
        return nil, fmt.Errorf("unknown provider type: %s", providerType)
    }
}
```

---

### 5. NanoClaw - Container-based Provider 架构

NanoClaw 采用独特的容器化架构，不依赖传统 Provider 抽象：

```
┌─────────────────────────────────────────────────────────────────┐
│                  NanoClaw Container Architecture                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Host Process                                           │   │
│  │  - Config Manager (src/config.ts)                       │   │
│  │  - Container Orchestrator                               │   │
│  │  - Message Router                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Docker Container (Linux VM)                        │ │ │
│  │  │                                                     │ │ │
│  │  │  ┌─────────────────────────────────────────────┐   │ │ │
│  │  │  │ Agent SDK Runtime                           │   │ │ │
│  │  │  │ - Claude Agent SDK                          │   │ │ │
│  │  │  │ - Tool Execution                            │   │ │ │
│  │  │  │ - Provider calls (via SDK)                  │   │ │ │
│  │  │  └─────────────────────────────────────────────┘   │ │ │
│  │  │                                                     │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### 配置方式

```typescript
// src/config.ts
export interface ContainerConfig {
    CONTAINER_IMAGE: string;
    CONTAINER_TIMEOUT: number;
    CONTAINER_MAX_OUTPUT_SIZE: number;
    MAX_CONCURRENT_CONTAINERS: number;
}

// Provider configuration through environment
export interface ProviderConfig {
    ANTHROPIC_API_KEY?: string;
    OPENAI_API_KEY?: string;
    DEFAULT_MODEL?: string;
}
```

**特点：**
- 使用 Claude Agent SDK 内置的 Provider 管理
- 容器隔离保证安全性
- 不直接管理 Provider 生命周期

---

### 6. Nanobot - Python ABC + LiteLLM 架构

#### LLMProvider ABC

```python
# nanobot/providers/base.py
from abc import ABC, abstractmethod
from typing import AsyncGenerator, List, Optional

class LLMProvider(ABC):
    """Abstract base class for LLM providers."""

    @abstractmethod
    async def chat(
        self,
        messages: List[Message],
        model: Optional[str] = None,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        tools: Optional[List[Tool]] = None,
    ) -> LLMResponse:
        """Send chat request and return response."""
        pass

    @abstractmethod
    def get_default_model(self) -> str:
        """Return default model ID for this provider."""
        pass

    async def chat_stream(
        self,
        messages: List[Message],
        **kwargs
    ) -> AsyncGenerator[str, None]:
        """Stream chat response (optional)."""
        response = await self.chat(messages, **kwargs)
        yield response.content

    async def chat_with_retry(
        self,
        messages: List[Message],
        max_retries: int = 3,
        **kwargs
    ) -> LLMResponse:
        """Chat with exponential backoff retry."""
        delays = [1, 2, 4]  # Exponential backoff delays

        for attempt in range(max_retries + 1):
            try:
                return await self.chat(messages, **kwargs)
            except Exception as e:
                if attempt >= max_retries:
                    raise
                await asyncio.sleep(delays[attempt])
```

#### LiteLLM 集成

```python
# nanobot/providers/litellm_provider.py
import litellm
from litellm import acompletion

class LiteLLMProvider(LLMProvider):
    """Provider using LiteLLM for multi-provider support."""

    def __init__(self, model: str, api_key: Optional[str] = None):
        self.model = model
        self.api_key = api_key

    async def chat(
        self,
        messages: List[Message],
        model: Optional[str] = None,
        **kwargs
    ) -> LLMResponse:
        response = await acompletion(
            model=model or self.model,
            messages=[m.to_dict() for m in messages],
            api_key=self.api_key,
            **kwargs
        )

        return LLMResponse(
            content=response.choices[0].message.content,
            usage=TokenUsage(
                prompt_tokens=response.usage.prompt_tokens,
                completion_tokens=response.usage.completion_tokens,
                total_tokens=response.usage.total_tokens,
            ),
            cost=litellm.completion_cost(response),  # LiteLLM cost tracking
        )

    def get_default_model(self) -> str:
        return self.model
```

#### Custom OpenAI-compatible Provider

```python
# nanobot/providers/custom.py
from openai import AsyncOpenAI

class CustomProvider(LLMProvider):
    """Custom provider for OpenAI-compatible endpoints."""

    def __init__(self, base_url: str, api_key: str, model: str):
        self.client = AsyncOpenAI(base_url=base_url, api_key=api_key)
        self.model = model

    async def chat(self, messages: List[Message], **kwargs) -> LLMResponse:
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[m.to_dict() for m in messages],
            **kwargs
        )

        return LLMResponse(
            content=response.choices[0].message.content,
            usage=TokenUsage(
                prompt_tokens=response.usage.prompt_tokens,
                completion_tokens=response.usage.completion_tokens,
            ),
        )
```

---

### 7. MimiClaw - C Struct 嵌入式架构

#### LLM Proxy Header

```c
// main/llm/llm_proxy.h
#ifndef LLM_PROXY_H
#define LLM_PROXY_H

#include <stdbool.h>
#include "cJSON.h"

typedef struct {
    char *name;
    char *description;
    cJSON *parameters;
} llm_tool_t;

typedef struct {
    char *name;
    cJSON *arguments;
} llm_tool_call_t;

typedef struct {
    char *content;
    llm_tool_call_t *tool_calls;
    int tool_call_count;
    int prompt_tokens;
    int completion_tokens;
} llm_response_t;

// Provider selection
void llm_set_provider(const char *provider);
void llm_set_model(const char *model);
void llm_set_api_key(const char *api_key);

// Chat with tools
llm_response_t* llm_chat_tools(
    const char *system_prompt,
    const char *user_message,
    llm_tool_t *tools,
    int tool_count
);

// Streaming support with chunked encoding
void llm_chat_stream(
    const char *message,
    void (*on_chunk)(const char *chunk, void *user_data),
    void *user_data
);

void llm_response_free(llm_response_t *response);

#endif
```

#### ESP32 NVS 配置

```c
// Configuration stored in ESP32 NVS
#define NVS_NAMESPACE "llm_config"
#define KEY_PROVIDER "provider"      // "openai" or "anthropic"
#define KEY_MODEL "model"            // Model ID
#define KEY_API_KEY "api_key"        // Encrypted API key
#define KEY_TEMPERATURE "temp"       // Temperature setting
```

---

### 8. OpenClaw - Gateway-based Provider 架构

#### Gateway Provider 架构

```typescript
// Gateway-based provider routing
interface GatewayProvider {
    id: string;
    name: string;
    endpoint: string;
    authType: 'api_key' | 'oauth' | 'bearer';
    healthCheck: () => Promise<boolean>;
    route: (request: LLMRequest) => Promise<LLMResponse>;
}

// Provider registry with wizard-based setup
class ProviderRegistry {
    private providers: Map<string, GatewayProvider> = new Map();

    register(provider: GatewayProvider): void {
        this.providers.set(provider.id, provider);
    }

    async route(request: LLMRequest, preferredProvider?: string): Promise<LLMResponse> {
        if (preferredProvider && this.providers.has(preferredProvider)) {
            const provider = this.providers.get(preferredProvider)!;
            if (await provider.healthCheck()) {
                return await provider.route(request);
            }
        }

        // Fallback to first healthy provider
        for (const [id, provider] of this.providers) {
            if (await provider.healthCheck()) {
                return await provider.route(request);
            }
        }

        throw new Error('No healthy providers available');
    }
}
```

---

## 三、编程代理项目 Provider 详解

### 9. Aider - LiteLLM Wrapper 架构

#### Model 元数据与成本追踪

```python
# aider/models.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class ModelInfo:
    name: str
    max_tokens: int
    max_input_tokens: Optional[int] = None
    max_output_tokens: Optional[int] = None
    input_cost_per_token: float = 0.0
    output_cost_per_token: float = 0.0
    supports_vision: bool = False
    supports_tool_use: bool = False

# Built-in model registry
MODELS = {
    "gpt-4o": ModelInfo(
        name="gpt-4o",
        max_tokens=128000,
        max_output_tokens=4096,
        input_cost_per_token=5.0 / 1_000_000,
        output_cost_per_token=15.0 / 1_000_000,
        supports_vision=True,
        supports_tool_use=True,
    ),
    "claude-3-5-sonnet-20241022": ModelInfo(
        name="claude-3-5-sonnet-20241022",
        max_tokens=200000,
        max_output_tokens=8192,
        input_cost_per_token=3.0 / 1_000_000,
        output_cost_per_token=15.0 / 1_000_000,
        supports_vision=True,
        supports_tool_use=True,
    ),
    # ... many more models
}
```

#### LiteLLM 包装器

```python
# aider/llm.py
from litellm import completion, acompletion
import litellm

class LazyLiteLLM:
    """Lazy loading wrapper for LiteLLM."""

    def __init__(self):
        self._model = None

    def set_model(self, model_name: str):
        self._model = model_name
        litellm.drop_params = True  # Drop unsupported params

    def complete(self, messages: List[Dict], **kwargs) -> Dict:
        response = completion(
            model=self._model,
            messages=messages,
            **kwargs
        )
        return response

    async def acomplete(self, messages: List[Dict], **kwargs) -> Dict:
        response = await acompletion(
            model=self._model,
            messages=messages,
            **kwargs
        )
        return response

# Usage with retry
class LLM:
    def __init__(self, model_name: str):
        self.lazy_llm = LazyLiteLLM()
        self.lazy_llm.set_model(model_name)

    def chat(self, messages: List[Dict]) -> str:
        response = self.lazy_llm.complete(messages)
        return response.choices[0].message.content
```

---

### 10. Codex - Rust Provider Struct 架构

#### Provider 配置

```rust
// codex-rs/codex-api/src/provider.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Provider {
    pub name: String,
    pub base_url: String,
    pub api_key: Option<String>,
    pub default_model: String,
    pub retry_config: RetryConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RetryConfig {
    pub max_attempts: u32,
    pub base_delay_ms: u64,
    pub retry_429: bool,       // Retry on rate limit
    pub retry_5xx: bool,       // Retry on server error
    pub retry_transport: bool, // Retry on transport error
}

impl Default for RetryConfig {
    fn default() -> Self {
        RetryConfig {
            max_attempts: 3,
            base_delay_ms: 1000,
            retry_429: true,
            retry_5xx: true,
            retry_transport: true,
        }
    }
}
```

#### ModelClient 与会话管理

```rust
// Session-scoped API client
pub struct ModelClient {
    provider: Provider,
    http_client: reqwest::Client,
}

impl ModelClient {
    pub async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, Error> {
        let mut attempt = 0;

        loop {
            match self.try_complete(&request).await {
                Ok(response) => return Ok(response),
                Err(e) if self.should_retry(&e, attempt) => {
                    attempt += 1;
                    if attempt >= self.provider.retry_config.max_attempts {
                        return Err(e);
                    }
                    let delay = self.calculate_backoff(attempt);
                    tokio::time::sleep(delay).await;
                }
                Err(e) => return Err(e),
            }
        }
    }

    fn should_retry(&self, error: &Error, attempt: u32) -> bool {
        if attempt >= self.provider.retry_config.max_attempts {
            return false;
        }

        match error {
            Error::RateLimit if self.provider.retry_config.retry_429 => true,
            Error::ServerError(_) if self.provider.retry_config.retry_5xx => true,
            Error::Transport(_) if self.provider.retry_config.retry_transport => true,
            _ => false,
        }
    }

    fn calculate_backoff(&self, attempt: u32) -> Duration {
        let base = self.provider.retry_config.base_delay_ms;
        let delay = base * 2_u64.pow(attempt);
        Duration::from_millis(delay)
    }
}

// Turn-scoped streaming
pub struct ModelClientSession {
    client: ModelClient,
    stream: Option<WebSocketStream>,
}

impl ModelClientSession {
    pub async fn stream_completion(
        &mut self,
        request: CompletionRequest,
    ) -> Result<impl Stream<Item = StreamEvent>, Error> {
        // WebSocket prewarm and connection reuse
        if self.stream.is_none() {
            self.stream = Some(self.connect_websocket().await?);
        }

        // Send request and return stream
        let stream = self.stream.as_mut().unwrap();
        // ... streaming logic
    }
}
```

---

### 11. Pi-mono - TypeScript ApiProvider Interface 架构

#### KnownApi 与 KnownProvider 类型

```typescript
// packages/ai/src/types.ts
export type KnownApi =
    | 'openai-completions'
    | 'openai-responses'
    | 'azure-openai-responses'
    | 'anthropic-messages'
    | 'bedrock-converse-stream'
    | 'google-generative-ai'
    | 'mistral-conversations'
    | 'xai';

export type KnownProvider =
    | 'amazon-bedrock'
    | 'anthropic'
    | 'google'
    | 'openai'
    | 'azure-openai-responses'
    | 'github-copilot'
    | 'xai'
    | 'groq'
    | 'cerebras'
    | 'openrouter'
    | 'vercel-ai-gateway'
    | 'mistral'
    | 'minimax'
    | 'huggingface';

export interface ApiProvider {
    api: string;
    stream: boolean;
    streamSimple: boolean;

    // Core methods
    generate(request: GenerateRequest): Promise<GenerateResponse>;
    stream(request: StreamRequest): AsyncIterable<StreamEvent>;
}
```

#### AssistantMessageEventStream 协议

```typescript
// packages/ai/src/streams.ts
export type AssistantMessageEvent =
    | { type: 'text_delta'; content: string }
    | { type: 'thinking_delta'; content: string }
    | { type: 'toolcall_start'; id: string; name: string }
    | { type: 'toolcall_end'; id: string; result: unknown }
    | { type: 'usage'; usage: Usage }
    | { type: 'finish'; reason: FinishReason };

export interface Usage {
    input: number;
    output: number;
    cacheRead?: number;
    cacheWrite?: number;
    totalTokens: number;
    cost: {
        input: number;      // Cost in USD
        output: number;
        cacheRead?: number;
        cacheWrite?: number;
        total: number;
    };
}

// Stream implementation with comprehensive event types
export class AssistantMessageEventStream {
    async *[Symbol.asyncIterator](): AsyncIterator<AssistantMessageEvent> {
        // SSE or WebSocket transport
        for await (const chunk of this.transport) {
            yield this.parseEvent(chunk);
        }
    }
}
```

#### Model Registry with Cost Calculation

```typescript
// packages/ai/src/registry.ts
interface ModelDefinition {
    id: string;
    provider: KnownProvider;
    api: KnownApi;
    contextWindow: number;
    maxOutputTokens: number;
    pricing: {
        input: number;          // $ per million tokens
        output: number;         // $ per million tokens
        cacheRead?: number;     // $ per million tokens
        cacheWrite?: number;    // $ per million tokens
    };
    capabilities: {
        vision: boolean;
        toolUse: boolean;
        streaming: boolean;
    };
}

export function calculateCost(
    model: ModelDefinition,
    usage: { input: number; output: number; cacheRead?: number; cacheWrite?: number }
): number {
    const inputCost = (usage.input / 1_000_000) * model.pricing.input;
    const outputCost = (usage.output / 1_000_000) * model.pricing.output;
    const cacheReadCost = usage.cacheRead
        ? (usage.cacheRead / 1_000_000) * (model.pricing.cacheRead ?? 0)
        : 0;
    const cacheWriteCost = usage.cacheWrite
        ? (usage.cacheWrite / 1_000_000) * (model.pricing.cacheWrite ?? 0)
        : 0;

    return inputCost + outputCost + cacheReadCost + cacheWriteCost;
}
```

---

### 12. Qwen-code - ModelRegistry OAuth 架构

#### ModelRegistry 类

```typescript
// packages/core/src/models/modelRegistry.ts
export type AuthType = 'QWEN_OAUTH' | 'USE_OPENAI';

export interface ModelConfig {
    id: string;
    name: string;
    authType: AuthType;
    contextWindow: number;
    capabilities: {
        toolUse: boolean;
        vision: boolean;
    };
}

export class ModelRegistry {
    private models: Map<string, ModelConfig> = new Map();

    constructor() {
        this.registerBuiltInModels();
    }

    private registerBuiltInModels(): void {
        // Qwen models
        this.register({
            id: 'qwen-coder-plus',
            name: 'Qwen Coder Plus',
            authType: 'QWEN_OAUTH',
            contextWindow: 128000,
            capabilities: { toolUse: true, vision: true },
        });

        this.register({
            id: 'qwen-coder',
            name: 'Qwen Coder',
            authType: 'QWEN_OAUTH',
            contextWindow: 128000,
            capabilities: { toolUse: true, vision: false },
        });

        // OpenAI-compatible models
        this.register({
            id: 'user-openai-model',
            name: 'Custom OpenAI Model',
            authType: 'USE_OPENAI',
            contextWindow: 128000,
            capabilities: { toolUse: true, vision: true },
        });
    }

    getModelsForAuthType(authType: AuthType): ModelConfig[] {
        return Array.from(this.models.values()).filter(m => m.authType === authType);
    }

    getDefaultModelForAuthType(authType: AuthType): ModelConfig {
        const models = this.getModelsForAuthType(authType);
        return models[0]; // First model as default
    }

    getModel(id: string): ModelConfig | undefined {
        return this.models.get(id);
    }
}
```

#### OAuth 认证流程

```typescript
// OAuth-based authentication for Qwen
export class QwenAuthProvider {
    async authenticate(): Promise<AuthToken> {
        // OAuth 2.0 flow
        const code = await this.startOAuthFlow();
        const token = await this.exchangeCode(code);
        return token;
    }

    async refreshToken(token: AuthToken): Promise<AuthToken> {
        // Token refresh
        const response = await fetch('https://qwen.aliyun.com/oauth/refresh', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ refresh_token: token.refreshToken }),
        });
        return await response.json();
    }
}
```

---

### 13. Gemini-cli - Policy Chain 架构

#### Model 策略链

```typescript
// packages/core/src/availability/policyCatalog.ts
export type ModelChain = string[]; // Ordered list of models to try

export const DEFAULT_CHAIN: ModelChain = [
    'gemini-2.5-pro',
    'gemini-2.0-flash', // Last resort
];

export const FLASH_LITE_CHAIN: ModelChain = [
    'gemini-2.0-flash-lite',
    'gemini-2.0-flash',
    'gemini-2.5-pro',
];

export interface PolicyAction {
    type: 'prompt' | 'silent';
    message?: string;
}

export interface ModelPolicy {
    chain: ModelChain;
    onError: {
        rate_limit: PolicyAction;
        quota_exceeded: PolicyAction;
        service_unavailable: PolicyAction;
        unknown: PolicyAction;
    };
}

export class PolicyCatalog {
    private policies: Map<string, ModelPolicy> = new Map();

    getModelPolicyChain(policyName: string): ModelChain {
        const policy = this.policies.get(policyName);
        return policy?.chain ?? DEFAULT_CHAIN;
    }

    createSingleModelChain(model: string): ModelChain {
        return [model];
    }

    validateModelPolicyChain(chain: ModelChain): boolean {
        // Ensure all models in chain exist
        return chain.every(model => this.isValidModel(model));
    }
}
```

#### 模型别名解析

```typescript
// packages/core/src/config/models.ts
export type ModelAlias = 'auto' | 'pro' | 'flash' | 'flash-lite';

const MODEL_ALIASES: Record<ModelAlias, string> = {
    'auto': 'gemini-2.0-flash',
    'pro': 'gemini-2.5-pro',
    'flash': 'gemini-2.0-flash',
    'flash-lite': 'gemini-2.0-flash-lite',
};

export function resolveModel(alias: ModelAlias | string): string {
    if (alias in MODEL_ALIASES) {
        return MODEL_ALIASES[alias as ModelAlias];
    }
    return alias; // Return as-is if not an alias
}

export function isPreviewModel(model: string): boolean {
    return model.includes('preview') || model.includes('exp');
}

export function isProModel(model: string): boolean {
    return model.includes('pro');
}
```

---

### 14. Kimi-cli - kosong Protocol 架构

#### ChatProvider Protocol

```python
# packages/kosong/src/kosong/chat_provider/__init__.py
from typing import Protocol, runtime_checkable, AsyncIterator, Sequence
from pydantic import BaseModel

@runtime_checkable
class ChatProvider(Protocol):
    """The interface of chat providers."""
    name: str

    @property
    def model_name(self) -> str: ...

    @property
    def thinking_effort(self) -> "ThinkingEffort | None": ...

    async def generate(
        self,
        system_prompt: str,
        tools: Sequence[Tool],
        history: Sequence[Message],
    ) -> "StreamedMessage": ...

    def with_thinking(self, effort: ThinkingEffort) -> Self: ...


@runtime_checkable
class RetryableChatProvider(Protocol):
    """Optional interface for providers that support retry."""
    def on_retryable_error(self, error: Exception, attempt: int) -> None: ...
```

#### StreamedMessage Protocol

```python
class StreamedMessagePart(BaseModel):
    """Base class for streamed message parts."""
    pass

class TextPart(StreamedMessagePart):
    text: str

class ToolCallPart(StreamedMessagePart):
    tool_name: str
    arguments: dict

class ReasoningPart(StreamedMessagePart):
    reasoning: str

class TokenUsage(BaseModel):
    input_other: int
    output: int
    input_cache_read: int = 0
    input_cache_creation: int = 0

class StreamedMessage:
    """Protocol for streamed message responses."""
    async def __aiter__(self) -> AsyncIterator[StreamedMessagePart]: ...
    async def get_full_response(self) -> str: ...
    def usage(self) -> TokenUsage: ...
```

#### 支持的 Provider Types

```python
# Provider type enumeration
PROVIDER_TYPES = {
    'kimi': KimiChatProvider,
    'openai_legacy': OpenAILegacyChatProvider,
    'openai_responses': OpenAIResponsesChatProvider,
    'anthropic': AnthropicChatProvider,
    'gemini': GeminiChatProvider,
    'vertexai': VertexAIChatProvider,
    '_echo': EchoChatProvider,          # Testing
    '_scripted_echo': ScriptedEchoProvider,  # Testing
    '_chaos': ChaosChatProvider,        # Failure testing
}
```

---

### 15. Opencode - Vercel AI SDK 架构

#### Provider Factory

```typescript
// packages/opencode/src/provider/provider.ts
import { createOpenAI } from '@ai-sdk/openai';
import { createAnthropic } from '@ai-sdk/anthropic';
import { createGoogleGenerativeAI } from '@ai-sdk/google';
import { createAmazonBedrock } from '@ai-sdk/amazon-bedrock';
import { createOpenRouter } from '@openrouter/ai-sdk-provider';
// ... 15+ providers

export interface Provider {
    chat: {
        model: LanguageModelV2;
        capabilities: ModelCapabilities;
    };
}

export async function fromConfig(config: Config.Info): Promise<Provider> {
    const client = await getClient(config);

    return {
        chat: {
            model: client,
            capabilities: getCapabilities(config.provider),
        },
    };
}

async function getClient(config: Config.Info): Promise<LanguageModelV2> {
    switch (config.provider) {
        case 'openai':
            return createOpenAI({ apiKey: config.apiKey })(config.model);
        case 'anthropic':
            return createAnthropic({ apiKey: config.apiKey })(config.model);
        case 'google':
            return createGoogleGenerativeAI({ apiKey: config.apiKey })(config.model);
        case 'bedrock':
            return createAmazonBedrock({ ...config.credentials })(config.model);
        case 'openrouter':
            return createOpenRouter({ apiKey: config.apiKey })(config.model);
        case 'github_copilot':
            return createGitHubCopilotOpenAICompatible({ token: config.token })(config.model);
        // ... 12+ more providers
        default:
            throw new Error(`Unknown provider: ${config.provider}`);
    }
}
```

#### Model 定义与成本追踪

```typescript
// packages/opencode/src/provider/models.ts
import { z } from 'zod';

export const Model = z.object({
    id: z.string(),
    name: z.string(),
    cost: z.object({
        input: z.number(),           // Input token cost per 1M
        output: z.number(),          // Output token cost per 1M
        cache_read: z.number().optional(),
        cache_write: z.number().optional(),
        context_over_200k: z.object({
            input: z.number(),
            output: z.number(),
        }).optional(),
    }).optional(),
    limit: z.object({
        context: z.number(),         // Context window size
        input: z.number().optional(),
        output: z.number(),
    }),
    // Capabilities
    tool_call: z.boolean(),
    reasoning: z.boolean(),
    attachment: z.boolean(),
});

export type Model = z.infer<typeof Model>;

// Fetched from models.dev API
export async function fetchModels(): Promise<Model[]> {
    const response = await fetch('https://models.dev/api/models');
    const data = await response.json();
    return data.models.map(Model.parse);
}
```

#### Streaming with Vercel AI SDK

```typescript
// packages/opencode/src/session/processor.ts
import { streamText } from 'ai';

const stream = await streamText({
    model: provider.chat.model,
    messages,
    tools,
    onChunk: (chunk) => {
        switch (chunk.type) {
            case 'text-delta':
                // Handle text streaming
                break;
            case 'tool-call':
                // Handle tool call
                await handleToolCall(chunk);
                break;
            case 'tool-result':
                // Handle tool result
                await handleToolResult(chunk);
                break;
        }
    },
});

// Full stream with event types
for await (const value of stream.fullStream) {
    switch (value.type) {
        case 'start':
            SessionStatus.set(input.sessionID, { type: 'busy' });
            break;
        case 'reasoning-start':
            // Handle reasoning start
            break;
        case 'reasoning-delta':
            // Stream reasoning content
            break;
        case 'tool-call':
            await handleToolCall(value);
            break;
        case 'finish':
            // Complete
            return;
    }
}
```

---

## 四、Streaming 实现对比

### 个人助手项目 Streaming 对比

| 项目 | 实现方式 | 事件类型 | 传输协议 |
|------|---------|---------|----------|
| **CoPaw** | AgentScope AsyncGenerator | ChatResponse | HTTP/SSE |
| **IronClaw** | SSE Stream | StreamChunk | SSE |
| **NanoClaw** | N/A (Container) | N/A | N/A |
| **ZeroClaw** | SSE Stream | StreamChunk | SSE |
| **MimiClaw** | HTTP Chunked | Text chunks | HTTP |
| **PicoClaw** | Go Channels | StreamEvent | Channels/SSE |
| **Nanobot** | Async Generators | Token chunks | AsyncIterable |
| **OpenClaw** | SSE/WebSocket | MessageEvent | SSE/WebSocket |

### 编程代理项目 Streaming 对比

| 项目 | 实现方式 | 事件类型 | 传输协议 |
|------|---------|---------|----------|
| **Aider** | LiteLLM Streaming | Token chunks | LiteLLM |
| **Codex** | WebSocket | StreamEvent | WebSocket |
| **Qwen-code** | SSE | Token chunks | SSE |
| **Gemini-cli** | Event-based | Gemini events | HTTP |
| **Pi-mono** | AsyncIterator | AssistantMessageEvent | SSE/WebSocket |
| **Kimi-cli** | AsyncIterator | StreamedMessagePart | AsyncIterable |
| **Opencode** | Vercel AI SDK | StreamChunk | Vercel SDK |

### Streaming 事件类型对比

```python
# CoPaw (AgentScope)
ChatResponse(
    content=[{"type": "text", "text": "..."}],
    usage=ChatUsage(input_tokens=10, output_tokens=20)
)

# IronClaw (Rust)
StreamChunk::Text(String)
StreamChunk::ToolCall(ToolCall)
StreamChunk::Usage(TokenUsage)
StreamChunk::Finish(FinishReason)

# PicoClaw (Go)
type StreamEvent struct {
    Type    string      // "text", "tool_call", "usage", "error"
    Content interface{}
}

# Nanobot (Python)
async def chat_stream(...) -> AsyncGenerator[str, None]:
    yield "token1"
    yield "token2"

# Pi-mono (TypeScript)
type AssistantMessageEvent =
    | { type: 'text_delta'; content: string }
    | { type: 'thinking_delta'; content: string }
    | { type: 'toolcall_start'; id: string; name: string }
    | { type: 'toolcall_end'; id: string; result: unknown }
    | { type: 'usage'; usage: Usage }
    | { type: 'finish'; reason: FinishReason };

# Opencode (Vercel AI SDK)
type StreamChunk =
    | { type: 'text-delta'; textDelta: string }
    | { type: 'tool-call'; toolCall: ToolCall }
    | { type: 'tool-result'; toolResult: ToolResult }
    | { type: 'finish'; finishReason: FinishReason };
```

---

## 五、成本追踪机制对比

### 个人助手项目成本追踪

| 项目 | 追踪粒度 | 存储方式 | 特色功能 |
|------|---------|---------|----------|
| **CoPaw** | Provider + Model + Date | JSON 文件 | TokenUsageManager，多维度聚合 |
| **IronClaw** | Token level | In-memory + DB | Decimal precision，Decorator 层计算 |
| **NanoClaw** | None | N/A | Container-based，no tracking |
| **ZeroClaw** | Basic tokens | Memory | TokenUsage struct |
| **MimiClaw** | None | N/A | Embedded system，no tracking |
| **PicoClaw** | Basic | In-memory | Per-provider cost calc |
| **Nanobot** | Via LiteLLM | LiteLLM | LiteLLM ecosystem |
| **OpenClaw** | Usage metrics | Session storage | Usage tracking |

### 编程代理项目成本追踪

| 项目 | 追踪粒度 | 存储方式 | 特色功能 |
|------|---------|---------|----------|
| **Aider** | Model level | YAML config | Model-based pricing |
| **Codex** | Azure-specific | Azure metrics | Azure cost tracking |
| **Qwen-code** | Basic | Memory | Qwen usage tracking |
| **Gemini-cli** | API-side | Google Cloud | Gemini API billing |
| **Pi-mono** | $/M tokens | Registry | Comprehensive cost calc |
| **Kimi-cli** | TokenUsage | In-memory | Cache tracking |
| **Opencode** | Model config | models.dev API | Auto-refresh pricing |

### 成本计算示例

```python
# CoPaw Multi-dimensional aggregation
class TokenUsageSummary(BaseModel):
    total_prompt_tokens: int
    total_completion_tokens: int
    total_calls: int
    by_model: dict[str, TokenUsageByModel]
    by_provider: dict[str, TokenUsageStats]
    by_date: dict[str, TokenUsageStats]

# IronClaw Decimal precision
fn calculate_cost(&self, usage: &TokenUsage, model_id: &str) -> Option<Decimal> {
    let pricing = self.get_pricing(model_id)?;
    let input_cost = Decimal::from(usage.prompt_tokens) * pricing.input_per_token;
    let output_cost = Decimal::from(usage.completion_tokens) * pricing.output_per_token;
    Some(input_cost + output_cost)
}

# Pi-mono Comprehensive cost calculation
function calculateCost(model: ModelDefinition, usage: Usage): number {
    const inputCost = (usage.input / 1_000_000) * model.pricing.input;
    const outputCost = (usage.output / 1_000_000) * model.pricing.output;
    const cacheReadCost = usage.cacheRead
        ? (usage.cacheRead / 1_000_000) * (model.pricing.cacheRead ?? 0)
        : 0;
    return inputCost + outputCost + cacheReadCost;
}

# Opencode Model config with cost
const Model = z.object({
    cost: z.object({
        input: z.number(),              // per 1M tokens
        output: z.number(),
        cache_read: z.number().optional(),
        cache_write: z.number().optional(),
        context_over_200k: z.object({   // Special pricing for large context
            input: z.number(),
            output: z.number(),
        }).optional(),
    }),
});
```

---

## 六、Fallback 与 Retry 机制对比

### Fallback 机制对比表

| 项目 | 机制类型 | 实现方式 | 特色功能 |
|------|---------|---------|----------|
| **CoPaw** | Retry Wrapper | RetryChatModel class | 指数退避，状态码检测 |
| **IronClaw** | Decorator Chain | Layered providers | Full chain: Retry → SmartRouting → Failover → CircuitBreaker |
| **NanoClaw** | Concurrent Limit | MAX_CONCURRENT_CONTAINERS | Resource management |
| **ZeroClaw** | None | N/A | No fallback |
| **MimiClaw** | None | N/A | No fallback (embedded) |
| **PicoClaw** | FallbackChain | ProviderCandidate list | Cooldown tracking |
| **Nanobot** | LiteLLM | Built-in retry | LiteLLM ecosystem |
| **OpenClaw** | Gateway Routing | Health check-based | Session-level failover |
| **Aider** | LiteLLM | Built-in | LiteLLM retry |
| **Codex** | RetryConfig | Configurable | Retry 429, 5xx, transport |
| **Qwen-code** | Auth-type switch | ModelRegistry | Within auth type only |
| **Gemini-cli** | Policy Chains | ModelChain | DEFAULT_CHAIN, FLASH_LITE_CHAIN |
| **Pi-mono** | Configurable retry | maxRetryDelayMs | Per-provider config |
| **Kimi-cli** | RetryableChatProvider | Protocol-based | on_retryable_error callback |
| **Opencode** | Provider error handling | Model-level | Auto-refresh from models.dev |

### 推荐 Retry Wrapper 实现

```typescript
// Recommended Retry Model (融合 CoPaw + IronClaw + Pi-mono)
class RetryModel implements LanguageModel {
    constructor(
        private inner: LanguageModel,
        private maxRetries: number = 3,
        private baseDelayMs: number = 1000,
        private maxDelayMs: number = 60000,
        private retryableStatusCodes: Set<number> = new Set([429, 500, 502, 503, 504]),
    ) {}

    async doGenerate(options: GenerateOptions): Promise<GenerateResult> {
        for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
            try {
                return await this.inner.doGenerate(options);
            } catch (error) {
                if (!this.isRetryable(error) || attempt === this.maxRetries) {
                    throw error;
                }
                const delay = this.calculateBackoff(attempt);
                await this.sleep(delay);
            }
        }
        throw new Error("Max retries exceeded");
    }

    private isRetryable(error: Error): boolean {
        if (error instanceof RateLimitError) return true;
        if (error instanceof ServerError) return true;
        const statusCode = (error as any).statusCode;
        return statusCode && this.retryableStatusCodes.has(statusCode);
    }

    private calculateBackoff(attempt: number): number {
        // Exponential backoff with jitter
        const exponential = this.baseDelayMs * Math.pow(2, attempt - 1);
        const jitter = Math.random() * 1000;
        return Math.min(exponential + jitter, this.maxDelayMs);
    }

    private sleep(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Recommended Failover Chain (融合 IronClaw + PicoClaw + Gemini-cli)
class FailoverChain implements LanguageModel {
    private providers: ProviderCandidate[] = [];
    private cooldowns: Map<string, Date> = new Map();
    private cooldownDuration: number = 5 * 60 * 1000; // 5 minutes

    addProvider(provider: LanguageModel, priority: number = 0): void {
        this.providers.push({ provider, priority });
        this.providers.sort((a, b) => a.priority - b.priority);
    }

    async doGenerate(options: GenerateOptions): Promise<GenerateResult> {
        const available = this.getAvailableProviders();

        for (const candidate of available) {
            try {
                const result = await candidate.provider.doGenerate(options);
                this.clearCooldown(candidate.provider);
                return result;
            } catch (error) {
                this.markFailed(candidate.provider, error);
                continue;
            }
        }

        throw new Error("All providers failed");
    }

    private getAvailableProviders(): ProviderCandidate[] {
        return this.providers.filter(c => !this.isInCooldown(c.provider));
    }

    private isInCooldown(provider: LanguageModel): boolean {
        const cooldownEnd = this.cooldowns.get(provider.toString());
        return cooldownEnd ? new Date() < cooldownEnd : false;
    }

    private markFailed(provider: LanguageModel, error: Error): void {
        const duration = this.getCooldownDuration(error);
        this.cooldowns.set(
            provider.toString(),
            new Date(Date.now() + duration)
        );
    }

    private getCooldownDuration(error: Error): number {
        if (error instanceof RateLimitError) return 5 * 60 * 1000; // 5 min
        if (error instanceof AuthError) return 60 * 60 * 1000; // 1 hour
        return this.cooldownDuration;
    }
}
```

---

## 七、推荐统一 Provider 架构

基于15个项目的最佳实践，推荐以下融合架构：

```
┌─────────────────────────────────────────────────────────────────┐
│              Recommended Unified Provider Architecture          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Application Layer                                        │   │
│  │  - ProviderManager (Singleton/Registry)                   │   │
│  │  - ModelRouter (Local/Cloud/Subagent)                    │   │
│  │  - CostTracker (Multi-dimensional)                       │   │
│  │  - PolicyEngine (Chains + Fallbacks)                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Decorator Layer (IronClaw style)                   │ │ │
│  │  │                                                     │ │ │
│  │  │  ┌─────────────────────────────────────────────┐   │ │ │
│  │  │  │ RetryProvider                               │   │ │ │
│  │  │  │ - Exponential backoff + jitter              │   │ │ │
│  │  │  │ - Configurable retryable errors             │   │ │ │
│  │  │  └─────────────────────────────────────────────┘   │ │ │
│  │  │                          │                          │ │ │
│  │  │  ┌───────────────────────┼───────────────────────┐ │ │ │
│  │  │  │                       ▼                       │ │ │ │
│  │  │  │  ┌─────────────────────────────────────────┐ │ │ │ │
│  │  │  │  │ CircuitBreakerProvider                  │ │ │ │ │
│  │  │  │  │ - Health checking                         │ │ │ │ │
│  │  │  │  │ - Automatic recovery                      │ │ │ │ │
│  │  │  │  └─────────────────────────────────────────┘ │ │ │ │ │
│  │  │  │                      │                      │ │ │ │ │
│  │  │  │  ┌───────────────────┼───────────────────┐ │ │ │ │ │
│  │  │  │  │                   ▼                   │ │ │ │ │ │
│  │  │  │  │  ┌───────────────────────────────┐   │ │ │ │ │ │
│  │  │  │  │  │ FailoverProvider              │   │ │ │ │ │ │
│  │  │  │  │  │ - Multi-provider chains       │   │ │ │ │ │ │
│  │  │  │  │  │ - Cooldown management         │   │ │ │ │ │ │
│  │  │  │  │  │ - Policy-based routing        │   │ │ │ │ │ │
│  │  │  │  │  └───────────────────────────────┘   │ │ │ │ │ │
│  │  │  │  └───────────────────────────────────────┘ │ │ │ │ │
│  │  │  └─────────────────────────────────────────────┘ │ │ │ │
│  │  └───────────────────────────────────────────────────┘ │ │ │
│  └─────────────────────────────────────────────────────────┘ │ │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Provider Interface (Vercel AI SDK + Pi-mono)       │ │ │
│  │  │                                                     │ │ │
│  │  │  interface LanguageModel {                        │ │ │
│  │  │    doGenerate(options): Promise<GenerateResult>   │ │ │
│  │  │    doStream(options): Promise<Stream<Chunk>>      │ │ │
│  │  │    getCapabilities(): ModelCapabilities           │ │ │
│  │  │    calculateCost(usage): CostBreakdown            │ │ │
│  │  │  }                                                │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Provider Implementations                           │ │ │
│  │  │  - OpenAI Provider                                  │ │ │
│  │  │  - Anthropic Provider                               │ │ │
│  │  │  - Google Provider                                  │ │ │
│  │  │  - LiteLLM Provider (Aider/Nanobot style)          │ │ │
│  │  │  - OpenAI-Compatible Provider                       │ │ │
│  │  │  - Local Providers (Ollama, llama.cpp, MLX)        │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│  ┌───────────────────────────┼───────────────────────────────┐ │
│  │                           ▼                               │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Configuration (IronClaw + openclaw style)          │ │ │
│  │  │  - providers.json (Registry)                        │ │ │
│  │  │  - active_model.json                                │ │ │
│  │  │  - policy_chains.json (Gemini-cli style)           │ │ │
│  │  │  - token_usage/                                     │ │ │
│  │  │  - Environment variable overrides                   │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键实现代码

```typescript
// Recommended Unified Provider Interface
interface LanguageModel {
    readonly specificationVersion: "v1";
    readonly modelId: string;
    readonly provider: string;

    doGenerate(options: GenerateOptions): Promise<GenerateResult>;
    doStream(options: StreamOptions): Promise<StreamResult>;
    getCapabilities(): ModelCapabilities;
    calculateCost(usage: TokenUsage): CostBreakdown;
}

interface GenerateResult {
    text: string;
    toolCalls: ToolCall[];
    usage: TokenUsage;
    cost?: CostBreakdown;
    finishReason: FinishReason;
}

interface TokenUsage {
    promptTokens: number;
    completionTokens: number;
    cacheReadTokens?: number;
    cacheWriteTokens?: number;
    totalTokens: number;
}

interface CostBreakdown {
    input: number;
    output: number;
    cacheRead?: number;
    cacheWrite?: number;
    total: number;
    currency: string; // "USD"
}

// Recommended Provider Manager
class ProviderManager {
    private static instance: ProviderManager;
    private providers: Map<string, Provider> = new Map();
    private activeModel: ModelRef;
    private costTracker: CostTracker;
    private policyEngine: PolicyEngine;

    static getInstance(): ProviderManager {
        if (!ProviderManager.instance) {
            ProviderManager.instance = new ProviderManager();
        }
        return ProviderManager.instance;
    }

    async activateModel(providerId: string, modelId: string): Promise<void> {
        const provider = this.providers.get(providerId);
        if (!provider) {
            throw new Error(`Provider ${providerId} not found`);
        }
        if (!provider.hasModel(modelId)) {
            throw new Error(`Model ${modelId} not found in provider ${providerId}`);
        }
        this.activeModel = { provider: providerId, model: modelId };
        await this.saveActiveModel();
    }

    async generateWithFallback(options: GenerateOptions): Promise<GenerateResult> {
        const chain = this.policyEngine.getPolicyChain('default');

        for (const modelRef of chain) {
            try {
                const provider = this.getDecoratedProvider(modelRef.provider);
                const result = await provider.doGenerate({ ...options, model: modelRef.model });

                // Track cost
                await this.costTracker.record(result.usage, modelRef);

                return result;
            } catch (error) {
                this.policyEngine.markFailed(modelRef, error);
                continue;
            }
        }

        throw new Error("All providers in chain failed");
    }

    private getDecoratedProvider(providerId: string): LanguageModel {
        const raw = this.providers.get(providerId);

        // Apply decorator chain
        let decorated: LanguageModel = raw;
        decorated = new RetryModel(decorated);
        decorated = new CircuitBreakerModel(decorated);
        decorated = new CostTrackingModel(decorated, this.costTracker);

        return decorated;
    }
}

// Recommended Cost Tracker
class CostTracker {
    private records: TokenUsageRecord[] = [];
    private modelPricing: Map<string, ModelPricing>;

    async record(usage: TokenUsage, model: ModelRef): Promise<void> {
        const pricing = this.modelPricing.get(`${model.provider}/${model.model}`);
        const cost = this.calculateCost(usage, pricing);

        this.records.push({
            timestamp: new Date(),
            provider: model.provider,
            model: model.model,
            promptTokens: usage.promptTokens,
            completionTokens: usage.completionTokens,
            cacheReadTokens: usage.cacheReadTokens,
            cacheWriteTokens: usage.cacheWriteTokens,
            cost,
        });

        await this.persist();
    }

    private calculateCost(usage: TokenUsage, pricing?: ModelPricing): CostBreakdown {
        if (!pricing) {
            return { input: 0, output: 0, total: 0, currency: "USD" };
        }

        const inputCost = (usage.promptTokens / 1_000_000) * pricing.inputPerMillion;
        const outputCost = (usage.completionTokens / 1_000_000) * pricing.outputPerMillion;
        const cacheReadCost = usage.cacheReadTokens
            ? (usage.cacheReadTokens / 1_000_000) * (pricing.cacheReadPerMillion ?? 0)
            : 0;
        const cacheWriteCost = usage.cacheWriteTokens
            ? (usage.cacheWriteTokens / 1_000_000) * (pricing.cacheWritePerMillion ?? 0)
            : 0;

        return {
            input: inputCost,
            output: outputCost,
            cacheRead: cacheReadCost,
            cacheWrite: cacheWriteCost,
            total: inputCost + outputCost + cacheReadCost + cacheWriteCost,
            currency: "USD",
        };
    }

    getSummary(filters?: UsageFilters): UsageSummary {
        // Aggregate by provider, model, date
        const filtered = this.filterRecords(filters);

        return {
            totalCost: filtered.reduce((sum, r) => sum + r.cost.total, 0),
            totalTokens: filtered.reduce((sum, r) => sum + r.promptTokens + r.completionTokens, 0),
            byProvider: this.aggregateByProvider(filtered),
            byModel: this.aggregateByModel(filtered),
            byDate: this.aggregateByDate(filtered),
        };
    }
}
```

---

## 八、总结

### 各项目优势汇总

#### 个人助手项目

| 项目 | Provider 优势 | 独特机制 |
|------|-------------|---------|
| **CoPaw** | 完整的 Provider 管理 + 包装器 | TokenUsageManager + RoutingChatModel |
| **IronClaw** | 最完整的装饰器链 | Retry → SmartRouting → Failover → CircuitBreaker → Cached → Recording |
| **NanoClaw** | 容器隔离安全 | Container-based execution |
| **ZeroClaw** | 简洁的 Trait 设计 | async_trait with capabilities |
| **MimiClaw** | 嵌入式 ESP32 支持 | C struct-based，HTTP chunked |
| **PicoClaw** | Go Interface + FallbackChain | Cooldown tracking，20+ protocols |
| **Nanobot** | LiteLLM 生态集成 | ABC + LiteLLM |
| **OpenClaw** | Gateway-based 路由 | Health check-based failover |

#### 编程代理项目

| 项目 | Provider 优势 | 独特机制 |
|------|-------------|---------|
| **Aider** | LiteLLM Wrapper | Lazy loading，model metadata |
| **Codex** | WebSocket streaming | RetryConfig，ModelClientSession |
| **Qwen-code** | OAuth + ModelRegistry | Auth-type switching |
| **Gemini-cli** | Policy Chains | DEFAULT_CHAIN，FLASH_LITE_CHAIN |
| **Pi-mono** | 最全面的 API 支持 | 15+ providers，comprehensive cost calc |
| **Kimi-cli** | Protocol-based | ChatProvider Protocol，test providers |
| **Opencode** | Vercel AI SDK 生态 | models.dev API，auto-refresh |

### 推荐技术选型

| 组件 | 推荐实现 | 来源 |
|------|---------|------|
| **Provider Interface** | LanguageModelV2 (Vercel style) + Pi-mono events | Opencode + Pi-mono |
| **Decorator Chain** | Retry → CircuitBreaker → Failover → CostTracking | IronClaw |
| **Provider Manager** | Singleton + Registry + ModelRef | CoPaw + IronClaw |
| **Configuration** | JSON Registry + TOML user config | IronClaw + ZeroClaw |
| **Retry/Fallback** | Exponential backoff + Policy chains | CoPaw + Gemini-cli |
| **Cost Tracking** | Multi-dimensional + Decimal precision | IronClaw + Pi-mono |
| **Streaming** | Standardized event types | Vercel AI SDK + Pi-mono |
| **Local Models** | Ollama + llama.cpp + MLX + OpenAI-compatible | CoPaw + PicoClaw |
| **Multi-provider** | LiteLLM fallback + Custom providers | Nanobot + Aider |

### 最终推荐架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    最终推荐 Provider 架构                      │
├─────────────────────────────────────────────────────────────────┤
│ 1. 接口设计:    Vercel AI SDK LanguageModelV2 + Pi-mono Events │
│ 2. 装饰器链:    Retry → CircuitBreaker → Failover → CostTrack  │
│ 3. 管理器:      Singleton ProviderManager + JSON Registry      │
│ 4. 配置:        TOML user config + JSON Registry + Env vars    │
│ 5. 成本:        Decimal precision + Multi-dimensional (Iron)   │
│ 6. 流式:        Standardized event types (Vercel + Pi-mono)    │
│ 7. 本地模型:    Ollama + llama.cpp + MLX + OpenAI-compatible   │
│ 8. Fallback:    Policy chains + Cooldown tracking (Gemini+Pico)│
│ 9. 多 Provider: LiteLLM ecosystem + Custom implementations     │
│ 10. 测试:       Echo + ScriptedEcho + Chaos providers (Kimi)   │
└─────────────────────────────────────────────────────────────────┘
```

此方案融合了15个项目的最佳实践，兼顾标准化、扩展性、成本追踪和容错能力，适用于生产级 AI Agent 系统。
