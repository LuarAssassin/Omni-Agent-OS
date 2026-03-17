# Qwen Code 架构决策记录

> 记录项目中的关键架构决策及其理由

---

## ADR-001: Monorepo 架构

### 状态
已接受

### 背景
项目包含 CLI、SDK、IDE 扩展、Web UI 等多个相关组件，需要统一管理依赖和构建流程。

### 决策
使用 npm workspaces 实现 Monorepo 架构。

### 理由

1. **依赖共享**: 多个包可以共享依赖，避免版本冲突
2. **代码复用**: core 包的功能可以被 CLI、SDK、IDE 扩展复用
3. **统一构建**: 可以一次性构建所有包
4. **原子提交**: 跨包修改可以在一个提交中完成
5. **版本对齐**: 相关包的版本可以统一管理

### 结构

```
qwen-code/
├── packages/
│   ├── cli/                 # CLI 主程序
│   ├── core/                # 核心库
│   ├── sdk-typescript/      # TypeScript SDK
│   ├── vscode-ide-companion/# VS Code 扩展
│   └── ...
└── package.json             # Workspaces 配置
```

### 权衡

- **优点**: 便于协作、统一版本、代码复用
- **缺点**: 仓库体积增大、CI 时间增加

---

## ADR-002: React 终端 UI (Ink)

### 状态
已接受

### 背景
CLI 需要复杂的交互界面，传统的命令行交互难以满足需求。

### 决策
使用 Ink (React for Terminal) 构建终端 UI。

### 理由

1. **组件化**: 使用 React 组件模型构建 UI
2. **状态管理**: 利用 React 的状态管理能力
3. **生态丰富**: 可以使用 React 生态的工具
4. **实时更新**: 支持流式响应的实时渲染
5. **测试友好**: 组件可以独立测试

### 实现

```typescript
// 使用 Ink 渲染 React 组件
import { render } from 'ink'
import { App } from './ui/App.js'

render(<App />)
```

### 权衡

- **优点**: 开发体验好、组件可复用、响应式更新
- **缺点**: 增加包体积、需要学习 Ink API

---

## ADR-003: 工具系统架构

### 状态
已接受

### 背景
AI 需要执行各种操作（文件操作、Shell、搜索等），需要一个统一的工具系统。

### 决策
实现声明式工具系统，每个工具包含：
- 名称和描述
- JSON Schema 参数定义
- 执行函数

### 理由

1. **类型安全**: JSON Schema 定义参数类型
2. **自文档**: 工具描述可直接用于 AI prompt
3. **可扩展**: 新工具可以通过扩展添加
4. **统一接口**: 所有工具遵循相同接口

### 实现

```typescript
interface Tool {
  name: string
  description: string
  parameters: JSONSchema
  execute(args: any): Promise<ToolResult>
}
```

### 权衡

- **优点**: 一致性强、易于扩展、类型安全
- **缺点**: 需要维护 JSON Schema

---

## ADR-004: MCP 协议支持

### 状态
已接受

### 背景
Model Context Protocol (MCP) 是 Anthropic 推出的标准协议，用于 AI 工具集成。

### 决策
原生支持 MCP 协议，允许用户连接 MCP 服务器。

### 理由

1. **标准化**: 遵循行业标准协议
2. **生态兼容**: 可以使用社区 MCP 服务器
3. **动态发现**: 支持运行时工具发现
4. **隔离性**: MCP 服务器运行在独立进程

### 实现

```typescript
class MCPManager {
  addServer(config: MCPServerConfig): void
  discoverTools(serverName: string): Promise<Tool[]>
  executeTool(serverName: string, toolName: string, args: any): Promise<any>
}
```

### 权衡

- **优点**: 标准化、生态丰富、功能扩展性强
- **缺点**: 增加系统复杂度、需要管理子进程

---

## ADR-005: 配置层级设计

### 状态
已接受

### 背景
用户需要在不同层级配置应用（全局、项目、命令行）。

### 决策
实现四层配置优先级：
1. CLI 参数
2. 环境变量
3. 项目级配置 (.qwen/settings.json)
4. 用户级配置 (~/.qwen/settings.json)

### 理由

1. **灵活性**: 不同场景使用不同配置
2. **可移植性**: 项目配置可以版本控制
3. **个性化**: 用户级配置保存个人偏好
4. **临时覆盖**: CLI 参数可临时覆盖配置

### 实现

```typescript
async function loadConfig(): Promise<Config> {
  const defaultConfig = getDefaultConfig()
  const userConfig = await loadUserConfig()
  const projectConfig = await loadProjectConfig()
  const envConfig = loadEnvConfig()
  const cliConfig = parseCliArgs()

  return merge(defaultConfig, userConfig, projectConfig, envConfig, cliConfig)
}
```

---

## ADR-006: 流式响应处理

### 状态
已接受

### 背景
AI 响应可能很长，需要实时显示给用户。

### 决策
使用流式响应，边接收边渲染。

### 理由

1. **用户体验**: 即时反馈，减少等待感
2. **内存效率**: 不需要缓存完整响应
3. **实时工具调用**: 可以在响应过程中处理工具调用
4. **取消支持**: 用户可以中途取消

### 实现

```typescript
for await (const chunk of llmClient.stream(params)) {
  if (chunk.type === 'text') {
    appendText(chunk.content)
  } else if (chunk.type === 'tool_call') {
    await handleToolCall(chunk.toolCall)
  }
}
```

---

## ADR-007: 扩展系统

### 状态
已接受

### 背景
需要允许第三方扩展功能。

### 决策
实现基于 npm 的扩展系统。

### 理由

1. **生态友好**: 利用 npm 生态
2. **版本管理**: npm 自动处理版本
3. **隔离性**: 扩展运行在独立作用域
4. **安全性**: 可以选择性启用/禁用

### 实现

```typescript
interface Extension {
  id: string
  name: string
  activate(context: ExtensionContext): void
}

class ExtensionManager {
  async install(extensionId: string): Promise<void>
  async enable(extensionId: string): Promise<void>
  async disable(extensionId: string): Promise<void>
}
```

---

## ADR-008: 子代理系统

### 状态
已接受

### 背景
复杂任务需要分解给专业代理处理。

### 决策
实现子代理系统，支持任务委托。

### 理由

1. **任务分解**: 复杂任务可以分解执行
2. **专业化**: 不同子代理专注不同领域
3. **并行化**: 子代理可以并行执行
4. **可观测性**: 子代理执行过程可监控

### 实现

```typescript
interface Subagent {
  name: string
  description: string
  execute(task: string): Promise<string>
}

class SubagentManager {
  async delegate(task: string, subagentName: string): Promise<string>
}
```

---

## ADR-009: 技能系统

### 状态
已接受

### 背景
AI 需要领域知识来更好地完成任务。

### 决策
实现基于 Markdown 的技能系统。

### 理由

1. **可读性**: Markdown 易于编写和维护
2. **版本控制**: 可以 Git 管理
3. **层级组织**: 按目录结构自动分类
4. **搜索友好**: 支持 BM25 搜索

### 结构

```
.qwen/skills/
├── programming/
│   ├── typescript.md
│   ├── react.md
│   └── nodejs.md
├── tools/
│   ├── git.md
│   └── docker.md
└── workflows/
    └── code-review.md
```

---

## ADR-010: TypeScript 严格模式

### 状态
已接受

### 背景
需要保证代码的类型安全。

### 决策
启用 TypeScript 严格模式。

### 配置

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true
  }
}
```

### 理由

1. **类型安全**: 捕获更多运行时错误
2. **代码质量**: 强制明确的类型定义
3. **重构安全**: 重构时有类型检查保护
4. **IDE 支持**: 更好的自动补全和提示

### 权衡

- **优点**: 类型安全、代码质量高
- **缺点**: 开发初期需要更多类型定义工作

---

## ADR-011: 测试策略

### 状态
已接受

### 背景
需要保证代码质量和稳定性。

### 决策
采用单元测试 + 集成测试的双层测试策略。

### 结构

```
packages/
├── cli/
│   └── src/
│       └── **/*.test.ts      # 单元测试 (与源文件同目录)
├── core/
│   └── src/
│       └── **/*.test.ts      # 单元测试
└── integration-tests/         # 集成测试
    ├── file-system.test.ts
    ├── run_shell_command.test.ts
    └── ...
```

### 理由

1. **单元测试**: 快速验证单个模块
2. **集成测试**: 验证模块间协作
3. **同目录**: 方便查找和维护
4. **Vitest**: 现代测试框架，支持 ESM

---

## ADR-012: ESBuild 构建

### 状态
已接受

### 背景
需要将 TypeScript 打包为可执行文件。

### 决策
使用 ESBuild 进行打包。

### 理由

1. **速度快**: ESBuild 是 Go 编写，构建速度极快
2. **Tree Shaking**: 自动删除未使用代码
3. **Bundle**: 输出单文件，便于分发
4. **Source Map**: 支持源码映射

### 配置

```javascript
module.exports = {
  entryPoints: ['packages/cli/index.ts'],
  outfile: 'dist/cli.js',
  bundle: true,
  platform: 'node',
  target: 'node20',
  format: 'esm',
}
```

---

## ADR-013: 多 IDE 集成

### 状态
已接受

### 背景
用户可能使用不同的 IDE。

### 决策
提供 VS Code、JetBrains、Zed 等多个 IDE 的集成。

### 实现

```typescript
interface IDEIntegration {
  name: string
  isAvailable(): Promise<boolean>
  getContext(): Promise<IDEContext>
  applyEdit(edit: Edit): Promise<void>
}
```

### 理由

1. **用户友好**: 支持用户习惯的 IDE
2. **上下文感知**: 可以获取 IDE 中的光标位置、选中内容等
3. **无缝体验**: 编辑可以直接应用到 IDE

---

## ADR-014: 遥测系统

### 状态
已接受

### 背景
需要了解产品使用情况以改进产品。

### 决策
实现可选的遥测系统，用户可以选择关闭。

### 理由

1. **隐私优先**: 默认开启，但可完全关闭
2. **匿名化**: 不包含敏感信息
3. **产品改进**: 帮助了解功能使用情况
4. **问题诊断**: 帮助发现和修复问题

---

## 待定决策

### ADR-PENDING-001: 插件热更新

- **问题**: 是否支持插件热更新，无需重启
- **考虑**: 复杂度 vs 用户体验
- **状态**: 评估中

### ADR-PENDING-002: 云端同步

- **问题**: 是否支持配置和历史云端同步
- **考虑**: 隐私 vs 便利性
- **状态**: 评估中

### ADR-PENDING-003: 协作模式

- **问题**: 是否支持多人协作编辑
- **考虑**: 复杂度 vs 使用场景
- **状态**: 评估中

---

*文档生成完成。如需补充特定决策，请告知。*
