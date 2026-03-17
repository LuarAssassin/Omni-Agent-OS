# Aider 依赖分析

> 技术栈与依赖项详细分析

## 核心依赖概览

| 类别 | 主要依赖 | 用途 |
|------|----------|------|
| LLM 交互 | `litellm` | 多模型统一接口 |
| Git 操作 | `gitpython` | Git 仓库管理 |
| 终端 UI | `prompt-toolkit` | 交互式终端界面 |
| 代码解析 | `tree-sitter`, `grep-ast` | 代码结构分析 |
| Markdown | `rich`, `markdown` | 富文本渲染 |
| 配置管理 | `configargparse` | 多来源配置 |
| 图片处理 | `pillow` | 图片支持 |
| 语音输入 | `speech-recognition`, `sounddevice` | 语音转文字 |
| 网页抓取 | `playwright`, `beautifulsoup4` | URL 内容抓取 |

---

## 详细依赖清单

### LLM 交互层

#### litellm
```
版本: >=1.0.0
用途: 统一多厂商 LLM API
特性:
- 支持 100+ 模型
- 内置重试机制
- 成本追踪
- 日志记录
- 工具调用支持
```

**依赖传递**:
- `openai` - OpenAI API
- `anthropic` - Anthropic API
- `google-generativeai` - Google Gemini
- `boto3` - AWS Bedrock
- 等...

#### backoff
```
用途: 指数退避重试
场景: API 限流时自动重试
```

### Git 操作层

#### gitpython
```
版本: >=3.1.0
用途: Pythonic Git 操作
覆盖功能:
- 仓库初始化/克隆
- 提交/推送
- 分支管理
- 状态查询
- 配置管理
```

#### pathspec
```
用途: Git ignore 模式匹配
场景: .aiderignore 文件解析
```

### 终端 UI 层

#### prompt-toolkit
```
版本: >=3.0.0
用途: 高级终端交互
特性:
- 多行输入编辑
- 语法高亮
- 自动补全
- 历史记录
- Vi/Emacs 模式
```

**相关依赖**:
- `pygments` - 语法高亮
- `regex` - 正则支持

#### rich
```
版本: >=13.0.0
用途: 富文本终端输出
特性:
- Markdown 渲染
- 表格/面板
- 进度条
- 语法高亮
- 主题支持
```

### 代码解析层

#### tree-sitter
```
版本: >=0.20.0
用途: 增量代码解析
支持语言: Python, JS, TS, Go, Rust, Java, C/C++, 等 30+
功能:
- 语法树生成
- 符号提取
- 增量更新
```

#### tree-sitter-languages
```
用途: 预编译语言解析器
注意: 二进制依赖，特定平台
```

#### grep-ast
```
用途: AST 辅助搜索
依赖: tree-sitter
```

### 配置管理

#### configargparse
```
版本: >=1.5.0
用途: 参数 + 配置文件 + 环境变量
特性:
- YAML/INI 配置文件
- 自动环境变量前缀
- 与 argparse 兼容
```

#### python-dotenv
```
用途: .env 文件加载
场景: API 密钥管理
```

#### pyyaml
```
用途: YAML 解析
场景: 模型配置文件
```

### 图片处理

#### pillow (PIL)
```
版本: >=10.0.0
用途: 图片处理
功能:
- 图片格式转换
- 尺寸调整
- 视觉模型输入处理
```

### 语音输入

#### speech-recognition
```
用途: 语音转文字
后端支持:
- Google Speech API
- Whisper API
- 本地 Whisper
```

#### sounddevice
```
用途: 音频录制
平台: 跨平台 (PortAudio)
```

#### numpy
```
用途: 音频数据处理
```

### 网页抓取

#### playwright
```
版本: >=1.40.0
用途: 浏览器自动化
功能:
- 网页内容抓取
- 截图
- 动态内容等待
```

#### beautifulsoup4
```
用途: HTML/XML 解析
配合: playwright 获取页面内容后解析
```

### 其他工具依赖

#### diskcache
```
用途: 磁盘缓存
场景: RepoMap tags 缓存
```

#### json5
```
用途: JSON5 格式解析
场景: 宽松 JSON 解析
```

#### tqdm
```
用途: 进度条
场景: 长时间操作反馈
```

#### tiktoken
```
用途: OpenAI token 计数
场景: token 估算
```

#### requests
```
用途: HTTP 请求
场景: API 调用、下载
```

---

## 可选依赖

### 开发/测试

| 依赖 | 用途 |
|------|------|
| `pytest` | 测试框架 |
| `pytest-cov` | 覆盖率 |
| `black` | 代码格式化 |
| `flake8` | 代码检查 |
| `mypy` | 类型检查 |

### 额外功能

| 依赖 | 用途 | 触发条件 |
|------|------|----------|
| `watchfiles` | 文件监听 | `--watch` 模式 |
| `shtab` | Shell 补全 | 安装时 |
| `mixpanel` | 分析统计 | 启用分析 |
| `posthog` | 产品分析 | 启用分析 |

---

## 依赖关系图

### 核心调用链

```
aider/
├── litellm
│   ├── openai
│   ├── anthropic
│   ├── google-generativeai
│   └── ...
├── gitpython
│   └── git 二进制
├── prompt-toolkit
│   ├── pygments
│   └── regex
├── tree-sitter
│   └── tree-sitter-languages (二进制)
├── rich
│   └── pygments
└── configargparse
    └── pyyaml
```

### 功能依赖映射

| 功能 | 直接依赖 | 间接依赖 |
|------|----------|----------|
| LLM 调用 | litellm | openai, anthropic, ... |
| Git 集成 | gitpython | git 二进制 |
| RepoMap | tree-sitter, grep-ast | tree-sitter-languages |
| 终端 UI | prompt-toolkit, rich | pygments, regex |
| 图片支持 | pillow | - |
| 语音输入 | speech-recognition, sounddevice | numpy, portaudio |
| 网页抓取 | playwright, beautifulsoup4 | 浏览器二进制 |
| 配置系统 | configargparse, python-dotenv | pyyaml |
| 缓存 | diskcache | sqlite3 (内置) |

---

## 平台特定依赖

### macOS
```
# 语音输入
- portaudio (brew install portaudio)

# Tree-sitter
- 预编译 wheel 可用
```

### Linux
```
# 语音输入
- portaudio (apt-get install libportaudio2)
- python3-dev

# Git
- git

# 浏览器 (playwright)
- playwright install
```

### Windows
```
# Tree-sitter
- Visual C++ Redistributable

# 语音输入
- 可能需要额外驱动
```

---

## 依赖安全考虑

### 高权限依赖

| 依赖 | 风险 | 缓解措施 |
|------|------|----------|
| `gitpython` | 执行 git 命令 | 验证仓库路径 |
| `subprocess` | 执行 shell 命令 | 仅执行用户指定命令 |
| `playwright` | 浏览器自动化 | 隔离环境 |

### API 密钥管理

```
推荐方式:
1. 环境变量 (.env 文件)
2. `--set-env` 参数
3. 配置文件 (仅限本地)

避免:
- 硬编码密钥
- 提交密钥到 Git
```

---

## 依赖更新策略

### 自动更新

```bash
# 使用 uv 管理依赖
uv pip compile requirements.in -o requirements.txt

# 更新所有
uv pip compile --upgrade requirements.in -o requirements.txt
```

### 锁定文件

```
requirements.txt - 完整锁定版本
requirements/
├── requirements.in - 直接依赖
└── common-constraints.txt - 版本约束
```

---

## 最小依赖安装

### 核心功能（推荐）

```bash
pip install aider-chat
```

包含: litellm, gitpython, prompt-toolkit, tree-sitter, rich

### 开发安装

```bash
pip install aider-chat[dev]
```

额外包含: pytest, black, flake8

### 完整安装

```bash
pip install aider-chat[all]
```

包含: 语音、浏览器、所有额外功能
