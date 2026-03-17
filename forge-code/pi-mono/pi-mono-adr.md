# Pi-Mono 架构决策记录 (ADR)

> 记录项目中的关键架构决策及其理由
> 项目: pi-mono (Pi Monorepo)
> 维护者: Mario Zechner

---

## ADR-001: 采用 Monorepo 结构

### 状态
已接受

### 背景
项目包含 7 个相互关联的包，需要协调版本、共享配置和代码。

### 决策
使用 npm workspaces 构建 Monorepo，所有包统一版本号 (0.58.4)。

### 理由
1. **版本一致性**: 所有包一起发布，避免版本冲突
2. **代码共享**: 便于共享类型定义和工具函数
3. **原子变更**: 跨包修改可以一次性提交
4. **构建优化**: 可以统一构建流程

### 权衡
- **优点**: 简化依赖管理，统一开发体验
- **缺点**: 仓库变大，构建时间增加

### 结果
- 使用 npm workspaces
- 根 package.json 定义统一脚本
- 依赖升级可以批量进行

---

## ADR-002: 分层包设计

### 状态
已接受

### 背景
需要支持从底层 LLM 调用到完整编码 Agent 的多层抽象。

### 决策
将代码分为 4 个层次：
```
pi-ai → pi-agent-core → pi-tui/pi-coding-agent → pi-web-ui/pi-mom
```

### 理由
1. **关注点分离**: 每层解决不同问题
2. **可组合性**: 用户可以选择需要的层次
3. **测试便利**: 每层可以独立测试
4. **演进独立**: 底层变更不影响上层（只要接口稳定）

### 结果
- `pi-ai`: 纯 LLM API，无 Agent 概念
- `pi-agent-core`: Agent 状态机和事件系统
- `pi-coding-agent`: 编码特定工作流
- `pi-web-ui`: UI 组件库

---

## ADR-003: TypeScript + ES Modules

### 状态
已接受

### 背景
Node.js 生态正在向 ES Modules 迁移，需要决定模块系统。

### 决策
- 使用 TypeScript 5.9+
- 采用 ES Modules (`"type": "module"`)
- 目标 Node.js >= 20
- 输出 `.js` + `.d.ts`

### 理由
1. **未来兼容**: ES Modules 是标准
2. **Tree Shaking**: 更好的打包优化
3. **异步加载**: 支持顶层 await
4. **类型安全**: TypeScript 提供编译时检查

### 权衡
- **优点**: 现代标准，工具支持好
- **缺点**: 部分旧库兼容性需要处理

---

## ADR-004: 统一 LLM Provider 抽象

### 状态
已接受

### 背景
需要支持 15+ 个 LLM 提供商，每个有不同的 API 格式。

### 决策
创建统一的 `StreamFunction` 抽象：
```typescript
type StreamFunction<TApi, TOptions> = (
  model: Model<TApi>,
  context: Context,
  options?: TOptions
) => AssistantMessageEventStream;
```

### 理由
1. **一致性**: 所有提供商使用相同接口
2. **可替换性**: 切换提供商只需改模型 ID
3. **可测试性**: 容易 mock
4. **流式统一**: 统一处理流式响应

### 实现细节
- 内部使用 EventStream 抽象响应流
- 自动处理消息格式转换
- 支持 SSE 和 WebSocket

---

## ADR-005: Agent 状态集中管理

### 状态
已接受

### 背景
Agent 有复杂的状态（消息、工具、流状态等），需要确定管理方式。

### 决策
使用集中式状态管理：`Agent` 类持有所有状态，通过方法修改。

```typescript
class Agent {
  private _state: AgentState;
  setSystemPrompt(v: string) { /* ... */ }
  setModel(m: Model) { /* ... */ }
  // ...
}
```

### 理由
1. **可控性**: 所有变更通过类方法，可添加验证
2. **可观测**: 容易实现订阅模式
3. **可预测**: 状态变更路径清晰

### 权衡
- **优点**: 易于调试，状态变更可追溯
- **缺点**: 灵活性不如分散式

---

## ADR-006: 事件驱动架构

### 状态
已接受

### 背景
Agent 需要与 UI 层通信，同时支持扩展系统。

### 决策
使用事件总线模式，定义完整的 Agent 事件流：
```
agent_start → turn_start → message_start/update/end →
tool_execution_start/update/end → turn_end → agent_end
```

### 理由
1. **解耦**: Agent 不直接依赖 UI
2. **扩展**: 扩展可以订阅事件
3. **测试**: 可以监听事件验证行为
4. **调试**: 事件序列可记录和回放

### 实现
```typescript
type AgentEvent =
  | { type: "agent_start" }
  | { type: "message_start"; message: AgentMessage }
  // ...

subscribe(fn: (e: AgentEvent) => void): () => void;
```

---

## ADR-007: 差分 TUI 渲染

### 状态
已接受

### 背景
终端 UI 需要高性能渲染，避免闪烁和卡顿。

### 决策
实现差分渲染：只更新变化的单元格，而非全屏重绘。

### 理由
1. **性能**: 减少终端输出量
2. **体验**: 减少闪烁
3. **带宽**: 远程连接时尤其重要
4. **电池**: 减少 CPU 使用

### 实现细节
- 维护前一帧状态
- 比较新旧状态生成差异
- 使用 ANSI 转义序列精确定位光标

---

## ADR-008: 消息队列 (Steering/FollowUp)

### 状态
已接受

### 背景
需要支持在 Agent 运行时插入消息（用户打断、自动跟进）。

### 决策
实现双队列系统：
- `steeringQueue`: 高优先级，立即中断当前执行
- `followUpQueue`: 低优先级，当前执行完成后处理

### 理由
1. **灵活性**: 支持用户打断和自动任务
2. **有序**: 保证消息处理顺序
3. **可控**: 可以清空或修改队列

### 模式支持
- `all`: 一次性发送队列中所有消息
- `one-at-a-time`: 每轮只发送一条

---

## ADR-009: 扩展系统

### 状态
已接受

### 背景
用户需要自定义 Agent 行为，添加自定义工具和集成。

### 决策
基于 JavaScript/TypeScript 文件的扩展系统：
- 扩展是导出一组钩子的模块
- 支持自定义工具注册
- 支持生命周期钩子

### 理由
1. **灵活**: 几乎无限制的定制能力
2. **熟悉**: 用户使用相同语言
3. **生态**: 可以利用 npm 生态
4. **隔离**: 扩展在独立上下文运行

### API 设计
```typescript
export default {
  name: "my-extension",
  setup(api: ExtensionAPI) {
    api.registerTool(/* ... */);
    api.onEvent("message_end", handler);
  }
};
```

---

## ADR-010: 上下文压缩 (Compaction)

### 状态
已接受

### 背景
长会话会超出 LLM 上下文限制，需要管理历史消息。

### 决策
实现基于分支摘要的上下文压缩：
1. 检测上下文使用超过阈值
2. 生成历史消息摘要
3. 用摘要替换详细历史
4. 保留最近 N 条完整消息

### 理由
1. **自动化**: 用户无需手动管理
2. **保留信息**: 摘要保留关键信息
3. **可配置**: 阈值和策略可调

---

## ADR-011: Web Components 架构

### 状态
已接受

### 背景
`pi-web-ui` 需要提供可复用的 AI 聊天界面。

### 决策
基于 Lit 的 Web Components：
- 框架无关（React/Vue/Angular 均可使用）
- 原生浏览器标准
- 样式封装

### 理由
1. **通用性**: 任何框架都能使用
2. **标准**: Web Components 是 W3C 标准
3. **性能**: 原生支持，无额外开销

---

## ADR-012: OAuth 内置支持

### 状态
已接受

### 背景
多个提供商需要 OAuth 认证（GitHub Copilot, Google 等）。

### 决策
在 `pi-ai` 中内置 OAuth 流程支持：
- 本地 HTTP 服务器接收回调
- PKCE 流程支持
- Token 自动刷新

### 理由
1. **用户体验**: 一键认证
2. **安全**: 本地回调，不暴露 token
3. **标准**: 遵循 OAuth 2.0 规范

---

## ADR-013: TypeBox 运行时验证

### 状态
已接受

### 背景
需要运行时类型验证（工具参数、配置等）。

### 决策
使用 TypeBox 替代 Zod：
- 编译时类型推断
- 运行时验证
- JSON Schema 输出

### 理由
1. **性能**: TypeBox 比 Zod 更快
2. **标准**: 输出标准 JSON Schema
3. **类型**: 更好的 TypeScript 集成

---

## ADR-014: 统一版本策略

### 状态
已接受

### 背景
Monorepo 中多个包需要版本管理。

### 决策
所有包使用统一版本号：
- 一起发布
- 版本号同步
- 破坏性变更统一升级

### 理由
1. **简单**: 用户无需考虑版本匹配
2. **一致**: API 变更统一反映
3. **可预测**: 版本号有明确含义

### 工具
- `npm version` with workspaces
- 自定义脚本同步版本

---

## 决策总结

| ADR | 决策 | 影响范围 | 风险 |
|-----|------|----------|------|
| 001 | Monorepo | 整体架构 | 低 |
| 002 | 分层设计 | 包结构 | 低 |
| 003 | ES Modules | 构建系统 | 中 |
| 004 | Provider 抽象 | pi-ai | 低 |
| 005 | 集中状态 | pi-agent | 低 |
| 006 | 事件驱动 | 整体架构 | 中 |
| 007 | 差分渲染 | pi-tui | 低 |
| 008 | 消息队列 | pi-agent | 中 |
| 009 | 扩展系统 | pi-coding | 中 |
| 010 | 上下文压缩 | pi-coding | 中 |
| 011 | Web Components | pi-web-ui | 低 |
| 012 | OAuth 内置 | pi-ai | 低 |
| 013 | TypeBox | 类型系统 | 低 |
| 014 | 统一版本 | 发布流程 | 低 |

---

*文档生成时间: 2026-03-17*
