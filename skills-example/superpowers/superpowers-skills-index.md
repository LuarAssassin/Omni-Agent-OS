# Superpowers Skills 完整索引

> Superpowers AI 开发工作流系统的 14 个核心技能详解
> 生成时间: 2026-03-17

---

## 目录

- [技能概览](#技能概览)
- [元技能 (Meta Skills)](#元技能-meta-skills)
- [设计阶段技能 (Design Phase)](#设计阶段技能-design-phase)
- [环境管理技能 (Environment Management)](#环境管理技能-environment-management)
- [开发执行技能 (Development Execution)](#开发执行技能-development-execution)
- [质量保证技能 (Quality Assurance)](#质量保证技能-quality-assurance)
- [工作流收尾技能 (Workflow Completion)](#工作流收尾技能-workflow-completion)
- [进阶技能 (Advanced Skills)](#进阶技能-advanced-skills)
- [技能关系图谱](#技能关系图谱)
- [使用决策树](#使用决策树)

---

## 技能概览

| # | 技能名称 | 类型 | 触发条件 | 核心原则 |
|---|---------|------|---------|---------|
| 1 | `using-superpowers` | 元技能 | 每次会话开始 | 1% 可能适用就必须调用 |
| 2 | `brainstorming` | 设计 | 构建新功能/修复前 | 先理解再构建 |
| 3 | `writing-plans` | 设计 | 有 spec/需求时 | 零上下文假设 |
| 4 | `using-git-worktrees` | 环境 | 开始功能开发 | 隔离工作区 |
| 5 | `subagent-driven-development` | 开发 | 执行计划任务 | 子代理 + 两阶段评审 |
| 6 | `executing-plans` | 开发 | 执行实现计划 | 严格按计划执行 |
| 7 | `test-driven-development` | 质量 | 任何代码实现 | 红-绿-重构铁律 |
| 8 | `requesting-code-review` | 质量 | 完成任务/功能 | 评审早期、频繁 |
| 9 | `receiving-code-review` | 质量 | 收到评审反馈 | 技术验证而非表演 |
| 10 | `systematic-debugging` | 质量 | 调试问题 | 系统化调试流程 |
| 11 | `finishing-a-development-branch` | 收尾 | 开发完成 | 验证→展示选项→执行 |
| 12 | `verification-before-completion` | 质量 | 声称完成前 | 证据先于声明 |
| 13 | `dispatching-parallel-agents` | 进阶 | 2+ 独立任务 | 并行处理 |
| 14 | `writing-skills` | 进阶 | 创建/编辑技能 | 文档的 TDD |

---

## 元技能 (Meta Skills)

### 1. using-superpowers

**定位**: 所有技能的入口和守门人

**核心职责**:
- 建立技能发现机制
- 强制执行技能调用规则
- 管理指令优先级层次

**关键规则**:
```
IF 技能可能适用（哪怕只有 1% 概率）:
    必须调用 Skill 工具
    不允许有任何借口绕过
```

**指令优先级**:
1. 用户显式指令 (CLAUDE.md, GEMINI.md, 直接请求)
2. Superpowers 技能
3. 默认系统提示

**反思维度表**:

| 危险思维 | 现实 |
|---------|------|
| "这只是个简单问题" | 问题就是任务，检查技能 |
| "我需要先获取上下文" | 技能检查先于澄清问题 |
| "让我先探索代码库" | 技能告诉你如何探索 |
| "这个不需要正式技能" | 如果有技能，就用它 |
| "我记得这个技能" | 技能在演进，读当前版本 |
| "这感觉很有生产力" | 无纪律的行动浪费时间 |

---

## 设计阶段技能 (Design Phase)

### 2. brainstorming

**目的**: 探索需求、提出方案、输出设计文档

**9 步检查清单**:
1. **理解** - 用自己的话重述问题
2. **明确范围** - 定义明确的成功标准
3. **探索约束** - 列出技术/业务/时间限制
4. **研究** - 查阅现有代码/文档/模式
5. **多方案** - 提出 2-3 个解决方案
6. **评估权衡** - 每个方案的优缺点
7. **推荐** - 推荐一个方案并说明理由
8. **获取确认** - 确保人类伙伴同意
9. **创建设计文档** - 输出到 `docs/superpowers/design/`

**输出规范**:
- 文件: `docs/superpowers/design/YYYY-MM-DD-<feature>.md`
- 结构: 问题/方案/权衡/建议/架构决策

**关键原则**: 先探索，再提交；没有设计文档不写代码

---

### 3. writing-plans

**目的**: 将 spec/需求拆分为可执行的详细计划

**核心假设**: 执行者对我们的代码库零了解，品味堪忧

**计划文档头**:
```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development

**Goal:** [一句话描述目标]
**Architecture:** [2-3 句方法描述]
**Tech Stack:** [关键技术/库]
```

**任务粒度**: 每步 2-5 分钟
- "写失败测试" - 一步
- "运行确认失败" - 一步
- "写最小实现让测试通过" - 一步
- "提交" - 一步

**文件结构优先**: 先映射文件结构，再定义任务
- 明确每个文件的责任
- 按责任拆分，不按技术层拆分
- 遵循现有项目模式

**审核循环**:
1. 写完整计划
2. 调度 plan-document-reviewer 子代理
3. 如有问题 → 修复 → 重新审核
4. 循环超过 3 次 → 升级人工

---

## 环境管理技能 (Environment Management)

### 4. using-git-worktrees

**目的**: 创建隔离的 git worktree 工作区

**目录选择优先级**:
```
1. 检查已存在: .worktrees/ (优先) 或 worktrees/
2. 检查 CLAUDE.md 配置
3. 询问用户
```

**安全检查** (项目本地目录):
```bash
git check-ignore -q .worktrees  # 必须确认被忽略
```

如果未被忽略 → 立即添加到 .gitignore 并提交

**创建流程**:
1. 检测项目名称
2. 创建 worktree: `git worktree add <path> -b <branch>`
3. 运行项目设置 (npm install, cargo build 等)
4. 验证基线测试通过
5. 报告位置和状态

**关键原则**: 系统化目录选择 + 安全验证 = 可靠隔离

---

## 开发执行技能 (Development Execution)

### 5. subagent-driven-development

**目的**: 使用子代理执行独立任务，带两阶段代码评审

**执行模式**:
```
┌─────────────────────────────────────────┐
│  Phase 1: Spec 合规性检查                │
│  - 是否按要求实现？                      │
│  - 是否有功能偏差？                      │
│  IF 不合规 → 返回修改                   │
└─────────────────────────────────────────┘
                    ↓ IF 通过
┌─────────────────────────────────────────┐
│  Phase 2: 代码质量检查                   │
│  - 代码质量评估                         │
│  - 识别问题                             │
│  IF 有关键问题 → 修复后再提交           │
└─────────────────────────────────────────┘
                    ↓ IF 通过
            [标记任务完成，继续下一任务]
```

**关键原则**:
- 每个任务一个新鲜子代理
- 零上下文继承 - 精确构建所需信息
- 两阶段评审 - 先合规，后质量
- 任务间评审 - 捕获问题防止累积

**何时使用**:
- ✅ 有明确计划的多步骤任务
- ✅ 子代理可用的平台 (Claude Code, Codex)
- ❌ 无子代理平台 → 使用 executing-plans

---

### 6. executing-plans

**目的**: 在当前会话中执行已编写的实现计划

**流程**:
1. **加载和评审计划**
   - 阅读计划文件
   - 批判性评审
   - 如有疑虑 → 先与人类确认

2. **执行任务**
   - 每项任务标记 in_progress
   - 严格遵循计划步骤
   - 运行指定验证
   - 标记 completed

3. **完成开发**
   - 必须调用 finishing-a-development-branch

**停止条件**:
- 遇到阻碍（缺少依赖、测试失败、指令不清）
- 计划有关键缺口
- 不理解某个指令
- 验证反复失败

**何时停止并求助** → 不要猜测

---

## 质量保证技能 (Quality Assurance)

### 7. test-driven-development

**目的**: 严格遵循红-绿-重构的 TDD 循环

**铁律**:
```
NO CODE WITHOUT A FAILING TEST FIRST
```

**红-绿-重构循环**:
```
RED: 写测试 → 运行 → 确认失败
     ↓
GREEN: 写最小代码让测试通过
     ↓
REFACTOR: 清理代码，保持测试通过
     ↓
[循环]
```

**违规即删除**:
```
写代码在测试前？删除它。重新开始。
没有例外：
- 不要保留作为"参考"
- 不要"调整"它来写测试
- 不要看它
- 删除就是删除
```

**危险信号 - 停止并重新开始**:
- 测试在代码之后
- "我已经手动测试过"
- "测试后能达到同样目的"
- "这是关于精神而非仪式"
- "这次不同因为..."

**核心原则**: TDD 不是测试策略，是设计策略

---

### 8. requesting-code-review

**目的**: 调度代码评审子代理捕获问题

**何时请求**:
- **强制**: 子代理开发每任务后、完成主要功能、合并前
- **可选但有价值**: 卡住时、重构前、修复复杂 bug 后

**请求流程**:
1. 获取 git SHAs
   ```bash
   BASE_SHA=$(git rev-parse HEAD~1)
   HEAD_SHA=$(git rev-parse HEAD)
   ```

2. 调度 code-reviewer 子代理

3. 根据反馈行动:
   - Critical → 立即修复
   - Important → 继续前修复
   - Minor → 稍后记录

**关键原则**: 早期评审、频繁评审

---

### 9. receiving-code-review

**目的**: 以技术严谨性处理评审反馈

**响应模式**:
```
WHEN 收到代码评审反馈:
  1. READ: 完整阅读不反应
  2. UNDERSTAND: 用自己的话重述（或询问）
  3. VERIFY: 对照代码库现实检查
  4. EVALUATE: 对这个代码库技术上正确？
  5. RESPOND: 技术确认或有理由的反驳
  6. IMPLEMENT: 一次一项，每项测试
```

**禁止回应**:
- ❌ "You're absolutely right!"
- ❌ "Great point!"
- ❌ "Let me implement that now" (验证前)

**正确回应**:
- ✅ 重述技术要求
- ✅ 询问澄清问题
- ✅ 如错误则用技术理由反驳
- ✅ 直接开始工作（行动 > 言语）

**何时反驳**:
- 建议破坏现有功能
- 评审者缺少完整上下文
- 违反 YAGNI
- 技术上对该栈不正确
- 存在遗留/兼容性原因

**核心原则**: 技术验证而非表演式同意

---

### 10. systematic-debugging

**目的**: 系统化调试方法论

**四阶段流程**:

| 阶段 | 目标 | 输出 |
|-----|------|------|
| **Phase 1: 收集证据** | 收集具体观察 | 事实列表 |
| **Phase 2: 形成假设** | 基于证据解释 | 假设列表 |
| **Phase 3: 设计实验** | 测试每个假设 | 测试计划 |
| **Phase 4: 验证修复** | 确认修复并防止回归 | 验证结果 |

**证据收集清单**:
- 确切错误消息
- 失败测试及输出
- 相关代码段
- 环境信息
- 最近变更
- 重现步骤

**假设形成**:
- 基于证据，非猜测
- 多个假设并行考虑
- 按可能性排序

**实验设计**:
- 每个假设一个测试
- 能证伪假设
- 最小变更原则

---

### 12. verification-before-completion

**目的**: 声称完成前强制验证

**铁律**:
```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

**门函数**:
```
BEFORE 任何状态声称或满意表达:

1. IDENTIFY: 什么命令能证明这个声称？
2. RUN: 执行完整命令（新鲜、完整）
3. READ: 完整输出，检查退出码，计数失败
4. VERIFY: 输出是否确认声称？
   - 否: 用证据陈述实际状态
   - 是: 用证据陈述声称
5. ONLY THEN: 做出声称
```

**常见失败**:

| 声称 | 需要 | 不足够 |
|-----|------|--------|
| 测试通过 | 测试命令输出: 0 失败 | 之前运行，"应该通过" |
| 构建成功 | 构建命令: exit 0 | Linter 通过，日志看起来好 |
| Bug 修复 | 测试原始症状: 通过 | 代码变更，假设修复 |

**危险信号 - 停止**:
- 使用 "should", "probably", "seems to"
- 验证前表达满意
- 未经验证就提交/推送/PR
- 信任代理成功报告
- 依赖部分验证

**核心原则**: 证据先于声明，总是

---

## 工作流收尾技能 (Workflow Completion)

### 11. finishing-a-development-branch

**目的**: 开发工作完成后的系统化收尾

**流程**:
```
1. 最终验证
   └── 运行测试套件
       ├── 通过 → 继续
       └── 失败 → 修复或询问

2. 展示选项
   └── 向用户呈现选择:
       ├── 提交到当前分支
       ├── 创建 PR
       ├── 合并到主分支
       └── 其他选项

3. 执行选择
   └── 根据用户选择执行
       └── 清理 worktree
```

**必须验证**:
- 所有测试通过
- Linter 清洁
- 构建成功
- 需求已满足

**清理步骤**:
```bash
# 返回主仓库
cd /path/to/main/repo

# 移除 worktree
git worktree remove <path>

# 删除分支 (如已合并)
git branch -d <branch>
```

---

## 进阶技能 (Advanced Skills)

### 13. dispatching-parallel-agents

**目的**: 并行调度多个子代理处理独立任务

**何时使用**:
```
Multiple failures? → Yes → Are they independent? → Yes → Can they work in parallel? → Yes
                                                                     ↓
                                                           Parallel dispatch
```

**使用场景**:
- ✅ 3+ 测试文件因不同根因失败
- ✅ 多个子系统独立损坏
- ✅ 每个问题无需其他上下文即可理解
- ✅ 无共享状态

**不使用场景**:
- ❌ 失败相关（修复一个可能修复其他）
- ❌ 需要理解完整系统状态
- ❌ 代理会互相干扰

**代理提示结构**:
1. **Focused** - 一个清晰问题域
2. **Self-contained** - 所有需要的上下文
3. **Specific about output** - 代理应返回什么

**实际案例**:
- 6 个失败跨 3 个文件
- 3 个代理并行调度
- 所有调查并发完成
- 零冲突

---

### 14. writing-skills

**目的**: 创建高质量的技能文档

**核心原则**: 编写技能是应用于流程文档的 TDD

**TDD 映射**:

| TDD 概念 | 技能创建 |
|---------|---------|
| 测试用例 | 子代理压力场景 |
| 生产代码 | 技能文档 (SKILL.md) |
| 测试失败 (RED) | 无技能时代理违规 |
| 测试通过 (GREEN) | 有技能时代理遵守 |
| 重构 | 关闭漏洞同时保持合规 |

**SKILL.md 结构**:
```markdown
---
name: skill-name-with-hyphens
description: Use when [特定触发条件和症状]
---

# Skill Name

## Overview
1-2 句话的核心原则

## When to Use
[如决策不明显，小型内联流程图]

## Core Pattern
Before/after 代码对比

## Quick Reference
表格或要点供扫描

## Implementation
简单模式内联代码
复杂参考或工具链接到文件

## Common Mistakes
什么会出错 + 修复

## Real-World Impact (optional)
具体结果
```

**Claude 搜索优化 (CSO)**:

1. **丰富的描述字段**:
   - 只描述触发条件，不描述流程
   - 以 "Use when..." 开头
   - 包含具体症状和情境

2. **关键词覆盖**:
   - 错误消息、症状、同义词、工具

3. **描述性命名**:
   - 主动语态，动词开头
   - `creating-skills` 而非 `skill-creation`

4. **令牌效率**:
   - getting-started 工作流: <150 词
   - 频繁加载技能: <200 词
   - 其他技能: <500 词

**铁律**:
```
NO SKILL WITHOUT A FAILING TEST FIRST
```

适用于新技能和现有技能的编辑。

**技能类型**:

| 类型 | 示例 | 测试方法 |
|-----|------|---------|
| 纪律执行 | TDD, verification | 学术问题 + 压力场景 |
| 技术指南 | condition-based-waiting | 应用场景 + 变体 |
| 思维模式 | reducing-complexity | 识别 + 应用 + 反例 |
| 参考资料 | API 文档 | 检索 + 应用 + 缺口 |

---

## 技能关系图谱

```
                                    ┌─────────────────┐
                                    │ using-superpowers│
                                    │   (入口守门人)   │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
                    ▼                        ▼                        ▼
            ┌───────────────┐        ┌───────────────┐        ┌───────────────┐
            │ brainstorming │        │  development  │        │    debug      │
            │  (设计阶段)    │        │   (执行阶段)   │        │  (问题解决)    │
            └───────┬───────┘        └───────┬───────┘        └───────┬───────┘
                    │                        │                        │
        ┌───────────┴───────────┐   ┌────────┴────────┐              │
        │                       │   │                 │              │
        ▼                       ▼   ▼                 ▼              ▼
┌───────────────┐      ┌───────────────┐    ┌───────────────┐ ┌─────────────────┐
│writing-plans  │      │using-git-     │    │subagent-driven│ │systematic-      │
│              │      │   worktrees   │    │development    │ │   debugging     │
└───────────────┘      └───────┬───────┘    └───────┬───────┘ └─────────────────┘
                               │                    │
                               │        ┌───────────┴───────────┐
                               │        │                       │
                               │        ▼                       ▼
                               │ ┌───────────────┐      ┌───────────────┐
                               │ │executing-plans│      │requesting-    │
                               │ │(无子代理时)   │      │code-review    │
                               │ └───────────────┘      └───────┬───────┘
                               │                                │
                               │        ┌───────────────────────┘
                               │        │
                               ▼        ▼
                         ┌─────────────────────────┐
                         │ test-driven-development │
                         │   (质量铁律)             │
                         └───────────┬─────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
         ┌─────────────────┐ ┌──────────────┐ ┌─────────────────┐
         │verification-    │ │receiving-    │ │finishing-a-dev- │
         │before-completion│ │code-review   │ │   branch        │
         └─────────────────┘ └──────────────┘ └─────────────────┘
                                                    │
                                                    ▼
                                           ┌───────────────┐
                                           │ writing-skills│
                                           │(技能创建 TDD) │
                                           └───────────────┘
                                                    │
                                                    ▼
                                           ┌───────────────┐
                                           │dispatching-   │
                                           │parallel-agents│
                                           │  (进阶优化)    │
                                           └───────────────┘
```

---

## 使用决策树

### 会话开始
```
任何新会话
    │
    ▼
┌──────────────────────┐
│ using-superpowers    │
│ - 加载技能发现机制    │
│ - 检查是否有适用技能  │
└──────────┬───────────┘
           │
           ▼
    1% 可能适用？
    /          \
   是            否
   |             |
   ▼             ▼
调用 Skill    正常响应
```

### 新功能开发
```
"我想构建 X"
    │
    ▼
┌──────────────────────┐
│ brainstorming        │
│ - 探索需求            │
│ - 输出设计文档        │
└──────────┬───────────┘
           │
           ▼
    需要实现计划？
    /          \
   是            否
   |             |
   ▼             ▼
┌──────────────┐   [简单任务直接实现]
│writing-plans │
│ - 拆分任务    │
│ - 详细计划    │
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ using-git-worktrees  │
│ - 创建隔离工作区      │
│ - 验证基线            │
└──────────┬───────────┘
           │
           ▼
    有子代理支持？
    /          \
   是            否
   |             |
   ▼             ▼
┌─────────────────────┐ ┌─────────────────┐
│subagent-driven-dev  │ │executing-plans  │
│- 子代理执行任务      │ │- 当前会话执行   │
│- 两阶段评审          │ │- 按计划步骤     │
└──────────┬──────────┘ └────────┬────────┘
           │                      │
           └──────────┬───────────┘
                      │
                      ▼
            ┌─────────────────┐
            │ finishing-a-dev │
            │    branch       │
            │ - 最终验证       │
            │ - 清理 worktree  │
            └─────────────────┘
```

### Bug 修复
```
"修复这个 bug"
    │
    ▼
┌──────────────────────┐
│ systematic-debugging │
│ - 收集证据            │
│ - 形成假设            │
│ - 设计实验            │
│ - 验证修复            │
└──────────┬───────────┘
           │
           ▼
    需要写代码修复？
    /          \
   是            否
   |             |
   ▼             ▼
┌──────────────────────┐    [配置/环境问题]
│ test-driven-develop  │
│ - 写失败测试          │
│ - 修复代码            │
│ - 验证回归测试        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│verification-before-  │
│   completion         │
│ - 运行验证命令        │
│ - 确认输出            │
│ - 才声称完成          │
└──────────────────────┘
```

### 代码评审
```
"请评审这段代码"
或
完成任务后
    │
    ▼
┌──────────────────────┐
│ requesting-code-     │
│      review          │
│ - 调度评审子代理      │
│ - 获取反馈            │
└──────────┬───────────┘
           │
           ▼
    收到反馈
           │
           ▼
┌──────────────────────┐
│ receiving-code-      │
│      review          │
│ - 技术验证            │
│ - 实施或反驳          │
└──────────────────────┘
```

---

## 技能分类矩阵

### 按严格性分类

| 严格性 | 技能 | 特征 |
|-------|------|------|
| **刚性** | TDD, verification-before-completion, systematic-debugging | 必须严格遵守，不可变通 |
| **灵活** | brainstorming, writing-plans | 可根据上下文调整 |
| **混合** | subagent-driven-development | 流程刚性，内容灵活 |

### 按阶段分类

| 阶段 | 技能 | 目的 |
|-----|------|------|
| 准备 | using-superpowers | 系统引导 |
| 设计 | brainstorming, writing-plans | 需求探索、方案设计 |
| 环境 | using-git-worktrees | 隔离工作区 |
| 开发 | subagent-driven-development, executing-plans | 实现执行 |
| 质量 | TDD, requesting-code-review, receiving-code-review, systematic-debugging, verification-before-completion | 质量保证 |
| 收尾 | finishing-a-development-branch | 工作流收尾 |
| 进阶 | dispatching-parallel-agents, writing-skills | 效率优化、技能创建 |

### 按依赖关系分类

| 层级 | 技能 | 依赖 |
|-----|------|------|
| 基础 | using-superpowers | 无 |
| 设计 | brainstorming, writing-plans | using-superpowers |
| 环境 | using-git-worktrees | using-superpowers |
| 执行 | subagent-driven-development, executing-plans | using-git-worktrees, writing-plans |
| 质量 | TDD, requesting-code-review, systematic-debugging | subagent-driven-development |
| 收尾 | finishing-a-development-branch | 所有开发和质量技能 |
| 进阶 | writing-skills | TDD, 所有其他技能 |

---

## 关键要点总结

1. **所有技能通过 using-superpowers 入口** - 1% 概率适用就必须调用

2. **设计先于实现** - brainstorming → writing-plans → 开发

3. **隔离先于修改** - using-git-worktrees 创建干净工作区

4. **两阶段评审** - subagent-driven-development 要求先 spec 合规，后代码质量

5. **红-绿-重构铁律** - TDD 不是可选的，是强制的

6. **证据先于声明** - verification-before-completion 要求验证后才声称完成

7. **技能即 TDD** - writing-skills 要求先测试（压力场景），后写技能

8. **并行化独立任务** - dispatching-parallel-agents 处理无依赖的多个问题

---

*技能系统的设计理念: 将软件工程最佳实践封装为可重用的、自执行的流程，确保 AI 编码代理始终遵循系统化、高质量的开发方法。*
