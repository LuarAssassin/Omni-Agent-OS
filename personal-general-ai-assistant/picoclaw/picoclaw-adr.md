# PicoClaw 架构决策记录

> 记录项目中的关键架构决策及其理由

---

## ADR-001: 使用 Go 语言重构

### 状态
已接受

### 背景
原项目 nanobot 基于 TypeScript/Node.js，内存占用较高（~100MB+），无法在低端硬件上运行。

### 决策
使用 Go 语言完全重写，目标是 <10MB 内存占用。

### 理由
1. **性能**: Go 的内存占用远低于 Node.js
2. **部署**: 单二进制文件，无依赖
3. **并发**: goroutine 模型适合 I/O 密集型任务（多通道消息处理）
4. **生态**: 丰富的网络/云原生库

### 结果
- 内存占用: ~10MB（目标达成）
- 启动时间: <1s
- 二进制大小: ~20-50MB（取决于功能）

---

## ADR-002: 双架构模式 (Node.js + Rust)

### 状态
已接受（仅适用于 agent-browser 项目，非 PicoClaw）

### 背景
（注：此决策来自 CLAUDE.md，可能不适用于 PicoClaw，但记录于此供参考）

---

## ADR-003: JSONL 会话存储

### 状态
已接受

### 背景
需要持久化会话历史，但 SQLite 可能过于重量级。

### 决策
使用 JSONL (JSON Lines) 格式存储会话，每行一个消息。

### 理由
1. **可读性**: 纯文本，便于调试
2. **可追加**: O(1) 追加写入
3. **兼容性**: 易于导入/导出
4. **轻量**: 无数据库依赖

### 实现
```go
// pkg/memory/jsonl.go
type JSONLStore struct {
    file *os.File
    mu   sync.Mutex
}
```

### 回退策略
如果 JSONL 初始化失败，自动回退到内存存储（session.SessionManager）。

---

## ADR-004: MCP 工具动态发现

### 状态
已接受

### 背景
MCP 服务器可能提供大量工具，全部加载会超出上下文窗口。

### 决策
实现动态发现机制：
1. BM25 搜索匹配相关工具
2. 正则模式匹配
3. TTL 机制控制工具生命周期

### 理由
1. **上下文效率**: 只加载相关工具
2. **响应速度**: 减少 Token 消耗
3. **可扩展性**: 支持大量工具而不会超载

### 实现
```go
// 工具注册表版本控制
type ToolRegistry struct {
    version atomic.Uint64
}

// TTL 机制
func (r *ToolRegistry) TickTTL() {
    // 递减所有非核心工具的 TTL
    // TTL <= 0 时移出可用工具集
}
```

---

## ADR-005: 模型路由 (Model Routing)

### 状态
已接受

### 背景
复杂查询需要强大的模型（如 Claude Opus），简单查询可以用轻量模型（如 Haiku）节省成本。

### 决策
实现智能路由系统：
1. 为每个消息计算复杂度评分
2. 根据阈值选择轻量或重量级模型
3. 支持配置化的阈值和模型选择

### 理由
1. **成本优化**: 简单查询使用便宜模型
2. **响应速度**: 轻量模型响应更快
3. **灵活性**: 用户可自定义规则

### 实现
```go
type Router struct {
    LightModel string
    Threshold  float64
}

func (r *Router) Score(msg Message) float64 {
    // 基于消息长度、复杂度、历史模式评分
}
```

---

## ADR-006: 纯 Go SQLite

### 状态
已接受

### 背景
需要 SQLite 但 CGO 增加构建复杂性。

### 决策
使用 `modernc.org/sqlite` - 纯 Go 实现的 SQLite。

### 理由
1. **无 CGO**: 简化跨平台构建
2. **单二进制**: 无需外部依赖
3. **兼容性**: 与标准 `database/sql` 接口兼容

### 权衡
- 性能略低于 CGO 版本（但对本项目影响可忽略）

---

## ADR-007: 消息总线架构

### 状态
已接受

### 背景
多通道消息需要统一处理和路由。

### 决策
实现发布-订阅模式的消息总线。

### 理由
1. **解耦**: 通道与处理器解耦
2. **扩展性**: 易于添加新通道
3. **测试性**: 便于模拟消息流

### 实现
```go
type MessageBus struct {
    subscribers map[string][]chan Message
    mu          sync.RWMutex
}
```

---

## ADR-008: 技能系统基于 Markdown

### 状态
已接受

### 背景
需要一种用户友好的方式来定义 AI 技能。

### 决策
使用 Markdown 文件定义技能，支持层级加载。

### 理由
1. **可读性**: Markdown 易于编写
2. **版本控制**: 便于 Git 管理
3. **层级组织**: `skills/编程/Go/并发.md` 自动成为分类
4. **BM25 搜索**: 基于内容相关性搜索

### 结构
```
skills/
├── 编程/
│   ├── Go/
│   │   ├── 并发.md
│   │   └── 性能优化.md
│   └── Python/
│       └── 最佳实践.md
└── 工具/
    └── Git.md
```

---

## ADR-009: 降级策略 (Fallback)

### 状态
已接受

### 背景
LLM API 可能失败或限流。

### 决策
实现多级降级策略：主模型 → Fallback 1 → Fallback 2 → ...

### 理由
1. **可靠性**: 单点故障不会导致服务中断
2. **灵活性**: 可配置不同模型的降级链
3. **透明性**: 用户可看到降级日志

### 实现
```go
type FallbackCandidate struct {
    Provider LLMProvider
    Model    string
}

func ExecuteWithFallback(candidates []FallbackCandidate, fn func(LLMProvider) error) error {
    for _, c := range candidates {
        if err := fn(c.Provider); err != nil {
            log.Printf("Fallback: %s failed, trying next", c.Model)
            continue
        }
        return nil
    }
    return errors.New("all candidates failed")
}
```

---

## ADR-010: 工作区隔离

### 状态
已接受

### 背景
多代理需要隔离的工作环境。

### 决策
每个代理拥有独立的工作目录（workspace）。

### 理由
1. **安全性**: 代理间文件隔离
2. **并发性**: 避免文件冲突
3. **可管理性**: 便于清理和迁移

### 结构
```
~/.picoclaw/
├── workspace/           # 默认工作区
│   ├── sessions/        # 会话存储
│   ├── skills/          # 技能文件
│   └── files/           # 工作文件
├── workspace-agent1/    # agent1 工作区
└── workspace-agent2/    # agent2 工作区
```

---

## ADR-011: 工具权限控制

### 状态
已接受

### 背景
AI 工具可能存在安全风险（如删除文件）。

### 决策
实现细粒度的工具权限控制：
1. 工作区限制（RestrictToWorkspace）
2. 读写路径白名单
3. 执行命令黑名单

### 理由
1. **安全性**: 限制 AI 的访问范围
2. **可配置性**: 用户自定义安全策略
3. **透明性**: 清晰的权限边界

### 实现
```go
type ToolRegistry struct {
    workspace        string
    restrictRead     bool
    restrictWrite    bool
    allowReadPaths   []*regexp.Regexp
    allowWritePaths  []*regexp.Regexp
}
```

---

## ADR-012: TUI 启动器设计

### 状态
已接受

### 背景
用户需要友好的方式来管理多个通道和配置。

### 决策
开发独立的 TUI 启动器 `picoclaw-launcher-tui`。

### 理由
1. **易用性**: 图形化配置比 CLI 更友好
2. **功能丰富**: 通道管理、日志查看、状态监控
3. **独立性**: 不依赖 Web 浏览器

### 技术选型
- `rivo/tview`: TUI 组件库
- `gdamore/tcell`: 终端控制

---

## ADR-013: 配置版本管理

### 状态
已接受

### 背景
配置格式可能随版本演进。

### 决策
实现配置版本号和自动迁移机制。

### 理由
1. **向后兼容**: 旧配置可自动升级
2. **可靠性**: 迁移失败可回退
3. **可追溯**: 版本历史清晰

### 实现
```go
const CurrentConfigVersion = "1.0"

type Config struct {
    Version string `json:"version"`
    // ...
}

func MigrateConfig(old *Config) (*Config, error) {
    switch old.Version {
    case "0.9":
        // 执行 0.9 -> 1.0 迁移
    // ...
    }
}
```

---

## ADR-014: 通道占位消息

### 状态
已接受

### 背景
AI 响应可能需要较长时间，用户需要即时反馈。

### 决策
实现占位消息机制：先发送"思考中..."，完成后更新为实际内容。

### 理由
1. **用户体验**: 即时反馈减少等待焦虑
2. **通道兼容**: 支持消息编辑的平台可更新占位符
3. **可配置**: 可启用/禁用

### 实现
```go
type Placeholder struct {
    ID        string
    ChannelID string
    MessageID string
    mu        sync.Mutex
}

func (p *Placeholder) Update(content string) error {
    // 调用平台 API 更新消息
}
```

---

## 待定决策

### ADR-PENDING-001: WebRTC 语音支持
- **问题**: 是否添加实时语音通话支持
- **考虑**: 资源消耗 vs 功能完整性
- **状态**: 评估中

### ADR-PENDING-002: 分布式部署
- **问题**: 是否支持多实例分布式部署
- **考虑**: 复杂性 vs 可扩展性
- **状态**: 评估中
