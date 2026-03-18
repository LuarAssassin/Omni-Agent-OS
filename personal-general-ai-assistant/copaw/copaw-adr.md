# CoPaw 架构决策记录 (Architecture Decision Records)

> 记录项目关键设计决策及其背景、权衡和结果。
> 格式参考: [Architecture Decision Records](https://adr.github.io/)

---

## ADR-001: Workspace-per-Agent 多代理架构

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要支持多个 AI 代理同时运行，每个代理拥有独立的配置、记忆、工具和通信通道。早期设计考虑使用单进程共享状态，但面临以下问题：

1. 代理间状态隔离困难，容易产生冲突
2. 单个代理崩溃会影响整个系统
3. 无法独立扩展特定代理的资源
4. 通道配置复杂，难以管理

### 决策

采用 **Workspace-per-Agent** 架构：每个代理拥有完全独立的 Workspace 实例，包含完整的运行时组件：

```
Workspace
├── AgentRunner       # 代理请求处理
├── ChannelManager    # 通信通道管理
├── MemoryManager     # 对话记忆管理
├── MCPClientManager  # MCP 工具客户端
├── CronManager       # 定时任务调度
└── ChatManager       # 聊天会话管理
```

关键设计点：
- 目录隔离：每个 Workspace 拥有独立的工作目录
- 组件懒加载：按需初始化，避免启动时资源浪费
- 并发安全：使用 asyncio.Lock 保护共享状态
- 生命周期独立：支持运行时动态创建/销毁代理

### 权衡

| 优点 | 缺点 |
|------|------|
| 强隔离性，代理间互不影响 | 内存占用增加（每个 Workspace 约 50-100MB） |
| 配置独立，易于调试 | 进程间通信需通过 Manager 中转 |
| 故障隔离，单点失败不影响全局 | 启动时间略长（需初始化多个组件） |
| 支持异构配置（不同模型、通道） | 代码复杂度增加（需管理多实例） |

### 实现

- `src/copaw/app/workspace.py` - Workspace 类实现
- `src/copaw/app/multi_agent_manager.py` - 全局代理管理器
- `src/copaw/app/dynamic_multi_agent_runner.py` - 动态运行器

### 结果

实现了真正的多代理隔离，支持同时运行 10+ 个配置迥异的代理。每个代理可独立配置模型、通道、技能和 MCP 工具，且崩溃不会影响其他代理。

---

## ADR-002: MCP (Model Context Protocol) 原生支持

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要提供丰富的工具生态供 AI 代理使用。传统方式是内置所有工具，但这导致：

1. 核心代码臃肿，依赖爆炸
2. 用户无法使用社区开发的工具
3. 工具更新需升级整个系统
4. 缺乏标准化接口

MCP 作为 Anthropic 推出的开放协议，正成为 AI 工具的事实标准。

### 决策

**原生支持 MCP 协议**，将工具发现与执行完全委托给 MCP 生态系统：

1. 实现完整 MCP 客户端 (`MCPClientManager`)
2. 支持 Stdio 和 SSE 两种传输方式
3. 工具动态发现，无需重启代理
4. 与 AgentScope 的工具系统集成

```python
# MCP 工具发现流程
MCPClientManager
├── discover_tools()     # 从 MCP Server 获取工具列表
├── register_tools()     # 注册到 Agent 的工具集
└── execute_tool()       # 调用 MCP Server 执行
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 工具生态丰富（100+ MCP Servers） | 需维护 MCP 客户端连接 |
| 社区驱动，持续更新 | 工具调用增加网络延迟 |
| 核心代码保持精简 | MCP Server 故障影响工具可用性 |
| 标准化接口，互操作性强 | 需理解 MCP 协议概念 |

### 实现

- `src/copaw/app/mcp/mcp_manager.py` - MCPClientManager 实现
- `src/copaw/app/mcp/client.py` - MCP 客户端封装
- `src/copaw/app/mcp/config.py` - MCP 配置模型

### 结果

CoPaw 可接入丰富的 MCP 工具生态（文件系统、数据库、API 等），用户通过简单配置即可扩展代理能力，无需修改代码。

---

## ADR-003: Provider 抽象与模型路由

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要支持多种 LLM 提供商（OpenAI、Anthropic、Gemini、本地模型等）。早期设计直接硬编码各提供商 API，导致：

1. 新增提供商需修改多处代码
2. 无法灵活切换模型
3. 难以实现负载均衡和故障转移
4. 配置管理混乱

### 决策

**统一 Provider 抽象层**：

1. 定义 `Provider` 基类，统一接口
2. 使用单例 `ProviderManager` 管理所有提供商
3. 模型字符串标识（如 `gpt-4`, `claude-3-sonnet`）
4. 运行时动态切换模型

```python
class Provider(ABC):
    @abstractmethod
    async def generate(self, messages: List[Message]) -> str: ...

class ProviderManager:
    def get_provider(self, model: str) -> Provider
    def register_provider(self, name: str, provider: Provider)
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 统一接口，易于扩展新提供商 | 抽象层可能隐藏提供商特有功能 |
| 运行时切换模型，灵活性高 | 需维护映射表和配置 |
| 支持模型负载均衡 | 单例模式增加全局状态 |
| 配置集中管理 | 初始化时加载所有提供商 |

### 实现

- `src/copaw/providers/base.py` - Provider 抽象接口
- `src/copaw/providers/provider_manager.py` - 单例管理器
- `src/copaw/providers/*.py` - 各提供商实现（OpenAI, Anthropic, Gemini, Ollama 等）

### 结果

支持 13+ 主流 LLM 提供商，用户通过配置即可切换模型。ProviderManager 统一管理模型实例，支持并发访问和故障恢复。

---

## ADR-004: 多通道统一适配器模式

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要支持多种消息平台（钉钉、飞书、Discord、Telegram 等）。各平台 API 差异巨大：

- 协议不同：HTTP、WebSocket、Webhook
- 认证方式不同：Token、OAuth、签名
- 消息格式不同：XML、JSON、Protobuf
- 推送模式不同：主动轮询、被动推送

### 决策

**统一 Channel 抽象，适配器模式实现**：

1. 定义 `BaseChannel` 抽象基类
2. 统一消息模型（`Msg` 类）
3. 各平台实现具体 Channel 子类
4. 消息标准化转换层

```python
class BaseChannel(ABC):
    @abstractmethod
    async def receive(self) -> Msg: ...

    @abstractmethod
    async def send(self, msg: Msg) -> None: ...

# 具体实现
class DingTalkChannel(BaseChannel): ...
class DiscordChannel(BaseChannel): ...
class FeishuChannel(BaseChannel): ...
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 统一接口，新增通道只需实现基类 | 难以利用平台特有功能 |
| 消息模型标准化，代理层无感知 | 适配层增加转换开销 |
| 支持多通道同时监听 | 各 SDK 依赖增加包体积 |
| 易于测试（可 Mock Channel） | 抽象可能过于简化某些平台 |

### 实现

- `src/copaw/app/channels/base.py` - BaseChannel 抽象类
- `src/copaw/app/channels/*/channel.py` - 各平台适配器
- `src/copaw/agents/schema.py` - 统一消息模型

### 结果

支持 15+ 消息平台，用户可通过配置启用任意组合。新增平台只需实现 BaseChannel 接口，无需修改核心逻辑。

---

## ADR-005: Markdown 技能系统

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要让非开发者也能定义和扩展代理能力。传统方式（编写 Python 代码）门槛过高。调研发现：

1. Markdown 是开发者最熟悉的文档格式
2. 层级化结构适合组织知识
3. 易于版本控制和共享
4. AI 可自然理解 Markdown 格式

### 决策

**基于 Markdown 的技能系统**：

1. 技能以 Markdown 文件形式存储
2. 支持层级化目录结构（文件夹 = 命名空间）
3. 前置 YAML frontmatter 定义元数据
4. 热加载机制，无需重启

```markdown
---
name: 数据分析
version: 1.0.0
tags: [data, analysis]
---

# 数据分析技能

当用户请求数据分析时，请遵循以下步骤：

1. 确认数据来源和格式
2. 检查数据完整性
3. 选择合适的分析方法
...
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 门槛低，非开发者可编写 | 表达能力有限（无逻辑控制） |
| 易于版本控制和协作 | 复杂技能需大量文本 |
| 天然文档化 | 解析开销 |
| 热加载，实时生效 | 错误检查有限 |

### 实现

- `src/copaw/agents/skills/base.py` - 技能基类
- `src/copaw/agents/memory/agent_md_manager.py` - Markdown 管理器
- 目录：`~/.copaw/active_skills/`, `~/.copaw/customized_skills/`

### 结果

技能系统降低了扩展门槛，用户可像写文档一样定义 AI 行为。支持从 Skills Hub 下载社区技能，也可创建私有技能。

---

## ADR-006: ToolGuard 权限控制模型

**状态**: 已接受 (Accepted)

### 背景

AI 代理拥有工具调用能力，但某些工具具有破坏性：

- 文件删除 (`rm -rf /`)
- 系统命令执行
- 敏感数据访问
- 网络请求（可能泄露数据）

需要细粒度的权限控制，而非简单的"允许/拒绝"。

### 决策

**ToolGuard 三层权限模型**：

1. **白名单模式**：默认拒绝，显式允许
2. **风险分级**：Low / Medium / High / Critical
3. **审批机制**：高风险工具需用户确认

```python
class ToolGuardMixin:
    def check_permission(self, tool_name: str) -> PermissionResult:
        # 1. 检查白名单
        # 2. 评估风险等级
        # 3. 如需审批，等待用户确认
        # 4. 返回执行结果
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 细粒度控制，安全性高 | 增加用户交互（审批提示） |
| 风险分级，减少误报 | 配置复杂 |
| 可审计（记录每次授权） | 可能影响自动化流程 |
| 支持超时和取消 | 需维护工具风险数据库 |

### 实现

- `src/copaw/security/tool_guard/` - ToolGuard 模块
- `src/copaw/agents/react_agent.py` - CoPawAgent 继承 ToolGuardMixin

### 结果

有效防止 AI 误操作导致的数据丢失或系统损坏。用户可以精确控制每个工具的访问权限，高风险操作需人工确认。

---

## ADR-007: ReAct 代理循环设计

**状态**: 已接受 (Accepted)

### 背景

CoPaw 代理需要既能推理又能执行工具。早期简单设计（单次 LLM 调用）无法满足复杂任务需求。ReAct 模式（Reasoning + Acting）已被证明有效：

1. 交替进行推理和行动
2. 观察行动结果并调整策略
3. 支持多步骤任务分解

### 决策

**基于 AgentScope ReActAgent 实现 CoPawAgent**：

1. 最大 50 轮推理-行动循环
2. 集成 ToolGuard 安全检查
3. 支持 MCP 工具、内置工具、Skills
4. 记忆压缩（长对话自动摘要）

```
循环：
1. LLM 生成思考 (Thought)
2. 如需工具 → 生成工具调用 (Action)
3. 执行工具，获取观察 (Observation)
4. 将观察加入上下文
5. 循环直到生成最终答案
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 支持复杂多步任务 | 可能陷入循环（需设上限） |
| 可解释性强（思考过程可见） | Token 消耗较高 |
| 错误可恢复（观察反馈） | 延迟较大（多轮 API 调用） |
| 与 ToolGuard 集成安全 | 需精细的提示工程 |

### 实现

- `src/copaw/agents/react_agent.py` - CoPawAgent 主类
- 继承自 `agentscope.agents.ReActAgent`
- 混入 `ToolGuardMixin`

### 结果

代理可处理复杂任务（如"分析这份报告并发送摘要到钉钉"），自动分解为多步执行，每步都有安全检查。

---

## ADR-008: 配置热重载机制

**状态**: 已接受 (Accepted)

### 背景

用户需要频繁调整配置（更换模型、添加通道、修改技能），但重启代理会丢失对话上下文。传统方式（重启服务）体验差。

### 决策

**文件监听 + 热重载**：

1. 使用 `watchdog` 监听配置文件变更
2. 变更时触发组件重新加载
3. 保持对话记忆不变
4. 部分配置支持运行时修改

```python
class ConfigWatcher:
    def on_config_changed(self, path: str):
        # 1. 验证新配置
        # 2. 停止相关组件
        # 3. 应用新配置
        # 4. 重启组件（保持记忆）
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 无需重启，即时生效 | 状态一致性复杂 |
| 保持对话上下文 | 某些配置仍需重启（如端口变更） |
| 开发体验好 | 需处理并发修改 |
| 可回滚到之前配置 | 增加内存占用（监听线程） |

### 实现

- `src/copaw/app/workspace.py` - ConfigWatcher 集成
- `src/copaw/config/loader.py` - 配置加载与验证

### 结果

用户修改 `config.json` 后，代理自动重新加载配置，无需手动重启。对话记忆保持不变，实现无缝升级。

---

## ADR-009: FastAPI + React 控制台架构

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要一个现代化的管理界面，用于：

1. 可视化配置代理、通道、技能
2. 查看对话历史
3. 管理定时任务
4. 监控 MCP 工具

早期 CLI 工具对非技术用户不友好。

### 决策

**前后端分离架构**：

1. **后端**: FastAPI 提供 REST API
2. **前端**: React 18 + TypeScript + Ant Design
3. 状态管理: Zustand
4. API 通信: Axios
5. 构建: Vite

```
Frontend (React + TS)
    ↓ HTTP/WebSocket
Backend (FastAPI)
    ↓
Core (CoPaw Agent)
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 现代化 UI，用户体验好 | 增加开发和维护成本 |
| API 可复用（CLI 也能调用） | 前端依赖 Node.js 生态 |
| 热重载开发体验 | 增加包体积（~100MB node_modules） |
| 可扩展性强 | 需处理前后端版本兼容 |

### 实现

- `console/` - React 前端项目
- `src/copaw/app/routers/` - FastAPI 路由
- `src/copaw/cli/app_cmd.py` - 服务启动命令

### 结果

功能完善的 Web 控制台，支持代理管理、通道配置、技能管理、MCP 工具浏览、定时任务设置等，大幅降低使用门槛。

---

## ADR-010: 组件懒加载策略

**状态**: 已接受 (Accepted)

### 背景

Workspace 包含多个重量级组件（Memory、MCP、Cron），全部初始化会导致：

1. 启动时间过长（>10秒）
2. 内存占用过高（即使某些功能未使用）
3. 不必要的依赖加载（如未使用 MCP 却初始化客户端）

### 决策

**按需懒加载**：

1. 组件首次访问时才初始化
2. 使用 `@property` 包装器
3. 异步初始化支持
4. 缓存已初始化实例

```python
class Workspace:
    @property
    def memory_manager(self) -> MemoryManager:
        if self._memory_manager is None:
            self._memory_manager = MemoryManager(...)
        return self._memory_manager
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 启动速度快 | 首次访问有延迟 |
| 内存占用低 | 初始化逻辑分散 |
| 只加载需要的功能 | 需处理并发初始化 |
| 故障隔离（单组件失败不影响其他） | 调试时状态不透明 |

### 实现

- `src/copaw/app/workspace.py` - 所有组件使用懒加载
- 并发初始化使用 `asyncio.gather()`

### 结果

Workspace 启动时间从 ~10s 降至 ~2s，内存占用根据实际使用功能动态分配。未使用的组件零开销。

---

## ADR-011: 环境变量配置策略

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要平衡配置灵活性：

1. 配置文件（JSON）适合复杂结构化配置
2. 环境变量适合敏感信息（API keys）和部署参数
3. 需要类型安全（bool、int、float 转换）
4. 需要默认值和边界检查

### 决策

**EnvVarLoader 工具类**：

1. 统一封装环境变量读取
2. 支持类型转换（str、int、float、bool）
3. 边界检查（min/max）
4. "Define errors out of existence" 理念

```python
class EnvVarLoader:
    @staticmethod
    def get_float(env_var: str, default: float = 0.0,
                  min_value: float | None = None,
                  max_value: float | None = None) -> float:
        try:
            value = float(os.environ.get(env_var, str(default)))
            # 边界检查...
            return value
        except (TypeError, ValueError):
            return default  # 定义错误行为而非抛出
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 类型安全，避免运行时错误 | 增加代码量 |
| 统一默认值处理 | 环境变量名需全局管理 |
| 边界检查防止无效值 | 文档需同步维护 |
| 错误处理一致 | 新类型需扩展工具类 |

### 实现

- `src/copaw/constant.py` - EnvVarLoader 实现
- 所有配置项使用常量定义，支持环境变量覆盖

### 结果

统一、类型安全的环境变量处理，支持 Docker 部署和本地开发的不同配置需求。无效输入自动回退到默认值，不中断启动。

---

## ADR-012: 定时任务调度器选择

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要支持定时任务（Heartbeat、定时提醒、数据同步）。调研选项：

1. **APScheduler**: 纯 Python，功能丰富
2. **Celery**: 分布式任务队列，依赖 Redis
3. **系统 Cron**: 简单但缺乏动态管理
4. **自定义实现**: 工作量大，可靠性差

### 决策

**使用 APScheduler**：

1. 纯 Python，无外部依赖（如 Redis）
2. 支持 Cron 表达式
3. 支持动态添加/删除任务
4. 支持持久化（JSON 文件）

```python
class CronManager:
    def __init__(self):
        self.scheduler = AsyncIOScheduler()

    def add_job(self, func, trigger: str, **kwargs):
        # 支持 cron、interval、date 触发器
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 无额外依赖 | 非分布式（单机） |
| 功能丰富 | 精度受限于 Python 性能 |
| 易于集成 | 任务过多时性能下降 |
| 支持异步任务 | 持久化格式固定 |

### 实现

- `src/copaw/app/crons/cron_manager.py` - APScheduler 封装
- `src/copaw/app/crons/jobs.py` - 任务定义
- `src/copaw/app/crons/storage.py` - JSON 持久化

### 结果

支持灵活的定时任务调度，包括 Heartbeat（定期主动发送消息）、数据同步、定时提醒等。配置持久化，重启后自动恢复。

---

## ADR-013: CLI 设计原则

**状态**: 已接受 (Accepted)

### 背景

CoPaw 需要通过 CLI 管理所有功能。设计考量：

1. 功能繁多（代理、通道、技能、Cron、模型等）
2. 需要与 Web 控制台功能对等
3. 用户体验（帮助文档、错误提示）
4. 与 Python API 层复用

### 决策

**Click 框架 + 命令分组**：

1. 使用 Click 的 Group 机制组织命令
2. 命令分组：`copaw channels`, `copaw skills`, `copaw cron`
3. 自动帮助生成
4. 性能分析（`--timing` 显示模块导入时间）

```
copaw
├── app       # 启动 FastAPI 服务
├── channels  # 通道管理
├── skills    # 技能管理
├── cron      # 定时任务
├── providers # 模型管理
├── init      # 初始化配置
└── ...
```

### 权衡

| 优点 | 缺点 |
|------|------|
| 命令组织清晰 | 子命令嵌套可能过深 |
| 自动生成帮助 | 需维护大量命令文件 |
| 与 Python 集成好 | 性能受导入影响 |
| 支持参数验证 | 测试复杂 |

### 实现

- `src/copaw/cli/main.py` - CLI 入口
- `src/copaw/cli/*_cmd.py` - 各命令组实现
- 使用 `click.group()` 组织层级

### 结果

15+ 子命令覆盖全部功能，统一的命令风格，完整的帮助文档。`copaw init --defaults` 可一键初始化，降低上手门槛。

---

## 总结

| ADR | 决策 | 核心原则 |
|-----|------|----------|
| 001 | Workspace-per-Agent | 隔离性 > 资源效率 |
| 002 | MCP 原生支持 | 标准化 > 自建生态 |
| 003 | Provider 抽象 | 统一接口 > 特化优化 |
| 004 | 通道适配器模式 | 抽象 > 直接集成 |
| 005 | Markdown 技能系统 | 低门槛 > 表达能力 |
| 006 | ToolGuard 权限 | 安全 > 便利 |
| 007 | ReAct 循环 | 可解释性 > 速度 |
| 008 | 配置热重载 | 体验 > 简单性 |
| 009 | FastAPI + React | 用户体验 > 维护成本 |
| 010 | 组件懒加载 | 启动速度 > 运行时延迟 |
| 011 | EnvVarLoader | 健壮性 > 简洁 |
| 012 | APScheduler | 功能 > 分布式 |
| 013 | Click CLI | 组织性 > 扁平化 |

---

*ADR 记录持续更新，新增重大决策将追加到此文档。*
