# AI Agent 工具系统与 Skills 体系深度调研报告（完整版）

## 概述

本报告对 **15 个开源 AI Agent 项目**（8 个个人助手 + 7 个编程代理）的工具系统（Tool System）和 Agent Skills 体系进行深度技术分析，对比各自的实现策略、安全机制、扩展能力，并提出融合各项目最佳实践的推荐架构。

**分析范围：**

| 类别 | 项目 | 技术栈 |
|------|------|--------|
| **个人助手** | IronClaw | Rust |
| | ZeroClaw | Rust |
| | NanoClaw | TypeScript |
| | CoPaw | Python |
| | Nanobot | Python |
| | OpenClaw | TypeScript |
| | PicoClaw | Go |
| | MimiClaw | C |
| **编程代理** | Kimi-CLI | Python |
| | Pi-Mono | TypeScript |
| | OpenCode | TypeScript |
| | Gemini-CLI | TypeScript |
| | Codex | Rust |
| | Aider | Python |
| | Qwen-Code | TypeScript |

---

## 一、工具注册架构对比

### 1.1 架构模式分类

根据实现语言，工具注册架构可分为三大类：

#### A. Trait/Interface 模式（Rust、Go）

**IronClaw（Rust）** - 最完善的 Trait 系统：

```rust
// src/tools/tool.rs
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, params: serde_json::Value, ctx: &JobContext) -> Result<ToolOutput, ToolError>;
    fn requires_approval(&self, _params: &serde_json::Value) -> ApprovalRequirement;
    fn domain(&self) -> ToolDomain;
    fn rate_limit_config(&self) -> Option<ToolRateLimitConfig>;
}

// ToolRegistry 使用 RwLock 保证线程安全
pub struct ToolRegistry {
    tools: RwLock<HashMap<String, Arc<dyn Tool>>>,
}
```

**ZeroClaw（Rust）** - 简化版 Trait：

```rust
// src/tools/traits.rs
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
    fn spec(&self) -> ToolSpec;
}
```

**PicoClaw（Go）** - Interface + TTL：

```go
// pkg/tools/base.go
type Tool interface {
    Name() string
    Description() string
    Parameters() map[string]any
    Execute(ctx context.Context, args map[string]any) *ToolResult
}

type AsyncExecutor interface {
    Tool
    ExecuteAsync(ctx context.Context, args map[string]any, cb AsyncCallback) *ToolResult
}

// 带 TTL 的隐藏工具
type HiddenTool struct {
    Tool
    ExpiresAt time.Time
}
```

#### B. Class/ABC 模式（Python）

**Nanobot** - 抽象基类 + Schema 映射：

```python
# nanobot/nanobot/agent/tools/base.py
class Tool(ABC):
    _TYPE_MAP = {
        "string": str, "integer": int, "number": (int, float),
        "boolean": bool, "array": list, "object": dict,
    }

    @property
    @abstractmethod
    def name(self) -> str: ...

    @property
    @abstractmethod
    def description(self) -> str: ...

    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]: ...

    @abstractmethod
    async def execute(self, **kwargs: Any) -> str: ...
```

**Kimi-CLI** - 装饰器 + 自动发现：

```python
# src/kimi_cli/tools/__init__.py
@tool
def read_file(
    file_path: str,
    offset: int | None = None,
    limit: int | None = None,
) -> str:
    """Read a file from disk.

    Args:
        file_path: Absolute path to the file
        offset: Line number to start reading from (1-based)
        limit: Maximum number of lines to read
    """
    ...

def get_tools() -> list[Callable[..., Any]]:
    """Get all tool functions."""
    tools = []
    for name, obj in globals().items():
        if callable(obj) and hasattr(obj, "_is_tool"):
            tools.append(obj)
    return tools
```

#### C. Schema-First 模式（TypeScript）

**OpenCode** - Zod Schema：

```typescript
// packages/opencode/src/tool/tool.ts
export namespace Tool {
  export interface Info<Parameters extends z.ZodType = z.ZodType> {
    id: string
    init: (ctx?: InitContext) => Promise<{
      description: string
      parameters: Parameters
      execute(args: z.infer<Parameters>, ctx: Context): Promise<Result>
    }>
  }
}
```

**Pi-Mono** - TypeBox 验证：

```typescript
// packages/coding-agent/src/core/tools/index.ts
export type Tool = AgentTool<any>;
export const codingTools: Tool[] = [readTool, bashTool, editTool, writeTool];
export const readOnlyTools: Tool[] = [readTool, grepTool, findTool, lsTool];
```

**Gemini-CLI** - 注册表模式：

```typescript
// packages/core/src/tools/tool-registry.ts
class ToolRegistry {
  private allKnownTools: Map<string, AnyDeclarativeTool> = new Map();

  registerTool(tool: AnyDeclarativeTool): void {
    const fullyQualifiedName = tool.getFullyQualifiedName();
    if (this.allKnownTools.has(fullyQualifiedName)) {
      throw new Error(`Tool ${fullyQualifiedName} already registered`);
    }
    this.allKnownTools.set(fullyQualifiedName, tool);
  }

  sortTools(): void {
    // Built-in < Discovered < MCP
  }
}
```

### 1.2 Schema 定义方式对比

| 项目 | Schema 定义方式 | 示例 |
|------|----------------|------|
| **IronClaw** | JSON Schema + Rust 类型 | `parameters_schema()` 返回 `serde_json::Value` |
| **ZeroClaw** | JSON Schema | `parameters_schema()` 返回 `serde_json::Value` |
| **Nanobot** | Python dict + `_TYPE_MAP` | `parameters` 返回 `dict[str, Any]` |
| **Kimi-CLI** | Docstring + `generate_schema()` | 自动从 docstring 提取 |
| **OpenCode** | Zod | `z.object({ path: z.string() })` |
| **Pi-Mono** | TypeBox | `Type.Object({ path: Type.String() })` |
| **Gemini-CLI** | 装饰器元数据 | `@tool({ schema: {...} })` |
| **Codex** | JSON Schema | `parameters: serde_json::Value` |
| **PicoClaw** | Go map | `Parameters() map[string]any` |
| **MimiClaw** | C JSON 字符串 | `input_schema_json: *const c_char` |
| **CoPaw** | Docstring + type hints | AgentScope `Toolkit` |
| **Aider** | 隐式（Coder 类方法） | 方法参数定义 |

### 1.3 冲突解决策略

| 策略 | 项目 | 实现 |
|------|------|------|
| **Skip** | IronClaw, CoPaw | 跳过已存在工具 |
| **Override** | IronClaw, ZeroClaw | 替换已存在工具 |
| **Rename** | IronClaw | 自动生成唯一名称 |
| **Raise** | IronClaw, Gemini-CLI | 抛出异常 |
| **Namespace** | Codex | `namespace:tool_name` 格式 |

---

## 二、安全机制深度对比

### 2.1 审批模式分类

| 模式 | 项目 | 说明 |
|------|------|------|
| **Never** | IronClaw, ZeroClaw | 从不询问 |
| **Auto/UnlessAutoApproved** | IronClaw, Gemini-CLI | 自动批准（除非敏感） |
| **Prompt/Confirm** | IronClaw, Gemini-CLI, Kimi-CLI, OpenCode | 敏感操作询问 |
| **Always** | IronClaw | 总是询问 |
| **Plan** | Gemini-CLI | 计划模式（批量确认） |

**IronClaw ApprovalRequirement 枚举：**

```rust
pub enum ApprovalRequirement {
    Never,                    // 从不询问
    UnlessAutoApproved,       // 除非自动批准
    Always,                   // 总是询问
}
```

**Gemini-CLI 审批策略引擎：**

```typescript
enum ApprovalMode {
  AUTO = 'auto',
  CONFIRM = 'confirm',
  PLAN = 'plan',
}

enum PolicyDecision {
  ALLOW = 'allow',
  DENY = 'deny',
  ASK_USER = 'ask_user',
}

enum ConfirmationOutcome {
  ProceedOnce,
  ProceedAlways,
  ProceedAlwaysAndSave,
}
```

### 2.2 多层安全架构

**IronClaw - 分层安全模型：**

```
Layer 1: ToolDomain (Orchestrator vs Container)
Layer 2: ApprovalRequirement
Layer 3: Rate Limiting (ToolRateLimitConfig)
Layer 4: Sensitive Param Redaction
Layer 5: WASM Sandboxing (可选)
```

**CoPaw - ToolGuard 六层防护：**

```
Layer 1: Deny List（无条件拒绝）
Layer 2: Pre-approval（预授权缓存）
Layer 3: Guard Engine（规则引擎检测）
Layer 4: Approval Flow（人工审批）
Layer 5: Audit Log（审计日志）
Layer 6: Rate Limiting（速率限制）
```

**Codex - 沙箱策略：**

```rust
pub enum SandboxPolicy {
    ReadOnly,                 // 只读
    WorkspaceWrite,           // 工作区写入
    FullAccess,               // 完全访问
    NetworkRestricted,        // 网络受限
}

pub trait ToolHandler: Send + Sync {
    async fn is_mutating(&self, invocation: &ToolInvocation) -> bool;
    async fn handle(&self, invocation: ToolInvocation) -> Result<Self::Output, FunctionCallError>;
}
```

### 2.3 威胁检测与规则引擎

**IronClaw 威胁分类：**

| 分类 | 说明 |
|------|------|
| COMMAND_INJECTION | 命令注入 |
| DATA_EXFILTRATION | 数据外泄 |
| PATH_TRAVERSAL | 路径遍历 |
| SENSITIVE_FILE_ACCESS | 敏感文件访问 |
| NETWORK_ABUSE | 网络滥用 |
| CREDENTIAL_EXPOSURE | 凭证泄露 |
| PROMPT_INJECTION | 提示注入 |

**Nanobot 路径遍历检测：**

```python
def _is_path_traversal_attempt(self, path: str) -> bool:
    """Detect path traversal attempts."""
    normalized = os.path.normpath(path)
    return ".." in normalized or normalized.startswith("/")
```

**Gemini-CLI 策略配置：**

```typescript
interface PolicyRule {
  tool: string;
  condition?: (params: any) => boolean;
  decision: PolicyDecision;
}
```

### 2.4 隔离机制对比

| 项目 | 隔离机制 | 实现 |
|------|---------|------|
| **IronClaw** | WASM Sandboxing | `wasmtime` 运行时 |
| | Docker Container | `SandboxContainer` |
| **ZeroClaw** | 进程隔离 | 子进程执行 |
| **NanoClaw** | Docker Container | 只读挂载 + 凭证代理 |
| **Codex** | Sandbox Policy | 文件系统 + 网络策略 |
| **PicoClaw** | Context 隔离 | channel/chatID 隔离 |
| **MimiClaw** | SPIFFS 路径验证 | `MIMI_SPIFFS_BASE` 前缀检查 |

---

## 三、Skills 与提示扩展体系

### 3.1 Skills 定义格式

**IronClaw - SKILL.md + AGENTS.md：**

```markdown
---
name: github-pr-review
description: Review GitHub pull requests
trust: trusted  # trusted vs installed
---

# System Prompt

You are a PR reviewer. Focus on:
- Code quality
- Security issues
- Test coverage

## Tools

- `github_get_pr`: Get PR details
- `github_get_diff`: Get PR diff
```

**Kimi-CLI - SKILL.md + Hooks：**

```
.skills/
├── file_reader/
│   ├── SKILL.md          # 技能定义
│   └── hooks/            # 钩子脚本
│       ├── pre-tool.py   # 工具执行前
│       ├── post-tool.py  # 工具执行后
│       └── pre-respond.py  # 响应前
```

**OpenCode - 最完善的 Skills 系统：**

```markdown
---
name: github-pr-review
description: Review GitHub pull requests
type: skill
version: 1.0.0
author: opencode
---

# GitHub PR Review

## System Prompt

You are a PR reviewer...

## Tools

- `github_get_pr`
- `github_get_diff`

## Workflows

### Review PR

1. Get PR details
2. Get the diff
3. Analyze changes
4. Post comments
```

### 3.2 Skills 信任模型

| 项目 | 信任级别 | 说明 |
|------|---------|------|
| **IronClaw** | Trusted / Installed | 用户放置 vs 注册表安装 |
| **OpenCode** | 隐式信任 | 用户手动安装 |
| **Nanobot** | 无 | 不支持动态 skills |
| **CoPaw** | Builtin / Customized / Active | 三级目录结构 |

### 3.3 动态工具注册

**IronClaw 技能运行时：**

```rust
impl SkillManager {
    pub fn enable_skill(&mut self, name: &str) -> Result<(), SkillError> {
        let skill = self.skills.get(name)?;

        // Register skill tools
        for tool in &skill.tools {
            self.tool_registry.register(tool)?;
        }

        // Inject system prompt
        self.active_skills.insert(name.to_string());
        Ok(())
    }
}
```

**OpenCode 动态 Python 工具：**

```typescript
class SkillLoader {
    async loadSkill(skillPath: string): Promise<Skill> {
        const skillMdPath = path.join(skillPath, "SKILL.md");
        const skill = this.parseSkillMd(await fs.readFile(skillMdPath, "utf-8"));

        // Load Python code if exists
        const initPyPath = path.join(skillPath, "__init__.py");
        if (await fs.exists(initPyPath)) {
            const tools = await this.loadPythonTools(initPyPath);
            skill.tools.push(...tools);
        }

        return skill;
    }
}
```

### 3.4 Hook 系统对比

| 项目 | Hook 点 | 实现 |
|------|--------|------|
| **IronClaw** | BeforeInbound, BeforeToolCall, BeforeOutbound, OnSessionStart, OnSessionEnd, TransformResponse | Trait-based hooks |
| **Kimi-CLI** | pre-tool, post-tool, pre-respond | Python 脚本执行 |
| **OpenClaw** | Generic Hook | `HookManager.register()` |
| **OpenCode** | Skill-level hooks | `SkillHooks` interface |

**IronClaw Hook Trait：**

```rust
pub trait Hook: Send + Sync {
    fn name(&self) -> &str;
    async fn execute(&self, context: &mut HookContext) -> Result<(), HookError>;
}

pub enum HookPoint {
    BeforeInbound,
    BeforeToolCall,
    BeforeOutbound,
    OnSessionStart,
    OnSessionEnd,
    TransformResponse,
}
```

---

## 四、MCP (Model Context Protocol) 支持

### 4.1 MCP 客户端实现

| 项目 | 支持程度 | 传输方式 | 特殊功能 |
|------|---------|---------|---------|
| **IronClaw** | ⭐⭐⭐ 完整 | stdio / HTTP / SSE | 动态 client 注册，工具自动发现 |
| **Nanobot** | ⭐⭐⭐ 完整 | stdio / SSE / HTTP | `MCPToolWrapper`，`enabled_tools` 过滤 |
| **Codex** | ⭐⭐⭐ 完整 | stdio / HTTP | `ToolKind::Mcp`，命名空间支持 |
| **Gemini-CLI** | ⭐⭐⭐ 完整 | stdio / HTTP / SSE | `DiscoveredMCPTool`，服务器命名空间 |
| **PicoClaw** | ⭐⭐⭐ 完整 | stdio / HTTP | `MCPManager`，工具调用路由 |
| **Kimi-CLI** | ⭐⭐ 部分 | ACP | ACP (Agent Communication Protocol) |
| **OpenClaw** | ⭐ 有限 | stdio | 基础支持 |
| **其他** | ❌ 无 | - | - |

### 4.2 MCP 工具集成示例

**IronClaw MCP 管理器：**

```rust
pub struct MCPManager {
    clients: HashMap<String, Arc<dyn MCPClient>>,
    tool_registry: Arc<ToolRegistry>,
}

#[async_trait]
pub trait MCPClient: Send + Sync {
    async fn connect(&self) -> Result<(), MCPError>;
    async fn disconnect(&self) -> Result<(), MCPError>;
    async fn list_tools(&self) -> Result<Vec<ToolDefinition>, MCPError>;
    async fn call_tool(&self, name: &str, params: Value) -> Result<Value, MCPError>;
}

impl MCPManager {
    pub async fn register_client(&mut self, config: MCPConfig) -> Result<(), MCPError> {
        let client = self.create_client(&config).await?;
        client.connect().await?;

        // Discover and register tools
        let tools = client.list_tools().await?;
        for tool in tools {
            self.tool_registry.register(MCPToolWrapper::new(client.clone(), tool))?;
        }

        self.clients.insert(config.name, client);
        Ok(())
    }
}
```

**Nanobot MCP 包装器：**

```python
class MCPToolWrapper(Tool):
    """Wraps an MCP server tool as a nanobot Tool."""

    def __init__(self, server_name: str, mcp_tool: dict, client: MCPClient):
        self.server_name = server_name
        self.mcp_tool = mcp_tool
        self.client = client

    @property
    def name(self) -> str:
        return f"mcp_{self.server_name}_{self.mcp_tool['name']}"

    async def execute(self, **kwargs: Any) -> str:
        result = await self.client.call_tool(self.mcp_tool['name'], kwargs)
        return json.dumps(result)
```

**Codex MCP 集成：**

```rust
#[async_trait]
pub trait ToolHandler: Send + Sync {
    fn kind(&self) -> ToolKind;
    async fn handle(&self, invocation: ToolInvocation) -> Result<Self::Output, FunctionCallError>;
}

pub enum ToolKind {
    Function,  // 内置工具
    Mcp,       // MCP 工具
}

pub struct McpToolHandler {
    server_name: String,
    tool_name: String,
    connection_manager: Arc<McpConnectionManager>,
}
```

**Gemini-CLI MCP 命名空间：**

```typescript
class DiscoveredMCPTool implements AnyDeclarativeTool {
    constructor(
        private serverName: string,
        private tool: MCPTool,
    ) {}

    getFullyQualifiedName(): string {
        return `${this.serverName}:${this.tool.name}`;
    }

    async execute(params: unknown): Promise<unknown> {
        return await mcpClient.callTool(this.tool.name, params);
    }
}
```

---

## 五、内置工具集对比

### 5.1 通用工具矩阵

| 工具类别 | IronClaw | ZeroClaw | Nanobot | Kimi-CLI | Pi-Mono | Codex | OpenCode | Gemini-CLI |
|---------|----------|----------|---------|----------|---------|-------|----------|------------|
| **文件读取** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **文件写入** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **文件编辑** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **目录列表** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Shell 执行** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **代码搜索** | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **浏览器操作** | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **网络请求** | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| **内存搜索** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **子代理** | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ |
| **Git 操作** | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **图片查看** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ |

### 5.2 特殊工具功能

**IronClaw：**
- `ApplyPatchTool` - 统一补丁应用
- 内存工具四件套：`memory_search`, `memory_write`, `memory_read`, `memory_tree`
- 子代理工具：`spawn_agent`, `send_message`, `wait_for_agent`

**Kimi-CLI：**
- `D-Mail` - 时间回溯消息机制
- `CreateSubagent` - 子代理创建
- `Task`/`TaskList`/`TaskOutput` - 任务管理
- `Think` - 思考工具

**Codex：**
- 多代理工具：`spawn`, `send_input`, `wait`, `close`, `resume`
- JS REPL：`js_repl`
- 图片查看：`view_image`

**Gemini-CLI：**
- Plan 模式：`.md` 文件批量编辑
- 输出掩码：敏感数据脱敏

**Pi-Mono：**
- 输出截断：`DEFAULT_MAX_BYTES = 30KB`, `DEFAULT_MAX_LINES = 300`
- 进程树终止

**OpenCode：**
- 图片/PDF 支持：base64 编码
- LSP 集成：文件预热
- 二进制文件检测

---

## 六、综合对比矩阵

### 6.1 工具系统对比表

| 维度 | IronClaw | ZeroClaw | Nanobot | Kimi-CLI | Pi-Mono | Codex | OpenCode | Gemini-CLI |
|------|----------|----------|---------|----------|---------|-------|----------|------------|
| **注册方式** | Trait | Trait | ABC | 装饰器 | Factory | Trait | Zod Schema | 注册表 |
| **Schema 定义** | JSON Schema | JSON Schema | Python dict | Docstring | TypeBox | JSON Schema | Zod | 装饰器 |
| **冲突解决** | 4 种策略 | Override | 覆盖 | 覆盖 | 覆盖 | Namespace | 覆盖 | Raise |
| **线程安全** | RwLock | 无 | GIL | GIL | 单线程 | RwLock | 单线程 | Map |
| **异步支持** | ✅ async_trait | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **工具数量** | 15+ | 8+ | 10+ | 20+ | 7 | 12+ | 10+ | 15+ |

### 6.2 安全机制对比表

| 维度 | IronClaw | CoPaw | Codex | Gemini-CLI | OpenCode | Nanobot |
|------|----------|-------|-------|------------|----------|---------|
| **审批模式** | 4 级 | 6 层 | SandboxPolicy | 3 级 + Plan | 3 级 | Pattern |
| **威胁分类** | 7 种 | 7 种 | 基础 | 详细 | 基础 | 基础 |
| **规则引擎** | YAML | YAML | 代码 | JSON | 代码 | 代码 |
| **沙箱** | WASM/Docker | 无 | Docker | 无 | 无 | 无 |
| **审计日志** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **速率限制** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

### 6.3 Skills 体系对比表

| 维度 | IronClaw | Kimi-CLI | OpenCode | CoPaw | Nanobot |
|------|----------|----------|----------|-------|---------|
| **定义文件** | SKILL.md | SKILL.md | SKILL.md | SKILL.md | 无 |
| **信任模型** | Trusted/Installed | 隐式 | 隐式 | 三级 | 无 |
| **Hook 系统** | 6 阶段 | 3 阶段 | Skill 级 | 无 | 无 |
| **动态工具** | ✅ | ❌ | ✅ (Python) | ✅ (MCP) | ❌ |
| **运行时隔离** | WASM | 无 | 沙箱 | 无 | 无 |
| **CRUD API** | ✅ | 简单 | 简单 | 完整 | 无 |

### 6.4 MCP 支持对比表

| 项目 | 支持 | 传输 | 命名空间 | 动态发现 |
|------|------|------|---------|---------|
| **IronClaw** | ⭐⭐⭐ | stdio/HTTP/SSE | ✅ | ✅ |
| **Nanobot** | ⭐⭐⭐ | stdio/HTTP/SSE | ✅ | ✅ |
| **Codex** | ⭐⭐⭐ | stdio/HTTP | ✅ | ✅ |
| **Gemini-CLI** | ⭐⭐⭐ | stdio/HTTP/SSE | ✅ | ✅ |
| **PicoClaw** | ⭐⭐⭐ | stdio/HTTP | ❌ | ✅ |
| **Kimi-CLI** | ⭐⭐ | ACP | ❌ | ❌ |
| **OpenClaw** | ⭐ | stdio | ❌ | ❌ |
| **其他** | ❌ | - | - | - |

---

## 七、推荐融合架构

### 7.1 核心设计原则

基于对 15 个项目的深度分析，推荐采用以下融合架构：

1. **分层安全**：采用 IronClaw/CoPaw 的多层安全模型
2. **灵活注册**：采用 IronClaw 的 Trait + Codex 的命名空间
3. **Schema 优先**：采用 Zod/TypeBox 的类型安全方案
4. **动态扩展**：采用 OpenCode 的 Skill 动态工具注册
5. **标准协议**：采用 MCP 作为外部工具标准
6. **版本控制**：支持 Skill 版本管理和热更新

### 7.2 推荐架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     推荐工具系统架构 (Unified Tool System)               │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 1: Tool Registry                                                  │
│          - Trait-based 工具定义 (@tool 装饰器自动注册)                  │
│          - 命名空间支持 (namespace:tool_name)                           │
│          - 冲突解决策略 (override/skip/rename/raise)                    │
│          - 线程安全 (RwLock/Atomic)                                     │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 2: Schema System                                                  │
│          - Zod/TypeBox/JSON Schema 多后端支持                           │
│          - 自动从 Docstring/注释生成 Schema                             │
│          - 参数验证和类型转换                                           │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 3: Security Guard                                                 │
│          - Deny List（黑名单）                                           │
│          - Rule Engine（YAML/JSON 规则）                                │
│          - Pre-approval（预授权）                                        │
│          - Approval Flow（审批流: Never/Auto/Prompt/Always）            │
│          - Audit Log（审计日志）                                         │
│          - Rate Limiting（速率限制）                                     │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 4: Isolation Layer                                                │
│          - Native（本地执行）                                            │
│          - WASM Sandbox（wasmtime）                                      │
│          - Docker Container（完整隔离）                                  │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 5: Skill System                                                   │
│          - SKILL.md 定义                                                 │
│          - Python/TypeScript/Rust 代码扩展                              │
│          - 动态工具注册                                                  │
│          - 信任模型 (Trusted/Installed/Unknown)                         │
│          - 运行时隔离                                                    │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 6: Hook System                                                    │
│          - BeforeInbound                                                 │
│          - BeforeToolCall                                                │
│          - AfterToolCall                                                 │
│          - BeforeOutbound                                                │
│          - OnSessionStart/End                                            │
│          - TransformResponse                                             │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 7: MCP Integration                                                │
│          - stdio / HTTP / SSE 传输                                       │
│          - 动态 client 注册                                              │
│          - 工具自动发现                                                  │
│          - 服务器命名空间                                                │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 8: Execution Engine                                               │
│          - 工具执行生命周期管理                                          │
│          - 超时处理                                                      │
│          - 错误处理和重试                                                │
│          - 输出截断                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.3 关键实现代码

```rust
// ==================== Layer 1: Tool Registry ====================

pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn namespace(&self) -> Option<&str> { None }
    fn fully_qualified_name(&self) -> String {
        match self.namespace() {
            Some(ns) => format!("{}:{}", ns, self.name()),
            None => self.name().to_string(),
        }
    }
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, params: Value, ctx: &ToolContext) -> Result<ToolOutput, ToolError>;
    fn approval_requirement(&self, params: &Value) -> ApprovalRequirement;
    fn domain(&self) -> ToolDomain { ToolDomain::Native }
    fn rate_limit(&self) -> Option<RateLimitConfig> { None }
}

pub struct ToolRegistry {
    tools: RwLock<HashMap<String, Arc<dyn Tool>>>,
    conflict_strategy: ConflictStrategy,
}

impl ToolRegistry {
    pub fn register(&self, tool: Arc<dyn Tool>) -> Result<(), ToolError> {
        let name = tool.fully_qualified_name();
        let mut tools = self.tools.write();

        if tools.contains_key(&name) {
            match self.conflict_strategy {
                ConflictStrategy::Skip => return Ok(()),
                ConflictStrategy::Raise => return Err(ToolError::AlreadyExists(name)),
                ConflictStrategy::Override => {},
            }
        }

        tools.insert(name, tool);
        Ok(())
    }
}

// ==================== Layer 3: Security Guard ====================

pub enum ApprovalRequirement {
    Never,                    // 从不询问
    UnlessAutoApproved,       // 除非自动批准
    Always,                   // 总是询问
}

pub enum ToolDomain {
    Native,                   // 本地执行
    Wasm,                     // WASM 沙箱
    Container,                // Docker 容器
}

pub struct SecurityManager {
    rules: Vec<SecurityRule>,
    deny_list: HashSet<String>,
    pre_approvals: HashMap<String, PreApproval>,
    audit_log: AuditLogger,
}

impl SecurityManager {
    pub async fn check(&self, tool: &str, params: &Value) -> SecurityResult {
        // 1. Deny List
        if self.deny_list.contains(tool) {
            return SecurityResult::Denied("Tool in deny list");
        }

        // 2. Pre-approval
        let key = self.hash(tool, params);
        if let Some(approval) = self.pre_approvals.get(&key) {
            if !approval.expired() {
                return SecurityResult::Allowed(ApprovalSource::PreApproval);
            }
        }

        // 3. Rule Engine
        let findings: Vec<SecurityFinding> = self.rules
            .iter()
            .filter(|r| r.applies_to(tool))
            .filter(|r| r.matches(params))
            .map(|r| r.to_finding())
            .collect();

        if findings.iter().any(|f| f.severity == Severity::Critical) {
            self.audit_log.record(tool, params, &findings, Action::Deny);
            return SecurityResult::Denied("Critical finding");
        }

        if !findings.is_empty() {
            return SecurityResult::PendingApproval(findings);
        }

        SecurityResult::Allowed(ApprovalSource::Auto)
    }
}

// ==================== Layer 5: Skill System ====================

pub struct Skill {
    name: String,
    version: Version,
    trust_level: TrustLevel,      // Trusted / Installed / Unknown
    system_prompt: Option<String>,
    tools: Vec<Arc<dyn Tool>>,
    hooks: Option<SkillHooks>,
    runtime: SkillRuntime,        // Native / Wasm / Container
}

pub struct SkillHooks {
    before_tool: Option<Box<dyn Hook>>,
    after_tool: Option<Box<dyn Hook>>,
    before_response: Option<Box<dyn Hook>>,
}

pub struct SkillManager {
    skills: HashMap<String, Skill>,
    active_skills: HashSet<String>,
    tool_registry: Arc<ToolRegistry>,
}

impl SkillManager {
    pub async fn load(&mut self, path: &Path) -> Result<Skill, SkillError> {
        let skill_md = tokio::fs::read_to_string(path.join("SKILL.md")).await?;
        let skill = self.parse_skill_md(&skill_md)?;

        // Load code extension if exists
        if let Some(ext) = self.detect_extension(&path) {
            let ext_tools = ext.load_tools().await?;
            skill.tools.extend(ext_tools);
        }

        Ok(skill)
    }

    pub fn enable(&mut self, name: &str) -> Result<(), SkillError> {
        let skill = self.skills.get(name)?;

        // Check trust level
        if skill.trust_level == TrustLevel::Unknown {
            return Err(SkillError::Untrusted);
        }

        // Register tools
        for tool in &skill.tools {
            self.tool_registry.register(tool.clone())?;
        }

        self.active_skills.insert(name.to_string());
        Ok(())
    }
}

// ==================== Layer 7: MCP Integration ====================

#[async_trait]
pub trait MCPClient: Send + Sync {
    async fn connect(&self) -> Result<(), MCPError>;
    async fn disconnect(&self) -> Result<(), MCPError>;
    async fn list_tools(&self) -> Result<Vec<ToolDefinition>, MCPError>;
    async fn call_tool(&self, name: &str, params: Value) -> Result<Value, MCPError>;
}

pub struct MCPManager {
    clients: HashMap<String, Arc<dyn MCPClient>>,
    tool_registry: Arc<ToolRegistry>,
}

impl MCPManager {
    pub async fn register(&mut self, config: MCPConfig) -> Result<(), MCPError> {
        let client = self.create_client(&config).await?;
        client.connect().await?;

        // Discover tools
        let tools = client.list_tools().await?;
        for tool in tools {
            let wrapped = MCPToolWrapper::new(
                config.name.clone(),
                client.clone(),
                tool,
            );
            self.tool_registry.register(Arc::new(wrapped))?;
        }

        self.clients.insert(config.name, client);
        Ok(())
    }
}

// MCP Tool Wrapper
pub struct MCPToolWrapper {
    server_name: String,
    tool: ToolDefinition,
    client: Arc<dyn MCPClient>,
}

impl Tool for MCPToolWrapper {
    fn name(&self) -> &str {
        &self.tool.name
    }

    fn namespace(&self) -> Option<&str> {
        Some(&self.server_name)
    }

    async fn execute(&self, params: Value, _ctx: &ToolContext) -> Result<ToolOutput, ToolError> {
        let result = self.client.call_tool(&self.tool.name, params).await?;
        Ok(ToolOutput::new(result))
    }
}

// ==================== Layer 8: Execution Engine ====================

pub struct ExecutionEngine {
    registry: Arc<ToolRegistry>,
    security: Arc<SecurityManager>,
    skills: Arc<SkillManager>,
    mcp: Arc<MCPManager>,
}

impl ExecutionEngine {
    pub async fn execute(&self, call: ToolCall, ctx: &SessionContext) -> Result<ToolResult, ExecutionError> {
        let tool = self.registry.get(&call.name)?;

        // Security check
        let security_result = self.security.check(&call.name, &call.params).await;
        match security_result {
            SecurityResult::Denied(reason) => {
                return Ok(ToolResult::error(reason));
            }
            SecurityResult::PendingApproval(findings) => {
                let approved = ctx.request_approval(&call, &findings).await?;
                if !approved {
                    return Ok(ToolResult::error("User declined"));
                }
            }
            SecurityResult::Allowed(_) => {}
        }

        // Get active skill hooks
        let hooks = self.skills.get_hooks_for_session(&ctx.session_id);

        // Pre-tool hook
        if let Some(ref h) = hooks.before_tool {
            h.execute(ctx).await?;
        }

        // Execute
        let start = Instant::now();
        let output = tool.execute(call.params, &ctx.to_tool_context()).await?;

        // Post-tool hook
        if let Some(ref h) = hooks.after_tool {
            h.execute_with_result(ctx, &output).await?;
        }

        Ok(ToolResult::success(output, start.elapsed()))
    }
}
```

### 7.4 配置示例

```yaml
# tools.yaml
tools:
  registry:
    conflict_strategy: skip  # skip/override/rename/raise

  security:
    enabled: true
    default_mode: prompt     # never/auto/prompt/always
    deny_list:
      - dangerous_command
      - rm_rf
    rules_path: ./security-rules/
    rate_limits:
      shell: 10/minute
      file_write: 100/hour

  skills:
    directories:
      - ~/.agent/skills/builtin
      - ~/.agent/skills/custom
      - ~/.agent/skills/installed
    active:
      - file-reader
      - github-review
      - code-refactor

  mcp:
    clients:
      - name: filesystem
        transport: stdio
        command: npx
        args: ["-y", "@modelcontextprotocol/server-filesystem", "~"]

      - name: github
        transport: http
        url: https://api.github.com/mcp

      - name: postgres
        transport: sse
        url: http://localhost:3000/sse

  hooks:
    enabled:
      - audit_log
      - rate_limit
      - before_tool.validate_params
```

---

## 八、总结与最佳实践

### 8.1 各维度最佳实现

| 维度 | 最佳实现 | 推荐采用 |
|------|---------|---------|
| **工具注册** | IronClaw Trait + Codex 命名空间 | ✅ 推荐 |
| **Schema 定义** | Zod/TypeBox + Docstring 提取 | ✅ 推荐 |
| **安全体系** | IronClaw 分层 + CoPaw ToolGuard | ✅ 推荐 |
| **审批模式** | IronClaw 4 级 + Gemini Plan 模式 | ✅ 推荐 |
| **Skills 动态扩展** | OpenCode Python 代码 + IronClaw 信任模型 | ✅ 推荐 |
| **Hook 系统** | IronClaw 6 阶段 | ✅ 推荐 |
| **MCP 支持** | IronClaw/Nanobot/Codex 完整实现 | ✅ 推荐 |
| **沙箱隔离** | IronClaw WASM + Docker | ✅ 推荐 |
| **审计日志** | IronClaw/CoPaw 完整实现 | ✅ 推荐 |

### 8.2 最终推荐架构

```
┌──────────────────────────────────────────────────────────────┐
│                    推荐工具系统架构                           │
├──────────────────────────────────────────────────────────────┤
│ 1. 工具定义: Trait/Interface + @tool 装饰器                   │
│ 2. 命名空间: namespace:tool_name                              │
│ 3. Schema: Zod/TypeBox/JSON Schema                           │
│ 4. 安全: Deny → Rule Engine → Pre-approval → Approval        │
│ 5. 隔离: Native → WASM → Docker Container                    │
│ 6. Skills: SKILL.md + 代码扩展 + 信任模型                     │
│ 7. Hooks: 6 阶段生命周期                                      │
│ 8. MCP: stdio/HTTP/SSE + 自动发现                            │
│ 9. 审计: 完整日志 + 速率限制                                  │
└──────────────────────────────────────────────────────────────┘
```

### 8.3 各项目特色功能参考

| 项目 | 特色功能 | 可借鉴场景 |
|------|---------|-----------|
| **IronClaw** | 完整 Trait 系统、WASM 沙箱、6 阶段 Hooks | 企业级安全需求 |
| **Codex** | 多代理工具、命名空间、SandboxPolicy | 多代理协作场景 |
| **Gemini-CLI** | Plan 模式、输出掩码、策略引擎 | 批量操作场景 |
| **Kimi-CLI** | D-Mail、Checkpoint、任务管理 | 时间回溯需求 |
| **OpenCode** | 动态 Python 工具、图像支持 | 快速扩展需求 |
| **CoPaw** | ToolGuard 六层防护 | 高安全要求场景 |
| **Nanobot** | MCP 包装器、模糊匹配编辑 | 编辑类工具 |

此方案融合了 15 个项目的最佳实践，兼顾安全性、灵活性和扩展性，适用于生产级 AI Agent 系统。
