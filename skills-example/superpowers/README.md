# Superpowers 项目文档集

> 全面深入的 Superpowers AI 开发工作流系统分析
> 生成时间: 2026-03-17

---

## 文档列表

| 文档 | 描述 | 大小 |
|------|------|------|
| [`superpowers-analysis.md`](./superpowers-analysis.md) | **主分析报告** - 项目全景分析、架构设计、工作流程 | ~25KB |
| [`superpowers-skills-index.md`](./superpowers-skills-index.md) | **Skills 索引** - 14+ Skills 详细解析 | ~15KB |
| [`superpowers-design-philosophy.md`](./superpowers-design-philosophy.md) | **设计哲学** - 结合 Ousterhout 软件设计哲学分析 | ~12KB |
| [`superpowers-adr.md`](./superpowers-adr.md) | **架构决策** - 关键设计决策记录 | ~8KB |

---

## 快速导航

### 如果你是...

**项目新手**
→ 先看 [`superpowers-analysis.md`](./superpowers-analysis.md) 的「项目概述」和「核心工作流程」章节

**想了解 Skills 系统**
→ 查阅 [`superpowers-skills-index.md`](./superpowers-skills-index.md)

**关注设计质量**
→ 阅读 [`superpowers-design-philosophy.md`](./superpowers-design-philosophy.md)

**想了解架构决策**
→ 查看 [`superpowers-adr.md`](./superpowers-adr.md)

---

## 项目关键信息

```yaml
项目: Superpowers
用途: AI 编码代理的完整软件开发工作流系统
版本: 5.0.4
作者: Jesse Vincent (jesse@fsck.com)
许可: MIT
代码库: https://github.com/obra/superpowers
文档文件: 64+ 个 Markdown 文件
核心组件: 14+ 个 Skills
支持平台: Claude Code, Cursor, Codex, OpenCode, Gemini CLI
```

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户交互层                                   │
│  Claude Code │ Cursor │ Codex CLI │ OpenCode │ Gemini CLI          │
├─────────────────────────────────────────────────────────────────────┤
│                         插件适配层                                   │
│  .claude-plugin/ │ .cursor-plugin/ │ .codex/ │ .opencode/         │
│  │                 │                 │         │                     │
│  └── plugin.json   └── plugin.json  └── INSTALL.md                  │
│                                     └── hooks.json                  │
├─────────────────────────────────────────────────────────────────────┤
│                         Skills 核心层                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 元技能 (Meta)                                                  │  │
│  │ using-superpowers │ writing-skills                            │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 设计阶段 (Design)                                              │  │
│  │ brainstorming → writing-plans                                  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 开发阶段 (Development)                                         │  │
│  │ using-git-worktrees → subagent-driven-development/executing-plans│ │
│  │ test-driven-development → requesting-code-review               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 完成阶段 (Completion)                                          │  │
│  │ finishing-a-development-branch                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 辅助技能 (Auxiliary)                                           │  │
│  │ systematic-debugging │ receiving-code-review                   │  │
│  │ dispatching-parallel-agents                                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                         支撑工具层                                   │
│  hooks/ (session-start, run-hook.cmd)                               │
│  commands/ (brainstorm.md, write-plan.md, execute-plan.md)          │
│  agents/ (code-reviewer.md)                                         │
│  docs/ (design docs, plans, specs)                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 核心工作流程

```
用户请求
    │
    ▼
┌──────────────┐
│ brainstorming│──→ 探索需求 → 提出方案 → 设计文档
└──────┬───────┘
       │
       ▼
┌──────────────┐
│writing-plans │──→ 拆分任务 → 详细计划 → 实现文档
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│using-git-worktrees│──→ 创建隔离工作区 → 验证基线
└──────┬───────────┘
       │
       ▼
┌──────────────────────────┐
│subagent-driven-development│──→ 分任务执行 → 两阶段评审
│   /executing-plans       │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│requesting-code-review    │──→ 代码评审 → 问题修复
│test-driven-development   │──→ TDD 循环 (Red-Green-Refactor)
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│finishing-a-development-  │──→ 测试验证 → 合并/PR/清理
│         branch           │
└──────────────────────────┘
```

---

## Skills 分类

### 按阶段分类

| 阶段 | Skills | 用途 |
|------|--------|------|
| **准备** | `using-superpowers` | 系统引导，技能发现机制 |
| **设计** | `brainstorming`, `writing-plans` | 需求探索、方案设计、计划编写 |
| **环境** | `using-git-worktrees` | 隔离工作区创建 |
| **开发** | `subagent-driven-development`, `executing-plans`, `test-driven-development` | 实现执行 |
| **评审** | `requesting-code-review`, `receiving-code-review` | 代码评审流程 |
| **调试** | `systematic-debugging` | 系统化调试方法论 |
| **完成** | `finishing-a-development-branch` | 工作流收尾 |
| **进阶** | `dispatching-parallel-agents`, `writing-skills` | 并行代理、技能编写 |

### 按性质分类

| 类型 | Skills | 特性 |
|------|--------|------|
| **刚性 (Rigid)** | TDD, debugging | 必须严格遵守，不可变通 |
| **灵活 (Flexible)** | brainstorming, patterns | 可据上下文调整 |

---

## 关键设计原则

### 软件设计哲学应用 (Ousterhout)

1. **深度模块 (Deep Modules)**
   - 每个 Skill = 简单接口 + 丰富功能
   - 用户只需 `@skill`，内部流程全自动

2. **信息隐藏 (Information Hiding)**
   - Skill 隐藏实现细节
   - 用户无需知道子代理如何调度

3. **向下吸收复杂度 (Pull Complexity Downward)**
   - 复杂逻辑封装在 Skill 内部
   - 用户获得简单接口

4. **定义错误不存在 (Define Errors Out of Existence)**
   - 预设流程减少错误可能
   - 系统化调试技能处理异常

---

## 项目统计

| 指标 | 数值 |
|------|------|
| Skills 数量 | 14+ |
| 文档文件数 | 64+ |
| 支持平台 | 5 (Claude, Cursor, Codex, OpenCode, Gemini) |
| 核心工作流步骤 | 7 |
| 设计文档示例 | 6+ |

---

## 相关资源

- **GitHub**: https://github.com/obra/superpowers
- **Marketplace**: https://github.com/obra/superpowers-marketplace
- **博客**: https://blog.fsck.com/2025/10/09/superpowers/
- **Discord**: https://discord.gg/Jd8Vphy9jq

---

*文档生成完成。如需补充特定模块的分析，请告知。*
