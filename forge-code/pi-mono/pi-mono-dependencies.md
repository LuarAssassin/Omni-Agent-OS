# Pi-Mono 依赖分析

> 包依赖关系和外部依赖清单
> 项目: pi-mono
> 生成时间: 2026-03-17

---

## 一、内部依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                          内部依赖关系                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌──────────────┐                                            │
│    │   pi-ai      │ ◄───────────────────────┐                  │
│    │  (基础层)     │                        │                  │
│    └──────┬───────┘                        │                  │
│           │ 被所有上层依赖                    │                  │
│           ▼                                │                  │
│    ┌──────────────┐                        │                  │
│    │ pi-agent-core│ ◄──────────┐           │                  │
│    │  (Agent 层)   │            │           │                  │
│    └──────┬───────┘            │           │                  │
│           │                    │           │                  │
│     ┌─────┴─────┐              │           │                  │
│     ▼           ▼              │           │                  │
│ ┌───────┐  ┌────────┐         │           │                  │
│ │pi-tui │  │pi-web-ui│ ◄──────┘           │                  │
│ │(TUI)  │  │(Web UI)│                      │                  │
│ └───┬───┘  └───┬────┘                      │                  │
│     │          │                            │                  │
│     └────┬─────┘                            │                  │
│          ▼                                  │                  │
│    ┌──────────────┐                         │                  │
│    │ pi-coding    │ ◄───────────────────────┘                  │
│    │   -agent     │                                            │
│    └──────┬───────┘                                            │
│           │                                                    │
│     ┌─────┴─────┐                                              │
│     ▼           ▼                                              │
│ ┌───────┐  ┌────────┐                                          │
│ │pi-mom │  │pi-pods │                                          │
│ │(Slack)│  │(GPU)   │                                          │
│ └───────┘  └────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、各包依赖详情

### 2.1 @mariozechner/pi-ai

**无内部依赖**（基础层）

**外部依赖**:
| 依赖 | 用途 |
|------|------|
| `@anthropic-ai/sdk` | Anthropic API |
| `@aws-sdk/client-bedrock-runtime` | AWS Bedrock |
| `@google/genai` | Google Gemini |
| `@mistralai/mistralai` | Mistral API |
| `openai` | OpenAI API |
| `@sinclair/typebox` | 运行时类型验证 |
| `chalk` | 终端颜色 |
| `undici` | HTTP 客户端 |
| `proxy-agent` | 代理支持 |
| `zod-to-json-schema` | Zod 转 JSON Schema |
| `zx` | 脚本工具 |

**关键设计**: 作为基础层，pi-ai 不依赖任何内部包，可以被独立使用。

---

### 2.2 @mariozechner/pi-agent-core

**内部依赖**:
| 依赖 | 关系 |
|------|------|
| `@mariozechner/pi-ai` | 核心依赖，所有 LLM 调用通过 pi-ai |

**外部依赖**: 无额外外部依赖（依赖由 pi-ai 传递）

**关键设计**: Agent 层只依赖 pi-ai，保持独立性。

---

### 2.3 @mariozechner/pi-tui

**无内部依赖**（可独立使用）

**外部依赖**:
| 依赖 | 用途 |
|------|------|
| `chalk` | 终端颜色 |
| `marked` | Markdown 解析 |
| `get-east-asian-width` | 东亚字符宽度计算 |
| `mime-types` | MIME 类型检测 |
| `koffi` (可选) | 原生图像处理 |

**关键设计**: TUI 库完全独立，可用于任何终端应用。

---

### 2.4 @mariozechner/pi-coding-agent

**内部依赖**:
| 依赖 | 关系 |
|------|------|
| `@mariozechner/pi-ai` | LLM API 调用 |
| `@mariozechner/pi-agent-core` | Agent 运行时 |
| `@mariozechner/pi-tui` | 终端 UI |

**外部依赖**:
| 依赖 | 用途 |
|------|------|
| `chalk` | 终端颜色 |
| `diff` | 差异计算 |
| `glob` | 文件匹配 |
| `ignore` | gitignore 解析 |
| `marked` | Markdown 渲染 |
| `minimatch` | 模式匹配 |
| `yaml` | YAML 解析 |
| `strip-ansi` | ANSI 去除 |
| `@mariozechner/jiti` | TypeScript 运行时加载 |

**关键设计**: 依赖最多，是功能最完整的包。

---

### 2.5 @mariozechner/pi-mom

**内部依赖**:
| 依赖 | 关系 |
|------|------|
| `@mariozechner/pi-ai` | LLM API |
| `@mariozechner/pi-agent-core` | Agent 运行时 |
| `@mariozechner/pi-coding-agent` | 工具和环境 |

**外部依赖**:
| 依赖 | 用途 |
|------|------|
| `@slack/socket-mode` | Slack Socket 模式 |
| `@slack/web-api` | Slack Web API |
| `@anthropic-ai/sandbox-runtime` | 沙盒运行时 |

**关键设计**: 复用 pi-coding-agent 的工具系统。

---

### 2.6 @mariozechner/pi-pods

**内部依赖**:
| 依赖 | 关系 |
|------|------|
| `@mariozechner/pi-agent-core` | Agent 运行时 |

**外部依赖**:
| 依赖 | 用途 |
|------|------|
| `chalk` | 终端颜色 |

**关键设计**: 轻量级，仅依赖 Agent 核心。

---

### 2.7 @mariozechner/pi-web-ui

**内部依赖**:
| 依赖 | 关系 |
|------|------|
| `@mariozechner/pi-ai` | 类型定义 |
| `@mariozechner/pi-tui` | 共享类型 |

**外部依赖**:
| 依赖 | 用途 |
|------|------|
| `@mariozechner/mini-lit` | Lit 轻量版 |
| `lit` | Web Components 框架 |
| `@lmstudio/sdk` | LM Studio 集成 |
| `ollama` | Ollama 集成 |
| `docx-preview` | DOCX 预览 |
| `jszip` | ZIP 处理 |
| `pdfjs-dist` | PDF 渲染 |
| `xlsx` | Excel 处理 |
| `lucide` | 图标库 |

**Peer Dependencies**:
| 依赖 | 用途 |
|------|------|
| `@mariozechner/mini-lit` | 轻量 Lit 实现 |
| `lit` | Web Components |

---

## 三、依赖热力图

```
                    被依赖次数
    pi-ai              ████████████████████  6 次
    pi-agent-core      ██████████████        4 次
    pi-tui             ██████████            2 次
    pi-coding-agent    █████                 1 次
```

**分析**:
- `pi-ai` 是最核心依赖，被所有上层包使用
- `pi-agent-core` 是 Agent 功能的基石
- `pi-tui` 和 `pi-web-ui` 是独立的 UI 层
- `pi-coding-agent` 是唯一被其他包依赖的应用层

---

## 四、外部依赖分类

### 4.1 LLM SDK
| 包 | 用途 | 依赖包 |
|----|------|--------|
| `@anthropic-ai/sdk` | Anthropic API | pi-ai |
| `@aws-sdk/client-bedrock-runtime` | AWS Bedrock | pi-ai |
| `@google/genai` | Google Gemini | pi-ai |
| `@mistralai/mistralai` | Mistral | pi-ai |
| `openai` | OpenAI API | pi-ai |
| `@lmstudio/sdk` | LM Studio | web-ui |
| `ollama` | Ollama | web-ui |

### 4.2 类型验证
| 包 | 用途 | 依赖包 |
|----|------|--------|
| `@sinclair/typebox` | 运行时类型 | pi-ai |
| `zod-to-json-schema` | Schema 转换 | pi-ai |

### 4.3 终端/UI
| 包 | 用途 | 依赖包 |
|----|------|--------|
| `chalk` | 终端颜色 | 多个包 |
| `marked` | Markdown | tui, coding-agent |
| `get-east-asian-width` | 字符宽度 | tui |

### 4.4 文件/工具
| 包 | 用途 | 依赖包 |
|----|------|--------|
| `glob` | 文件匹配 | coding-agent |
| `ignore` | gitignore | coding-agent |
| `yaml` | YAML 解析 | coding-agent |
| `diff` | 差异计算 | coding-agent |

### 4.5 通信
| 包 | 用途 | 依赖包 |
|----|------|--------|
| `undici` | HTTP 客户端 | pi-ai |
| `proxy-agent` | 代理 | pi-ai |
| `@slack/socket-mode` | Slack | mom |
| `@slack/web-api` | Slack | mom |

### 4.6 文档处理 (Web UI)
| 包 | 用途 |
|----|------|
| `docx-preview` | Word 文档 |
| `pdfjs-dist` | PDF |
| `jszip` | ZIP |
| `xlsx` | Excel |

---

## 五、关键技术选型说明

### 5.1 TypeBox vs Zod

**选择 TypeBox 的原因**:
1. **性能**: TypeBox 编译为纯函数，比 Zod 快
2. **标准**: 输出标准 JSON Schema
3. **类型**: 更好的 TypeScript 推断

**使用位置**:
- 主要类型定义在 `pi-ai`
- 工具参数验证
- 配置验证

### 5.2 Lit for Web Components

**选择 Lit 的原因**:
1. **标准**: 基于 Web Components 标准
2. **轻量**: 体积小，性能好
3. **生态**: Google 维护，生态成熟

**版本策略**:
- 使用 `@mariozechner/mini-lit` 轻量版
- 兼容标准 `lit`

### 5.3 ES Modules

**选择 ES Modules 的原因**:
1. **未来**: 标准模块系统
2. **Tree Shaking**: 更好的打包优化
3. **顶层 await**: 更简洁的异步代码

**兼容性处理**:
- 目标 Node.js >= 20
- 使用 `.js` 扩展名配合 `"type": "module"`

### 5.4 npm Workspaces

**选择原因**:
1. **原生**: npm 内置，无需额外工具
2. **简单**: 配置简单，学习成本低
3. **统一**: 统一依赖管理

---

## 六、依赖风险分析

### 低风险 ✓
| 依赖 | 原因 |
|------|------|
| `chalk` | 成熟稳定，广泛使用的终端库 |
| `marked` | Markdown 解析标准库 |
| `glob` | Node.js 文件匹配标准 |

### 中风险 △
| 依赖 | 原因 | 缓解措施 |
|------|------|----------|
| `@sinclair/typebox` | 相对较新 | 版本锁定 |
| `@mariozechner/mini-lit` | 私人维护 | 备用标准 lit |
| `koffi` | 可选依赖 | 功能降级 |

### 需要关注 ⚠
| 依赖 | 原因 | 建议 |
|------|------|------|
| LLM SDKs | API 变更频繁 | 抽象层隔离 |
| `@anthropic-ai/sandbox-runtime` | 实验性 | 监控更新 |

---

## 七、依赖优化建议

### 7.1 减少重复依赖

**现状**: `chalk` 被多个包重复依赖

**建议**: 考虑通过 pi-ai 或 pi-agent-core 暴露日志/颜色工具

### 7.2 可选依赖优化

**现状**: `koffi` 是 pi-tui 的可选依赖

**建议**: 明确文档化可选依赖的功能降级行为

### 7.3 版本统一

**现状**: 各包可能有不同的间接依赖版本

**建议**: 在根 package.json 使用 `overrides` 统一关键依赖版本

---

## 八、依赖安装统计

| 包 | 生产依赖 | 开发依赖 | 总大小 |
|----|----------|----------|--------|
| pi-ai | ~15 | 5 | ~50MB |
| pi-agent | 1 | 3 | ~5MB |
| pi-tui | 5 | 3 | ~10MB |
| pi-coding-agent | 10+ | 5 | ~30MB |
| pi-web-ui | 15+ | 5 | ~40MB |
| pi-mom | 3 | 3 | ~15MB |
| pi-pods | 2 | 3 | ~5MB |

**总计**: 生产依赖 ~50 个，开发依赖 ~20 个

---

*如需查看完整依赖树，运行 `npm ls` 或 `npm ls --prod`*
