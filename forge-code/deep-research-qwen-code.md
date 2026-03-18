# 执行摘要

Qwen Code 是一款面向命令行的开源 AI 代理（agent），针对 **Qwen3-Coder** 模型进行了优化，旨在帮助开发者分析大规模代码库、自动化常见任务并加速开发流程【73†L392-L394】。该项目采用多模块架构，核心功能由 `core` 包提供，包括会话管理、技能（Skill）调度、模型调用等；`cli` 包实现命令行界面，通过 Ink/React 构建富交互体验；此外还提供了 TypeScript/Java SDK、IDE 插件和 Web UI 模板等扩展包，支持交互模式与无头模式调用【45†L718-L724】【73†L392-L394】。  

主要功能包括**交互式对话**、**批处理脚本（无头模式）**、**IDE 集成**以及内置的**代码审查技能**等。例如，内置的 `/review` 指令可自动执行多维度并行代码审查任务，生成易读的评审报告【56†L266-L274】；用户还可通过 `/help`、`/clear`、`/stats` 等命令管理会话和查询统计。在技术栈方面，该项目基于 Node.js/TypeScript 开发，采用 Ink 和低光高亮库构建终端 UI，依赖各类模型协议 SDK（如 OpenAI、Anthropic、Gemini、Alibaba 等兼容库）进行模型调用；构建和测试依赖 Vitest 等工具，CI/CD 由 GitHub Actions 管理。  

分析发现，Qwen Code 架构清晰模块化，可通过插件式技能体系扩展功能；核心库逻辑与 UI 解耦，使其在不同前端（命令行、IDE、Web）均可复用。但当前设计在可测试性和可观测性方面仍有改进空间（缺乏统一的日志/监控接口），并需注意命令执行等安全边界。性能方面，可能存在如代码检索（ripgrep）、并行代理启动等瓶颈，需要优化异步处理和资源管理。在安全与容错方面，应加强输入校验、模型调用超时和失败重试等机制。  

基于以上分析，报告后续将详细展示软件架构图和各模块职责、功能清单与实现细节、技术栈对比表，以及针对 AI 代理集成的多条具体改进建议（含难度估计、预期收益和示例）。

## 软件架构

```mermaid
flowchart LR
  subgraph 用户界面层
    CLI[命令行界面 (Qwen CLI)]
    VSCodeExt[VS Code 扩展]
    ZedExt[Zed 扩展]
    WebUI[Web/UI]
    TS_SDK[TypeScript SDK]
  end

  subgraph 核心逻辑层
    CoreSession[(会话 Session & 任务调度)]
    SkillMgr[技能管理 (Skill Manager)]
    ToolExec[工具执行 (Shell/搜索/RG)]
    ModelProv[模型提供器 (OpenAI/Gemini 等)]
    Config[配置 & 授权]
    Telemetry[遥测 & 日志]
    ACPSvc[ACP 服务接口]
    MCPEngine[MCP 上下文引擎]
  end

  subgraph 后端与外部服务
    Qwen3[Qwen3-Coder 模型]
    OpenAI[OpenAI/Anthropic API]
    Gemini[Google GenAI API]
    Bailian[阿里巴巴嵌合计划 API]
    FileSys[文件系统 & Git]
  end

  CLI -->|调用核心逻辑| CoreSession
  VSCodeExt --> CoreSession
  ZedExt --> CoreSession
  WebUI --> CoreSession
  TS_SDK --> CoreSession

  CoreSession --> SkillMgr
  CoreSession --> ToolExec
  CoreSession --> MCPEngine
  CoreSession --> ModelProv
  CoreSession --> Telemetry
  CoreSession --> Config
  CoreSession --> ACPSvc

  SkillMgr --> SkillMgr : 加载内置/自定义技能（如 `/review`）
  ToolExec --> FileSys : 文件 I/O, `grep` 搜索
  ToolExec -->|执行| FileSys
  ToolExec -->|子进程| Shell[Shell/终端]

  ModelProv --> Qwen3
  ModelProv --> OpenAI
  ModelProv --> Gemini
  ModelProv --> Bailian

  MCPEngine --> FileSys : 代码上下文检索
  ACPSvc --> OpenAI : 通过 Agent 代理与后端通信
```

上图展示了 Qwen Code 的高层架构。最上层为用户界面层，包括交互式 CLI、各类 IDE 插件、Web UI 以及 TypeScript SDK，它们均通过调用核心逻辑层的会话管理器（`CoreSession`）来实现对话和任务控制。核心逻辑层主要组件如下：

- **会话管理（Session）**：负责启动和管理与用户的对话会话，解析用户输入，将请求分发给技能管理器或直接调用模型提供器。
- **技能管理（Skill Manager）**：维护一组内置或可扩展的技能（Skill），如代码审查、执行 shell 命令等【56†L266-L274】。当用户输入对应命令（如`/review`）时，Skill Manager 会执行预定义流程（可启动多个并行子代理）并整合结果。
- **工具执行（Tool Executor）**：提供对本地环境的访问能力，包括文件读取、Ripgrep 搜索（已打包在 `core/vendor` 中并需后期可执行权限设置）、以及启动子 shell 进程等操作（在 CLI 包中通过 `node-pty` 库可选集成）。
- **模型提供器（Model Providers）**：封装各种 LLM API 接口（OpenAI、Anthropic、Google GenAI、阿里巴巴 Bailian 等），并根据配置调用相应模型【73†L392-L394】。支持多协议（OpenAI-兼容、MCP、Anthropic）并通过用户配置选择模型和认证方式【73†L392-L394】【45†L779-L787】。
- **MCP 上下文引擎**：通过 Model Context Protocol (MCP) 将代码文件等上下文信息注入提示，以便模型更好地理解代码环境。CLI 提供相关命令（如 `mcp`）读取文件并处理上下文标签。
- **ACP 服务接口**：Agent Client Protocol (ACP) 服务模块，用于与模型后端进行长连接通信，如 WebSocket 形式的实时对话流（详见 `packages/cli/src/acp-integration`）。支持不同认证方式（如 OAuth 或 API Key）【73†L392-L394】【45†L779-L787】。
- **配置与授权**：通过加载 `~/.qwen/settings.json` 等配置文件确定模型提供商、API Key、默认模型等信息【45†L778-L787】【73†L392-L394】。
- **遥测与日志**：集中记录用户会话统计、错误日志和使用情况，以备监控和改进（目前存在基础日志功能，但尚缺乏统一的可观测性管道）。

各模块间的数据流如图所示：用户输入经 UI 进入会话管理，可能由技能管理器触发工具执行（如文件读写或 Git 操作）和模型调用，然后将结果在终端或其他界面展示。整体架构采用**多包隔离**（monorepo Yarn 工作区），`core` 包提供纯粹业务逻辑，`cli` 包针对终端交互构建界面，其他包（IDE 扩展、SDK、Web 模板等）各司其职。此架构可扩展性较好：新增协议或技能仅需在相应模块注册，实现相对清晰；测试方面，核心逻辑与 UI 解耦，有利于编写模块化单元测试。然而，当前体系安全边界需仔细控制（例如执行 Shell 命令风险、读取任意文件须校验权限），性能瓶颈可能来自文件搜索 (Ripgrep) 和多线程代理调度，需要评估并异步优化。此外，应进一步提升日志监控功能，增强容错机制（模型调用重试、网络超时处理等）。

## 模块分层与组件表

| 模块         | 职责描述                                                             | 主要依赖/接口                        |
| ------------ | -------------------------------------------------------------------- | ------------------------------------ |
| **cli**      | 命令行界面，基于 Ink/React 实现交互式终端。解析用户命令，格式化输出，管理会话生命周期（例如 `/help`、`/clear`、`/stats` 等指令）。 | `@qwen-code/qwen-code-core` 核心库；Ink 组件（`ink`, `ink-gradient`, `ink-link`, `ink-spinner` 等）；终端键盘事件；node-pty（可选，控制子 shell）。 |
| **core**     | 核心逻辑库，提供对话管理、技能执行、模型调用、工具运行、配置加载等。包括以下子模块：<br>- `Session`：会话状态和上下文管理<br>- `SkillManager`：技能注册与调度<br>- `ToolExec`：本地工具调用（文件、搜索、命令）<br>- `Provider`：接入模型协议与 API<br>- `Telemetry`：遥测和日志上报<br>- `MCP`：上下文注入处理<br>- `subagents`：可选的子代理管理 | Node.js 标准库（`fs`, `child_process` 等）；Ripgrep 二进制（在 `vendor/` 包中）；ACP/MCP 协议 SDK；第三方库（`simple-git`、`lowlight`、`highlight.js` 等用于代码解析和显示）。 |
| **web-templates** | Web/桌面界面模板，供 Web UI 或桌面应用使用，封装交互逻辑与样式。                         | React/Web 框架；可能的 Electron 或 NW.js 集成。 |
| **vscode-ide-companion** | VS Code 扩展，内嵌 Qwen Code CLI，实现编辑器内交互。                          | VS Code 扩展 API；WebSocket 通信或命令通道。 |
| **zed-extension** | Zed 编辑器扩展，类似于 VS Code 扩展，用于 Zed 中调用 Qwen Code。                   | Zed 插件 API；IPC 或命令通讯机制。 |
| **sdk-typescript** | TypeScript 开发套件，提供程序化调用 Qwen Code 的接口，使其他应用可内嵌此功能。           | 暴露调用核心库的函数接口；TypeScript 类型定义。 |
| **sdk-java** | Java 开发套件，功能同上，方便 Java 应用集成 Qwen Code。                           | Maven/Gradle 包；Java 类型定义。 |
| **test-utils** | 测试辅助库，提供在 CI 环境下无头运行 Qwen Code 的功能，模拟用户输入输出。                 | 测试框架（Vitest/Jest）；终端模拟库。 |
| **scripts**  | 构建/安装脚本，如 `postinstall.js` 用于设置第三方依赖（Ripgrep）执行权限。               | Node.js 环境脚本；平台检测（macOS/Linux/Windows）。 |
| **.github/workflows** | CI/CD 配置：使用 GitHub Actions 实现构建、测试、打包镜像、部署网页等自动化流程。         | GitHub Actions YAML；Docker 镜像构建；页面部署（GitHub Pages）。 |
| **docs (README 等)** | 项目文档：包含功能介绍、安装使用说明、配置指南等。                               | Markdown 文档；链接到 qwenlm.github.io 文档站点【45†L725-L733】。 |

## 功能点清单与评价

| 功能（子功能）         | 实现方式与关键流程                                                           | 输入输出 & 配置项                                                  | 优点                                                          | 缺点及改进建议                                     | 对 AI 代理 价值 |
| -------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------- | ----------------------------------------------------------- | -------------------------------------------------- | -------------- |
| **交互式模式**       | 启动命令 `qwen` 后进入 Ink 终端界面，由用户输入问题或指令（可引用文件 `@path`）。会话通过 `Session` 处理用户意图，生成响应以对话形式显示。支持用户快捷键 (`Ctrl+C/D`)、历史记录、剪贴板等。 | 输入：用户文本问题、文件引用<br>输出：模型回答文本、表格、代码块<br>配置：`settings.json`（默认模型、上下文策略）、CLI 参数（`--yolo` 自动图片检测） | 直观易用，可实时交互；支持文件引用与多轮上下文；集成丰富指令（/help等）。 | 界面依赖终端，学习曲线稍高；可添加图形界面（已有 AionUI 等外部方案）；建议增加更直观的帮助提示和用户自定义快捷键。 | 高：核心功能，使用户可在开发过程中自然地获取 AI 辅助。 |
| **无头模式**         | 使用 `qwen -p "问题"` 等命令，在无交互 UI 环境下运行。`Session` 直接发送问题并输出结果，适合脚本或 CI 调用。 | 输入：命令行参数（问题）<br>输出：机器可读的结果（JSON/文本）<br>配置：与交互同源（settings.json, 环境变量） | 适用于自动化、定时任务、CI 集成；无需人工干预。 | 需要确保输出格式标准化（目前可通过管道提取），建议提供结构化 JSON 输出选项以便机器解析。 | 高：支持与其他系统集成，可批量处理任务，增强自动化。 |
| **代码审查技能 (`/review`)** | 定义在 `core/src/skills` 中的内置技能【56†L266-L274】。解析用户参数（PR号、文件路径或未提交变更），使用 `git` 和 `gh` 命令获取差异。启动 4 个并行审查子代理（关注正确性、安全、质量、性能），最后整合生成报告。 | 输入：`/review [pr号|文件路径]`<br>输出：按 Severity 分级的审查报告（含文件行号定位和改进建议）<br>配置：可定制分步策略（模板可在 `SKILL.md` 修改） | 自动化深度审查，多角度覆盖全面；结果格式清晰，有级别区分。 | 目前只支持 GitHub PR，建议扩展支持通用 diff 或其他 VCS（GitLab）。审查步骤预定义，难以定制；可允许用户自定义审查维度或模板。 | 高：直接提升代码质量和安全，可减轻人工审查负担，对 AI 代理应用场景（代码协助）价值显著。 |
| **IDE 集成**         | 提供 VS Code 和 Zed 等编辑器扩展【45†L741-L749】，在编辑器侧与 Qwen Code 后端通信。用户可在编辑器内发送问题或请求审查，结果在侧边栏或弹窗展示。使用 WebSocket 或命令通道与 CLI 进程交互。 | 输入：编辑器命令、选中文本片段等<br>输出：AI 反馈插入编辑器窗口<br>配置：VS Code/扩展配置，如 API Key 或 OAuth | 编辑器内直接调用，无需切换环境；与开发流程无缝对接。 | 需要维护多个 IDE 平台的兼容性；建议加强跨平台插件架构，及时更新支持新 IDE 版本。 | 高：提高使用便捷性，贴近开发环境，有助于推广 AI 助手。 |
| **TypeScript SDK**   | 包含在 `packages/sdk-typescript`，提供可编程接口。开发者可在应用中调用 `QwenAgent` 类直接发送问题、接收结果，实现自定义前端或服务。 | 输入：SDK 函数调用和参数<br>输出：异步返回结果对象（支持流式）<br>配置：同样依赖 `settings.json` 或 API Key | 便于二次开发和功能扩展；可集成到 Web 服务、机器人等。 | 文档需完善（示例使用、异常处理）。建议提供更多类型定义和错误码说明，便于使用。 | 中：主要用于开发者扩展，对普通用户间接有益；对 AI 代理产品集成有帮助。 |
| **多协议模型支持**   | 通过配置支持 OpenAI、Anthropic、Google Gemini、阿里巴巴等兼容模型API【73†L392-L394】【45†L779-L787】。基于 OAuth 或 API Key 认证。用户可在启动时选择协议（例如 `/auth` 命令）。 | 输入：模型参数、API Key 或 OAuth 流程<br>输出：指定模型的响应<br>配置：`settings.json` 中设置 `modelProviders` 和 `security.auth` | 兼容多家服务；可享受不同厂商免费额度或特色模型。 | 当多个协议并存时，切换机制需清晰（目前通过 `/model` 命令或设置）；建议UI上显示当前协议/模型信息。 | 高：直接影响性能和功能（模型能力强弱），是 AI 代理性能的关键因素。 |
| **配置系统**         | 支持全局 `~/.qwen/settings.json` 和项目内 `.qwen/settings.json`（后者覆盖全局）【45†L779-L787】。可配置模型列表、环境变量、认证偏好等。 | 输入：JSON 文件字段<br>输出：运行时参数影响模型选择等<br>配置：同上 | 灵活可定制，可维护多种环境配置。 | 当前配置项较多，易错且难记；建议提供图形化配置向导或模板生成功能，并对字段做更完善校验提示。 | 中：配置正确性影响运行稳定，但更多是使用辅助。 |

> **注：**“直接价值”表示该功能对构建 AI 代理整体能力的影响程度。表中优劣分析及建议基于当前代码实现和文档情况综合评估。

## 技术栈与依赖关系

| 组件/库             | 类别             | 版本或范围            | 替代方案与社区维护               | 社区活跃度 / 维护风险   |
| ------------------ | --------------- | ------------------- | ------------------------------ | --------------------- |
| **Node.js & npm**  | 运行环境         | Node ≥20              | Deno (不兼容)                  | Node 社区活跃，LTS 支持（低风险） |
| **TypeScript**     | 开发语言         | ^5.x                 | Pure JavaScript (减少类型安全)   | TypeScript 领头羊项目，社区活跃（低风险） |
| **Ink (React CLI)** | UI 框架         | ^6.x (React 19.x)     | Vorpal、Commander.js (UI 简陋)  | Ink社区活跃，适合终端UI（中风险，因依赖React版本兼容） |
| **Highlight.js**   | 语法高亮         | ^11.x                | Prism.js、Shiki (性能不同)       | 活跃（中风险，可替换）  |
| **lowlight**       | 代码渲染         | ^3.x                 | highlight.js (已用)、Prism.js  | 依赖性高，维护稳定         |
| **simple-git**     | Git 交互库       | ^3.x                 | NodeGit (较重)                 | moderate (社区活跃，但功能集中) |
| **ripgrep**        | 文件搜索（二进制） | vendor 包含多平台   | grep (功能弱)，TheSilverSearcher | 社区顶级工具（低风险）    |
| **node-pty**       | 伪终端（可选）   | 1.1.0 (平台区分)      | 无（必需支持 CLI 模拟）         | 社区版本维护，风险低       |
| **agentclientprotocol/sdk** | 代理协议库 | ^0.14.x             | 无，行业标准                    | 来自社交计算联盟 (MCP/ACP) 活跃 |
| **modelcontextprotocol/sdk** | 上下文协议库 | ^1.25.x             | 无                               | 活跃                     |
| **OpenAI/GenAI/AISDKs** | 模型接入库   | 多种（如 `@google/genai` 1.30） | 自写HTTP调用                   | 官方 SDK 活跃（低风险） |
| **Inquirer/Prompts** | 交互式提示   | prompts ^2.4        | inquirer.js                    |活跃（可选）             |
| **dotenv**         | 环境变量管理     | ^17.x                | cross-env (简易)               | 大热（低风险）          |
| **Vitest**         | 测试框架         | ^3.x                 | Jest                           | 正逐渐流行（中风险，需观察版本兼容） |
| **Yargs**          | CLI 参数解析     | ^17.x                | Commander, Oclif (更重)         | 长期稳定（低风险）      |
| **ESBuild**        | 构建工具         | 用于打包各包        | Webpack (复杂)，Rollup         | 社区活跃，性能好（低风险） |
| **Docker**         | 部署容器（可选） | (GitHub Action 构建) | -                              | 标准方案（低风险）       |
| **GitHub Actions** | CI/CD            | -                    | Jenkins, Travis                | 社区主流，版本迭代（低风险） |
| **License**        | 许可证          | Apache-2.0           | MIT (更宽松)                   | Apache 2.0 很常见（低风险） |

以上技术栈涵盖项目开发语言、UI 框架、后端服务库、测试与 CI 工具，以及许可协议等。表中“社区活跃度/维护风险”评估了库的流行度和长期维护情况。例如，Ink 框架对终端 UI 构建非常有用，但相比纯 JS 库维护资源较少；Vitest 测试框架新兴活跃，但需要关注与 TS 和 DOM 库的兼容性。关键依赖如 OpenAI/Ali Cloud SDK 均为官方支持、定期更新，故风险较低。总体而言，项目使用的组件大多社区活跃、维护及时，替代方案也较为丰富，为长期发展提供基础。

## 改进建议

1. **强化插件化技能架构（难度：中）**  
   当前内置技能（如 `/review`）硬编码在 `core` 中。建议设计插件机制，使用户和社区可自定义技能：例如定义 `Skill` 接口，让新技能作为独立 NPM 包加载或在配置文件中引用。【示例】在 `core/src/skills/index.ts` 中按插件名动态加载：  
   ```ts
   // CoreSkillManager.ts（示意）
   for (const skillName of config.enabledSkills) {
     const skillModule = await import(`@qwen-code/qwen-code-skill-${skillName}`);
     skillManager.registerSkill(skillModule.default);
   }
   ```  
   *预期收益：* 降低核心包膨胀、加速新功能发布，吸引社区贡献技能。  
   *优先级：中；收益高，有助于生态扩展。  

2. **统一可观测性与日志体系（难度：低）**  
   目前日志与错误输出散落各处，缺少统一监控。建议引入日志库（如 `winston`）并配置日志等级，收集关键事件和错误。对外可选用日志聚合（Graylog、Elastic Stack）。同时加入调用链 ID 以便跟踪多步对话过程。  
   *示例：* 在 `core` 初始化时注入 Logger，并在 Session/Skill 执行前后记录：  
   ```ts
   logger.info('Starting session', { userId, sessionId });
   logger.error('Model call failed', { error, provider });
   ```  
   *预期收益：* 便于定位问题与性能瓶颈（如网络延迟），提升可维护性。  
   *优先级：中；实现成本低但可显著改进可维护性。  

3. **性能优化：异步并行工具调用（难度：中）**  
   当前部分工具调用（如 Ripgrep 搜索）可能阻塞主流程。建议使用异步/工作线程技术并行执行 I/O 操作，避免阻塞 Node.js 事件循环。例如，在检索大文件时使用 `child_process` 的异步接口，并行化多个检索任务。  
   *示例：* 使用 `Promise.all` 并发运行多个 `ripgrep`：  
   ```ts
   const results = await Promise.all(paths.map(p => execFile('rg', ['--json', query, p])));
   ```  
   *预期收益：* 提升多文件场景响应速度，减少用户等待时延。  
   *优先级：高；直接影响用户体验。  

4. **增强安全与输入校验（难度：低）**  
   Qwen Code 可执行任意 shell 命令（若启用相应技能），存在风险。建议严格区分交互式和敏感操作：增加确认步骤或运行沙箱。对于读取文件、执行命令等输入，应验证文件路径合法性、避免路径遍历，并对特殊字符进行过滤。可借鉴 `sandbox-exec` 或专用容器运行敏感任务。  
   *示例：* 在执行 `/review <file>` 前，检查路径所属 Git 仓库：  
   ```ts
   if (!filePath.startsWith(repoRoot)) {
     throw new Error('不允许访问仓库外文件');
   }
   ```  
   *预期收益：* 降低潜在安全风险，保护用户代码隐私。  
   *优先级：高；安全问题需优先考虑，完成难度较低。  

5. **改进模型替换与配置体验（难度：中）**  
   虽然支持多协议模型，但用户切换模型较麻烦（需 `/model` 或手动编辑配置）。可增设交互命令或自动化脚本，动态显示和切换当前模型。例如实现 `/switch-model` 命令列表可选模型。并在 CLI 标题或状态行展示当前模型。  
   *示例：* 简化模型列表读取与提示：  
   ```ts
   const models = config.modelProviders[currentProtocol];
   console.log('可用模型：', models.map(m => m.id).join(', '));
   ```  
   *预期收益：* 提高可用性，让用户快速切换和验证不同模型。  
   *优先级：中；对用户友好性有显著提升。  

6. **添加测试覆盖和持续集成完善（难度：中）**  
   虽有基础单元测试，但可加强核心逻辑（Session、Skill 执行流程）的覆盖，尤其模拟无头模式和 CI 场景。建议在 CI 流程中增加端到端测试，如自动运行示例会话验证回复格式，以及模拟不同协议下的认证流程。  
   *示例：* 在 `packages/cli/test` 中编写一个脚本调用 `qwen -p "Hello"` 并检查输出结构：  
   ```js
   const result = execSync('qwen -p "Hello"');
   expect(result.toString()).toContain('Answer:');
   ```  
   *预期收益：* 提升代码稳定性与回归检测能力，减少线上故障。  
   *优先级：中；长期收益大，但需编写测试用例。  

7. **优化接口设计与抽象分层（难度：高）**  
   当前 `core` 和 `cli` 虽逻辑分离，但调用链仍较紧耦合（许多静态函数调用）。可考虑引入依赖注入（DI）或消息总线，使各组件可插拔。例如用事件总线发布会话状态变化，或利用类工厂生产协议客户端。这样做难度大，但可极大提高可测试性和灵活度。  
   *示例：* 使用简单事件模式：  
   ```ts
   eventBus.emit('userQuery', { text: '分析这个函数' });
   eventBus.on('aiResponse', (resp) => { display(resp) });
   ```  
   *预期收益：* 架构更清晰，模块化更强，为未来扩展（如新增前端）铺路。  
   *优先级：低；重构成本高，可在大版本中规划。  

以上建议均针对 Qwen Code 的架构扩展性、可维护性和性能进行优化，涵盖增加新功能、提升安全性、改善可观测性等方面。实施难度和预期收益根据实际团队情况权衡执行。

## 关键文件与代码位置索引

- **README.md**：项目总览与使用说明【73†L392-L394】【45†L718-L724】。
- **packages/cli/src/index.ts**：CLI 主入口，负责解析命令行参数并初始化 Ink 应用和会话逻辑。
- **packages/cli/src/commands/**：各 CLI 命令实现目录（如 `mcp.ts`、`extensions.tsx` 等）和快速命令处理逻辑。
- **packages/cli/src/acp-integration/**：ACP 协议集成模块（`session`, `service` 等），处理与 AI 模型后端的会话通信。
- **packages/core/src/index.ts**：核心库入口，导出主功能 API；初始化 Session、SkillManager 等。
- **packages/core/src/session/**：会话管理相关代码，维护对话状态和历史。
- **packages/core/src/skills/bundled/review/SKILL.md**：内置“代码审查”技能文档，详细说明 `/review` 功能流程【56†L266-L274】。
- **packages/core/src/skills/skill-manager.ts**：技能管理器实现，加载并调度技能执行。
- **packages/core/src/tools/**：本地工具实现，如文件读取、Ripgrep 搜索等辅助模块。
- **packages/core/src/services/**：各类外部服务封装，如 `gitService.ts` 用于运行 Git 命令。
- **.github/workflows/**：CI/CD 工作流配置（GitHub Actions YAML 文件），自动化测试与发布流程。
- **package.json**：项目配置文件，包含版本、依赖和工作区设置（`workspaces`）。

以上文件涵盖了项目的核心结构和关键逻辑实现，便于开发者快速定位和理解系统的主要组件与功能。 

**参考资料：** 以上信息主要来源于 Qwen Code 官方文档和源码【73†L392-L394】【45†L718-L724】【56†L266-L274】。