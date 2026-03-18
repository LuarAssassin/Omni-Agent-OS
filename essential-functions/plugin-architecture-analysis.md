# Plugin Architecture Analysis

## Executive Summary

This document provides comprehensive analysis of plugin and extension architectures across all 15 AI agent projects in the Omni-Agent-OS ecosystem. The analysis reveals five distinct architectural patterns for extensibility:

| Maturity Level | Projects | Pattern |
|---------------|----------|---------|
| **Level 4: Full WASM + MCP** | IronClaw | Complete sandboxed extension system with wasmtime |
| **Level 3: Multi-Extension SDK** | OpenClaw, ZeroClaw | Comprehensive SDK with multiple extension points |
| **Level 2: MCP + Hooks** | CoPaw, Kimi-CLI, OpenCode | MCP integration with lifecycle hooks |
| **Level 1: Skills + Tools** | Codex, Gemini CLI, Aider, Pi-Mono | Skill-based or tool-based extension |
| **Level 0: Limited/None** | NanoClaw, PicoClaw, Nanobot, Mimiclaw | Minimal or no plugin architecture |

---

## 1. Core Concepts

### Plugin Architecture Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Plugin Architecture Stack                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     Discovery Layer                                  │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │   │
│  │  │  Filesystem  │ │  npm Registry│ │   GitHub     │ │   MCP Hub    │ │   │
│  │  │    Scan      │ │    Search    │ │   Repos      │ │  Registry    │ │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Loading Layer                                   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │   │
│  │  │   Node.js    │ │     jiti     │ │    WASM      │ │   Python     │ │   │
│  │  │   import()   │ │   (TS/JS)    │ │  (wasmtime)  │ │  importlib   │ │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Lifecycle Management                              │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │   │
│  │  │   Install    │ │    Enable    │ │    Load      │ │  Uninstall   │ │   │
│  │  │  (download)  │ │   (activate) │ │  (runtime)   │ │   (remove)   │ │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     Extension Points                                 │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │   │
│  │  │  Tools   │ │  Hooks   │ │ Channels │ │ Providers│ │   CLI    │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                   Security & Isolation                               │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │   │
│  │  │   Path       │ │   Resource   │ │   Network    │ │  Permission  │ │   │
│  │  │ Validation   │ │    Limits    │ │  Allowlist   │ │    System    │ │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Extension Point Taxonomy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Extension Point Types                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  TOOL EXTENSIONS                      Used by: All 15 projects       │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                 │   │
│  │  │ Built-in     │ │ External     │ │ MCP Tools    │                 │   │
│  │  │ Tools        │ │ Tools        │ │ (Dynamic)    │                 │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  HOOK EXTENSIONS                      Used by: IronClaw, OpenCode    │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                 │   │
│  │  │ Before       │ │ Transform    │ │ After        │                 │   │
│  │  │ Inbound      │ │ Response     │ │ Outbound     │                 │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CHANNEL EXTENSIONS                   Used by: CoPaw, IronClaw, etc  │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                 │   │
│  │  │ Messaging    │ │ Webhook      │ │ WASM         │                 │   │
│  │  │ Adapters     │ │ Endpoints    │ │ Channels     │                 │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  PROVIDER EXTENSIONS                  Used by: IronClaw, ZeroClaw    │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                 │   │
│  │  │ LLM          │ │ Embedding    │ │ Success      │                 │   │
│  │  │ Providers    │ │ Providers    │ │ Evaluators   │                 │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. WASM Plugin Systems

### IronClaw: Production-Grade WASM Sandbox

IronClaw implements the most sophisticated WASM plugin system using wasmtime with full sandboxing capabilities.

```rust
// src/tools/wasm/runtime.rs
use wasmtime::{Engine, Module, Store, Instance, Memory, Func};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder};

pub struct WasmRuntime {
    engine: Engine,
    module_cache: Arc<RwLock<HashMap<String, Module>>>,
    config: WasmSandboxConfig,
}

pub struct WasmSandboxConfig {
    pub max_memory_pages: u32,      // 128MB default (2048 pages)
    pub max_fuel: u64,               // Execution fuel limit
    pub enable_wasi: bool,
    pub network_allowlist: Vec<String>,
}

impl WasmRuntime {
    pub fn new(config: WasmSandboxConfig) -> Result<Self> {
        let mut engine_config = wasmtime::Config::new();

        // Enable fuel metering for deterministic execution
        engine_config.consume_fuel(true);

        // Enable WASI for standard APIs
        if config.enable_wasi {
            engine_config.wasm_backtrace_details(wasmtime::WasmBacktraceDetails::Enable);
        }

        let engine = Engine::new(&engine_config)?;

        Ok(Self {
            engine,
            module_cache: Arc::new(RwLock::new(HashMap::new())),
            config,
        })
    }

    pub async fn load_tool(&self, wasm_path: &Path) -> Result<WasmTool> {
        // 1. Read WASM bytes
        let wasm_bytes = tokio::fs::read(wasm_path).await?;

        // 2. Compile module (cached)
        let module = self.get_or_compile_module(wasm_path, &wasm_bytes).await?;

        // 3. Create WASI context with restricted capabilities
        let wasi = WasiCtxBuilder::new()
            .inherit_env()
            .inherit_stdio()
            .preopened_dir(
                self.config.allowed_workspace_dir(),
                "/workspace",
                DirPerms::READ | DirPerms::WRITE,
                FilePerms::READ | FilePerms::WRITE,
            )?
            .build();

        // 4. Create store with fuel limit
        let mut store = Store::new(&self.engine, wasi);
        store.add_fuel(self.config.max_fuel)?;

        // 5. Instantiate with host functions
        let instance = self.linker.instantiate(&mut store, &module)?;

        // 6. Extract tool metadata
        let metadata = self.extract_metadata(&mut store, &instance)?;

        Ok(WasmTool {
            instance,
            store,
            metadata,
            wasm_path: wasm_path.to_path_buf(),
        })
    }
}
```

#### WASM Tool Wrapper

```rust
// src/tools/wasm/wrapper.rs
#[async_trait]
impl Tool for WasmToolWrapper {
    fn name(&self) -> &str {
        &self.metadata.name
    }

    fn description(&self) -> &str {
        &self.metadata.description
    }

    fn parameters(&self) -> &serde_json::Value {
        &self.metadata.parameters
    }

    async fn execute(&self, input: ToolInput) -> Result<ToolOutput, ToolError> {
        // 1. Create fresh store for isolation
        let mut store = self.create_isolated_store().await?;

        // 2. Get exported "execute" function
        let execute_func = self.instance
            .get_typed_func::<(i32, i32), (i32, i32)>(&mut store, "execute")?;

        // 3. Serialize input to WASM memory
        let input_json = serde_json::to_string(&input)?;
        let (input_ptr, input_len) = self.write_string_to_memory(&mut store, &input_json)?;

        // 4. Execute with fuel consumption tracking
        let (output_ptr, output_len) = execute_func.call(&mut store, (input_ptr, input_len))
            .map_err(|e| ToolError::ExecutionFailed(e.to_string()))?;

        // 5. Read output from WASM memory
        let output_json = self.read_string_from_memory(&store, output_ptr, output_len)?;

        // 6. Deserialize result
        let output: ToolOutput = serde_json::from_str(&output_json)?;

        Ok(output)
    }
}
```

#### WASM Host Functions

```rust
// src/tools/wasm/host.rs
pub fn register_host_functions(linker: &mut Linker<WasiCtx>) -> Result<()> {
    // Logging host function
    linker.func_wrap(
        "host",
        "log",
        |mut caller: Caller<'_, WasiCtx>, ptr: i32, len: i32| {
            let memory = caller.get_export("memory").unwrap().into_memory().unwrap();
            let data = memory.read(&caller, ptr as usize, len as usize)?;
            let message = String::from_utf8(data)?;
            tracing::info!(target: "wasm_tool", "{}", message);
            Ok(())
        },
    )?;

    // Time host function
    linker.func_wrap(
        "host",
        "time_now_ms",
        || -> i64 {
            std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)
                .unwrap()
                .as_millis() as i64
        },
    )?;

    // HTTP fetch with allowlist
    linker.func_wrap_async(
        "host",
        "http_fetch",
        |mut caller: Caller<'_, WasiCtx>,
         url_ptr: i32, url_len: i32,
         body_ptr: i32, body_len: i32| {
            Box::new(async move {
                let memory = caller.get_export("memory")
                    .unwrap()
                    .into_memory()
                    .unwrap();

                // Read URL
                let url_data = memory.read(&caller, url_ptr as usize, url_len as usize)?;
                let url = String::from_utf8(url_data)?;

                // Validate against allowlist
                if !is_url_allowed(&url, &caller.data().network_allowlist) {
                    return Err(anyhow!("URL not in allowlist: {}", url));
                }

                // Perform HTTP request
                let response = reqwest::get(&url).await?;
                let body = response.text().await?;

                // Write response back to WASM memory
                let (resp_ptr, resp_len) = write_string_to_memory(&mut caller, &body)?;
                Ok((resp_ptr, resp_len))
            })
        },
    )?;

    Ok(())
}
```

### ZeroClaw: Trait-Based WASM Channels

ZeroClaw supports WASM-based channel implementations alongside native Rust channels.

```rust
// src/channels/wasm/runtime.rs
pub struct WasmChannelRuntime {
    engine: wasmtime::Engine,
    runtime_config: RuntimeConfig,
}

pub struct WasmChannel {
    instance: wasmtime::Instance,
    store: wasmtime::Store<ChannelState>,
}

#[async_trait]
impl Channel for WasmChannel {
    async fn initialize(&mut self, config: ChannelConfig) -> Result<()> {
        let init_func = self.instance
            .get_typed_func::<(i32, i32), i32>(&mut self.store, "initialize")?;

        let config_json = serde_json::to_string(&config)?;
        let (ptr, len) = self.write_to_memory(&config_json)?;

        let result = init_func.call(&mut self.store, (ptr, len)).await?;

        if result != 0 {
            return Err(ChannelError::InitializationFailed(result));
        }

        Ok(())
    }

    async fn send_message(&self, message: OutgoingMessage) -> Result<()> {
        // Serialize and send through WASM exported function
        let json = serde_json::to_string(&message)?;
        let (ptr, len) = self.write_to_memory(&json)?;

        let send_func = self.instance
            .get_typed_func::<(i32, i32), i32>(&mut self.store, "send_message")?;

        send_func.call(&mut self.store, (ptr, len)).await?;
        Ok(())
    }
}
```

---

## 3. MCP (Model Context Protocol) Integration

### CoPaw: MCP Client Implementation

```python
# src/copaw/tools/mcp_client.py
from mcp import ClientSession, StdioServerParameters
from mcp.types import (
    TextContent, ImageContent, EmbeddedResource,
    CallToolResult, ListToolsResult
)
import asyncio
from contextlib import AsyncExitStack

class MCPClientManager:
    """Manages MCP server connections and tool exposure."""

    def __init__(self):
        self.sessions: dict[str, ClientSession] = {}
        self.exit_stack = AsyncExitStack()
        self._tools_cache: dict[str, dict] = {}

    async def connect_stdio_server(
        self,
        server_id: str,
        command: str,
        args: list[str] = None,
        env: dict[str, str] = None
    ) -> ClientSession:
        """Connect to MCP server via stdio transport."""
        server_params = StdioServerParameters(
            command=command,
            args=args or [],
            env={**os.environ, **(env or {})}
        )

        # Create stdio transport
        stdio_transport = await self.exit_stack.enter_async_context(
            stdio_client(server_params)
        )
        stdio, write = stdio_transport

        # Create session
        session = await self.exit_stack.enter_async_context(
            ClientSession(stdio, write)
        )

        # Initialize
        await session.initialize()

        self.sessions[server_id] = session
        return session

    async def connect_http_server(
        self,
        server_id: str,
        url: str,
        headers: dict[str, str] = None
    ) -> ClientSession:
        """Connect to MCP server via HTTP/SSE transport."""
        from mcp.client.sse import sse_client

        # Create SSE transport
        sse_transport = await self.exit_stack.enter_async_context(
            sse_client(url, headers=headers)
        )
        read, write = sse_transport

        # Create session
        session = await self.exit_stack.enter_async_context(
            ClientSession(read, write)
        )

        await session.initialize()
        self.sessions[server_id] = session
        return session

    async def list_all_tools(self) -> list[dict]:
        """Aggregate tools from all connected MCP servers."""
        all_tools = []

        for server_id, session in self.sessions.items():
            result: ListToolsResult = await session.list_tools()

            for tool in result.tools:
                tool_def = {
                    "name": f"{server_id}:{tool.name}",
                    "original_name": tool.name,
                    "server_id": server_id,
                    "description": tool.description,
                    "parameters": tool.inputSchema,
                }
                all_tools.append(tool_def)
                self._tools_cache[tool_def["name"]] = tool_def

        return all_tools

    async def call_tool(self, tool_name: str, arguments: dict) -> CallToolResult:
        """Execute tool via MCP."""
        if tool_name not in self._tools_cache:
            raise ValueError(f"Unknown MCP tool: {tool_name}")

        tool_def = self._tools_cache[tool_name]
        session = self.sessions[tool_def["server_id"]]

        result = await session.call_tool(
            tool_def["original_name"],
            arguments=arguments
        )

        return result

class MCPToolAdapter:
    """Adapts MCP tools to CoPaw's tool interface."""

    def __init__(self, mcp_manager: MCPClientManager):
        self.mcp = mcp_manager

    async def to_copaw_tool(self, mcp_tool_def: dict) -> Tool:
        """Convert MCP tool definition to CoPaw Tool."""
        async def execute(**kwargs) -> ToolResult:
            result = await self.mcp.call_tool(
                mcp_tool_def["name"],
                kwargs
            )

            # Convert MCP content to CoPaw format
            content_parts = []
            for item in result.content:
                if isinstance(item, TextContent):
                    content_parts.append({
                        "type": "text",
                        "text": item.text
                    })
                elif isinstance(item, ImageContent):
                    content_parts.append({
                        "type": "image",
                        "mime_type": item.mimeType,
                        "data": item.data
                    })

            return ToolResult(
                content=content_parts,
                is_error=result.isError if hasattr(result, 'isError') else False
            )

        return Tool(
            name=mcp_tool_def["name"],
            description=mcp_tool_def["description"],
            parameters=mcp_tool_def["parameters"],
            execute=execute
        )
```

### Kimi-CLI: MCP Tool Integration via Kosong

```python
# packages/kosong/src/kosong/tooling/mcp.py
from pydantic import BaseModel, Field
from typing import Any, Callable
from contextlib import asynccontextmanager

class MCPServerConfig(BaseModel):
    """Configuration for MCP server connection."""
    transport: str = Field(..., description="stdio, http, or unix")
    command: str | None = None
    args: list[str] = Field(default_factory=list)
    url: str | None = None
    socket_path: str | None = None
    env: dict[str, str] = Field(default_factory=dict)
    headers: dict[str, str] = Field(default_factory=dict)

class MCPToolAdapter:
    """Adapter for MCP (Model Context Protocol) tools."""

    def __init__(self, session: ClientSession):
        self.session = session
        self._tool_cache: dict[str, CallableTool] = {}

    async def list_tools(self) -> list[CallableTool]:
        """List all available MCP tools and cache them."""
        tools_result = await self.session.list_tools()

        tools = []
        for mcp_tool in tools_result.tools:
            callable_tool = self._convert_tool(mcp_tool)
            self._tool_cache[mcp_tool.name] = callable_tool
            tools.append(callable_tool)

        return tools

    def _convert_tool(self, mcp_tool) -> CallableTool:
        """Convert MCP tool definition to Kosong CallableTool."""

        async def tool_executor(**kwargs) -> ToolReturnValue:
            result: CallToolResult = await self.session.call_tool(
                mcp_tool.name,
                arguments=kwargs
            )

            # Convert MCP content types
            content_parts = []
            for content in result.content:
                if isinstance(content, TextContent):
                    content_parts.append({
                        "type": "text",
                        "text": content.text
                    })
                elif isinstance(content, ImageContent):
                    content_parts.append({
                        "type": "image",
                        "mime_type": content.mimeType,
                        "data": content.data
                    })
                elif isinstance(content, EmbeddedResource):
                    content_parts.append({
                        "type": "resource",
                        "resource": content.resource
                    })

            return ToolReturnValue(
                content=content_parts,
                is_error=getattr(result, 'isError', False)
            )

        return CallableTool(
            name=mcp_tool.name,
            description=mcp_tool.description or "",
            parameters=mcp_tool.inputSchema,
            callable=tool_executor,
            source="mcp"
        )

    @asynccontextmanager
    async def session_scope(self):
        """Context manager for MCP session lifecycle."""
        try:
            await self.session.initialize()
            yield self
        finally:
            await self.session.close()

async def create_mcp_adapter(config: MCPServerConfig) -> MCPToolAdapter:
    """Factory function to create MCP adapter from config."""
    if config.transport == "stdio":
        from mcp.client.stdio import stdio_client

        server_params = StdioServerParameters(
            command=config.command,
            args=config.args,
            env={**os.environ, **config.env}
        )

        async with stdio_client(server_params) as (read, write):
            async with ClientSession(read, write) as session:
                return MCPToolAdapter(session)

    elif config.transport == "http":
        from mcp.client.sse import sse_client

        async with sse_client(config.url, headers=config.headers) as (read, write):
            async with ClientSession(read, write) as session:
                return MCPToolAdapter(session)

    else:
        raise ValueError(f"Unsupported transport: {config.transport}")
```

### IronClaw: Full MCP Server Support

IronClaw can act as both MCP client and MCP server.

```rust
// src/tools/mcp/client.rs
use mcp_sdk::{Client, Transport};

pub struct McpClientManager {
    clients: Arc<RwLock<HashMap<String, Client>>>,
    config: McpConfig,
}

impl McpClientManager {
    pub async fn connect_stdio(
        &self,
        server_id: &str,
        command: &str,
        args: &[String],
    ) -> Result<Client> {
        let transport = StdioTransport::new(command, args);
        let client = Client::new(transport);

        client.initialize().await?;

        self.clients.write().await.insert(server_id.to_string(), client.clone());

        Ok(client)
    }

    pub async fn connect_http(
        &self,
        server_id: &str,
        url: &str,
    ) -> Result<Client> {
        let transport = SseTransport::new(url);
        let client = Client::new(transport);

        client.initialize().await?;

        self.clients.write().await.insert(server_id.to_string(), client.clone());

        Ok(client)
    }

    pub async fn list_all_tools(&self) -> Result<Vec<McpTool>> {
        let mut all_tools = Vec::new();

        for (server_id, client) in self.clients.read().await.iter() {
            let tools = client.list_tools().await?;

            for tool in tools {
                all_tools.push(McpTool {
                    name: format!("{}:{}", server_id, tool.name),
                    original_name: tool.name,
                    server_id: server_id.clone(),
                    description: tool.description,
                    parameters: tool.input_schema,
                });
            }
        }

        Ok(all_tools)
    }
}
```

#### IronClaw as MCP Server

```rust
// src/mcp/server.rs
use mcp_sdk::{Server, Router, Request, Response};

pub struct IronClawMcpServer {
    tool_registry: Arc<ToolRegistry>,
    memory_store: Arc<MemoryStore>,
}

#[async_trait]
impl Router for IronClawMcpServer {
    async fn handle_request(&self, request: Request) -> Result<Response> {
        match request.method.as_str() {
            "tools/list" => {
                let tools = self.tool_registry.list_tools().await?;
                Ok(Response::success(json!({ "tools": tools })))
            }
            "tools/call" => {
                let tool_name = request.params["name"].as_str().unwrap();
                let arguments = request.params["arguments"].clone();

                let result = self.tool_registry
                    .execute_tool(tool_name, arguments)
                    .await?;

                Ok(Response::success(json!({
                    "content": [{"type": "text", "text": result}]
                })))
            }
            "resources/list" => {
                let resources = self.memory_store.list_resources().await?;
                Ok(Response::success(json!({ "resources": resources })))
            }
            _ => Err(anyhow!("Method not found: {}", request.method))
        }
    }
}
```

---

## 4. Skill Systems

### CoPaw: Directory-Based Skill System

```python
# src/copaw/agents/skills_manager.py
from pathlib import Path
import yaml
import shutil
from dataclasses import dataclass
from typing import Literal

@dataclass
class SkillInfo:
    name: str
    path: str
    source: Literal["builtin", "customized", "active"]
    description: str
    content: str
    references: dict
    scripts: dict

class SkillsManager:
    """Manage skills lifecycle and synchronization."""

    def __init__(self):
        self.builtin_skills_dir = Path(__file__).parent / "skills"
        self.customized_skills_dir = Path(
            os.getenv("CUSTOMIZED_SKILLS_DIR", Path.home() / ".copaw" / "customized_skills")
        )
        self.active_skills_dir = Path(
            os.getenv("ACTIVE_SKILLS_DIR", Path.cwd() / ".copaw" / "active_skills")
        )
        self.working_dir = Path.cwd()

    def _collect_skills_from_dir(self, directory: Path) -> dict[str, Path]:
        """Scan directory for valid skills (must contain SKILL.md)."""
        skills: dict[str, Path] = {}
        if directory.exists():
            for skill_dir in directory.iterdir():
                if skill_dir.is_dir() and (skill_dir / "SKILL.md").exists():
                    skills[skill_dir.name] = skill_dir
        return skills

    def _parse_skill_md(self, skill_md_path: Path) -> dict:
        """Parse SKILL.md with YAML frontmatter."""
        content = skill_md_path.read_text()

        # Parse YAML frontmatter
        if content.startswith("---"):
            _, frontmatter, body = content.split("---", 2)
            metadata = yaml.safe_load(frontmatter)
        else:
            metadata = {}
            body = content

        return {
            "description": metadata.get("description", ""),
            "content": body.strip(),
            "references": metadata.get("references", {}),
            "scripts": metadata.get("scripts", {}),
            "tags": metadata.get("tags", []),
            "author": metadata.get("author"),
            "version": metadata.get("version", "0.1.0"),
        }

    def list_skills(self) -> list[SkillInfo]:
        """List all skills from all sources."""
        builtin = self._collect_skills_from_dir(self.builtin_skills_dir)
        customized = self._collect_skills_from_dir(self.customized_skills_dir)
        active = self._collect_skills_from_dir(self.active_skills_dir)

        skills = []
        for name, path in {**builtin, **customized, **active}.items():
            skill_info = self._parse_skill_md(path / "SKILL.md")

            # Determine source
            if name in builtin:
                source = "builtin"
            elif name in customized:
                source = "customized"
            else:
                source = "active"

            skills.append(SkillInfo(
                name=name,
                path=str(path),
                source=source,
                **skill_info
            ))

        return skills

    async def enable_skill(self, skill_name: str) -> None:
        """Enable skill by copying to active directory."""
        # 1. Find skill source
        source_path = None
        for source_dir in [self.builtin_skills_dir, self.customized_skills_dir]:
            candidate = source_dir / skill_name
            if candidate.exists():
                source_path = candidate
                break

        if not source_path:
            raise SkillNotFoundError(skill_name)

        # 2. Copy to active directory
        target_path = self.active_skills_dir / skill_name
        if target_path.exists():
            shutil.rmtree(target_path)
        shutil.copytree(source_path, target_path)

    async def sync_skills_to_working_dir(self, force: bool = False) -> None:
        """Sync all enabled skills to working directory."""
        active_skills = self._collect_skills_from_dir(self.active_skills_dir)

        for name, source_path in active_skills.items():
            target_path = self.working_dir / ".copaw" / "skills" / name

            # Check if sync needed
            if target_path.exists() and not force:
                source_mtime = max(f.stat().st_mtime for f in source_path.rglob("*"))
                target_mtime = max(f.stat().st_mtime for f in target_path.rglob("*"))
                if target_mtime >= source_mtime:
                    continue

            # Sync
            if target_path.exists():
                shutil.rmtree(target_path)
            shutil.copytree(source_path, target_path)
```

### Kimi-CLI: Layered Skill Resolution

```python
# src/kimi_cli/skill/__init__.py
class SkillResolver:
    """Resolve skills from layered directories with priority order."""

    PRIORITY_ORDER = ["project", "user", "builtin"]

    def get_user_skills_dir_candidates(self) -> tuple[Path, ...]:
        """Standard skill directory locations (cross-tool compatible)."""
        return (
            Path.home() / ".config" / "agents" / "skills",
            Path.home() / ".agents" / "skills",
            Path.home() / ".kimi" / "skills",
            Path.home() / ".claude" / "skills",
            Path.home() / ".codex" / "skills",
        )

    def resolve_skill(
        self,
        name: str,
        work_dir: Path | None = None,
    ) -> Skill | None:
        """Resolve skill by name with priority: Project > User > Built-in."""
        # 1. Project level (highest priority)
        if work_dir:
            project_paths = [
                work_dir / ".kimi" / "skills" / name,
                work_dir / ".agents" / "skills" / name,
            ]
            for path in project_paths:
                if path.exists():
                    return self._load_skill(path)

        # 2. User level
        for dir_candidate in self.get_user_skills_dir_candidates():
            path = dir_candidate / name
            if path.exists():
                return self._load_skill(path)

        # 3. Built-in level
        builtin_path = Path(__file__).parent / "skills" / name
        if builtin_path.exists():
            return self._load_skill(builtin_path)

        return None

    def list_all_skills(
        self,
        work_dir: Path | None = None,
    ) -> dict[str, list[SkillSource]]:
        """List all skills from all layers, grouped by name."""
        skills: dict[str, list[SkillSource]] = {}

        # Collect built-in
        for skill in self._list_builtin_skills():
            skills.setdefault(skill.name, []).append(
                SkillSource(skill=skill, layer="builtin")
            )

        # Collect user level
        for skill in self._list_user_skills():
            skills.setdefault(skill.name, []).append(
                SkillSource(skill=skill, layer="user")
            )

        # Collect project level
        if work_dir:
            for skill in self._list_project_skills(work_dir):
                skills.setdefault(skill.name, []).append(
                    SkillSource(skill=skill, layer="project")
                )

        return skills

    def get_merged_skill(self, name: str, work_dir: Path | None = None) -> Skill | None:
        """Merge skill from all layers (project overrides user overrides builtin)."""
        all_sources = self.list_all_skills(work_dir).get(name, [])

        if not all_sources:
            return None

        # Sort by priority
        all_sources.sort(key=lambda s: self.PRIORITY_ORDER.index(s.layer))

        # Start with lowest priority
        merged = all_sources[0].skill

        # Apply overrides
        for source in all_sources[1:]:
            merged = self._merge_skill(merged, source.skill)

        return merged
```

### IronClaw: Trust-Based Skill System

```rust
// src/skills/manager.rs
pub struct SkillManager {
    trusted_skills_dir: PathBuf,
    installed_skills_dir: PathBuf,
    builtin_skills: Vec<Skill>,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum SkillTrust {
    Trusted,    // User-placed in ~/.ironclaw/skills/ - full tool access
    Installed,  // From registry - read-only tools only
    Builtin,    // Bundled with app
}

impl SkillManager {
    pub async fn load_skill(&self, path: &Path) -> Result<Skill> {
        let skill_md_path = path.join("SKILL.md");
        let content = tokio::fs::read_to_string(&skill_md_path).await?;

        // Parse frontmatter
        let (frontmatter, body) = self.parse_frontmatter(&content)?;

        // Determine trust level
        let trust = if path.starts_with(&self.trusted_skills_dir) {
            SkillTrust::Trusted
        } else if path.starts_with(&self.installed_skills_dir) {
            SkillTrust::Installed
        } else {
            SkillTrust::Builtin
        };

        // Parse requirements (gating)
        let requirements = self.parse_requirements(&frontmatter)?;

        // Check if requirements met
        self.validate_requirements(&requirements).await?;

        Ok(Skill {
            name: frontmatter.name.clone(),
            description: frontmatter.description,
            content: body,
            trust,
            requirements,
            tags: frontmatter.tags,
            tools_allowed: self.determine_tool_ceiling(trust),
        })
    }

    fn determine_tool_ceiling(&self, trust: SkillTrust) -> Vec<String> {
        match trust {
            SkillTrust::Trusted => vec!["*".to_string()], // All tools
            SkillTrust::Installed => vec![
                "memory_read".to_string(),
                "web_fetch".to_string(),
                "json".to_string(),
            ],
            SkillTrust::Builtin => vec!["*".to_string()], // Full access
        }
    }

    pub async fn select_skills_for_session(
        &self,
        query: &str,
        max_tokens: usize,
    ) -> Result<Vec<Skill>> {
        let all_skills = self.list_all_skills().await?;

        // 1. Gating: filter by requirements
        let gated: Vec<_> = all_skills.into_iter()
            .filter(|s| self.check_requirements(&s.requirements))
            .collect();

        // 2. Scoring: match against query
        let mut scored: Vec<_> = gated.into_iter()
            .map(|s| {
                let score = self.score_skill(&s, query);
                (s, score)
            })
            .filter(|(_, score)| *score > 0.0)
            .collect();

        scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

        // 3. Budget: fit within max_tokens
        let mut selected = Vec::new();
        let mut token_budget = max_tokens;

        for (skill, _) in scored {
            let tokens = self.estimate_tokens(&skill.content);
            if tokens <= token_budget {
                selected.push(skill);
                token_budget -= tokens;
            }
            if token_budget < 100 { // Minimum remaining
                break;
            }
        }

        // 4. Attenuation: apply trust-based tool ceiling
        for skill in &mut selected {
            skill.available_tools = self.determine_tool_ceiling(skill.trust);
        }

        Ok(selected)
    }
}
```

---

## 5. Tool Registry Architectures

### Codex: Rust Tool Registry

```rust
// codex-rs/core/src/tools/registry.rs
use std::collections::HashMap;
use std::sync::Arc;

pub struct ToolRegistry {
    tools: RwLock<HashMap<String, Arc<dyn Tool>>>,
    mcp_clients: RwLock<HashMap<String, McpClient>>,
}

#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters(&self) -> &serde_json::Value;
    async fn execute(&self, input: ToolInput) -> Result<ToolOutput, ToolError>;
}

impl ToolRegistry {
    pub fn new() -> Self {
        let registry = Self {
            tools: RwLock::new(HashMap::new()),
            mcp_clients: RwLock::new(HashMap::new()),
        };

        // Register built-in tools
        registry.register_builtin_tools();

        registry
    }

    fn register_builtin_tools(&self) {
        let builtins: Vec<Arc<dyn Tool>> = vec![
            Arc::new(ShellTool),
            Arc::new(FileReadTool),
            Arc::new(FileWriteTool),
            Arc::new(FileEditTool),
            Arc::new(FileBlockEditTool),
            Arc::new(LinterTool),
            Arc::new(CodeSearchTool),
        ];

        for tool in builtins {
            self.register(tool);
        }
    }

    pub fn register(&self, tool: Arc<dyn Tool>) {
        self.tools.write().insert(tool.name().to_string(), tool);
    }

    pub async fn register_mcp_server(&self, server_id: &str, config: McpConfig) -> Result<()> {
        let client = McpClient::connect(config).await?;
        let tools = client.list_tools().await?;

        for tool in tools {
            let mcp_tool = McpToolAdapter {
                client: client.clone(),
                tool_name: tool.name,
                description: tool.description,
                parameters: tool.input_schema,
            };

            self.register(Arc::new(mcp_tool));
        }

        self.mcp_clients.write().insert(server_id.to_string(), client);

        Ok(())
    }

    pub fn get(&self, name: &str) -> Option<Arc<dyn Tool>> {
        self.tools.read().get(name).cloned()
    }

    pub fn list_all(&self) -> Vec<ToolDefinition> {
        self.tools.read().values()
            .map(|t| ToolDefinition {
                name: t.name().to_string(),
                description: t.description().to_string(),
                parameters: t.parameters().clone(),
            })
            .collect()
    }
}
```

### OpenClaw: TypeScript Plugin Registry

```typescript
// src/plugins/registry.ts
export interface PluginRecord {
  id: string;
  name: string;
  version?: string;
  description?: string;
  kind?: PluginKind;
  source: string;
  origin: PluginOrigin;  // "bundled" | "workspace" | "npm"
  enabled: boolean;
  status: "loaded" | "disabled" | "error";
  error?: string;

  // Registered extensions
  toolNames: string[];
  hookNames: string[];
  channelIds: string[];
  providerIds: string[];
  httpRoutes: string[];
  commands: string[];

  manifest?: PluginManifest;
  config?: Record<string, unknown>;
}

export interface PluginRegistry {
  plugins: Map<string, PluginRecord>;

  // Registration methods
  registerTool: (toolId: string, tool: ToolDefinition, opts?: ToolOptions) => void;
  registerHook: (event: string, handler: HookHandler, opts?: HookOptions) => string;
  registerChannel: (channel: ChannelRegistration) => void;
  registerProvider: (provider: ProviderDefinition) => void;
  registerHttpRoute: (route: HttpRouteDefinition) => void;
  registerCommand: (command: CommandDefinition) => void;
}

class PluginRegistryImpl implements PluginRegistry {
  plugins = new Map<string, PluginRecord>();
  tools = new Map<string, RegisteredTool>();
  hooks = new Map<string, HookRegistration[]>();
  channels = new Map<string, Channel>();
  providers = new Map<string, Provider>();
  httpRoutes = new Map<string, HttpRoute>();
  commands = new Map<string, Command>();

  registerTool(
    pluginId: string,
    toolId: string,
    tool: ToolDefinition,
    opts?: ToolOptions
  ): void {
    const record = this.plugins.get(pluginId);
    if (!record) throw new Error(`Plugin not found: ${pluginId}`);

    const fullToolId = `${pluginId}:${toolId}`;

    this.tools.set(fullToolId, {
      ...tool,
      id: fullToolId,
      pluginId,
      options: opts || {},
    });

    record.toolNames.push(fullToolId);
  }

  registerHook(
    pluginId: string,
    event: string,
    handler: HookHandler,
    opts?: HookOptions
  ): string {
    const hookId = `${pluginId}:${event}:${Date.now()}`;

    const registration: HookRegistration = {
      id: hookId,
      pluginId,
      event,
      handler,
      priority: opts?.priority || 0,
    };

    const existing = this.hooks.get(event) || [];
    existing.push(registration);

    // Sort by priority (higher first)
    existing.sort((a, b) => b.priority - a.priority);

    this.hooks.set(event, existing);

    const record = this.plugins.get(pluginId)!;
    record.hookNames.push(hookId);

    return hookId;
  }

  async executeHook(event: string, context: HookContext): Promise<HookContext> {
    const handlers = this.hooks.get(event) || [];

    for (const registration of handlers) {
      if (!this.isPluginEnabled(registration.pluginId)) continue;

      try {
        const result = await registration.handler(context);
        if (result) {
          context = { ...context, ...result };
        }
      } catch (error) {
        console.error(`Hook ${registration.id} failed:`, error);
        if (registration.options?.abortOnError) {
          throw error;
        }
      }
    }

    return context;
  }
}
```

### Aider: Python Function-Based Tools

```python
# aider/coders/base_coder.py
class ToolRegistry:
    """Simple function-based tool registry."""

    def __init__(self):
        self.tools = {}

    def register(self, name: str, func: Callable, desc: str, params: dict):
        """Register a tool function."""
        self.tools[name] = {
            "function": func,
            "description": desc,
            "parameters": params,
        }

    def get_tool_defs(self) -> list[dict]:
        """Get tool definitions for LLM."""
        return [
            {
                "type": "function",
                "function": {
                    "name": name,
                    "description": info["description"],
                    "parameters": info["parameters"],
                }
            }
            for name, info in self.tools.items()
        ]

    def execute(self, name: str, arguments: dict) -> str:
        """Execute a tool by name."""
        if name not in self.tools:
            raise ValueError(f"Unknown tool: {name}")

        tool = self.tools[name]
        return tool["function"](**arguments)

# Usage in Aider
coder = ToolRegistry()

# Register built-in tools
coder.register(
    name="read_file",
    func=commands.cmd_read,
    desc="Read a file's contents",
    params={
        "type": "object",
        "properties": {
            "filename": {"type": "string"},
        },
        "required": ["filename"],
    }
)

coder.register(
    name="write_file",
    func=commands.cmd_write,
    desc="Write content to a file",
    params={
        "type": "object",
        "properties": {
            "filename": {"type": "string"},
            "content": {"type": "string"},
        },
        "required": ["filename", "content"],
    }
)
```

---

## 6. Hook Systems

### IronClaw: Lifecycle Hook System

```rust
// src/hooks/mod.rs
pub enum HookPoint {
    BeforeInbound,    // Before processing incoming message
    BeforeToolCall,   // Before executing a tool
    BeforeOutbound,   // Before sending response
    OnSessionStart,   // When session starts
    OnSessionEnd,     // When session ends
    TransformResponse, // Modify response before sending
}

pub struct HookRegistry {
    hooks: HashMap<HookPoint, Vec<RegisteredHook>>,
}

pub struct RegisteredHook {
    pub id: String,
    pub plugin_id: String,
    pub priority: i32,
    pub handler: Box<dyn HookHandler>,
}

#[async_trait]
pub trait HookHandler: Send + Sync {
    async fn handle(&self, context: &mut HookContext) -> Result<HookAction>;
}

pub enum HookAction {
    Continue,      // Continue to next hook
    Modify(Box<dyn Any>), // Modify context and continue
    Halt(Box<dyn Any>),   // Stop hook chain
}

impl HookRegistry {
    pub fn register(
        &mut self,
        point: HookPoint,
        plugin_id: &str,
        handler: Box<dyn HookHandler>,
        priority: i32,
    ) -> String {
        let id = format!("{}:{:?}:{}", plugin_id, point, uuid::Uuid::new_v4());

        let hook = RegisteredHook {
            id: id.clone(),
            plugin_id: plugin_id.to_string(),
            priority,
            handler,
        };

        let hooks = self.hooks.entry(point).or_default();
        hooks.push(hook);

        // Sort by priority (higher first)
        hooks.sort_by(|a, b| b.priority.cmp(&a.priority));

        id
    }

    pub async fn execute(
        &self,
        point: HookPoint,
        mut context: HookContext,
    ) -> Result<HookContext> {
        let hooks = self.hooks.get(&point).cloned().unwrap_or_default();

        for hook in hooks {
            match hook.handler.handle(&mut context).await? {
                HookAction::Continue => continue,
                HookAction::Modify(new_ctx) => {
                    context = *new_ctx.downcast().unwrap();
                }
                HookAction::Halt(final_ctx) => {
                    return Ok(*final_ctx.downcast().unwrap());
                }
            }
        }

        Ok(context)
    }
}
```

### OpenCode: Event-Driven Hooks

```typescript
// packages/opencode/src/plugin/index.ts
export interface Hooks {
  // Event handling
  event?: (input: { event: Event }) => Promise<void>;

  // Configuration modification
  config?: (input: Config) => Promise<void>;

  // Tool registration
  tool?: { [key: string]: ToolDefinition };

  // Authentication provider
  auth?: {
    provider: string;
    authenticate: (input: AuthInput) => Promise<AuthOutput>;
  };

  // Chat lifecycle hooks
  "chat.message"?: (
    input: { sessionID: string; message: Message },
    output: { message: Message },
  ) => Promise<void>;

  "chat.params"?: (
    input: { sessionID: string; params: ChatParams },
    output: { params: ChatParams },
  ) => Promise<void>;

  "chat.headers"?: (
    input: { sessionID: string; headers: Record<string, string> },
    output: { headers: Record<string, string> },
  ) => Promise<void>;

  // Permission handling
  "permission.ask"?: (
    input: { sessionID: string; permission: Permission },
    output: { approved: boolean },
  ) => Promise<void>;

  // Command lifecycle
  "command.execute.before"?: (
    input: { command: string; args: string[] },
    output: { skip?: boolean },
  ) => Promise<void>;

  "command.execute.after"?: (
    input: { command: string; args: string[]; output: string },
  ) => Promise<void>;

  // Tool lifecycle
  "tool.execute.before"?: (
    input: { sessionID: string; tool: string; input: unknown },
    output: { skip?: boolean; modifiedInput?: unknown },
  ) => Promise<void>;

  "tool.execute.after"?: (
    input: { sessionID: string; tool: string; result: unknown },
    output: { modifiedResult?: unknown },
  ) => Promise<void>;

  // Shell environment
  "shell.env"?: (
    input: { cwd: string },
    output: { env: Record<string, string> },
  ) => Promise<void>;
}

// Plugin loader
export namespace Plugin {
  const state = Instance.state(async () => {
    const hooks: Hooks[] = [];
    const seen = new Set<PluginInstance>();

    const input: PluginInput = {
      client: Instance.client,
      project: Instance.project,
      worktree: Instance.worktree,
      directory: Instance.directory,
      serverUrl: Server.url ?? new URL("http://localhost:4096"),
      $: Bun.$,
    };

    // Load internal plugins
    for (const plugin of INTERNAL_PLUGINS) {
      const init = await plugin(input).catch((err) => {
        console.error("Failed to load internal plugin:", err);
        return null;
      });
      if (init) hooks.push(init);
    }

    // Load external plugins
    const config = await Config.get();
    for (const pluginRef of config.plugins || []) {
      let pluginPath = pluginRef;

      // npm package installation
      if (!pluginRef.startsWith("file://")) {
        const [pkg, version] = pluginRef.split("@");
        pluginPath = await BunProc.install(pkg, version).catch((err) => {
          console.error(`Failed to install plugin ${pluginRef}:`, err);
          return null;
        });
      }

      if (!pluginPath) continue;

      // Dynamic import
      await import(pluginPath).then(async (mod) => {
        for (const [name, fn] of Object.entries<PluginInstance>(mod)) {
          if (seen.has(fn)) continue;
          seen.add(fn);

          const hook = await fn(input).catch((err) => {
            console.error(`Failed to initialize plugin ${name}:`, err);
            return null;
          });

          if (hook) hooks.push(hook);
        }
      });
    }

    return hooks;
  });
}
```

---

## 7. Security & Sandboxing

### IronClaw: Multi-Layer Security Model

```rust
// src/tools/wasm/allowlist.rs
pub struct NetworkAllowlist {
    allowed_hosts: Vec<HostPattern>,
    allowed_ips: Vec<IpRange>,
}

pub enum HostPattern {
    Exact(String),
    Wildcard(String),  // *.example.com
    Regex(Regex),
}

impl NetworkAllowlist {
    pub fn is_allowed(&self, url: &Url) -> bool {
        let host = url.host_str().unwrap_or("");

        // Check host patterns
        for pattern in &self.allowed_hosts {
            match pattern {
                HostPattern::Exact(h) if h == host => return true,
                HostPattern::Wildcard(pattern) => {
                    if Self::matches_wildcard(host, pattern) {
                        return true;
                    }
                }
                HostPattern::Regex(re) if re.is_match(host) => return true,
                _ => continue,
            }
        }

        // Check IP if URL has IP host
        if let Some(ip) = url.host() {
            if let Host::Ipv4(addr) = ip {
                for range in &self.allowed_ips {
                    if range.contains(addr) {
                        return true;
                    }
                }
            }
        }

        false
    }
}

// src/tools/wasm/limits.rs
pub struct ResourceLimiter {
    max_memory_pages: u32,
    max_fuel: u64,
    max_execution_time: Duration,
}

impl wasmtime::ResourceLimiter for ResourceLimiter {
    fn memory_growing(
        &mut self,
        current: usize,
        desired: usize,
        maximum: Option<usize>,
    ) -> Result<bool> {
        let max_bytes = self.max_memory_pages as usize * 64 * 1024;

        if desired > max_bytes {
            tracing::warn!(
                "WASM memory limit exceeded: desired {} > max {}",
                desired,
                max_bytes
            );
            return Ok(false);
        }

        Ok(true)
    }

    fn table_growing(
        &mut self,
        current: u32,
        desired: u32,
        maximum: Option<u32>,
    ) -> Result<bool> {
        Ok(desired <= 10000) // Max 10K table entries
    }
}
```

### OpenClaw: Path Boundary Validation

```typescript
// src/plugins/security.ts
export interface PluginSecurityPolicy {
  // Filesystem permissions
  allowFileRead: boolean;
  allowFileWrite: boolean;
  allowedPaths: string[];
  blockedPaths: string[];

  // Network permissions
  allowNetwork: boolean;
  allowedHosts: string[];
  blockedHosts: string[];

  // System permissions
  allowShell: boolean;
  allowProcessSpawn: boolean;
  allowedCommands: string[];

  // Prompt injection protection
  allowPromptInjection: boolean;
}

export function openBoundaryFileSync(
  pluginId: string,
  filePath: string,
  options?: { encoding?: BufferEncoding; flag?: string },
): string {
  // 1. Validate path is within allowed boundaries
  if (!isPathInside(filePath, getPluginAllowedPaths(pluginId))) {
    throw new PluginSecurityError(
      `Access denied: ${filePath} is outside plugin boundaries`,
    );
  }

  // 2. Resolve symlinks
  const resolved = fs.realpathSync(filePath);

  // 3. Re-validate resolved path (prevent traversal via symlink)
  if (!isPathInside(resolved, getPluginAllowedPaths(pluginId))) {
    throw new PluginSecurityError(
      `Path traversal detected: ${filePath} -> ${resolved}`,
    );
  }

  // 4. Reject hardlinks for non-bundled plugins
  if (!isBundledPlugin(pluginId) && isHardLink(resolved)) {
    throw new PluginSecurityError(
      `Hard links are not allowed for non-bundled plugins`,
    );
  }

  // 5. Read file
  return fs.readFileSync(resolved, options);
}

export function isPathInside(childPath: string, parentPaths: string[]): boolean {
  const resolved = path.resolve(childPath);
  return parentPaths.some((p) => resolved.startsWith(path.resolve(p)));
}

// Prompt injection protection
function registerTypedHook<K extends PluginHookName>(
  record: PluginRecord,
  hookName: K,
  handler: PluginHookHandlerMap[K],
  opts?: { priority?: number },
  policy?: PluginTypedHookPolicy,
): string {
  // Check policy
  if (hookName === "before_prompt_build" && !policy?.allowPromptInjection) {
    throw new PluginSecurityError(
      `Prompt injection hooks require explicit policy approval`,
    );
  }

  return registry.registerHook(hookName, handler, opts);
}
```

---

## 8. Comparison Matrices

### 8.1 Plugin Architecture Maturity

| Project | Level | WASM | MCP | Hooks | Skills | Tool Registry | Security |
|---------|-------|------|-----|-------|--------|---------------|----------|
| **IronClaw** | 4 | Full (wasmtime) | Client + Server | 6 lifecycle hooks | Trust-based | Yes | Path + Network + Resource |
| **ZeroClaw** | 3 | Channels only | Client | No | No | Yes | Path validation |
| **OpenClaw** | 3 | No | No | Event-based | SKILL.md | Full SDK | Path + Provenance |
| **CoPaw** | 2 | No | Client | No | Directory-based | Yes | Directory sandbox |
| **Kimi-CLI** | 2 | No | Client (via Kosong) | No | Layered | Yes | None |
| **OpenCode** | 2 | No | No | 10+ hooks | No | Via hooks | Trust |
| **Codex** | 1 | No | Client | No | No | Yes | None |
| **Gemini CLI** | 1 | No | Client | No | No | Yes | None |
| **Aider** | 1 | No | No | No | No | Function-based | None |
| **Pi-Mono** | 1 | No | No | No | No | Class-based | None |
| **Qwen Code** | 1 | No | No | No | No | Tool array | None |
| **NanoClaw** | 0 | No | No | No | No | Built-in only | None |
| **PicoClaw** | 0 | No | No | No | No | Built-in only | None |
| **Nanobot** | 0 | No | No | No | No | Built-in only | None |
| **Mimiclaw** | 0 | No | No | No | No | Built-in only | None |

### 8.2 Extension Point Coverage

| Extension Point | IronClaw | ZeroClaw | OpenClaw | CoPaw | Kimi-CLI | OpenCode |
|----------------|----------|----------|----------|-------|----------|----------|
| **Tools** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Hooks** | ✓ | - | ✓ | - | - | ✓ |
| **Channels** | ✓ | ✓ | ✓ | ✓ | - | - |
| **LLM Providers** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Embedding Providers** | ✓ | ✓ | - | - | - | - |
| **HTTP Routes** | - | - | ✓ | - | - | - |
| **CLI Commands** | - | ✓ | ✓ | ✓ | - | - |
| **Success Evaluators** | ✓ | - | - | - | - | - |
| **Observers** | ✓ | ✓ | - | - | - | - |

### 8.3 Skill System Comparison

| Feature | CoPaw | Kimi-CLI | IronClaw | OpenClaw |
|---------|-------|----------|----------|----------|
| **Format** | SKILL.md | SKILL.md | SKILL.md | SKILL.md |
| **Layering** | Source-based | 3-layer | Trust-based | Single |
| **Priority** | Source | Project > User > Builtin | Trust level | Single |
| **Gating** | No | No | Requirements check | No |
| **Scoring** | No | No | Keyword matching | No |
| **Budget** | No | No | Token-based | No |
| **Attenuation** | No | No | Tool ceiling | No |
| **Sync** | Yes | No | No | No |

### 8.4 MCP Implementation Comparison

| Feature | CoPaw | Kimi-CLI | IronClaw | Codex | OpenCode |
|---------|-------|----------|----------|-------|----------|
| **Stdio Transport** | ✓ | ✓ | ✓ | ✓ | - |
| **HTTP/SSE Transport** | ✓ | ✓ | ✓ | ✓ | - |
| **Unix Socket** | - | - | ✓ | - | - |
| **Server Mode** | - | - | ✓ | - | - |
| **Tool Discovery** | ✓ | ✓ | ✓ | ✓ | - |
| **Resource Support** | ✓ | ✓ | ✓ | - | - |
| **Multiple Servers** | ✓ | ✓ | ✓ | ✓ | - |

### 8.5 Security Mechanism Comparison

| Mechanism | IronClaw | ZeroClaw | OpenClaw | CoPaw |
|-----------|----------|----------|----------|-------|
| **Path Validation** | ✓ (realpath) | ✓ | ✓ (symlink check) | ✓ |
| **Network Allowlist** | ✓ (wildcard) | - | ✓ | - |
| **Resource Limits** | ✓ (fuel) | - | - | - |
| **Memory Limits** | ✓ (128MB) | - | - | - |
| **Provenance Tracking** | ✓ | - | ✓ | ✓ |
| **Hardlink Prevention** | - | - | ✓ | - |
| **Symlink Validation** | ✓ | - | ✓ | - |

---

## 9. Recommended Unified Plugin Architecture

Based on analysis of all 15 projects, here's a recommended unified architecture combining best practices:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Unified Plugin Architecture                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Discovery (Multi-Source)                          │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │   │
│  │  │ Filesystem   │ │ MCP Registry │ │ npm/GitHub   │ │   Bundled    │ │   │
│  │  │  (Layered)   │ │              │ │              │ │              │ │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Loading (Sandboxed)                               │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                 │   │
│  │  │ Native       │ │ WASM         │ │ MCP          │                 │   │
│  │  │ (Rust/Go/TS) │ │ (wasmtime)   │ │ (External)   │                 │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Lifecycle Management                              │   │
│  │                                                                     │   │
│  │   Install → Enable → Load → [Active] → Disable → Uninstall          │   │
│  │      │        │       │       │         │          │                │   │
│  │      ▼        ▼       ▼       ▼         ▼          ▼                │   │
│  │   [Download] [Deps] [Init] [Running] [Cleanup] [Remove]             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Extension Points                                  │   │
│  │  ┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐ │   │
│  │  │  Tools   │  Hooks   │ Channels │ Providers│  Skills  │   CLI    │ │   │
│  │  │          │          │          │          │          │          │ │   │
│  │  │ • Shell  │ • Before │ • Discord│ • LLM    │• Markdown│ • Cmds   │ │   │
│  │  │ • File   │ • After  │ • Slack  │ • Embed  │ • Schema │ • Flags  │ │   │
│  │  │ • Web    │ • Transform│ •Telegram│ • STT   │ • Scripts│ • Args   │ │   │
│  │  └──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Security Layer                                    │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │   │
│  │  │ Path         │ │ Network      │ │ Resource     │ │ Permission   │ │   │
│  │  │ Validation   │ │ Allowlist    │ │ Limits       │ │ Model        │ │   │
│  │  │ (realpath)   │ │ (wildcard)   │ │ (fuel/mem)   │ │ (trust)      │ │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Recommended Configuration

```yaml
# plugin-config.yaml
plugin_system:
  discovery:
    sources:
      - type: filesystem
        paths:
          - "./.agent/plugins"      # Project level
          - "~/.config/agent/plugins" # User level
          - "/usr/share/agent/plugins" # System level
        priority: [project, user, system]

      - type: mcp_registry
        url: "https://registry.mcp.io"
        auto_discover: true

      - type: npm
        registry: "https://registry.npmjs.org"
        scope: "@agent-plugin"

  loading:
    wasm:
      enabled: true
      runtime: wasmtime
      max_memory_mb: 128
      max_fuel: 1000000000
      enable_wasi: true

    native:
      allowed_extensions: [".so", ".dylib", ".dll"]
      require_signed: false

  lifecycle:
    auto_enable: false
    verify_on_install: true
    rollback_on_error: true

  security:
    default_policy:
      filesystem:
        read: true
        write: false
        allowed_paths: ["./workspace"]

      network:
        allow: false
        allowed_hosts: ["api.openai.com", "*.anthropic.com"]

      system:
        allow_shell: false
        allowed_commands: ["git", "npm"]

    trust_levels:
      builtin:
        policy: unrestricted

      installed:
        policy: restricted
        filesystem: read_only
        network: allowlisted

      trusted:
        policy: permissive
        filesystem: read_write
        network: allowlisted

  extension_points:
    tools:
      auto_register: true
      prefix_with_plugin_id: true

    hooks:
      points:
        - before_inbound
        - before_tool_call
        - before_outbound
        - on_session_start
        - on_session_end

      execution:
        order: priority
        abort_on_error: false

    skills:
      format: markdown
      frontmatter_schema: skill-v1
      max_tokens_per_session: 4000
```

---

## 10. Appendix: Key Implementation Files

| Project | File | Purpose |
|---------|------|---------|
| **IronClaw** | `src/tools/wasm/runtime.rs` | WASM runtime with wasmtime |
| **IronClaw** | `src/tools/wasm/host.rs` | Host functions for WASM |
| **IronClaw** | `src/tools/mcp/client.rs` | MCP client implementation |
| **IronClaw** | `src/hooks/mod.rs` | Lifecycle hook system |
| **IronClaw** | `src/skills/manager.rs` | Trust-based skill system |
| **ZeroClaw** | `src/channels/wasm/runtime.rs` | WASM channel runtime |
| **ZeroClaw** | `src/tools/traits.rs` | Tool trait definitions |
| **CoPaw** | `src/copaw/agents/skills_manager.py` | Skill lifecycle management |
| **CoPaw** | `src/copaw/tools/mcp_client.py` | MCP client integration |
| **Kimi-CLI** | `src/kimi_cli/skill/__init__.py` | Layered skill resolution |
| **Kimi-CLI** | `packages/kosong/src/kosong/tooling/mcp.py` | MCP tooling adapter |
| **OpenClaw** | `src/plugins/loader.ts` | Plugin loading with jiti |
| **OpenClaw** | `src/plugins/registry.ts` | Extension registry |
| **OpenClaw** | `src/plugins/security.ts` | Security policies |
| **OpenCode** | `packages/opencode/src/plugin/index.ts` | Hook-based plugins |
| **Codex** | `codex-rs/core/src/tools/registry.rs` | Rust tool registry |
| **Gemini CLI** | `src/mcp/client.ts` | MCP client |
| **Aider** | `aider/coders/base_coder.py` | Function-based tools |

---

*Report Generation: 2026-03-18*
*Analysis Coverage: 15/15 AI Agent Projects*
*Categories: WASM Plugins, MCP Integration, Skill Systems, Tool Registries, Hook Systems, Security*
