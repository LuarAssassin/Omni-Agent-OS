# Kimi CLI 架构决策记录 (ADR)

> 分析时间: 2026-03-17
> 项目路径: `/Users/luarassassin/reference-projects-me/kimi-cli`
> KLIP 文档数: 16 个

---

## 关于 KLIP

**KLIP** (Kimi CLI Improvement Proposal) 是 Kimi CLI 的架构决策记录系统，存放于 `klips/` 目录。每个 KLIP 记录一项重要的架构决策、设计原则或技术选型。

命名来源：
- **Kosong** (马来语"空")：LLM 抽象层
- **YAMAHA** 框架：Yet Another Moonshot AI Handler Architecture
- KLIP 致敬 Python 的 PEP 流程

---

## KLIP 索引

| KLIP | 标题 | 状态 | 类别 |
|------|------|------|------|
| 0 | KLIP 流程与项目起源 | Active | Process |
| 1 | Kimi CLI Monorepo 结构 | Active | Structure |
| 2 | ACPKaos: ACP 客户端集成 | Active | Architecture |
| 3 | Kimi CLI 用户文档结构 | Active | Documentation |
| 6 | 自动刷新模型配置 (/setup) | Active | Feature |
| 7 | Kimi SDK 设计 | Active | API Design |
| 8 | 配置与技能目录布局 | Active | Configuration |
| 9 | Shell UI 闪烁缓解 | Active | UX |
| 10 | Agent Flow (流程图技能) | Active | Feature |
| 11 | Kimi Code 品牌重命名 | Active | Branding |
| 12 | Wire 初始化外部工具 | Active | Protocol |
| 14 | Kimi Code OAuth 登录 | Active | Auth |
| 15 | kagent Sidecar 集成 | Active | Architecture |

---

## 详细决策记录

### KLIP-0: KLIP 流程与项目起源

**决策**: 建立正式的架构决策流程，强调数据结构和关系设计优先于代码。

**背景**:
- Kimi CLI 源于内部 side project "Ensoul"
- 最初是探索性项目，用于理解 Moonshot AI API
- 核心架构师认为："数据结构、数据关系、数据流转的设计，比代码更重要"

**核心原则**:
1. **Data over Code** - 数据结构设计优先
2. **关系定义行为** - 数据间的关系决定系统行为
3. **文档驱动** - 重大变更需先写 KLIP

**设计哲学契合**:
- 信息隐藏：通过数据结构定义清晰的模块边界
- 深模块：数据结构接口简单，内部关系复杂

---

### KLIP-1: Kimi CLI Monorepo 结构

**决策**: 采用 UV workspace 管理多包仓库，整合 Kosong、PyKAOS 等核心依赖。

**仓库结构**:
```
kimi-cli/
├── packages/
│   ├── kosong/          # LLM 抽象层
│   ├── pykaos/          # Python KAOS 实现
│   ├── kaos/            # KAOS 协议定义
│   ├── kimi-code/       # 元包占位
│   └── kimi-sdk/        # SDK 封装
├── src/kimi_cli/        # 主包源码
└── pyproject.toml       # Workspace 配置
```

**标签约定**:
| 包 | 标签格式 | 示例 |
|----|----------|------|
| kimi-cli | `x.xx` | `0.68` |
| kosong | `kosong-x.xx.x` | `kosong-0.20.0` |
| pykaos | `pykaos-x.x.x` | `pykaos-0.2.0` |

**设计理由**:
- 减少版本兼容性问题
- 统一 CI/CD 流程
- 便于跨包重构

---

### KLIP-2: ACPKaos - ACP 客户端集成

**决策**: 创建 LocalKaos 变体 ACPKaos，将文件操作重定向到 ACP 客户端。

**问题背景**:
- Zed、JetBrains 等 IDE 无法直接访问本地文件系统
- 需要统一协议让 IDE 和 CLI 共享文件操作

**解决方案**:
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Kimi CLI  │────►│   ACPKaos   │────►│  ACP Client │
│  (Runtime)  │     │  (Adapter)  │     │ (IDE Plugin)│
└─────────────┘     └─────────────┘     └─────────────┘
                            │
                            ▼ (capability-based fallback)
                     ┌─────────────┐
                     │  LocalKaos  │
                     └─────────────┘
```

**关键设计**:
- Capability-based 能力协商
- 透明 fallback 到本地操作
- 支持批量文件操作优化

---

### KLIP-3: Kimi CLI 用户文档结构

**决策**: 建立分层文档结构，区分用户指南和参考手册。

**文档架构**:
```
docs/
├── guides/              # 指南 (how-to)
│   ├── getting-started/
│   ├── common-use-cases/
│   ├── interaction/
│   ├── sessions/
│   ├── ide-usage/
│   └── integrations/
├── customization/       # 定制配置
│   ├── mcp/
│   ├── skills/
│   ├── agents/
│   ├── print-mode/
│   └── wire-mode/
├── configuration/       # 配置参考
│   ├── files/
│   ├── providers-models/
│   ├── overrides/
│   ├── environment-variables/
│   └── data-locations/
└── reference/           # 参考手册
    ├── commands/
    ├── tools/
    ├── exit-codes/
    └── keyboard-shortcuts/
```

**设计哲学**: 区分"学习"(guides)和"查询"(reference)两种使用模式。

---

### KLIP-6: 自动刷新模型配置 (/setup)

**决策**: 引入 `/setup` 命令和托管命名空间实现模型配置的自动刷新。

**核心概念**:
- **Managed Namespace**: `managed:<platform-id>` 格式的特殊 provider ID
- **Platform Definition**: 包含 `allowed_prefixes` 用于模型过滤
- **自动刷新触发**: `/model` 命令时检查过期配置

**配置示例**:
```toml
[[providers]]
id = "managed:kimi-code"
name = "Kimi Code"
```

**设计理由**:
- 减少用户手动配置成本
- 支持平台级模型管理策略
- 向后兼容非托管配置

---

### KLIP-7: Kimi SDK 设计

**决策**: 设计简洁的 SDK 层，提供扁平化的 API 接口。

**API 设计**:
```python
from kimi_sdk import Kimi, generate, step, Message

# 函数式 API
response = generate("Hello, world!")

# 面向对象 API
kimi = Kimi()
for chunk in kimi.generate_stream("Hello"):
    print(chunk)
```

**设计原则**:
1. **Flat is better than nested** - 单层导入
2. **Phase 1 直接依赖 Kosong** - 严格版本约束
3. **独立版本管理** - `kimi-sdk-*` 标签前缀

---

### KLIP-8: 配置与技能目录布局

**决策**: 统一技能发现机制，支持分层覆盖和目录查找两种模式。

**发现层级** (优先级从高到低):
1. CLI 参数: `--skills-dir`
2. 项目级: `work_dir/.agents/skills/`
3. 用户级 (按优先级):
   - `~/.config/agents/skills/`
   - `~/.kimi/skills/`
   - `~/.claude/skills/`
4. 内置技能: `kimi_cli/skills/`

**两层逻辑**:
1. **分层合并** (Layered Merge): 低优先级 → 高优先级覆盖
2. **目录查找** (Directory Lookup): 按优先级顺序搜索

**设计哲学**: 符合 XDG 规范，兼容 Claude Code 技能目录。

---

### KLIP-9: Shell UI 闪烁缓解

**决策**: 通过 Pager 扩展和固定显示预算解决终端渲染闪烁问题。

**问题分析**:
- 当 Live display 超过视口高度时，Rich 库重绘导致闪烁
- 快速更新的内容加剧问题

**解决方案**:
1. **Pager 扩展**: Ctrl+E 展开完整内容
2. **统一预算**: 固定 4 行内容区域
3. **ShellDisplayBlock**: 专用 Shell 命令显示组件
4. **KeyboardListener**: 支持暂停/恢复输入监听

**实现位置**:
- `ui/shell/visualize.py`
- `ui/shell/keyboard.py`

---

### KLIP-10: Agent Flow (流程图技能)

**决策**: 扩展技能系统支持基于流程图的交互式执行 (Mermaid/D2)。

**技能类型**:
```yaml
# standard - 标准 Markdown 技能
type: standard

# flow - 流程图技能
type: flow
flow_format: mermaid  # 或 d2
```

**FlowRunner 架构**:
```
BEGIN ──► task ──► decision ──► task ──► END
              │         │
              └────┬────┘
              branch selection
```

**节点类型**:
- `BEGIN`: 流程入口
- `END`: 流程结束
- `task`: 执行任务，支持技能调用
- `decision`: 条件分支，LLM 选择路径

**Ralph 模式**:
- 自动迭代循环，直到任务完成
- 通过 `<choice>{value}</choice>` 标签输出选择

---

### KLIP-11: Kimi Code 品牌重命名

**决策**: 对外品牌从 "Kimi CLI" 调整为 "Kimi Code CLI"，内部保持兼容。

**变更范围**:
| 范围 | 变更 |
|------|------|
| 包名 | 保持不变 (`kimi-cli`, `kimi_cli`) |
| 命令 | 保持不变 (`kimi`) |
| 用户文档 | 使用 "Kimi Code CLI" |
| `kimi-code` 包 | 保留作为占位包 |

**设计理由**:
- 与 "Kimi Code" 产品线统一
- 最小化破坏性变更
- 保留生态兼容性

---

### KLIP-12: Wire 初始化外部工具

**决策**: 扩展 Wire 协议，支持初始化握手和外部工具注册。

**协议流程**:
```
Client                              Server (Kimi CLI)
  │ ────── initialize ─────────────► │
  │     { external_tools: [...] }    │
  │                                  │
  │ ◄──────── initialized ────────── │
  │     { slash_commands: [...] }    │
```

**新消息类型**:
- `ToolCallRequest`: 客户端执行外部工具
- `ApprovalResponse`: 统一的请求/响应语义 (替代 ApprovalRequestResolved)

**兼容性**:
- 向后兼容 Wire v1.0
- 能力协商决定可用功能

---

### KLIP-14: Kimi Code OAuth 登录

**决策**: 实现 OAuth 2.0 Device Authorization Grant (RFC 8628) 支持 Kimi Code 平台登录。

**认证流程**:
```
┌─────────┐                                  ┌─────────────┐
│ 用户输入 │ ───── /login ────► │ Device Code │
│ /login  │                                  │ Request     │
└─────────┘                                  └──────┬──────┘
      │                                              │
      │ ◄──────── Device Code,                     │
      │            Verification URI                 │
      │                                              │
      │  用户访问 verification_uri 授权              │
      │                                              │
      │ ◄──────── Token (polling 成功后) ────────── │
```

**Token 存储**:
1. 首选: keyring (系统密钥环)
2. 回退: `~/.kimi/credentials/kimi-code.json`

**Token 刷新策略**:
- 提前 5 分钟刷新
- 失败时使用刷新令牌
- 热更新无需重启

**设备信息头**:
- `X-Msh-Platform`
- `X-Msh-Version`
- `X-Msh-Device-Name`
- `X-Msh-Device-Id`

---

### KLIP-15: kagent Sidecar 集成

**决策**: 将 Rust 实现的 kagent 内核作为 sidecar 进程集成，通过 stdio 通信。

**架构**:
```
┌─────────────────────────────────────────────┐
│              Kimi CLI (Python)              │
│  ┌─────────────┐      ┌─────────────────┐   │
│  │  KimiSoul   │◄────►│ WireBackedSoul  │   │
│  │  (Runtime)  │      │    (Proxy)      │   │
│  └─────────────┘      └────────┬────────┘   │
└────────────────────────────────┼────────────┘
                                 │ stdio
┌────────────────────────────────┼────────────┐
│           kagent (Rust)        │            │
│  ┌─────────────────────────────▼────────┐   │
│  │         Rust Kernel Core             │   │
│  │  (High-performance Agent Engine)     │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**运行时选择**:
- 命令行: `--kernel rust|python`
- 环境变量: `KIMI_KERNEL`
- 配置文件: `kernel` 字段

**打包**:
- 使用 maturin 构建多平台 wheel
- 内嵌预编译二进制
- 失败时回退到 Python 内核

**设计理由**:
- Rust 内核性能更高
- 渐进式迁移策略
- 保持 API 兼容

---

## 架构模式总结

### 1. 分层抽象模式

```
User Interface (Shell/Web/ACP)
    │
    ▼
Wire Protocol (Message Bus)
    │
    ▼
Agent Runtime (KimiSoul)
    │
    ▼
Abstraction Layer (Kosong/Kaos)
    │
    ▼
Infrastructure (LLM API/File System)
```

### 2. 插件扩展模式

| 扩展点 | 机制 | 示例 |
|--------|------|------|
| 工具 | MCP Protocol | 外部工具服务器 |
| 技能 | Markdown + Flow | 自定义工作流 |
| 内核 | Sidecar Process | kagent (Rust) |
| UI | Wire Protocol | Web UI, ACP |

### 3. 降级容错模式

- **ACPKaos**: ACP 不可用时 fallback 到 LocalKaos
- **kagent**: Rust 内核失败时 fallback 到 Python
- **Token 存储**: keyring 失败时 fallback 到文件

---

## 设计哲学反思

| 原则 | 体现 |
|------|------|
| **深度模块** | Wire 协议封装复杂消息路由，暴露简单 send/receive |
| **信息隐藏** | Kosong 隐藏 LLM 提供商差异，Kaos 隐藏存储差异 |
| **通用模块** | Kosong、Kaos 设计为可复用组件 |
| **拉低复杂度** | 配置验证、路径安全在底层统一处理 |
| **定义错误不存在** | `is_empty()` 处理各种边界情况而不抛异常 |

---

## 参考

- [KLIP 目录](../../kimi-cli/klips/)
- [软件设计哲学分析](./kimi-cli-analysis.md#四软件设计哲学分析)
