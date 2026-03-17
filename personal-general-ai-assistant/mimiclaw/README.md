# Mimiclaw 项目文档

Mimiclaw 是一个基于 ESP32-S3 的嵌入式 AI Agent 固件项目，实现了一个完整的 ReAct（Reasoning + Acting）智能体系统，支持多通道交互（Telegram、Feishu）、工具调用和自主任务调度。

## 文档索引

| 文档 | 说明 |
|------|------|
| [mimiclaw-analysis.md](./mimiclaw-analysis.md) |  comprehensive code analysis following software design philosophy principles |
| [mimiclaw-code-index.md](./mimiclaw-code-index.md) | Source file navigation and module organization |
| [mimiclaw-dependencies.md](./mimiclaw-dependencies.md) | External dependencies and component relationships |
| [mimiclaw-adr.md](./mimiclaw-adr.md) | Architecture Decision Records |

## 项目概述

### 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                        Application Layer                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Telegram  │  │   Feishu    │  │      Cron Jobs      │  │
│  │    Bot      │  │   WebSocket │  │   (Scheduled Tasks) │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
└─────────┼────────────────┼────────────────────┼─────────────┘
          │                │                    │
          └────────────────┴────────────────────┘
                           │
                    ┌──────▼──────┐
                    │ Message Bus │  ← FreeRTOS xQueue
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────▼─────┐    ┌────▼────┐     ┌─────▼──────┐
    │  Agent    │    │  Tool   │     │   System   │
    │   Loop    │◄──►│ Registry│     │  Services  │
    │ (ReAct)   │    │         │     │            │
    └─────┬─────┘    └────┬────┘     └────────────┘
          │               │
          │    ┌──────────┼──────────┬──────────┬──────────┐
          │    │          │          │          │          │
          │  ┌─▼─┐    ┌──▼──┐   ┌───▼───┐  ┌──▼──┐   ┌───▼───┐
          └──►LLM│    │GPIO │   │ Files │  │Cron │   │ Web   │
               │  │    │     │   │       │  │     │   │Search │
               └──┘    └─────┘   └───────┘  └─────┘   └───────┘
```

### 技术栈

- **硬件平台**: ESP32-S3 (Dual-core Xtensa LX7, 512KB SRAM + 8MB PSRAM)
- **RTOS**: FreeRTOS (双核任务调度)
- **开发框架**: ESP-IDF v5.x
- **网络协议**: WiFi STA 模式, HTTP/HTTPS, WebSocket
- **存储**: SPIFFS (SPI Flash File System), NVS (Non-Volatile Storage)
- **JSON 处理**: cJSON
- **HTTP 客户端**: esp_http_client + esp_https_ota
- **代理支持**: HTTP CONNECT, SOCKS5

### 关键特性

| 特性 | 描述 |
|------|------|
| **ReAct Agent Loop** | 实现 Reasoning + Acting 循环，支持工具调用和迭代执行 |
| **多通道输入** | 支持 Telegram Bot (长轮询) 和 Feishu/Lark (WebSocket 事件流) |
| **工具系统** | 模块化工具注册表，支持 GPIO 控制、文件操作、定时任务、网络搜索等 |
| **代理调度** | 基于 Cron 的自主任务调度，支持周期性消息触发 |
| **网络代理** | 支持 HTTP CONNECT 和 SOCKS5 代理隧道，适配企业网络环境 |
| **WiFi 管理** | 自动重连、指数退避、NVS 凭证持久化 |
| **OTA 更新** | 通过 HTTPS 进行固件无线升级 |

## 快速导航

### 核心模块

- **Agent 层**: `agent/agent_loop.c`, `agent/context_builder.c`, `agent/llm_client.c`
- **消息总线**: `bus/message_bus.c`
- **工具注册**: `tools/tool_registry.c`
- **网络代理**: `proxy/http_proxy.c`
- **WiFi 管理**: `wifi/wifi_manager.c`

### 工具实现

- **GPIO 控制**: `tools/tool_gpio.c`
- **文件操作**: `tools/tool_files.c`
- **定时任务**: `tools/tool_cron.c`
- **网络搜索**: `tools/tool_web_search.c`
- **时间同步**: `tools/tool_get_time.c`

### 通道实现

- **Telegram**: `transport/telegram_transport.c`
- **Feishu**: `transport/feishu_transport.c`

## 设计哲学应用

本项目文档基于 John Ousterhout 的《A Philosophy of Software Design》原则进行分析：

1. **Deep Modules**: 识别并评估模块的深度（接口复杂度 vs 实现功能）
2. **Information Hiding**: 分析信息隐藏和泄露情况
3. **Pull Complexity Downward**: 评估复杂度下沉策略
4. **Define Errors Out of Existence**: 分析错误处理设计
5. **Code Should Be Obvious**: 评估代码的直观性和可维护性

详见 [mimiclaw-analysis.md](./mimiclaw-analysis.md) 中的详细分析。

## 构建与运行

```bash
# 设置 ESP-IDF 环境
. $HOME/esp/esp-idf/export.sh

# 构建
idf.py build

# 烧录
idf.py flash

# 监控日志
idf.py monitor
```

## 配置

关键配置项位于 `mimi_config.h`（或 `mimi_secrets.h` 用于敏感信息）：

- WiFi SSID/Password
- Telegram Bot Token
- Feishu App ID/Secret
- API Keys (Tavily, Brave Search)
- GPIO 允许列表
- 代理服务器设置
