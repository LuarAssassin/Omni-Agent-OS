# Channel Adapter Architecture Analysis: Multi-Platform Messaging Protocol

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Personal Assistant Channel Architectures](#personal-assistant-channel-architectures)
3. [Programming Agent I/O Patterns](#programming-agent-io-patterns)
4. [Architecture Comparison Matrix](#architecture-comparison-matrix)
5. [Recommended Unified Architecture](#recommended-unified-architecture)

---

## Core Concepts

### Channel Adapter Design Goals

| Challenge | Solution Direction |
|-----------|------------------|
| Multi-platform protocol differences | Unified message abstraction layer |
| Authentication method variations | Channel-independent config + unified auth interface |
| Media type differences | Content type system standardization |
| Message format incompatibility | Bidirectional converters (Native ↔ Internal) |
| Connection method variations | Connection management abstraction (WebSocket/HTTP/Long Polling) |

### Unified Protocol Key Components

```
┌─────────────────────────────────────────────────────────────────┐
│                 External Messaging Platforms                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │  Feishu │ │  Ding   │ │ Telegram│ │ Discord │ │ WhatsApp│   │
│  │WebSocket│ │ Stream  │ │  HTTP   │ │ Gateway │ │  Baileys│   │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘   │
└───────┼──────────┼──────────┼──────────┼──────────┼───────────┘
        │          │          │          │          │
        ▼          ▼          ▼          ▼          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Channel Adapter Layer                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Native Message Converter                               │   │
│  │  - Protocol parsing (Feishu Card / Ding Markdown)       │   │
│  │  - Authentication (Token/AppKey/SessionWebhook)         │   │
│  │  - Media handling (download/upload/format conversion)   │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Unified Message Format                                 │   │
│  │  - IncomingMessage (channel, sender_id, content)        │   │
│  │  - OutgoingMessage (content, attachments, thread_id)    │   │
│  │  - Attachment (id, kind, mime_type, data)               │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Agent Core Runtime                           │
│              (Processing, Memory, Tools, LLM)                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Personal Assistant Channel Architectures

### 1. CoPaw - Python BaseChannel with AgentScope

**Architecture Overview:**
CoPaw implements an 11-channel architecture using Python's AgentScope framework with a unified `BaseChannel` abstraction.

**Supported Channels:**
| Channel | Protocol | Features |
|---------|----------|----------|
| Feishu/Lark | WebSocket + Open API | Cards, mentions, image/file |
| DingTalk | Stream SDK | Group chat, markdown |
| Discord | Gateway WebSocket | Threads, reactions, slash commands |
| QQ | HTTP POST | Legacy support |
| iMessage | macOS Scripting Bridge | Native Apple integration |
| Matrix | Client-Server API | E2E encryption, federation |
| Mattermost | REST API | Enterprise team chat |
| WeCom | Callback + API | Enterprise WeChat |
| Telegram | Bot API | Webhooks, inline keyboards |
| MQTT | Pub/Sub | IoT integration |
| Console | stdin/stdout | Local debugging |

**Core BaseChannel Implementation:**
```python
# copaw/app/channels/base.py
class BaseChannel(ABC):
    """Unified channel interface for all messaging platforms."""

    def __init__(
        self,
        process: ProcessHandler,
        on_reply_sent: OnReplySent = None,
        show_tool_details: bool = True,
        dm_policy: str = "open",  # "open", "closed", "whitelist"
        group_policy: str = "open",
        allow_from: Optional[List[str]] = None,
        require_mention: bool = False,
    ):
        self.process = process
        self.dm_policy = dm_policy
        self.group_policy = group_policy
        self.allow_from = set(allow_from or [])
        self.require_mention = require_mention

    @abstractmethod
    async def start(self) -> None:
        """Start listening for messages (long-running)."""
        pass

    @abstractmethod
    async def send_reply(
        self,
        request: AgentRequest,
        contents: List[OutgoingContentPart],
    ) -> None:
        """Send response back to the channel."""
        pass

    async def _should_process(self, msg: IncomingMessage) -> bool:
        """Permission check: whitelist + mention policy."""
        if self.allow_from and msg.sender_id not in self.allow_from:
            return False
        if msg.is_group and self.require_mention and not msg.is_mentioned:
            return False
        return True
```

**Feishu Channel Implementation (WebSocket-based):**
```python
class FeishuChannel(BaseChannel):
    """Feishu/Lark: WebSocket receive + Open API send."""

    channel = "feishu"

    async def start(self) -> None:
        # WebSocket long connection for events
        ws_client = lark.Client.builder() \
            .app_id(self.app_id) \
            .app_secret(self.app_secret) \
            .build()

        ws_handler = lark.WebsocketHandler() \
            .on_message(self._on_message) \
            .on_error(self._on_error)

        await ws_handler.start()

    async def _on_message(self, event: P2ImMessageReceiveV1) -> None:
        # Convert Feishu message to unified format
        msg = IncomingMessage(
            channel=self.channel,
            sender_id=event.sender.sender_id.open_id,
            content=event.message.content,
            chat_type=event.message.chat_type,
            message_type=event.message.message_type,
        )

        if await self._should_process(msg):
            await self.process(msg)

    async def send_reply(self, request, contents) -> None:
        # Convert to Feishu interactive cards
        for part in contents:
            if part.type == ContentType.TEXT:
                await self._send_text(request, part.content)
            elif part.type == ContentType.IMAGE:
                await self._upload_and_send_image(request, part.path)
            elif part.type == ContentType.FILE:
                await self._upload_and_send_file(request, part.path)
```

---

### 2. NanoClaw - TypeScript Channel Registry

**Architecture Overview:**
NanoClaw uses a minimal TypeScript channel registry pattern with runtime channel discovery.

**Channel Registry Pattern:**
```typescript
// nanoclaw/src/channels/registry.ts
export interface Channel {
  name: string;
  start(): Promise<void>;
  stop(): Promise<void>;
  send(message: OutboundMessage): Promise<void>;
  isAllowed(senderId: string): boolean;
}

export class ChannelRegistry {
  private channels: Map<string, Channel> = new Map();

  register(name: string, factory: ChannelFactory): void {
    this.channels.set(name, factory);
  }

  async initializeEnabled(config: Config): Promise<Channel[]> {
    const enabled: Channel[] = [];
    for (const [name, factory] of this.channels) {
      if (config.channels[name]?.enabled) {
        enabled.push(await factory(config));
      }
    }
    return enabled;
  }
}
```

**Key Design Decisions:**
- Channels are loaded as separate skill packages
- Uses dependency injection for config/bus
- Supports hot-reloading of channel configurations

---

### 3. ZeroClaw - Rust Trait-Driven 25+ Channels

**Architecture Overview:**
ZeroClaw implements the most comprehensive channel architecture with 25+ channel implementations using Rust traits.

**Core Channel Trait:**
```rust
// zeroclaw/src/channels/traits.rs
#[async_trait]
pub trait Channel: Send + Sync {
    /// Channel name ("cli", "slack", "telegram", "http")
    fn name(&self) -> &str;

    /// Start listening for messages - returns stream
    async fn start(&self) -> Result<MessageStream, ChannelError>;

    /// Send response back to user
    async fn respond(
        &self,
        msg: &IncomingMessage,
        response: OutgoingResponse,
    ) -> Result<(), ChannelError>;

    /// Send status updates (typing, tool execution)
    async fn send_status(
        &self,
        status: StatusUpdate,
        metadata: &serde_json::Value,
    ) -> Result<(), ChannelError>;

    /// Broadcast proactive messages
    async fn broadcast(
        &self,
        user_id: &str,
        response: OutgoingResponse,
    ) -> Result<(), ChannelError>;

    /// Health check
    async fn health_check(&self) -> Result<(), ChannelError>;

    /// Get conversation context for LLM
    fn conversation_context(&self, metadata: &serde_json::Value)
        -> HashMap<String, String>;

    /// Graceful shutdown
    async fn shutdown(&self) -> Result<(), ChannelError>;
}

/// Hot-secret-swapping for zero-downtime credential updates
#[async_trait]
pub trait ChannelSecretUpdater: Send + Sync {
    async fn update_secret(&self, new_secret: Option<SecretString>);
}
```

**Supported Channels (ZeroClaw):**
| Category | Channels |
|----------|----------|
| Messaging | Telegram, Discord, Slack, Signal, WhatsApp, WeChat |
| Enterprise | Feishu/Lark, DingTalk, WeCom, Mattermost, Matrix |
| Social | Bluesky, Twitter/X, Reddit, Nostr, IRC |
| Protocol | Email, Webhook, MQTT, WebSocket |
| Local | CLI, TTS (Text-to-Speech) |
| Custom | ClawdTalk (inter-agent), Linq |

**Channel Message Types:**
```rust
pub struct IncomingMessage {
    pub id: Uuid,
    pub channel: String,
    pub user_id: String,
    pub user_name: Option<String>,
    pub content: String,
    pub thread_id: Option<String>,
    pub received_at: DateTime<Utc>,
    pub metadata: serde_json::Value,
    pub timezone: Option<String>,
    pub attachments: Vec<IncomingAttachment>,
}

pub struct IncomingAttachment {
    pub id: String,
    pub kind: AttachmentKind,  // Audio, Image, Document
    pub mime_type: String,
    pub filename: Option<String>,
    pub size_bytes: Option<u64>,
    pub source_url: Option<String>,
    pub storage_key: Option<String>,
    pub extracted_text: Option<String>,
    pub data: Vec<u8>,
    pub duration_secs: Option<u32>,
}

pub enum StatusUpdate {
    Thinking(String),
    ToolStarted { name: String },
    ToolCompleted { name: String, success: bool, error: Option<String> },
    ToolResult { name: String, preview: String },
    StreamChunk(String),
    ApprovalNeeded { request_id: String, tool_name: String, ... },
    AuthRequired { extension_name: String, ... },
    ImageGenerated { data_url: String, path: Option<String> },
}
```

---

### 4. IronClaw - Rust Channel Trait with Manager

**Architecture Overview:**
IronClaw uses a Rust trait-based channel system with a ChannelManager for stream merging.

**Channel Trait:**
```rust
// ironclaw/src/channels/channel.rs (simplified)
#[async_trait]
pub trait Channel: Send + Sync {
    fn name(&self) -> &str;
    async fn start(&self) -> Result<MessageStream, ChannelError>;
    async fn respond(&self, msg: &IncomingMessage, response: OutgoingResponse)
        -> Result<(), ChannelError>;
    async fn send_status(&self, status: StatusUpdate, metadata: &serde_json::Value)
        -> Result<(), ChannelError>;
    async fn broadcast(&self, user_id: &str, response: OutgoingResponse)
        -> Result<(), ChannelError>;
    async fn health_check(&self) -> Result<(), ChannelError>;
    fn conversation_context(&self, metadata: &serde_json::Value)
        -> HashMap<String, String>;
    async fn shutdown(&self) -> Result<(), ChannelError>;
}

pub struct IncomingMessage {
    pub id: Uuid,
    pub channel: String,
    pub user_id: String,
    pub user_name: Option<String>,
    pub content: String,
    pub thread_id: Option<String>,
    pub received_at: DateTime<Utc>,
    pub metadata: serde_json::Value,
    pub timezone: Option<String>,
    pub attachments: Vec<IncomingAttachment>,
}
```

**Channel Manager for Stream Merging:**
```rust
// ironclaw/src/channels/manager.rs
pub struct ChannelManager {
    channels: Vec<Arc<dyn Channel>>,
}

impl ChannelManager {
    pub async fn start_all(&self) -> Result<MessageStream, ChannelError> {
        let mut streams = Vec::new();
        for channel in &self.channels {
            streams.push(channel.start().await?);
        }
        // Merge all channel streams into one
        Ok(futures::stream::select_all(streams))
    }
}
```

---

### 5. PicoClaw - Go Channel Interface (15+ Channels)

**Architecture Overview:**
PicoClaw implements a Go interface-based channel system with 15+ channels and rate limiting.

**Core Channel Interface:**
```go
// picoclaw/pkg/channels/base.go
type Channel interface {
    Name() string
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Send(ctx context.Context, msg bus.OutboundMessage) error
    IsRunning() bool
    IsAllowed(senderID string) bool
    IsAllowedSender(sender bus.SenderInfo) bool
    ReasoningChannelID() string
}

// BaseChannel provides common functionality
type BaseChannel struct {
    config              any
    bus                 *bus.MessageBus
    running             atomic.Bool
    name                string
    allowList           []string
    maxMessageLength    int
    groupTrigger        config.GroupTriggerConfig
    mediaStore          media.MediaStore
    placeholderRecorder PlaceholderRecorder
    owner               Channel
    reasoningChannelID  string
}
```

**Channel Capabilities via Interface Segregation:**
```go
// picoclaw/pkg/channels/interfaces.go

// TypingCapable channels support typing indicators
type TypingCapable interface {
    Channel
    StartTyping(ctx context.Context, chatID string) (func(), error)
}

// ReactionCapable channels support emoji reactions
type ReactionCapable interface {
    Channel
    ReactToMessage(ctx context.Context, chatID, messageID string) (func(), error)
}

// PlaceholderCapable channels support "Thinking..." placeholders
type PlaceholderCapable interface {
    Channel
    SendPlaceholder(ctx context.Context, chatID string) (string, error)
}

// MessageEditor channels support message editing
type MessageEditor interface {
    Channel
    EditMessage(ctx context.Context, chatID, messageID, content string) error
}

// MediaCapable channels support media upload/download
type MediaCapable interface {
    Channel
    SetMediaStore(s media.MediaStore)
}
```

**Supported Channels:**
```go
// picoclaw/pkg/channels/manager.go
func (m *Manager) initChannels() {
    if m.config.Channels.Telegram.Enabled {
        m.initChannel("telegram", "Telegram")
    }
    if m.config.Channels.WhatsApp.Enabled {
        if m.config.Channels.WhatsApp.UseNative {
            m.initChannel("whatsapp_native", "WhatsApp Native")
        } else {
            m.initChannel("whatsapp", "WhatsApp")
        }
    }
    if m.config.Channels.Feishu.Enabled {
        m.initChannel("feishu", "Feishu")
    }
    if m.config.Channels.Discord.Enabled {
        m.initChannel("discord", "Discord")
    }
    if m.config.Channels.Slack.Enabled {
        m.initChannel("slack", "Slack")
    }
    if m.config.Channels.Matrix.Enabled {
        m.initChannel("matrix", "Matrix")
    }
    if m.config.Channels.LINE.Enabled {
        m.initChannel("line", "LINE")
    }
    if m.config.Channels.OneBot.Enabled {
        m.initChannel("onebot", "OneBot")
    }
    if m.config.Channels.WeCom.Enabled {
        m.initChannel("wecom", "WeCom")
    }
    // ... 15+ total channels
}
```

---

### 6. Nanobot - Python BaseChannel with Plugin Discovery

**Architecture Overview:**
Nanobot uses a plugin-based channel discovery system with pkgutil scanning and entry points.

**BaseChannel with Message Bus Integration:**
```python
# nanobot/channels/base.py
class BaseChannel(ABC):
    """Abstract base class for chat channel implementations."""

    name: str = "base"
    display_name: str = "Base"
    transcription_api_key: str = ""

    def __init__(self, config: Any, bus: MessageBus):
        self.config = config
        self.bus = bus
        self._running = False

    @abstractmethod
    async def start(self) -> None:
        """Start listening for messages."""
        pass

    @abstractmethod
    async def stop(self) -> None:
        """Stop the channel."""
        pass

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None:
        """Send a message."""
        pass

    def is_allowed(self, sender_id: str) -> bool:
        """Check if sender is permitted."""
        allow_list = getattr(self.config, "allow_from", [])
        if not allow_list:
            return False  # Empty list = deny all
        if "*" in allow_list:
            return True
        return str(sender_id) in allow_list

    async def _handle_message(
        self,
        sender_id: str,
        chat_id: str,
        content: str,
        media: list[str] | None = None,
        metadata: dict[str, Any] | None = None,
    ) -> None:
        if not self.is_allowed(sender_id):
            logger.warning("Access denied for sender %s on channel %s",
                         sender_id, self.name)
            return

        msg = InboundMessage(
            channel=self.name,
            sender_id=str(sender_id),
            chat_id=str(chat_id),
            content=content,
            media=media or [],
            metadata=metadata or {},
        )
        await self.bus.publish_inbound(msg)
```

**Channel Manager with Plugin Discovery:**
```python
# nanobot/channels/manager.py
class ChannelManager:
    def __init__(self, config: Config, bus: MessageBus):
        self.config = config
        self.bus = bus
        self.channels: dict[str, BaseChannel] = {}
        self._init_channels()

    def _init_channels(self) -> None:
        """Initialize channels via pkgutil scan + entry_points."""
        from nanobot.channels.registry import discover_all

        for name, cls in discover_all().items():
            section = getattr(self.config.channels, name, None)
            if not section or not section.get("enabled", False):
                continue
            try:
                channel = cls(section, self.bus)
                self.channels[name] = channel
                logger.info("%s channel enabled", cls.display_name)
            except Exception as e:
                logger.warning("%s channel not available: %s", name, e)

    async def _dispatch_outbound(self) -> None:
        """Dispatch outbound messages to appropriate channel."""
        while True:
            msg = await self.bus.consume_outbound()
            channel = self.channels.get(msg.channel)
            if channel:
                await channel.send(msg)
```

---

### 7. OpenClaw - Extension-Based Channel Architecture

**Architecture Overview:**
OpenClaw uses a sophisticated extension-based architecture with channel plugins.

**Channel Config Resolution:**
```typescript
// openclaw/src/channels/channel-config.ts
export type ChannelMatchSource = "direct" | "parent" | "wildcard";

export type ChannelEntryMatch<T> = {
  entry?: T;
  key?: string;
  wildcardEntry?: T;
  wildcardKey?: string;
  parentEntry?: T;
  parentKey?: string;
  matchKey?: string;
  matchSource?: ChannelMatchSource;
};

export function resolveChannelEntryMatchWithFallback<T>(params: {
  entries?: Record<string, T>;
  keys: string[];
  parentKeys?: string[];
  wildcardKey?: string;
  normalizeKey?: (value: string) => string;
}): ChannelEntryMatch<T> {
  // Try direct match first
  const direct = resolveChannelEntryMatch({...});
  if (direct.entry) return {...direct, matchSource: "direct"};

  // Try normalized match
  if (normalizeKey) { ... }

  // Try parent match (for nested channels like groups)
  if (parentKeys) { ... }

  // Fall back to wildcard
  if (direct.wildcardEntry) {
    return {...direct, entry: direct.wildcardEntry, matchSource: "wildcard"};
  }

  return direct;
}
```

**Extension Channel Example (Discord):**
```typescript
// openclaw/extensions/discord/src/channel.ts
export class DiscordChannel implements Channel {
  async start(): Promise<void> {
    this.client = new Client({
      intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.DirectMessages,
      ]
    });

    this.client.on(Events.MessageCreate, async (message) => {
      if (message.author.bot) return;

      const incoming: IncomingMessage = {
        channel: "discord",
        senderId: message.author.id,
        senderName: message.author.username,
        content: message.content,
        threadId: message.channel.isThread() ? message.channel.id : undefined,
        attachments: message.attachments.map(att => ({
          id: att.id,
          url: att.url,
          filename: att.name,
          size: att.size,
        })),
      };

      await this.handleIncoming(incoming);
    });

    await this.client.login(this.config.token);
  }

  async send(msg: OutboundMessage): Promise<void> {
    const channel = await this.client.channels.fetch(msg.threadId || msg.chatId);
    if (channel?.isTextBased()) {
      await channel.send(msg.content);
    }
  }
}
```

---

### 8. MimiClaw - ESP32/FreeRTOS Channel Abstraction

**Architecture Overview:**
MimiClaw runs on ESP32 with FreeRTOS and implements a minimal channel abstraction for resource-constrained environments.

**Lightweight Channel Interface:**
```c
// mimicloud/main/channels/base.h
typedef struct {
    const char* name;
    esp_err_t (*init)(void* config);
    esp_err_t (*start)(void);
    esp_err_t (*stop)(void);
    esp_err_t (*send)(const char* user_id, const char* content);
    bool (*is_allowed)(const char* sender_id);
} channel_interface_t;

// UART channel for wired debugging
static channel_interface_t uart_channel = {
    .name = "uart",
    .init = uart_channel_init,
    .start = uart_channel_start,
    .stop = uart_channel_stop,
    .send = uart_channel_send,
    .is_allowed = uart_channel_is_allowed,
};

// MQTT channel for cloud connectivity
static channel_interface_t mqtt_channel = {
    .name = "mqtt",
    .init = mqtt_channel_init,
    .start = mqtt_channel_start,
    .stop = mqtt_channel_stop,
    .send = mqtt_channel_send,
    .is_allowed = mqtt_channel_is_allowed,
};
```

---

## Programming Agent I/O Patterns

Programming agents (Aider, Codex, Qwen-code, Gemini-cli, Pi-mono, Kimi-cli, Opencode) typically don't use traditional messaging channel adapters. Instead, they use:

### 1. Aider - Git Committer as Channel

```python
# aider/coders/coder.py
class Coder:
    def __init__(self, ...):
        self.commit_before_message = []
        self.dirty_commits = []

    def run(self, message):
        """Main entry point - message is from CLI, file, or git history."""
        self.commit_before_message.append(message)

        while True:
            try:
                self.run_one(message)
                break
            except SwitchCoder as switch:
                # Handle coder switching
                return self.run_switch_coder(switch)

    def run_one(self, message):
        """Process single message turn."""
        # Get context from git repo, files, etc.
        # Send to LLM
        # Apply edits
        # Commit if configured
```

### 2. Codex - CLI with Sandbox I/O

```rust
// codex/codex-cli/src/chat.rs
pub struct ChatClient {
    pub sandbox: Option<Sandbox>,
}

impl ChatClient {
    pub async fn run(&mut self, initial_items: Vec<InputItem>) -> Result<(), Error> {
        // CLI input/output only
        // No traditional channel concept
        // Uses terminal UI for interaction
    }
}
```

### 3. Qwen-code - React + Ink Stream UI

```typescript
// qwen-code/src/ui/stream.tsx
export function StreamUI({ messages }: { messages: Message[] }) {
  return (
    <Box flexDirection="column">
      {messages.map((msg, i) => (
        <MessageComponent key={i} message={msg} />
      ))}
    </Box>
  );
}

// Input is from stdin/CLI arguments
// Output is terminal UI via Ink
```

### 4. Gemini-cli - A2A Protocol

```typescript
// gemini-cli/src/protocol/a2a.ts
export interface A2AMessage {
  role: "user" | "model" | "tool";
  content: string | Part[];
}

// Uses A2A (Agent-to-Agent) protocol
// Not traditional user messaging
```

### 5. Kimi-cli - Wire Protocol

```python
# kimi-cli/kosong/wire.py
class WireProtocol:
    """Binary wire protocol for efficient message encoding."""

    def encode(self, msg: Message) -> bytes:
        # Efficient binary encoding
        pass

    def decode(self, data: bytes) -> Message:
        pass
```

---

## Architecture Comparison Matrix

### Channel Abstraction Comparison

| Project | Language | Channel Count | Abstraction Pattern | Discovery |
|---------|----------|---------------|---------------------|-----------|
| CoPaw | Python | 11 | ABC BaseChannel | Static imports |
| NanoClaw | TypeScript | 10+ | Interface + Registry | Runtime skills |
| ZeroClaw | Rust | 25+ | Trait (async_trait) | Feature-gated modules |
| IronClaw | Rust | 8+ | Trait + Manager | Config-driven |
| PicoClaw | Go | 15+ | Interface segregation | Factory registry |
| Nanobot | Python | 12+ | ABC + MessageBus | pkgutil + entry_points |
| OpenClaw | TypeScript | 15+ | Extension plugins | Dynamic imports |
| MimiClaw | C | 2 | Struct function pointers | Static compile |

### Feature Support Matrix

| Feature | CoPaw | ZeroClaw | PicoClaw | OpenClaw | Nanobot |
|---------|-------|----------|----------|----------|---------|
| WebSocket | ✓ | ✓ | ✓ | ✓ | ✓ |
| Webhook | ✓ | ✓ | ✓ | ✓ | ✓ |
| Typing indicators | ✗ | ✓ | ✓ | ✓ | ✗ |
| Reactions | ✗ | ✓ | ✓ | ✓ | ✗ |
| Placeholders | ✗ | ✓ | ✓ | ✓ | ✗ |
| Media attachments | ✓ | ✓ | ✓ | ✓ | ✓ |
| Thread support | ✓ | ✓ | ✓ | ✓ | ✓ |
| Voice transcription | ✗ | ✓ | ✓ | ✓ | ✓ |
| Hot credential reload | ✗ | ✓ | ✗ | ✓ | ✗ |
| Rate limiting | ✗ | ✗ | ✓ | ✓ | ✗ |

### Message Format Comparison

| Project | Incoming Message | Outgoing Message | Metadata |
|---------|------------------|------------------|----------|
| CoPaw | `IncomingMessage` dataclass | `OutgoingContentPart` list | Dict |
| ZeroClaw | `IncomingMessage` struct | `OutgoingResponse` struct | serde_json::Value |
| PicoClaw | `InboundMessage` struct | `OutboundMessage` struct | map[string]string |
| Nanobot | `InboundMessage` dataclass | `OutboundMessage` dataclass | Dict |
| OpenClaw | `IncomingMessage` interface | `OutboundMessage` interface | Record<string, any> |

### Permission Model Comparison

| Project | Whitelist | Mention Required | Group Policies | Allowlist Format |
|---------|-----------|------------------|----------------|------------------|
| CoPaw | `allow_from` list | `require_mention` | `dm_policy`, `group_policy` | User IDs |
| ZeroClaw | Per-channel config | Configurable | Thread-scoped | User IDs + patterns |
| PicoClaw | `allowList` array | Group trigger config | Group trigger prefixes | Canonical IDs |
| Nanobot | `allow_from` list | N/A | N/A | User IDs or "*" |
| OpenClaw | Nested allowlist | Configurable | Per-channel | Platform-specific |

---

## Recommended Unified Architecture

### Hybrid Channel Architecture Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    Channel Gateway Layer                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   WebSocket │  │    HTTP     │  │   Webhook   │             │
│  │   Gateway   │  │   Server    │  │   Handler   │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
└─────────┼────────────────┼────────────────┼────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Protocol Adapters                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ Telegram │ │ Discord  │ │  Slack   │ │  Feishu  │           │
│  │ Adapter  │ │ Adapter  │ │ Adapter  │ │ Adapter  │           │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘           │
└───────┼────────────┼────────────┼────────────┼─────────────────┘
        │            │            │            │
        └────────────┴────────────┴────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Unified Message Bus                          │
│              (IncomingMessage / OutboundMessage)                │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Agent Runtime Core                           │
│              (ReAct Loop / Step Loop / Event Stream)            │
└─────────────────────────────────────────────────────────────────┘
```

### Recommended Channel Trait/Interface

```rust
// Recommended unified Channel trait (Rust-inspired, language-agnostic)
trait Channel {
    // Identification
    fn name(&self) -> &str;
    fn capabilities(&self) -> ChannelCapabilities;

    // Lifecycle
    async fn start(&self) -> Result<MessageStream, Error>;
    async fn stop(&self) -> Result<(), Error>;
    async fn health_check(&self) -> HealthStatus;

    // Messaging
    async fn send(&self, msg: OutboundMessage) -> Result<(), Error>;
    async fn send_typing(&self, chat_id: &str) -> Result<(), Error>;

    // Permissions
    fn is_allowed(&self, sender: &SenderInfo) -> bool;

    // Context
    fn conversation_context(&self, metadata: &Metadata) -> Context;
}

struct ChannelCapabilities {
    supports_typing: bool,
    supports_reactions: bool,
    supports_threads: bool,
    supports_media: bool,
    supports_voice: bool,
    supports_placeholders: bool,
    max_message_length: Option<usize>,
    supported_media_types: Vec<MimeType>,
}

struct UnifiedMessage {
    id: String,
    channel: String,
    sender: SenderInfo,
    content: Content,
    thread_id: Option<String>,
    timestamp: DateTime,
    metadata: Metadata,
    attachments: Vec<Attachment>,
}

struct SenderInfo {
    id: String,
    name: Option<String>,
    platform_id: String,  // e.g., "telegram:123456"
    canonical_id: String, // e.g., "user@example.com"
    is_bot: bool,
}
```

### Key Design Recommendations

1. **Use Interface Segregation**: Separate capabilities into optional traits (TypingCapable, ReactionCapable) rather than one large interface

2. **Canonical ID System**: Use "platform:id" format for cross-platform user identification

3. **Message Bus Pattern**: Decouple channels from agent core via async message bus

4. **Hot Reload**: Support credential updates without restart (ChannelSecretUpdater pattern)

5. **Rate Limiting**: Per-channel rate limiting with exponential backoff

6. **Structured Logging**: Channel-aware logging with context propagation

7. **Health Monitoring**: Per-channel health checks with auto-reconnect

8. **Media Pipeline**: Unified media download/upload with async processing

---

## Appendix: Channel Implementation Checklist

### Adding a New Channel

- [ ] Implement base channel interface/trait
- [ ] Add protocol-specific connection handling
- [ ] Implement message format conversion (native ↔ unified)
- [ ] Add authentication/authorization
- [ ] Implement media handling (if applicable)
- [ ] Add typing/reaction support (if applicable)
- [ ] Add health check mechanism
- [ ] Write tests
- [ ] Document configuration options
- [ ] Add to channel registry/factory
