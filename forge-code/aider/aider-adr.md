# Aider 架构决策记录 (ADR)

> Architecture Decision Records for Aider

## ADR-001: 使用策略模式实现多编辑格式支持

### 状态
已接受 (Accepted)

### 背景
Aider 需要支持多种 LLM 输出格式来修改代码：
- SEARCH/REPLACE 块（diff 格式）
- 完整文件重写（whole 格式）
- Unified Diff（udiff 格式）
- Function Calling（function 格式）

每种格式有不同的解析逻辑和应用方式。

### 决策
使用 **Strategy Pattern（策略模式）**，将每种编辑格式实现为独立的 `Coder` 子类。

```python
# 基类定义接口
class Coder:
    edit_format = None

    def get_edits(self):
        raise NotImplementedError

    def apply_edits(self, edits):
        raise NotImplementedError

# 具体实现
class EditBlockCoder(Coder):
    edit_format = "diff"
    # 实现 SEARCH/REPLACE 解析

class WholeFileCoder(Coder):
    edit_format = "whole"
    # 实现全文件解析
```

### 后果

**正面**:
- ✅ 新格式可通过添加子类实现，无需修改现有代码（开闭原则）
- ✅ 每种格式的复杂性封装在独立模块中
- ✅ 易于单元测试（每个 Coder 可独立测试）
- ✅ 格式特定逻辑不会泄漏到其他部分

**负面**:
- ⚠️ 类数量增加（目前 15+ Coder 类）
- ⚠️ 需要在 `__init__.py` 中维护导出列表

### 替代方案考虑

| 方案 | 拒绝原因 |
|------|----------|
| 单一 Coder + 策略对象 | 增加一层抽象，没有明显收益 |
| 函数式（每个格式一个函数） | 丢失状态管理能力，难以扩展 |

---

## ADR-002: RepoMap 作为代码库上下文管理机制

### 状态
已接受 (Accepted)

### 背景
大型代码库往往包含数万文件，远超 LLM 上下文限制。需要一种机制：
1. 选择最相关的代码片段提供给 LLM
2. 让 LLM 理解代码库结构而不需要看到所有代码

### 决策
实现独立的 `RepoMap` 模块，使用以下技术：

1. **Tree-sitter 解析**: 提取符号（函数、类、变量）
2. **引用分析**: 基于符号引用关系排序相关性
3. **分层缓存**: SQLite 缓存避免重复解析
4. **Token 预算**: 在配置限制内选择最优代码片段

```python
class RepoMap:
    def get_repo_map(self, chat_files, other_files, max_map_tokens):
        # 1. 获取所有文件的符号
        tags = self.get_tags(files)
        # 2. 基于引用关系排序
        ranked = self.rank_tags(tags, mentioned_idents)
        # 3. 在 token 限制内选择
        return self.select_within_budget(ranked, max_map_tokens)
```

### 后果

**正面**:
- ✅ 与 Coder 逻辑解耦，可独立演进
- ✅ 缓存机制显著提升性能
- ✅ 支持多种编程语言（Tree-sitter 生态）

**负面**:
- ⚠️ 依赖 Tree-sitter 二进制库，增加安装复杂性
- ⚠️ 需要维护 Tree-sitter 语言解析器

---

## ADR-003: 使用 LiteLLM 作为模型抽象层

### 状态
已接受 (Accepted)

### 背景
需要支持多种 LLM 提供商：OpenAI、Anthropic、DeepSeek、Gemini、本地模型等。

### 决策
集成 **LiteLLM** 作为统一抽象层，而非自建多供应商支持。

```python
# aider/llm.py
import litellm

# 使用 LiteLLM 的统一接口
response = litellm.completion(
    model=model_name,
    messages=messages,
    ...
)
```

### 后果

**正面**:
- ✅ 100+ 模型开箱即用
- ✅ 内置重试、日志、成本追踪
- ✅ 统一的消息格式和工具调用支持
- ✅ 社区维护，持续更新

**负面**:
- ⚠️ 依赖外部项目的发展
- ⚠️ 需要跟进 LiteLLM 的破坏性更新
- ⚠️ 部分高级功能可能受限

### 替代方案考虑

| 方案 | 拒绝原因 |
|------|----------|
| 自建多供应商抽象 | 维护成本过高，重复造轮子 |
| 每个供应商独立实现 | 代码重复，难以保持一致性 |

---

## ADR-004: 双模型架构 (ArchitectCoder)

### 状态
已接受 (Accepted)

### 背景
某些模型（如 Claude 3.7 Sonnet）擅长规划和设计，但输出格式不够精确；其他模型（如 GPT-4o）擅长精确编辑但规划能力稍弱。

### 决策
实现 `ArchitectCoder`，采用双模型协作模式：

1. **Architect 模型**: 负责规划和设计，输出自然语言修改方案
2. **Editor 模型**: 负责执行修改，将方案转换为精确代码变更

```python
class ArchitectCoder(AskCoder):
    def reply_completed(self):
        # 1. Architect 模型生成方案
        content = self.partial_response_content

        # 2. 创建 Editor Coder
        editor_coder = Coder.create(
            main_model=self.main_model.editor_model,
            edit_format=self.main_model.editor_edit_format,
            ...
        )

        # 3. Editor 执行修改
        editor_coder.run(with_message=content)
```

### 后果

**正面**:
- ✅ 充分发挥不同模型的优势
- ✅ 规划阶段的输出不受格式限制，更自然
- ✅ 执行阶段使用最适合编辑的模型

**负面**:
- ⚠️ 成本增加（两次 API 调用）
- ⚠️ 延迟增加
- ⚠️ 需要维护两个模型的配置

---

## ADR-005: Git 集成作为一等公民

### 状态
已接受 (Accepted)

### 背景
AI 编程助手会频繁修改代码，需要可靠的版本控制集成。

### 决策
将 Git 集成深度整合到核心流程：

1. **自动提交**: 每次修改后自动 git commit
2. **智能归因**: 区分人类提交和 AI 提交
3. **变更追踪**: 使用 git 状态管理文件变更
4. **提交信息生成**: 使用 LLM 生成提交信息

```python
class GitRepo:
    def commit(self, fnames, context, message, aider_edits):
        # 自动暂存、生成提交信息、提交
        ...
```

### 后果

**正面**:
- ✅ 用户可随时回滚 AI 修改
- ✅ 透明地记录所有变更
- ✅ 符合开发者现有工作流

**负面**:
- ⚠️ 需要处理各种 Git 配置边界情况
- ⚠️ 增加了对 GitPython 的依赖

---

## ADR-006: Prompt 模板分离

### 状态
已接受 (Accepted)

### 背景
不同编辑格式需要不同的系统提示词（system prompts）。

### 决策
每个 Coder 关联独立的 Prompts 类：

```python
class EditBlockCoder(Coder):
    gpt_prompts = EditBlockPrompts()

class EditBlockPrompts:
    system_reminder = "..."
    main_system = "..."
    files_content_prefix = "..."
```

### 后果

**正面**:
- ✅ Prompt 与代码逻辑分离
- ✅ 易于测试和版本控制不同 Prompt 变体
- ✅ 支持多语言 Prompt（国际化基础）

**负面**:
- ⚠️ 类数量翻倍（Coder + Prompts）

---

## ADR-007: 配置系统的分层设计

### 状态
已接受 (Accepted)

### 背景
Aider 有大量配置选项，需要支持多种配置来源。

### 决策
采用分层配置系统（优先级从高到低）：

1. 命令行参数
2. 环境变量（`AIDER_*`）
3. `.aider.conf.yml` 配置文件
4. 默认值

使用 `configargparse` 库实现：

```python
parser = configargparse.ArgumentParser(
    auto_env_var_prefix="AIDER_",
    default_config_files=[".aider.conf.yml"],
)
```

### 后果

**正面**:
- ✅ 配置来源清晰，优先级明确
- ✅ 支持配置文件、环境变量、命令行混用
- ✅ 自动生成 shell 补全

**负面**:
- ⚠️ `args.py` 文件膨胀（~800 行）

---

## ADR-008: 输入输出抽象层 (IO 模块)

### 状态
已接受 (Accepted)

### 背景
需要支持多种交互方式：终端、脚本、IDE 集成、Web 界面。

### 决策
创建 `InputOutput` 类作为所有交互的抽象层：

```python
class InputOutput:
    def tool_output(self, msg):
        # 输出工具消息

    def tool_error(self, msg):
        # 输出错误

    def confirm_ask(self, msg):
        # 确认提问

    def read_text(self, fname):
        # 读取文件（支持编码检测）
```

### 后果

**正面**:
- ✅ 底层交互细节完全隐藏
- ✅ 易于 mock 测试
- ✅ 支持非终端环境（如 CI/CD）

**负面**:
- ⚠️ 需要维护与 prompt-toolkit 的集成

---

## 决策时间线

| ADR | 日期 | 作者 | 状态 |
|-----|------|------|------|
| ADR-001 | 2023 | Paul Gauthier | 已接受 |
| ADR-002 | 2023 | Paul Gauthier | 已接受 |
| ADR-003 | 2023 | Paul Gauthier | 已接受 |
| ADR-004 | 2024 | Paul Gauthier | 已接受 |
| ADR-005 | 2023 | Paul Gauthier | 已接受 |
| ADR-006 | 2023 | Paul Gauthier | 已接受 |
| ADR-007 | 2023 | Paul Gauthier | 已接受 |
| ADR-008 | 2023 | Paul Gauthier | 已接受 |
