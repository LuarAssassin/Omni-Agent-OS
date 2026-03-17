# Aider 代码索引

> 核心模块导航与功能速查

## 快速导航

| 模块 | 行数 | 职责 | 设计模式 |
|------|------|------|----------|
| `coders/base_coder.py` | ~1000 | Coder 基类 | 模板方法模式 |
| `coders/editblock_coder.py` | ~300 | SEARCH/REPLACE 编辑 | 策略模式 |
| `coders/wholefile_coder.py` | ~145 | 全文件替换 | 策略模式 |
| `coders/architect_coder.py` | ~49 | 双模型协作 | 装饰器模式 |
| `repomap.py` | ~700 | 代码库地图 | 缓存模式 |
| `repo.py` | ~600 | Git 操作封装 | 外观模式 |
| `io.py` | ~1000 | 交互抽象 | 适配器模式 |
| `models.py` | ~900 | 模型管理 | 工厂模式 |
| `commands.py` | ~1500 | 斜杠命令 | 命令模式 |
| `args.py` | ~800 | 参数解析 | 建造者模式 |
| `main.py` | ~1100 | 应用入口 | 协调器 |

---

## 核心类索引

### Coder 继承体系

```
Coder (base_coder.py)
├── HelpCoder (help_coder.py)
├── AskCoder (ask_coder.py)
│   ├── ArchitectCoder (architect_coder.py)
│   ├── ContextCoder (context_coder.py)
│   └── AskCoder 基础问答
├── WholeFileCoder (wholefile_coder.py)
│   ├── WholeFileFunctionCoder (wholefile_func_coder.py)
│   └── SingleWholeFileFunctionCoder (single_wholefile_func_coder.py)
├── EditBlockCoder (editblock_coder.py)
│   ├── EditBlockFencedCoder (editblock_fenced_coder.py)
│   ├── EditBlockFunctionCoder (editblock_func_coder.py)
│   ├── EditorEditBlockCoder (editor_editblock_coder.py)
│   └── PatchCoder (patch_coder.py)
├── UnifiedDiffCoder (udiff_coder.py)
│   └── UnifiedDiffSimpleCoder (udiff_simple.py)
├── EditorWholeFileCoder (editor_whole_coder.py)
├── EditorDiffFencedCoder (editor_diff_fenced_coder.py)
└── ...
```

### Prompts 类对应关系

每个 Coder 对应一个 Prompts 类：

| Coder | Prompts | 用途 |
|-------|---------|------|
| `EditBlockCoder` | `EditBlockPrompts` | SEARCH/REPLACE 指令 |
| `WholeFileCoder` | `WholeFilePrompts` | 全文件重写指令 |
| `ArchitectCoder` | `ArchitectPrompts` | 架构师规划指令 |
| `UnifiedDiffCoder` | `UnifiedDiffPrompts` | udiff 格式指令 |
| `HelpCoder` | `HelpPrompts` | 帮助系统指令 |
| `AskCoder` | `AskPrompts` | 问答模式指令 |

---

## 模块详细索引

### coders/ 目录

#### base_coder.py (核心)
```python
class Coder:
    """所有 Coder 的抽象基类"""

    # 关键方法
    @classmethod
    def create(cls, main_model, edit_format, io, ...) -> Coder
    # 工厂方法，根据 edit_format 创建对应 Coder

    def run(self, with_message=None)
    # 主循环：发送消息 → 获取编辑 → 应用编辑 → 提交

    def send_message(self, message)
    # 发送消息到 LLM，处理流式响应

    def get_edits(self) -> List[Edit]
    # 抽象方法：子类实现具体编辑解析

    def apply_edits(self, edits)
    # 抽象方法：子类实现具体编辑应用

    def get_inchat_relative_files(self) -> List[str]
    # 获取当前对话中的文件列表
```

#### editblock_coder.py
```python
class EditBlockCoder(Coder):
    """使用 SEARCH/REPLACE 块编辑代码"""

    edit_format = "diff"

    def get_edits(self):
        # 解析 SEARCH/REPLACE 块
        # 格式: <<<<<<< SEARCH\n...\n=======\n...\n>>>>>>> REPLACE

    def apply_edits(self, edits):
        # 执行文本替换
        # 处理模糊匹配（前导空格容错）
```

**核心函数**:
- `find_original_update_blocks()` - 解析编辑块
- `do_replace()` - 执行替换
- `perfect_replace()` - 精确匹配替换
- `replace_part_with_missing_leading_whitespace()` - 容错替换

#### wholefile_coder.py
```python
class WholeFileCoder(Coder):
    """全文件重写模式"""

    edit_format = "whole"

    def get_edits(self, mode="update"):
        # 解析 ```filename.ext 格式的代码块
        # 支持 diff 模式预览变更
```

#### architect_coder.py
```python
class ArchitectCoder(AskCoder):
    """架构师-编辑者双模型模式"""

    def reply_completed(self):
        # 1. Architect 模型生成方案
        # 2. 创建 Editor Coder 执行修改
        # 3. 合并结果
```

### repomap.py

```python
class RepoMap:
    """代码库地图生成器"""

    def __init__(self, map_tokens, root, main_model, ...)
    # 初始化，加载缓存

    def get_repo_map(self, chat_files, other_files, ...)
    # 生成代码库地图

    def get_ranked_tags_map(self, ...)
    # 基于引用关系排序的符号地图

    def get_tags(self, fname)
    # 使用 Tree-sitter 提取文件符号

    def token_count(self, text)
    # 估算文本 token 数（采样优化）
```

**关键数据结构**:
```python
Tag = namedtuple("Tag", "rel_fname fname line name kind")
# rel_fname: 相对路径
# fname: 绝对路径
# line: 行号
# name: 符号名
# kind: 类型 (def/ref)
```

### repo.py

```python
class GitRepo:
    """Git 仓库操作封装"""

    def __init__(self, io, fnames, git_dname, ...)
    # 初始化，查找 git 仓库根目录

    def commit(self, fnames, context, message, aider_edits)
    # 智能提交：生成提交信息、处理归属

    def get_tracked_files(self)
    # 获取跟踪的文件列表

    def is_dirty(self, fname)
    # 检查文件是否有未提交变更

    def ignored_file(self, fname)
    # 检查文件是否被 .aiderignore 忽略
```

### io.py

```python
class InputOutput:
    """输入输出抽象层"""

    def __init__(self, pretty, yes, ...)
    # 初始化交互配置

    def tool_output(self, msg)
    # 输出工具消息

    def tool_error(self, msg)
    # 输出错误信息

    def tool_warning(self, msg)
    # 输出警告

    def confirm_ask(self, msg, ...)
    # 确认询问（支持 --yes 模式）

    def read_text(self, fname)
    # 读取文件（编码自动检测）

    def write_text(self, fname, content)
    # 写入文件

    def get_coder_completer(self, ...)
    # 获取自动补全器
```

### models.py

```python
@dataclass
class ModelSettings:
    """模型配置数据类"""
    name: str
    edit_format: str = "whole"
    weak_model_name: Optional[str] = None
    use_repo_map: bool = False
    cache_control: bool = False
    ...

class Model:
    """LLM 模型封装"""

    def __init__(self, model_name, ...)
    # 初始化，加载模型配置

    def token_count(self, text)
    # 计算文本 token 数

    def get_repo_map_tokens(self)
    # 获取 RepoMap 推荐 token 数

    def commit_message_models(self)
    # 获取提交信息生成模型
```

### commands.py

```python
class Commands:
    """斜杠命令处理器"""

    def __init__(self, io, coder, ...)

    def cmd_add(self, args)
    # /add - 添加文件到对话

    def cmd_drop(self, args)
    # /drop - 从对话移除文件

    def cmd_commit(self, args)
    # /commit - 提交变更

    def cmd_model(self, args)
    # /model - 切换模型

    def cmd_chat_mode(self, args)
    # /chat-mode - 切换编辑格式

    def cmd_undo(self, args)
    # /undo - 撤销最后一次修改

    def cmd_help(self, args)
    # /help - 显示帮助
```

### args.py

```python
def get_parser(default_config_files, git_root):
    """创建参数解析器"""
    parser = configargparse.ArgumentParser(
        auto_env_var_prefix="AIDER_",
        ...
    )
    # 约 100+ 个参数定义
```

**主要参数组**:
- Main model: `--model`, `--api-key`, ...
- Model settings: `--weak-model`, `--editor-model`, ...
- Git settings: `--auto-commits`, `--dirty-commits`, ...
- Repomap: `--map-tokens`, `--map-refresh`, ...
- Output: `--pretty`, `--no-pretty`, `--stream`, ...

### main.py

```python
def main():
    """应用入口"""
    # 1. 解析参数
    # 2. 初始化 IO
    # 3. 初始化模型
    # 4. 初始化 Git
    # 5. 创建 Coder
    # 6. 启动交互循环

def setup_git(git_root, io)
# 设置 Git 仓库

def check_gitignore(git_root, io)
# 检查 .gitignore 配置

def load_dotenv_files(git_root)
# 加载 .env 文件
```

---

## 关键工具函数索引

### utils.py

| 函数 | 用途 |
|------|------|
| `format_content()` | 格式化内容显示 |
| `format_messages()` | 格式化消息历史 |
| `format_tokens()` | 格式化 token 数显示 |
| `is_image_file()` | 检测是否为图片文件 |
| `touch_file()` | 创建空文件 |
| `find_common_root()` | 查找共同根目录 |
| `safe_abs_path()` | 安全获取绝对路径 |

### run_cmd.py

| 函数 | 用途 |
|------|------|
| `run_cmd()` | 执行 shell 命令 |

### diffs.py

| 函数 | 用途 |
|------|------|
| `diff_partial_update()` | 生成部分更新 diff |
| `unified_diff()` | 生成统一 diff |

---

## 配置与资源

### 模型配置

```
aider/resources/
├── model-settings.yml    # 模型默认配置
└── ...
```

### Prompts 模板

```
aider/coders/
├── base_prompts.py
├── editblock_prompts.py
├── wholefile_prompts.py
├── architect_prompts.py
└── ...
```

---

## 调用关系图

### 启动流程

```
main.py:main()
├── args.py:get_parser()          # 解析参数
├── io.py:InputOutput()           # 初始化 IO
├── models.py:Model()             # 初始化主模型
├── repo.py:GitRepo()             # 初始化 Git
└── coders/base_coder.py:Coder.create()  # 创建 Coder
    └── 根据 edit_format 选择具体 Coder
```

### 对话流程

```
Coder.run()
├── send_message()
│   └── llm.py:litellm.completion()
├── get_edits()                   # 子类实现
│   ├── EditBlockCoder: 解析 SEARCH/REPLACE
│   ├── WholeFileCoder: 解析 ```filename
│   └── ...
├── apply_edits()                 # 子类实现
│   └── io.py:write_text()
└── repo.py:commit()              # 自动提交
```

### RepoMap 生成流程

```
RepoMap.get_repo_map()
├── get_tags()                    # Tree-sitter 解析
├── rank_tags()                   # 引用排序
└── select_within_budget()        # Token 预算筛选
```
