# Aider 项目文档

> AI Pair Programming in Your Terminal
>
> 深度架构分析与设计模式研究

## 文档导航

| 文档 | 内容 | 适用读者 |
|------|------|----------|
| [aider-analysis.md](./aider-analysis.md) | 架构深度分析、设计模式评估、软件设计哲学应用 | 架构师、高级开发者 |
| [aider-adr.md](./aider-adr.md) | 架构决策记录 (ADRs) | 技术负责人、贡献者 |
| [aider-code-index.md](./aider-code-index.md) | 代码索引、模块导航、API 速查 | 开发者、贡献者 |
| [aider-dependencies.md](./aider-dependencies.md) | 依赖分析、技术栈 | 运维、开发者 |

## 项目速览

### 基本信息

```yaml
名称: Aider
定位: 终端 AI 结对编程工具
语言: Python
代码规模: ~13,000+ 行（核心）
License: Apache 2.0
GitHub: https://github.com/Aider-AI/aider
```

### 核心特性

- **多模型支持**: OpenAI, Anthropic, DeepSeek, Gemini, 100+ 模型
- **RepoMap**: 智能代码库上下文管理
- **多种编辑格式**: diff, whole, udiff, architect 等
- **Git 集成**: 自动提交、变更追踪
- **终端 UI**: prompt-toolkit 驱动的交互界面
- **扩展性**: 插件化 Coder 架构

### 架构亮点

```
┌─────────────────────────────────────────────┐
│                  Terminal                    │
│           (prompt-toolkit + rich)           │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│              InputOutput                     │
│              (交互抽象层)                     │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│                 Coder                        │
│  ┌──────────┬──────────┬──────────────────┐ │
│  │EditBlock │  Whole   │    Architect     │ │
│  │  Coder   │  Coder   │     Coder        │ │
│  └──────────┴──────────┴──────────────────┘ │
│            (策略模式实现)                     │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────┬──────────┬───────────┐
│       RepoMap       │   Repo   │  Models   │
│    (代码库地图)      │  (Git)   │  (LLM)    │
└─────────────────────┴──────────┴───────────┘
```

## 快速理解 Aider 架构

### 1. 核心设计模式

**策略模式 (Strategy Pattern)** 是整个项目的架构核心：

```python
# 不同的编辑格式 = 不同的策略
Coder.create(edit_format="diff")    # → EditBlockCoder
Coder.create(edit_format="whole")   # → WholeFileCoder
Coder.create(edit_format="architect") # → ArchitectCoder
```

### 2. 模块职责

| 模块 | 职责 | 设计哲学评价 |
|------|------|-------------|
| `coders/` | 编辑策略实现 | Deep Module，信息隐藏优秀 |
| `repomap.py` | 代码库上下文 | 算法模块，缓存设计精良 |
| `io.py` | 交互抽象 | 适配器模式，终端细节隐藏 |
| `repo.py` | Git 操作 | 外观模式，简化 Git 使用 |
| `models.py` | LLM 管理 | 工厂模式，配置集中 |

### 3. 关键流程

```
用户输入 → IO 层 → Coder → LLM API
                         ↓
                    解析响应 → 应用编辑 → Git 提交
```

## 软件设计哲学应用

基于 John Ousterhout 《A Philosophy of Software Design》评估：

### 做得好的方面

| 原则 | 体现位置 | 说明 |
|------|----------|------|
| **深度模块** | `coders/base_coder.py` | 简单接口隐藏复杂编辑逻辑 |
| **信息隐藏** | `repomap.py` | Tree-sitter 细节完全封装 |
| **策略模式** | `coders/` 目录 | 扩展编辑格式无需改核心 |
| **复杂度下沉** | `base_coder.__init__` | 缓存预热、令牌计算内部化 |

### 改进空间

| 位置 | 建议 |
|------|------|
| `args.py` (~800行) | 拆分为子模块或配置类 |
| `base_coder.__init__` | 使用配置对象减少参数数量 |
| `utils.py` | 按功能拆分为多个工具模块 |

## 学习价值

### 优秀实践

1. **策略模式解耦**: 15+ 种编辑格式的优雅组织
2. **缓存设计**: RepoMap 的多级缓存策略
3. **双模型架构**: ArchitectCoder 的创新模式
4. **终端 UI**: prompt-toolkit 的最佳实践

### 可借鉴模式

```python
# 1. 工厂 + 策略组合
class Coder:
    @classmethod
    def create(cls, edit_format, ...):
        for coder in coders.__all__:
            if coder.edit_format == edit_format:
                return coder(...)

# 2. 模板方法定义流程
class Coder:
    def run(self):
        self.send_message()
        edits = self.get_edits()      # 子类实现
        self.apply_edits(edits)       # 子类实现

# 3. 智能缓存
class RepoMap:
    def get_tags(self, fname):
        if fname in self.cache:
            return self.cache[fname]
        tags = self._parse_with_treesitter(fname)
        self.cache[fname] = tags
        return tags
```

## 贡献指南

### 添加新编辑格式

1. 在 `coders/` 创建新类继承 `Coder`
2. 实现 `get_edits()` 和 `apply_edits()`
3. 创建对应的 `Prompts` 类
4. 在 `coders/__init__.py` 注册

示例：

```python
# coders/myformat_coder.py
class MyFormatCoder(Coder):
    edit_format = "myformat"
    gpt_prompts = MyFormatPrompts()

    def get_edits(self):
        # 解析 LLM 响应
        pass

    def apply_edits(self, edits):
        # 应用编辑
        pass
```

### 代码规范

- 遵循 PEP 8
- 使用类型注解（关键接口）
- 添加文档字符串
- 保持向后兼容

## 相关资源

### 官方资源

- [官网](https://aider.chat/)
- [GitHub](https://github.com/Aider-AI/aider)
- [文档](https://aider.chat/docs/)
- [HISTORY](https://aider.chat/HISTORY.html)

### 社区

- [Discord](https://discord.gg/Y7X7bhMQFV)
- [GitHub Issues](https://github.com/Aider-AI/aider/issues)
- [GitHub Discussions](https://github.com/Aider-AI/aider/discussions)

### 相关项目

| 项目 | 说明 | 关系 |
|------|------|------|
| [LiteLLM](https://github.com/BerriAI/litellm) | 多模型统一接口 | 核心依赖 |
| [Tree-sitter](https://github.com/tree-sitter/tree-sitter) | 代码解析 | 核心依赖 |
| [prompt-toolkit](https://github.com/prompt-toolkit/python-prompt-toolkit) | 终端 UI | 核心依赖 |

---

*本文档基于 Aider 源码分析生成，遵循软件设计哲学原则进行架构评估。*
