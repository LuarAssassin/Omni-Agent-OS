# OpenCode 架构决策记录 (ADR)

> 分析时间: 2026-03-17
> 分析依据: A Philosophy of Software Design (John Ousterhout)

---

## ADR-001: 使用 Vercel AI SDK 作为 LLM 统一抽象层

### 状态
**Accepted**

### 背景
OpenCode 需要支持 20+ 不同的 LLM 提供商（Claude、OpenAI、Google、Groq 等），每个提供商都有独特的 API 格式、认证方式和响应结构。直接在代码中处理这些差异会导致严重的复杂性扩散。

### 决策
使用 Vercel AI SDK (`ai` 包) 作为统一的抽象层，通过 `@ai-sdk/*` 提供商适配器实现多模型支持。

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **模块深度** | ★★★★★ 极深的模块——简单接口 (`generateText`, `streamText`) 隐藏了20+提供商的复杂性 |
| **信息隐藏** | ★★★★★ 调用者完全无需了解底层提供商差异 |
| **通用性** | ★★★★☆ 通用接口支持多种用例，无需为每个提供商单独编码 |

### 权衡分析

```
✅ 优势:
   - 统一的调用接口，切换模型只需修改 provider 参数
   - 内置流式响应支持 (streamText)
   - 标准化的工具调用接口
   - 完整的 TypeScript 类型支持
   - 活跃的社区维护和持续更新

⚠️ 成本:
   - 增加了一层抽象，需要理解其内部机制
   - 提供商特定的高级功能可能无法直接访问
```

### 代码示例

```typescript
// 使用 AI SDK 的统一接口
import { generateText } from "ai"
import { anthropic } from "@ai-sdk/anthropic"

// 无论使用哪个提供商，调用方式完全一致
const result = await generateText({
  model: anthropic("claude-3-7-sonnet-20250219"),
  messages: [...]
})

// 切换到 OpenAI 只需修改 provider
import { openai } from "@ai-sdk/openai"
const result = await generateText({
  model: openai("gpt-4"),
  messages: [...]
})
```

### 设计原则体现
> **"Pull complexity downward"** - AI SDK 将多提供商的复杂性吸收到库内部，让业务代码保持简洁。

---

## ADR-002: 使用 Effect 处理异步流程和错误

### 状态
**Accepted** (逐步迁移中)

### 背景
CLI 工具涉及大量复杂的异步流程（文件操作、网络请求、数据库访问、LSP 通信），传统的 try/catch 模式导致错误处理分散且难以追踪。需要一种更可靠的方式来管理异步复杂性和错误传播。

### 决策
新代码使用 Effect 库，遗留代码逐步迁移。Effect 提供类型安全的效果系统，将错误处理纳入类型系统。

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **模块深度** | ★★★★★ 深模块——复杂的异步控制隐藏在简洁的 `Effect.gen` 接口后 |
| **错误消除** | ★★★★☆ 通过类型系统强制处理错误，减少运行时异常 |
| **认知负荷** | ★★★☆☆ 学习曲线较陡峭，需要理解函数式编程概念 |

### 权衡分析

```
✅ 优势:
   - 类型安全的错误处理 (Effect<Success, Error>)
   - 可组合的异步流程 (Effect.gen, pipe)
   - 更好的并发控制和资源管理
   - 编译时保证错误被处理

⚠️ 成本:
   - 团队需要时间适应函数式编程思维
   - 与现有异常处理代码混用增加复杂度
   - 调试栈追踪不如传统 async/await 直观
```

### 代码对比

```typescript
// 传统方式 - 错误可能在运行时未处理
try {
  const data = await fetchData()
  const result = await process(data)
  return result
} catch (e) {
  // 错误处理分散，可能遗漏
}

// Effect 方式 - 错误在类型系统中追踪
const program = Effect.gen(function* () {
  const data = yield* fetchData  // 类型强制处理错误
  const result = yield* process(data)
  return result
})

// 运行时才处理错误
const result = await Effect.runPromise(program)
```

### 设计原则体现
> **"Define errors out of existence"** - Effect 将错误处理从异常转换为类型，让不可能的状态在编译时被消除。

---

## ADR-003: 使用 Drizzle ORM 而非 Prisma

### 状态
**Accepted**

### 背景
CLI 工具需要轻量级、与 Bun 运行时良好兼容的 ORM 解决方案。Prisma 的体积和架构不适合 CLI 场景。

### 决策
使用 Drizzle ORM 配合 Bun 内置的 SQLite 实现。

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **模块深度** | ★★★★★ 简洁 API 隐藏 SQL 构造复杂性 |
| **信息隐藏** | ★★★★☆ 类型安全的同时保留 SQL 表达力 |
| **依赖最小化** | ★★★★★ 轻量级，适合 CLI 场景 |

### 权衡分析

```
✅ 优势:
   - 更轻量，适合 CLI 工具场景
   - 与 Bun 兼容性更好 (使用 Bun:sqlite)
   - 运行时 schema 定义 (非代码生成)
   - 查询语法接近原始 SQL
   - 类型安全且支持复杂查询

⚠️ 成本:
   - 生态相对较新，部分功能不如 Prisma 完善
   - 没有 Prisma Studio 这样的可视化工具
   - 迁移功能不如 Prisma Migrate 成熟
```

### 代码示例

```typescript
// Schema 定义
export const sessions = sqliteTable("sessions", {
  id: text().primaryKey(),
  createdAt: integer({ mode: "timestamp" }).notNull(),
  updatedAt: integer({ mode: "timestamp" }).notNull(),
})

// 类型安全查询 - 编译时类型检查
const result = await db
  .select()
  .from(sessions)
  .where(eq(sessions.id, id))
```

### 设计原则体现
> **信息隐藏原则** - Drizzle 隐藏了 SQL 构造的复杂性，同时通过类型系统保证查询的正确性。

---

## ADR-004: 使用 SolidJS 而非 React

### 状态
**Accepted**

### 背景
TUI（终端 UI）和 Web UI 需要高性能的响应式框架。传统 React 的虚拟 DOM 在 TUI 场景下是多余的，且 bundle 体积较大。

### 决策
使用 SolidJS 作为响应式 UI 框架，配合 `@opentui/core` 实现终端 UI。

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **模块深度** | ★★★★☆ SolidJS 编译时转换隐藏了响应式实现的复杂性 |
| **性能** | ★★★★★ 无虚拟 DOM，直接更新，性能优异 |
| **体积** | ★★★★★ 更小的 bundle size，适合 CLI 工具 |

### 权衡分析

```
✅ 优势:
   - 更小的 bundle size
   - 无虚拟 DOM，直接更新真实 DOM
   - 响应式模型非常适合 TUI 场景
   - 与 @opentui 深度集成
   - 性能接近原生 JavaScript

⚠️ 成本:
   - 生态不如 React 丰富
   - 团队需要学习新的响应式模型
   - 部分 React 模式需要调整 (如 useEffect 等)
```

### 设计原则体现
> **"Different layers, different abstractions"** - SolidJS 提供了与 React 不同的响应式抽象，更适合 TUI 场景的需求。

---

## ADR-005: 使用 Zod 进行运行时验证

### 状态
**Accepted**

### 背景
AI 助手的工具调用涉及大量外部输入（LLM 生成的参数、用户输入、配置文件），需要可靠的运行时类型验证。

### 决策
使用 Zod 作为统一的运行时验证库，替代多个分散的验证方案。

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **模块深度** | ★★★★★ 声明式 schema 隐藏验证逻辑的复杂性 |
| **类型集成** | ★★★★★ 与 TypeScript 类型系统无缝集成 |
| **错误信息** | ★★★★☆ 友好的错误消息，易于调试 |

### 权衡分析

```
✅ 优势:
   - 优秀的 TypeScript 集成
   - 类型推导和运行时验证统一
   - JSON Schema 导出支持
   - 链式 API 易于使用
   - 广泛的生态支持

⚠️ 成本:
   - 运行时性能开销 (通常可接受)
   - 包体积增加 (~20KB)
```

### 代码示例

```typescript
// 工具参数定义
const parameters = z.object({
  content: z.string().describe("文件内容"),
  filePath: z.string().describe("绝对路径"),
})

// 类型推导
type WriteParams = z.infer<typeof parameters>

// 运行时验证
const result = parameters.parse(userInput)
```

### 设计原则体现
> **深度模块** - Zod 提供了声明式的 schema 定义，隐藏了验证逻辑、错误格式化、类型推导的复杂性。

---

## ADR-006: 使用 Bun 而非 Node.js

### 状态
**Accepted**

### 背景
CLI 工具对启动速度和运行时性能敏感。Node.js 的启动时间和 TypeScript 编译流程增加了开发复杂度。

### 决策
使用 Bun 1.3.10+ 作为运行时和包管理器。

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **开发体验** | ★★★★★ 内置 TypeScript 支持，无需编译 |
| **性能** | ★★★★★ 更快的启动速度和执行速度 |
| **兼容性** | ★★★★☆ 与 npm 生态基本兼容 |

### 权衡分析

```
✅ 优势:
   - 内置 TypeScript 支持，无需编译步骤
   - 更快的启动速度 (对 CLI 至关重要)
   - 内置 SQLite 支持 (Bun:sqlite)
   - 兼容 npm 生态 (bun install)
   - 单一工具链 (runtime + package manager + bundler)

⚠️ 成本:
   - 相对较新，部分工具链支持不完善
   - 与 Node.js 的某些差异需要适配
   - 某些原生模块可能需要重新编译
```

### 设计原则体现
> **消除复杂性** - Bun 消除了 TypeScript 编译、复杂的构建配置、多个工具链的复杂性。

---

## ADR-007: 使用原生 MCP 支持而非自定义协议

### 状态
**Accepted**

### 背景
AI 助手需要与外部工具集成（浏览器、数据库、API 等）。传统方式是每个工具单独适配，维护成本高。

### 决策
原生支持 Model Context Protocol (MCP)，作为外部工具集成的标准协议。

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **通用性** | ★★★★★ 统一协议支持任意 MCP 兼容工具 |
| **生态** | ★★★★☆ 快速增长的 MCP 工具生态 |
| **标准化** | ★★★★★ 避免 N×M 适配问题 |

### 权衡分析

```
✅ 优势:
   - 标准化协议，任意 MCP 服务器即插即用
   - 支持多种传输方式 (stdio, SSE, HTTP)
   - OAuth 认证流程完整支持
   - 动态工具发现，无需硬编码
   - 由 Anthropic 主导，社区活跃

⚠️ 成本:
   - 需要理解 MCP 协议规范
   - 协议仍在演进中
```

### 设计原则体现
> **通用模块更深** - MCP 提供了一个通用接口，比为每个工具单独写适配器要深得多。

---

## ADR-008: 配置系统的 6 层合并策略

### 状态
**Accepted**

### 背景
AI 助手需要处理多种配置来源：用户全局配置、项目配置、环境变量、组织默认值等。配置冲突和优先级管理是常见痛点。

### 决策
实现 6 层配置合并策略，每层可以覆盖或扩展上层配置。

### 配置层级 (低优先级 → 高优先级)

```
1. Remote .well-known/opencode     # 组织默认值
2. Global config (~/.config/opencode/)  # 用户全局
3. Custom config (OPENCODE_CONFIG)      # 环境变量指定
4. Project config (opencode.json)       # 项目级
5. .opencode directories                # 项目子目录
6. Inline config (OPENCODE_CONFIG_CONTENT)  # 内联
7. [Managed config]                     # 企业部署 (最高)
```

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **模块深度** | ★★★★★ 调用者只需 `Config.get()`，合并逻辑完全隐藏 |
| **灵活性** | ★★★★★ 支持复杂的配置覆盖场景 |
| **信息隐藏** | ★★★★★ 用户无需了解合并细节 |

### 设计原则体现
> **优秀的信息隐藏** - 复杂的合并逻辑被封装在 Config 模块内部，对外提供简单的 `get()` / `set()` 接口。

---

## ADR-009: Agent 系统的声明式配置

### 状态
**Accepted**

### 背景
AI 助手需要不同类型的代理：代码编辑、代码探索、计划制定等。每个 Agent 有不同的权限、模型和提示词需求。

### 决策
采用声明式 Agent 配置，内置 7 种 Agent 类型，支持用户自定义。

### Agent 类型

| Agent | 模式 | 用途 | 权限策略 |
|-------|------|------|----------|
| `build` | primary | 默认构建代理 | 允许编辑、提问、计划 |
| `plan` | primary | 计划模式 | 禁止编辑，只允许规划 |
| `general` | subagent | 通用研究 | 禁用 todo 工具 |
| `explore` | subagent | 代码库探索 | 只允许搜索、读取 |
| `compaction` | primary | 会话压缩 | 无权限（内部使用）|
| `title` | primary | 生成标题 | 无权限（内部使用）|
| `summary` | primary | 生成摘要 | 无权限（内部使用）|

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **配置驱动** | ★★★★★ 行为通过配置定义，无需修改代码 |
| **权限分离** | ★★★★★ 不同 Agent 有不同的权限边界 |
| **可扩展性** | ★★★★☆ 用户可以自定义 Agent 配置 |

### 设计原则体现
> **关注点分离** - 每个 Agent 专注于特定任务，权限控制明确，避免功能混杂。

---

## ADR-010: 文件系统操作的 LSP 集成

### 状态
**Accepted**

### 背景
AI 助手在修改代码后，需要了解修改是否引入了语法错误。传统的文件操作工具无法提供这种反馈。

### 决策
所有文件写入操作（write, edit）自动触发 LSP 诊断检查，返回错误信息供 LLM 修复。

### 流程

```
User Request → Write/Edit Tool → 文件修改 → LSP.touchFile() →
LSP.diagnostics() → 收集错误 → 返回 LLM → 自动修复
```

### 软件设计哲学评估

| 维度 | 评价 |
|------|------|
| **自动化** | ★★★★★ 诊断检查自动触发，无需用户干预 |
| **反馈闭环** | ★★★★★ 错误信息直接反馈给 LLM |
| **深度** | ★★★★☆ 复杂诊断逻辑隐藏在工具内部 |

### 设计原则体现
> **"Pull complexity downward"** - LSP 诊断的复杂性被隐藏在文件操作工具内部，对用户透明。

---

## 总结

| ADR | 决策 | 状态 | 设计原则 |
|-----|------|------|----------|
| 001 | Vercel AI SDK | Accepted | 深度模块、信息隐藏 |
| 002 | Effect | Accepted | 错误消除、向下拉取复杂性 |
| 003 | Drizzle ORM | Accepted | 信息隐藏、轻量依赖 |
| 004 | SolidJS | Accepted | 不同层不同抽象 |
| 005 | Zod | Accepted | 深度模块 |
| 006 | Bun | Accepted | 消除复杂性 |
| 007 | MCP 协议 | Accepted | 通用模块更深 |
| 008 | 6层配置 | Accepted | 信息隐藏 |
| 009 | Agent 系统 | Accepted | 关注点分离 |
| 010 | LSP 集成 | Accepted | 向下拉取复杂性 |

---

*文档生成完成。如需深入分析特定决策，请告知。*
