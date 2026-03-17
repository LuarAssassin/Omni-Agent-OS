# Codex 关键代码索引

> 快速导航到项目中的重要代码文件

---

## 入口与 CLI

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 主入口 | `codex-rs/cli/src/main.rs` | MultitoolCli、子命令分发 |
| Exec 子命令 | `codex-rs/exec/` | 非交互执行 |
| Login 子命令 | `codex-rs/cli/src/login` | 认证流程 |
| MCP 子命令 | `codex-rs/cli/src/mcp_cmd.rs` | MCP 管理 |

---

## 核心引擎 (core/)

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 库入口 | `codex-rs/core/src/lib.rs` | 模块导出 |
| CodexThread | `codex-rs/core/src/codex_thread.rs` | 主线程 |
| Agent | `codex-rs/core/src/agent.rs` | Agent 逻辑 |
| 配置 | `codex-rs/core/src/config/` | Config 结构 |
| 配置加载 | `codex-rs/core/src/config_loader/` | 加载逻辑 |
| 执行器 | `codex-rs/core/src/exec/` | 执行相关 |
| 工具注册表 | `codex-rs/core/src/tools/registry.rs` | ToolRegistry |
| 工具规范 | `codex-rs/core/src/tools/spec.rs` | 工具 JSON Schema |
| 工具路由 | `codex-rs/core/src/tools/router.rs` | ToolRouter |
| Shell 工具 | `codex-rs/core/src/tools/handlers/shell.rs` | ShellHandler |
| 统一执行 | `codex-rs/core/src/tools/handlers/unified_exec.rs` | UnifiedExecHandler |
| MCP 连接 | `codex-rs/core/src/mcp_connection_manager.rs` | MCP 管理 |
| 沙箱 | `codex-rs/core/src/sandboxing/` | 沙箱抽象 |
| Windows 沙箱 | `codex-rs/core/src/windows_sandbox.rs` | Windows 配置 |

---

## 协议

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 协议定义 | `codex-rs/protocol/src/protocol.rs` | Op, EventMsg |
| 协议类型 | `codex-rs/protocol/src/` | 各子模块 |

---

## TUI

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| TUI 主入口 | `codex-rs/tui/` | ratatui 应用 |
| App Server | `codex-rs/tui_app_server/` | JSON-RPC 服务 |

---

## 执行与沙箱

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 执行引擎 | `codex-rs/exec/` | 执行流程 |
| 执行策略 | `codex-rs/execpolicy/` | Starlark 策略 |
| Linux 沙箱 | `codex-rs/linux-sandbox/` | Landlock |
| Windows 沙箱 | `codex-rs/windows-sandbox-rs/` | Windows Sandbox |

---

## MCP

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| MCP 服务器 | `codex-rs/mcp-server/` | stdio MCP |
| RMCP 客户端 | `codex-rs/rmcp-client/` | RMCP 协议 |

---

## 技能系统

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 技能管理 | `codex-rs/skills/` | 技能加载、BM25 |
| 示例技能 | `codex-rs/skills/src/assets/samples/` | SKILL.md 示例 |

---

## 认证与状态

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 登录 | `codex-rs/login/` | ChatGPT 登录 |
| 状态存储 | `codex-rs/state/` | SQLite |

---

## 应用服务器

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| App Server | `codex-rs/app-server/` | 协议实现 |
| 协议类型 | `codex-rs/app-server-protocol/` | 共享类型 |
| 客户端 | `codex-rs/app-server-client/` | 客户端 |

---

## SDK

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| TypeScript SDK | `sdk/typescript/src/` | codex.ts, exec.ts, thread.ts |
| Python SDK | `sdk/python/src/codex_app_server/` | JSON-RPC 客户端 |

---

## 工具函数

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| Git | `codex-rs/utils/git/` | Git 操作 |
| PTY | `codex-rs/utils/pty/` | 伪终端 |
| 流解析 | `codex-rs/utils/stream-parser/` | 流解析 |
| CLI 工具 | `codex-rs/utils/cli/` | 配置覆盖 |

---

## 文档

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 协议 v1 | `codex-rs/docs/protocol_v1.md` | 协议说明 |
| 配置 | `docs/config.md` | 配置文档 |
| 安装 | `docs/install.md` | 安装说明 |
| 执行 | `docs/exec.md` | 执行模式 |

---

## 测试文件索引

### 核心测试
- `codex-rs/core/tests/` - Core 测试
- `codex-rs/core/tests/common/` - 测试支持

### App Server 测试
- `codex-rs/app-server/tests/` - App Server 测试

### MCP 测试
- `codex-rs/mcp-server/tests/` - MCP 测试

### SDK 测试
- `sdk/typescript/tests/` - TypeScript 测试
- `sdk/python/tests/` - Python 测试
