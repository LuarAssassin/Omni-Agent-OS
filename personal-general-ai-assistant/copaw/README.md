# CoPaw 项目文档索引

> 分析时间: 2026-03-17
> 项目路径: `/Users/luarassassin/reference-projects-me/CoPaw`
> Python 版本: >=3.10,<3.14

---

## 文档列表

| 文档 | 描述 |
|------|------|
| **[copaw-analysis.md](./copaw-analysis.md)** | 项目深度分析报告 - 包含架构设计、代码质量、设计哲学分析 |
| **[copaw-dependencies.md](./copaw-dependencies.md)** | 依赖分析 - Python/TypeScript 依赖详解 |
| **[copaw-code-index.md](./copaw-code-index.md)** | 代码导航索引 - 核心文件快速定位 |
| **[copaw-adr.md](./copaw-adr.md)** | 架构决策记录 - 关键设计决策说明 |

---

## 项目概览

**CoPaw** 是一个基于 Python 的个人 AI 助手，基于 AgentScope 框架构建，支持多通道通信（钉钉、飞书、QQ、Discord、iMessage 等）和定时任务调度。

### 核心特性

| 特性 | 描述 |
|------|------|
| **多通道支持** | 支持 15+ 消息平台（DingTalk、Feishu、QQ、Discord、Telegram、iMessage、Mattermost、MQTT、Matrix、Voice 等） |
| **多代理架构** | 每个代理拥有独立的 Workspace，包含 Runner、ChannelManager、MemoryManager、MCPManager、CronManager |
| **MCP 支持** | 原生支持 Model Context Protocol，可接入丰富的工具生态 |
| **技能系统** | 基于 Markdown 的层级化技能管理，支持自定义技能 |
| **Web 控制台** | React + TypeScript 构建的现代化管理界面 |
| **FastAPI 后端** | 高性能异步 API 服务 |
| **定时任务** | 基于 APScheduler 的 Cron 任务调度 |

### 项目统计

| 指标 | 数值 |
|------|------|
| Python 源文件数 | 270+ 个 |
| TypeScript/TSX 文件数 | 159+ 个 |
| 直接依赖数 | 35+ 个 |

---

## 快速导航

### 核心模块

```
CoPaw/src/copaw/
├── agents/          # 代理核心（ReActAgent、MemoryManager、Skills）
├── app/             # 应用层（Workspace、MultiAgentManager、Channels）
├── cli/             # 命令行接口（Click 框架）
├── config/          # 配置管理（Pydantic 模型）
├── providers/       # LLM 提供商管理（OpenAI、Anthropic、Gemini 等）
└── security/        # 安全模块（ToolGuard、SkillScanner）
```

### 前端控制台

```
CoPaw/console/src/
├── api/             # API 客户端
├── components/      # React 组件
├── pages/           # 页面组件
└── utils/           # 工具函数
```

---

## 设计哲学分析要点

基于 John Ousterhout《A Philosophy of Software Design》的核心理念，本分析重点关注：

1. **模块深度分析** - 识别深模块与浅模块
2. **信息隐藏** - 评估接口设计与信息泄露
3. **耦合与内聚** - 分析模块间依赖关系
4. **错误处理** - "Define errors out of existence" 实践
5. **命名与注释** - 代码可读性评估

详见 [copaw-analysis.md](./copaw-analysis.md) 第 XI 章节。

---

*文档生成完成。如需深入分析特定模块，请参阅具体文档。*
