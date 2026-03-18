# Superpowers 项目深度分析报告

> 分析时间: 2026-03-17
> 项目路径: `/Users/luarassassin/reference-projects-me/superpowers`
> 版本: 5.0.4

---

## 一、项目概述

### 1.1 项目定位

**Superpowers** 是一个为 AI 编码代理设计的完整软件开发工作流系统。它由 Jesse Vincent 创建，基于一系列可组合的 "Skills"（技能）构建，确保 AI 代理在编码时遵循系统化、高质量的开发流程。

不同于传统的代码生成工具，Superpowers 的核心理念是：**AI 不应直接跳进去写代码，而应首先理解用户真正想要构建什么**。

### 1.2 核心使命

| 使命 | 描述 |
|------|------|
| **系统化开发** | 将软件开发从即兴行为转变为系统化流程 |
| **质量保证** | 通过 TDD、代码评审、调试方法论确保质量 |
| **知识封装** | 将最佳实践封装为可重用的 Skills |
| **多平台支持** | 支持主流 AI 编码平台（Claude Code, Cursor, Codex, OpenCode, Gemini） |

### 1.3 设计哲学

- **Test-Driven Development** - 始终先写测试
- **Systematic over ad-hoc** - 流程胜过即兴
- **Complexity reduction** - 简化是首要目标
- **Evidence over claims** - 验证胜过声明

---

## 二、架构设计

### 2.1 分层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户交互层                                   │
│  ┌──────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │ Claude Code  │  │ Cursor          │  │ Codex CLI               │ │
│  │ (Skill tool) │  │ (Plugin)        │  │ (Hooks)                 │ │
│  └──────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  ┌──────────────┐  ┌─────────────────┐                              │
│  │ OpenCode     │  │ Gemini CLI      │                              │
│  │ (Plugin)     │  │ (Extension)     │                              │
│  └──────────────┘  └─────────────────┘                              │
├─────────────────────────────────────────────────────────────────────┤
│                         平台适配层                                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │.claude-plugin│ │.cursor-plugin│ │  .codex/    │ │ .opencode/  │   │
│  │             │ │             │ │             │ │             │   │
│  │marketplace. │ │  plugin.json│ │ INSTALL.md  │ │  INSTALL.md │   │
│  │json         │ │  hooks-     │ │  hooks.json │ │  plugins/   │   │
│  │plugin.json  │ │  cursor.json│ │             │ │  superpowers│   │
│  └─────────────┘ │             │ └─────────────┘ │  .js        │   │
│                  └─────────────┘                 └─────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                         Skills 核心层                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 元技能: using-superpowers (引导层)                             │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 设计层: brainstorming → writing-plans                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 环境层: using-git-worktrees                                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 开发层: subagent-driven-development / executing-plans         │  │
│  │         test-driven-development                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 质量层: requesting-code-review / receiving-code-review        │  │
│  │         systematic-debugging                                  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 完成层: finishing-a-development-branch                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 进阶层: dispatching-parallel-agents / writing-skills          │  │
│  └───────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                         支撑工具层                                   │
│  hooks/ (session-start, run-hook.cmd)                               │
│  commands/ (brainstorm.md, write-plan.md, execute-plan.md)          │
│  agents/ (code-reviewer.md)                                         │
│  docs/superpowers/ (design docs, plans, specs)                      │
│  skills/*/scripts/ (server.js, helper.js, etc.)                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心工作流程架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                      完整开发工作流                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 1: Discovery (探索)                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 触发: 用户提出任何开发需求                                     │   │
│  │ 技能: using-superpowers (自动触发)                            │   │
│  │      └── 检查是否有适用的 Skill                               │   │
│  │ 输出: 确定使用 brainstorming 技能                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  Phase 2: Design (设计)                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 技能: brainstorming                                          │   │
│  │ 步骤:                                                        │   │
│  │   1. 探索项目上下文                                           │   │
│  │   2. 提供可视化伴侣（可选）                                    │   │
│  │   3. 逐一询问澄清问题                                         │   │
│  │   4. 提出 2-3 种方案及权衡                                     │   │
│  │   5. 分章节展示设计，获取用户批准                               │   │
│  │   6. 编写设计文档到 docs/superpowers/specs/                   │   │
│  │   7. 规范评审循环（子代理评审）                                │   │
│  │   8. 用户评审设计文档                                          │   │
│  │ 输出: 获批的设计规格文档                                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  Phase 3: Planning (计划)                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 技能: writing-plans                                          │   │
│  │ 步骤:                                                        │   │
│  │   1. 分析设计文档，确定文件结构                                │   │
│  │   2. 拆分为 bite-sized 任务（2-5 分钟/任务）                  │   │
│  │   3. 每个任务包含: 文件路径、完整代码、验证步骤、提交命令        │   │
│  │   4. 计划评审循环（子代理评审）                                │   │
│  │ 输出: 详细的实现计划文档                                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  Phase 4: Environment Setup (环境)                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 技能: using-git-worktrees                                    │   │
│  │ 步骤:                                                        │   │
│  │   1. 检测工作目录优先级（.worktrees > worktrees > CLAUDE.md）  │   │
│  │   2. 验证目录被 gitignore 忽略                                │   │
│  │   3. 创建 git worktree 和新分支                               │   │
│  │   4. 自动运行项目设置（npm install / cargo build 等）          │   │
│  │   5. 验证测试基线通过                                         │   │
│  │ 输出: 隔离的、可工作的开发环境                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  Phase 5: Implementation (实现)                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 路径 A: subagent-driven-development（有子代理支持）           │   │
│  │   ├── 每个任务派独立子代理                                     │   │
│  │   ├── 两阶段评审: 规范符合性 → 代码质量                        │   │
│  │   └── 快速迭代，无人值守                                       │   │
│  │                                                                 │   │
│  │ 路径 B: executing-plans（无子代理支持）                        │   │
│  │   ├── 当前会话按批次执行                                       │   │
│  │   ├── 人工检查点                                               │   │
│  │   └── 适合平台不支持子代理的场景                                │   │
│  │                                                                 │   │
│  │ 贯穿始终: test-driven-development                             │   │
│  │   ├── RED: 编写失败的测试                                      │   │
│  │   ├── GREEN: 编写最小实现使测试通过                            │   │
│  │   └── REFACTOR: 重构，保持测试通过                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  Phase 6: Review (评审)                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 技能: requesting-code-review                                 │   │
│  │ 触发: 每个任务完成后 / 功能完成后 / 合并前                      │   │
│  │ 流程:                                                        │   │
│  │   1. 获取 git SHA 范围                                        │   │
│  │   2. 派 code-reviewer 子代理                                   │   │
│  │   3. 根据严重程度处理问题                                      │   │
│  │      - Critical: 立即修复，阻塞进展                            │   │
│  │      - Important: 修复后继续                                   │   │
│  │      - Minor: 记录，后续处理                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  Phase 7: Completion (完成)                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 技能: finishing-a-development-branch                         │   │
│  │ 步骤:                                                        │   │
│  │   1. 验证所有测试通过                                          │   │
│  │   2. 确定基础分支                                              │   │
│  │   3. 提供 4 个选项：                                           │   │
│  │      - 本地合并到基础分支                                      │   │
│  │      - 推送并创建 PR                                           │   │
│  │      - 保持分支现状（稍后处理）                                │   │
│  │      - 丢弃工作（需确认）                                      │   │
│  │   4. 执行用户选择的选项                                        │   │
│  │   5. 清理 worktree（如适用）                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 插件架构详解

Superpowers 采用多平台插件架构，每个平台有独立的适配层：

#### Claude Code 适配

```
.claude-plugin/
├── marketplace.json    # 插件市场注册信息
└── plugin.json         # 插件元数据（名称、版本、作者等）

hooks/
├── hooks.json          # Claude Code 钩子配置（SessionStart）
├── session-start       # 启动钩子脚本（bash）
└── run-hook.cmd        # Windows 钩子执行器
```

**SessionStart 钩子流程：**
1. 匹配 `startup|clear|compact` 事件
2. 执行 `session-start` 脚本
3. 脚本加载 `using-superpowers` Skill 内容到上下文

#### Cursor 适配

```
.cursor-plugin/
└── plugin.json         # Cursor 专用插件配置

hooks/
└── hooks-cursor.json   # Cursor 格式的钩子（camelCase）
```

**差异点：**
- Cursor 使用 camelCase 命名（`sessionStart` vs `SessionStart`）
- Cursor 同时设置 `CLAUDE_PLUGIN_ROOT` 和 `CURSOR_PLUGIN_ROOT`

#### Codex 适配

```
.codex/
├── INSTALL.md          # 手动安装指南
└── hooks.json          # 钩子配置
```

**安装方式：**
用户需手动下载并配置，通过 `fetch` 指令加载 INSTALL.md。

#### OpenCode 适配

```
.opencode/
├── INSTALL.md          # 安装指南
└── plugins/
    └── superpowers.js  # OpenCode 插件实现（Node.js）
```

**技术实现：**
- 使用 `experimental.chat.system.transform` 钩子注入系统提示
- 修改 `config.skills.paths` 动态注册 Skills 目录
- 避免符号链接，纯配置注入

---

## 三、Skills 系统设计

### 3.1 Skill 文件格式

每个 Skill 是一个 Markdown 文件，包含 YAML Frontmatter：

```markdown
---
name: skill-name
description: "精确描述何时使用此 Skill"
---

# Skill 标题

## 概述

核心原则声明。

## 何时使用

使用场景和条件。

## 流程

详细步骤...

## Red Flags

禁止事项...
```

### 3.2 Skill 触发机制

```
用户消息
    │
    ▼
using-superpowers Skill 自动触发
    │
    ├── 检查消息内容
    ├── 匹配 Skill 描述关键词
    └── 确定适用的 Skill
    │
    ▼
触发具体 Skill
    │
    ├── 刚性 Skill (TDD, Debugging)
    │   └── 必须严格遵守
    │
    └── 灵活 Skill (Brainstorming)
        └── 可据上下文调整
```

### 3.3 Skill 依赖关系

```
using-superpowers (元技能，所有入口)
    │
    ├── brainstorming
    │       └── writing-plans
    │               └── using-git-worktrees
    │                       ├── subagent-driven-development
    │                       │       ├── test-driven-development
    │                       │       ├── requesting-code-review
    │                       │       └── finishing-a-development-branch
    │                       │
    │                       └── executing-plans (替代方案)
    │                               └── (同上)
    │
    ├── systematic-debugging (独立触发)
    │
    ├── receiving-code-review (独立触发)
    │
    └── dispatching-parallel-agents (进阶)
            └── writing-skills (进阶)
```

---

## 四、核心 Skills 详解

### 4.1 brainstorming (头脑风暴)

**目的：** 在编写代码前将想法转化为完整设计

**核心流程：**
1. 探索项目上下文（文件、文档、最近提交）
2. 提供可视化伴侣（可选）
3. 逐一询问澄清问题
4. 提出 2-3 种方案及权衡
5. 分章节展示设计，获取用户批准
6. 编写设计文档
7. 规范评审循环（子代理评审，最多 3 轮）
8. 用户评审设计文档

**关键原则：**
- YAGNI（你不会需要它）- 从设计中删除不必要的功能
- 每个单元应有明确边界和定义良好的接口
- 遵循现有代码模式

**可视化伴侣：**
- 浏览器辅助工具，用于展示原型、图表、视觉选项
- 可选功能，需要用户同意
- 仅用于视觉内容（原型、布局），文本问题仍用终端

### 4.2 writing-plans (编写计划)

**目的：** 创建详细的实现计划，假设执行者零上下文

**计划文档结构：**
```markdown
# [功能名] 实现计划

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development...

**Goal:** 一句话描述

**Architecture:** 2-3 句话

**Tech Stack:** 关键技术/库

---

## Task N: [组件名]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: 编写失败的测试**
- [ ] **Step 2: 运行测试验证失败**
- [ ] **Step 3: 编写最小实现**
- [ ] **Step 4: 运行测试验证通过**
- [ ] **Step 5: 提交**
```

**任务粒度：**
- 每个步骤 2-5 分钟
- 每个任务自包含
- 频繁提交

### 4.3 subagent-driven-development (子代理驱动开发)

**目的：** 通过派生子代理并行执行任务，实现高质量快速迭代

**核心机制：**
```
读取计划 → 提取所有任务 → 创建 TodoWrite
    │
    ├── Task 1
    │   ├── 派实现者子代理 (implementer-prompt.md)
    │   ├── 实现者提问 → 回答 → 实现、测试、提交、自评审
    │   ├── 派规范评审子代理 (spec-reviewer-prompt.md)
    │   ├── 规范符合？→ 修复 → 重新评审
    │   ├── 派代码质量评审子代理 (code-quality-reviewer-prompt.md)
    │   ├── 质量通过？→ 修复 → 重新评审
    │   └── 标记任务完成
    │
    ├── Task 2 (同上)
    │
    └── ...
    │
    └── 所有任务完成
        ├── 派最终代码评审子代理
        └── 使用 finishing-a-development-branch
```

**两阶段评审：**
1. **规范符合性评审** - 确认代码符合计划要求
2. **代码质量评审** - 确认代码质量（模式、命名、测试等）

**模型选择策略：**
- 机械任务（1-2 文件，明确规范）→ 快速廉价模型
- 集成任务（多文件，需要判断）→ 标准模型
- 架构任务（设计、评审）→ 最强可用模型

### 4.4 test-driven-development (测试驱动开发)

**目的：** 确保代码质量，先写测试再实现

**Red-Green-Refactor 循环：**

```
┌─────────────────────────────────────────────────────────────────┐
│                        TDD 循环                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RED - 编写失败的测试                                            │
│  ├── 单一行为                                                    │
│  ├── 清晰命名                                                    │
│  └── 真实代码（避免过度 mock）                                   │
│           │                                                     │
│           ▼                                                     │
│  验证 RED - 观察测试失败                                         │
│  ├── 强制步骤，永不跳过                                          │
│  ├── 确认失败消息符合预期                                        │
│  └── 失败原因为功能缺失（非拼写错误）                             │
│           │                                                     │
│           ▼                                                     │
│  GREEN - 最小实现                                                │
│  ├── 编写使测试通过的最简单代码                                   │
│  ├── 不添加功能、不重构其他代码                                   │
│  └── 不超实现                                                    │
│           │                                                     │
│           ▼                                                     │
│  验证 GREEN - 观察测试通过                                       │
│  ├── 强制步骤                                                    │
│  ├── 确认其他测试仍通过                                          │
│  └── 输出干净（无错误、警告）                                     │
│           │                                                     │
│           ▼                                                     │
│  REFACTOR - 清理                                                 │
│  ├── 仅在通过后重构                                              │
│  ├── 删除重复、改进命名、提取辅助函数                             │
│  └── 保持测试通过，不添加行为                                     │
│           │                                                     │
│           └─────────────────────────────────────────────────────┘
```

**核心法则：**
```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
（没有失败的测试，就没有生产代码）
```

### 4.5 systematic-debugging (系统化调试)

**目的：** 找到根本原因而非症状

**铁律：**
```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
（没有根本原因调查，就没有修复）
```

**四阶段流程：**

| 阶段 | 关键活动 | 成功标准 |
|------|----------|----------|
| **1. 根本原因调查** | 阅读错误、复现、检查变更、收集证据 | 理解 WHAT 和 WHY |
| **2. 模式分析** | 找正常示例、对比参考实现、识别差异 | 确定差异点 |
| **3. 假设与测试** | 形成单一假设、最小化测试、验证 | 确认或新假设 |
| **4. 实现** | 创建失败测试、修复、验证 | Bug 解决，测试通过 |

**3+ 修复失败 = 架构问题：**
- 如果尝试 3 次修复都失败，停止修复
- 讨论架构是否根本错误
- 与用户讨论后再继续

### 4.6 using-git-worktrees (使用 Git Worktrees)

**目的：** 创建隔离的开发工作区

**目录选择优先级：**
1. 检查现有目录（`.worktrees` > `worktrees`）
2. 检查 CLAUDE.md 配置
3. 询问用户

**安全检查：**
- 项目本地目录必须被 gitignore 忽略
- 如果未忽略，立即修复（添加 gitignore 并提交）

**创建流程：**
1. 检测项目名称
2. 创建 worktree 和新分支
3. 运行项目设置（自动检测 npm/cargo/pip/go）
4. 验证测试基线
5. 报告位置

### 4.7 finishing-a-development-branch (完成开发分支)

**目的：** 规范地完成开发工作

**四选项：**
1. **本地合并** - 合并到基础分支，清理 worktree
2. **推送并创建 PR** - 保持 worktree
3. **保持现状** - 保持 worktree 和分支
4. **丢弃** - 需输入 "discard" 确认

**关键步骤：**
1. 验证测试通过（失败则不继续）
2. 确定基础分支
3. 提供选项
4. 执行选择
5. 清理 worktree（选项 1 和 4）

---

## 五、项目结构详解

### 5.1 完整目录树

```
superpowers/
│
├── .claude-plugin/                    # Claude Code 插件
│   ├── marketplace.json               # 市场注册信息
│   └── plugin.json                    # 插件元数据
│
├── .cursor-plugin/                    # Cursor 插件
│   └── plugin.json                    # Cursor 格式配置
│
├── .codex/                            # Codex CLI 支持
│   ├── INSTALL.md                     # 手动安装指南
│   └── hooks.json                     # 钩子配置
│
├── .opencode/                         # OpenCode 支持
│   ├── INSTALL.md                     # 安装指南
│   └── plugins/
│       └── superpowers.js             # OpenCode 插件实现
│
├── .github/
│   └── FUNDING.yml                    # GitHub Sponsors
│
├── agents/                            # 代理定义
│   └── code-reviewer.md               # 代码评审代理
│
├── commands/                          # 快捷命令（备用）
│   ├── brainstorm.md                  # 头脑风暴命令
│   ├── execute-plan.md                # 执行计划命令
│   └── write-plan.md                  # 编写计划命令
│
├── docs/                              # 文档
│   ├── README.codex.md                # Codex 专用 README
│   ├── README.opencode.md             # OpenCode 专用 README
│   ├── testing.md                     # 测试指南
│   ├── windows/
│   │   └── polyglot-hooks.md          # Windows 多语言钩子
│   ├── plans/                         # 历史计划文档
│   │   ├── 2025-11-22-opencode-support-design.md
│   │   ├── 2025-11-22-opencode-support-implementation.md
│   │   └── ...
│   └── superpowers/                   # 设计和规格文档
│       ├── plans/                     # 实现计划
│       └── specs/                     # 设计规格
│
├── hooks/                             # 钩子脚本
│   ├── hooks.json                     # Claude 格式钩子配置
│   ├── hooks-cursor.json              # Cursor 格式钩子配置
│   ├── session-start                   # 会话启动脚本（bash）
│   └── run-hook.cmd                   # Windows 钩子执行器
│
├── skills/                            # Skills 目录（核心）
│   │
│   ├── using-superpowers/             # 元技能：系统引导
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── codex-tools.md         # Codex 工具映射
│   │       └── gemini-tools.md        # Gemini 工具映射
│   │
│   ├── brainstorming/                 # 设计阶段
│   │   ├── SKILL.md
│   │   ├── visual-companion.md        # 可视化伴侣指南
│   │   ├── spec-document-reviewer-prompt.md
│   │   └── scripts/                   # Brainstorm 服务器
│   │       ├── frame-template.html
│   │       ├── helper.js
│   │       ├── server.js              # Node.js 服务器
│   │       ├── start-server.sh
│   │       └── stop-server.sh
│   │
│   ├── writing-plans/                 # 计划阶段
│   │   ├── SKILL.md
│   │   └── plan-document-reviewer-prompt.md
│   │
│   ├── using-git-worktrees/           # 环境准备
│   │   └── SKILL.md
│   │
│   ├── subagent-driven-development/   # 开发执行（子代理）
│   │   ├── SKILL.md
│   │   ├── implementer-prompt.md      # 实现者提示词
│   │   ├── spec-reviewer-prompt.md    # 规范评审提示词
│   │   └── code-quality-reviewer-prompt.md
│   │
│   ├── executing-plans/               # 开发执行（单会话）
│   │   └── SKILL.md
│   │
│   ├── test-driven-development/       # TDD
│   │   ├── SKILL.md
│   │   └── testing-anti-patterns.md   # 测试反模式
│   │
│   ├── requesting-code-review/        # 代码评审请求
│   │   ├── SKILL.md
│   │   └── code-reviewer.md           # 评审代理提示词
│   │
│   ├── receiving-code-review/         # 处理评审反馈
│   │   └── SKILL.md
│   │
│   ├── finishing-a-development-branch/ # 开发完成
│   │   └── SKILL.md
│   │
│   ├── systematic-debugging/          # 调试
│   │   ├── SKILL.md
│   │   ├── CREATION-LOG.md
│   │   ├── condition-based-waiting.md
│   │   ├── condition-based-waiting-example.ts
│   │   ├── defense-in-depth.md
│   │   ├── find-polluter.sh
│   │   ├── root-cause-tracing.md
│   │   ├── test-academic.md
│   │   ├── test-pressure-1.md
│   │   ├── test-pressure-2.md
│   │   └── test-pressure-3.md
│   │
│   ├── dispatching-parallel-agents/   # 并行代理（进阶）
│   │   └── SKILL.md
│   │
│   └── writing-skills/                # 编写 Skills（进阶）
│       └── SKILL.md
│
├── CHANGELOG.md                       # 变更日志
├── GEMINI.md                          # Gemini CLI 入口
├── LICENSE                            # MIT 许可证
├── README.md                          # 主文档
├── RELEASE-NOTES.md                   # 发布说明
├── gemini-extension.json              # Gemini 扩展配置
├── hooks/hooks.json                   # 兼容性钩子配置
├── package.json                       # Node.js 元数据
└── .gitignore                         # Git 忽略配置
```

---

## 六、代码质量与设计模式

### 6.1 设计模式应用

| 模式 | 应用位置 | 说明 |
|------|----------|------|
| **命令模式** | 每个 Skill | Skill = 封装命令，统一接口 |
| **策略模式** | subagent-driven-development vs executing-plans | 根据平台能力选择策略 |
| **模板方法** | TDD 流程 | 固定的 Red-Green-Refactor 骨架 |
| **观察者模式** | hooks/session-start | 会话启动事件触发 |
| **依赖注入** | OpenCode 插件 | 配置动态注入 |

### 6.2 代码质量特征

**清晰命名：**
- Skill 名称使用动词开头（using-, writing-, brainstorming）
- 描述精确说明触发条件
- 文件名与内容一致

**一致性：**
- 所有 Skill 使用相同的 Frontmatter 格式
- 所有流程使用 Graphviz DOT 图表示
- 所有 Red Flags 使用表格格式

**模块化：**
- 每个 Skill 独立，可单独使用
- 脚本分离（server.js, helper.js）
- 提示词模板分离（*prompt.md）

### 6.3 文档质量

**注释即设计：**
- 每个 Skill 的注释描述设计决策和意图
- 流程图先于代码（DOT 图定义流程）
- 使用 `<HARD-GATE>`、`<EXTREMELY-IMPORTANT>` 等标记强调

**示例丰富：**
- 每个流程都有具体示例
- Good/Bad 对比展示
- 真实场景的工作流示例

---

## 七、平台兼容性设计

### 7.1 多平台支持策略

```
统一核心 (skills/)
    │
    ├── Claude Code ──→ .claude-plugin/ + hooks/hooks.json
    │
    ├── Cursor ───────→ .cursor-plugin/ + hooks/hooks-cursor.json
    │
    ├── Codex ────────→ .codex/ + hooks.json
    │
    ├── OpenCode ─────→ .opencode/ + superpowers.js
    │
    └── Gemini ───────→ gemini-extension.json + GEMINI.md
```

### 7.2 技术挑战与解决

| 挑战 | 解决方案 |
|------|----------|
| **不同钩子格式** | 为 Cursor 单独维护 hooks-cursor.json（camelCase） |
| **工具名差异** | using-superpowers/references/ 下的工具映射文档 |
| **子代理支持** | 自动检测，无子代理时使用 executing-plans 替代 |
| **Windows 兼容** | 使用 `#!/usr/bin/env bash`，避免 `/bin/bash` 依赖 |
| **PID 命名空间** | Windows 下通过 `/proc/$PPID/winpid` 获取原生 PID |

### 7.3 跨平台工具映射

**Claude Code → OpenCode 映射：**
| Claude Code | OpenCode |
|-------------|----------|
| `TodoWrite` | `todowrite` |
| `Task` (subagent) | `@mention` (subagent) |
| `Skill` | `skill` (native) |
| `Read/Write/Edit/Bash` | Native tools |

---

## 八、项目演进历史

### 8.1 版本演进

**v5.0.4 (当前):**
- 稳定的 14+ Skills 体系
- 5 平台支持
- Windows 兼容性修复

**近期修复 (CHANGELOG.md):**
- Brainstorm server Windows 支持（检测 Git Bash）
- 可移植 shebangs（`#!/usr/bin/env bash`）
- POSIX-safe hook 脚本（dash 兼容）
- Bash 5.3+ hook 挂起修复（printf 替代 heredoc）
- Cursor hooks 支持

### 8.2 设计文档演进

```
docs/superpowers/
├── specs/
│   ├── 2026-01-22-document-review-system-design.md
│   ├── 2026-02-19-visual-brainstorming-refactor-design.md
│   └── 2026-03-11-zero-dep-brainstorm-server-design.md
│
└── plans/
    ├── 2026-01-22-document-review-system.md
    ├── 2026-02-19-visual-brainstorming-refactor.md
    └── 2026-03-11-zero-dep-brainstorm-server.md
```

**文档即代码：**
- 每个重大功能都有设计文档
- 设计文档经过评审循环
- 实现计划基于设计文档

---

## 九、项目亮点

### 9.1 技术创新

1. **Skill 系统** - 将软件开发最佳实践封装为可组合单元
2. **子代理驱动** - 通过委派子代理实现高质量并行开发
3. **两阶段评审** - 规范符合性 + 代码质量双重把关
4. **可视化伴侣** - 浏览器辅助的视觉设计工具
5. **多平台插件** - 统一核心 + 平台适配层架构

### 9.2 工程实践

1. **文档驱动** - 设计先于实现，规格文档化
2. **系统化的质量控制** - TDD + 代码评审 + 调试方法论
3. **平台抽象** - 统一 Skills，多平台适配
4. **持续集成** - 每个任务自动测试、提交
5. **错误预防** - Red Flags 系统防止常见错误

### 9.3 社区与生态

- **开源**: MIT 许可证，GitHub 公开开发
- **社区**: Discord 服务器支持
- **赞助**: GitHub Sponsors 支持
- **贡献**: 接受 Skill 贡献（遵循 writing-skills 指南）

---

## 十、总结

Superpowers 是一个**软件开发流程的工程化系统**。它将编码从即兴行为转变为系统化、可重复、高质量的过程。

### 核心优势

- **系统化流程** - 从设计到完成的完整工作流
- **质量保证** - TDD + 代码评审 + 调试方法论
- **平台无关** - 统一核心，多平台支持
- **可扩展** - Skill 系统支持自定义扩展
- **文档丰富** - 每个流程都有详细文档和示例

### 适用场景

- AI 辅助软件开发
- 需要高质量代码的团队
- 希望标准化开发流程的组织
- 多平台 AI 工具用户

### 设计哲学契合

Superpowers 完美体现了 John Ousterhout 的软件设计哲学：
- **深度模块** - 每个 Skill 简单接口，丰富功能
- **信息隐藏** - 复杂流程封装在 Skill 内部
- **向下吸收复杂度** - 用户获得简单接口
- **定义错误不存在** - 系统化流程减少错误

---

*文档生成完成。如需深入分析特定模块，请告知。*
