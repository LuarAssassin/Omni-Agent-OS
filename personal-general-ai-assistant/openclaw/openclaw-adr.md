# OpenClaw 架构决策记录 (ADR)

> 关键设计决策与架构演进

---

## 目录

1. [ADR-001: Gateway 控制平面架构](#adr-001-gateway-控制平面架构)
2. [ADR-002: 插件系统架构](#adr-002-插件系统架构)
3. [ADR-003: 配置系统分层](#adr-003-配置系统分层)
4. [ADR-004: ACP 协议集成](#adr-004-acp-协议集成)
5. [ADR-005: 多通道抽象设计](#adr-005-多通道抽象设计)
6. [ADR-006: 密钥管理降级策略](#adr-006-密钥管理降级策略)
7. [ADR-007: TypeScript ESM 迁移](#adr-007-typescript-esm-迁移)
8. [ADR-008: 测试策略 (Vitest)](#adr-008-测试策略-vitest)
9. [ADR-009: 代码质量工具 (Oxlint)](#adr-009-代码质量工具-oxlint)
10. [ADR-010: 移动端架构 (SwiftUI/Kotlin)](#adr-010-移动端架构-swiftuikotlin)
11. [ADR-011: Pi Agent 运行时集成](#adr-011-pi-agent-运行时集成)
12. [ADR-012: 文档站点 (Mintlify)](#adr-012-文档站点-mintlify)

---

## ADR-001: Gateway 控制平面架构

### 状态
已接受 (Accepted)

### 背景
OpenClaw 需要一个统一的控制平面来管理：
- 多 Agent 会话
- 多通道消息路由
- 工具注册与调用
- 实时事件分发

### 决策
采用 **WebSocket -based Gateway** 作为核心控制平面：

```
┌─────────────────────────────────────┐
│        Gateway (WebSocket)          │
│  ws://127.0.0.1:18789               │
├─────────────────────────────────────┤
│  Session Manager                    │
│  Channel Router                     │
│  Tool Registry                      │
│  Event Bus                          │
└─────────────────────────────────────┘
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| WebSocket Gateway | 实时双向、状态保持、低延迟 | 需要连接管理 | ✅ 选择 |
| HTTP REST API | 简单、无状态 | 实时性差、轮询开销 | ❌ 放弃 |
| gRPC | 高性能、强类型 | 浏览器支持有限 | ❌ 放弃 |

### 影响
- 所有客户端通过 WebSocket 连接 Gateway
- 通道适配器抽象为 Gateway 插件
- 支持本地与远程部署模式

---

## ADR-002: 插件系统架构

### 状态
已接受 (Accepted)

### 背景
需要支持 20+ 消息通道和功能扩展，同时保持核心轻量。

### 决策
采用 **Workspace Plugin 架构**：

```
openclaw/
├── src/                    # 核心 (必需)
├── extensions/             # 扩展插件 (可选)
│   ├── telegram/
│   ├── discord/
│   └── ...
```

插件清单 (`package.json`):
```json
{
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| Workspace 插件 | 版本一致、代码复用、类型安全 | 需构建时存在 | ✅ 选择 |
| npm 独立包 | 按需安装、独立版本 | 版本冲突风险 | ⚠️ 部分使用 |
| 动态加载 | 运行时灵活 | 类型安全难保证 | ❌ 放弃 |

### 影响
- 核心保持轻量 (< 50MB)
- 插件通过 `openclaw/plugin-sdk` 开发
- jiti 运行时转译支持 TypeScript 插件

---

## ADR-003: 配置系统分层

### 状态
已接受 (Accepted)

### 背景
配置系统需要支持：
- 复杂嵌套结构（Agent、通道、认证）
- 环境变量替换
- 遗留配置迁移
- 运行时验证

### 决策
采用 **分层类型 + 统一 IO** 设计：

```
src/config/
├── types.ts           # 类型聚合
├── types.base.ts      # 基础类型
├── types.agents.ts    # Agent 配置
├── types.channels.ts  # 通道配置
├── ...
├── io.ts              # 统一读写
└── validation.ts      # 验证逻辑
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| 分层类型文件 | 编辑局部性、易维护 | 文件数量多 | ✅ 选择 |
| 单一类型文件 | 集中 | 文件过大、冲突多 | ❌ 放弃 |
| 代码生成 | 自动化 | 构建复杂 | ❌ 放弃 |

### 影响
- 配置类型分散在 20+ 文件，每个 < 500 行
- 支持 JSON5 格式（注释、宽松语法）
- 遗留配置自动迁移

---

## ADR-004: ACP 协议集成

### 状态
已接受 (Accepted)

### 背景
需要标准化 Agent 与 Gateway 的通信协议。

### 决策
采用 **@agentclientprotocol/sdk** 标准实现：

```typescript
// ACP Server
import { AgentSideConnection } from "@agentclientprotocol/sdk";

export async function serveAcpGateway(opts: AcpServerOptions): Promise<void> {
  const connection = new AgentSideConnection(...);
  // ...
}
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| ACP 标准协议 | 生态兼容、规范维护 | 依赖外部标准 | ✅ 选择 |
| 私有协议 | 完全控制 | 生态封闭 | ❌ 放弃 |
| MCP 协议 | 生态增长 | 当时不成熟 | ⚠️ 后续考虑 |

### 影响
- Agent 与 Gateway 解耦
- 支持第三方 ACP 兼容 Agent
- 协议升级由 SDK 处理

---

## ADR-005: 多通道抽象设计

### 状态
已接受 (Accepted)

### 背景
需要统一 20+ 不同消息平台的接口差异。

### 决策
采用 **统一 ChannelMeta + 归一化层** 设计：

```typescript
// 统一抽象
interface ChannelMeta {
  id: string;
  label: string;
  docsPath: string;
  // ...
}

// 归一化处理
src/channels/plugins/normalize/
├── telegram.ts
├── discord.ts
└── shared.ts
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| 统一抽象 + 归一化 | 一致接口、易于扩展 | 需维护归一化层 | ✅ 选择 |
| 各自独立实现 | 无转换开销 | 重复逻辑、难维护 | ❌ 放弃 |
| 适配器模式 | 灵活 | 过度抽象 | ⚠️ 部分使用 |

### 影响
- 新通道接入只需实现归一化适配器
- 上层逻辑与具体通道解耦
- 共享命令/提及/白名单逻辑

---

## ADR-006: 密钥管理降级策略

### 状态
已接受 (Accepted)

### 背景
密钥解析失败（如文件权限、网络问题）不应导致服务中断。

### 决策
采用 **Last-Known-Good 降级模式**：

```typescript
const activateRuntimeSecrets = async (...) => {
  try {
    const prepared = await prepareSecretsRuntimeSnapshot({ config });
    activateSecretsRuntimeSnapshot(prepared);
  } catch (err) {
    // 降级：保持现有密钥快照
    if (!secretsDegraded) {
      emitSecretsStateEvent("SECRETS_RELOADER_DEGRADED", ...);
    }
    secretsDegraded = true;
  }
};
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| 降级模式 | 高可用、自恢复 | 可能使用过期密钥 | ✅ 选择 |
| 严格失败 | 数据一致性 | 服务中断 | ❌ 放弃 |
| 异步重试 | 自动恢复 | 复杂状态管理 | ⚠️ 部分使用 |

### 影响
- 密钥服务短暂故障不影响运行
- 状态事件通知外部监控
- 启动时失败则快速退出（fail-fast）

---

## ADR-007: TypeScript ESM 迁移

### 状态
已接受 (Accepted)

### 背景
需要现代模块化标准支持，同时保持开发体验。

### 决策
全面采用 **ES Modules (ESM)**：

```json
// package.json
{
  "type": "module",
  "exports": {
    ".": "./dist/index.js",
    "./plugin-sdk": "./dist/plugin-sdk/index.js"
  }
}
```

使用 jiti 支持运行时 TypeScript 加载。

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| ESM | 标准、tree-shaking、顶层 await | 生态迁移中 | ✅ 选择 |
| CommonJS | 生态兼容 | 过时、无静态分析 | ❌ 放弃 |
| 双模式 | 最大兼容 | 构建复杂 | ❌ 放弃 |

### 影响
- 所有源码使用 `.ts` + ESM 语法
- 依赖需支持 ESM
- jiti 转译插件代码

---

## ADR-008: 测试策略 (Vitest)

### 状态
已接受 (Accepted)

### 背景
需要高性能、现代 TypeScript 测试方案。

### 决策
采用 **Vitest + V8 Coverage**：

```json
{
  "scripts": {
    "test": "vitest",
    "test:coverage": "vitest --coverage"
  }
}
```

覆盖率阈值：70% (lines/branches/functions/statements)

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| Vitest | 快速、原生 TS、Vite 生态 | 相对新 | ✅ 选择 |
| Jest | 成熟、生态丰富 | 配置复杂、慢 | ❌ 放弃 |
| Node Test Runner | 内置、无依赖 | 功能有限 | ⚠️ 不采用 |

### 影响
- 测试文件 `*.test.ts` 与源码同目录
- E2E 测试 `*.e2e.test.ts`
- Live 测试需显式标记

---

## ADR-009: 代码质量工具 (Oxlint)

### 状态
已接受 (Accepted)

### 背景
需要快速、现代的 Lint 方案替代 ESLint。

### 决策
采用 **Oxlint + Oxfmt**：

```bash
pnpm check          # 运行所有检查
pnpm format         # 格式化检查
pnpm lint           # Oxlint 检查
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| Oxlint | 极快、Rust 实现 | 规则数少于 ESLint | ✅ 选择 |
| ESLint | 规则丰富、可配置 | 慢、配置复杂 | ❌ 放弃 |
| Biome | 快速、统一工具 | 当时不成熟 | ⚠️ 未采用 |

### 影响
- 构建速度提升
- 规则配置简化
- 部分 ESLint 插件需替代方案

---

## ADR-010: 移动端架构 (SwiftUI/Kotlin)

### 状态
已接受 (Accepted)

### 背景
需要原生 iOS/Android 应用作为 Gateway 的移动端控制界面。

### 决策
采用 **原生开发 + 共享 Gateway 核心**：

```
apps/
├── ios/           # SwiftUI
│   └── Sources/
├── android/       # Kotlin
│   └── app/src/
└── macos/         # SwiftUI
    └── Sources/
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| 原生开发 | 性能、平台集成 | 双端开发 | ✅ 选择 |
| React Native | 代码复用 | 性能、原生限制 | ❌ 放弃 |
| Flutter | 代码复用 | 生态、包体积 | ❌ 放弃 |

### 影响
- iOS 使用 SwiftUI + Observation 框架
- Android 使用 Kotlin + Compose
- 通过 Gateway WebSocket API 通信

---

## ADR-011: Pi Agent 运行时集成

### 状态
已接受 (Accepted)

### 背景
需要强大的本地 AI Agent 运行能力。

### 决策
集成 **@mariozechner/pi-agent-core**：

```typescript
import { PiAgent } from "@mariozechner/pi-agent-core";

// Gateway 通过 ACP 协议与 Pi Agent 通信
const agent = new PiAgent({
  sessionKey,
  tools,
  // ...
});
```

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| Pi Agent | 功能完整、本地运行、RPC 模式 | 外部依赖 | ✅ 选择 |
| 自研 Agent | 完全控制 | 开发成本高 | ❌ 放弃 |
| LangChain | 生态丰富 | 体积大、复杂 | ❌ 放弃 |

### 影响
- Pi Agent 作为独立进程运行
- 通过 ACP 协议通信
- 支持工具调用、代码执行、浏览器自动化

---

## ADR-012: 文档站点 (Mintlify)

### 状态
已接受 (Accepted)

### 背景
需要专业、易维护的文档站点。

### 决策
采用 **Mintlify** 托管文档：

```
docs/
├── mint.json        # Mintlify 配置
├── *.md             # 文档内容
└── zh-CN/           # i18n 翻译
```

部署地址: https://docs.openclaw.ai

### 理由

| 方案 | 优点 | 缺点 | 结论 |
|------|------|------|------|
| Mintlify | 美观、MDX、搜索、托管 | 商业服务 | ✅ 选择 |
| Docusaurus | 开源、可控 | 需自建、配置多 | ❌ 放弃 |
| GitBook | 易用 | 定制受限 | ❌ 放弃 |

### 影响
- 文档使用 MDX 格式
- 支持中文 i18n
- 自动化部署

---

## 待决策事项

### ADR-Candidate: MCP 协议支持

**背景**: Model Context Protocol (MCP) 生态快速发展

**选项**:
1. 完全迁移到 MCP
2. ACP + MCP 双协议支持
3. 保持 ACP，观望 MCP 发展

**状态**: 评估中

---

*文档完成 - 2026-03-17*
