# Raspberry Pi "Claw" AI Agent Projects 调研报告

> 针对树莓派生态的 AI Agent/助手项目全景扫描
> 生成时间: 2026-03-17

---

## 项目概览

| 项目 | 技术栈 | 定位 | 硬件要求 | 星级/影响力 |
|------|--------|------|----------|-------------|
| **OpenClaw** | Python + TypeScript | 全功能 AI Agent | Pi 4/5 (4GB+) | 高 |
| **ZeroClaw** | Rust | 轻量级自主代理框架 | Pi Zero 2W 及以上 | 27.4k ⭐ |
| **PicoClaw** | Go | 超轻量级个人助手 | Pi Zero 2W 及以上 | 活跃开发 |
| **PiZero-OpenClaw** | Python | 语音交互助手 | Pi Zero W + PiSugar | 社区项目 |
| **PicoClaw-AI-Switch** | Python | 智能家居控制器 | Pi Zero W | 社区项目 |

---

## 1. OpenClaw

**官网**: https://openclaw.ai
**GitHub**: https://github.com/openclaw-project/openclaw

### 简介
OpenClaw 是一个功能完整的 AI Agent，专为树莓派优化，可以执行复杂任务、运行命令、与 API 交互。

### 硬件要求
- **推荐**: Raspberry Pi 4 (4GB+) 或 Pi 5
- **支持**: Pi 3B+ (功能受限)
- **存储**: 32GB+ SD 卡

### 核心特性
| 特性 | 说明 |
|------|------|
| 离线运行 | 支持 Ollama/llama.cpp/LocalAI 本地部署 |
| 多模态 | 文本、图像、语音输入 |
| 工具集成 | 文件操作、Web 搜索、API 调用 |
| 安全防护 | 沙箱执行、权限控制 |
| 插件系统 | 可扩展的插件架构 |

### 安装命令
```bash
curl -fsSL https://openclaw.ai/install.sh | bash
# 或
git clone https://github.com/openclaw-project/openclaw && cd openclaw && ./install.sh
```

### 离线模式配置
```yaml
# config.yaml
llm:
  provider: ollama
  model: llama3.2:3b  # 适合 Pi 4 的轻量模型
  local: true
```

---

## 2. ZeroClaw

**GitHub**: https://github.com/zeroclaw-labs/zeroclaw

### 简介
ZeroClaw 是一个 Rust 编写的自主 AI Agent 框架，专为资源受限环境设计，强调安全性和最小资源占用。

### 技术亮点
| 特性 | 实现 |
|------|------|
| 内存安全 | Rust 所有权系统保证 |
| 并发模型 | async/await + tokio |
| 体积优化 | 静态链接，单二进制文件 ~5MB |
| 启动时间 | <500ms |

### 架构设计
```
┌─────────────────────────────────────┐
│           用户接口层                 │
│    CLI │ Web UI │ Voice Command     │
├─────────────────────────────────────┤
│           核心引擎                   │
│    Planner │ Executor │ Memory      │
├─────────────────────────────────────┤
│           工具层                     │
│    System │ Network │ File │ API    │
└─────────────────────────────────────┘
```

### 资源占用对比
| 指标 | ZeroClaw | OpenClaw | PicoClaw |
|------|----------|----------|----------|
| 内存占用 | ~8MB | ~100MB+ | ~10MB |
| 二进制大小 | ~5MB | ~50MB+ | ~20-50MB |
| 启动时间 | <500ms | ~2s | <1s |
| 依赖数量 | 0 (静态链接) | 较多 | 较少 |

---

## 3. PicoClaw (本仓库项目)

**GitHub**: https://github.com/sipeed/picoclaw
**本文档分析版本**: 本地 `/Users/luarassassin/reference-projects-me/picoclaw`

### 简介
PicoClaw 是专为树莓派设计的超轻量级个人 AI 助手，Go 语言编写，目标是运行在 $10 硬件上。

### 硬件兼容性
| 平台 | 状态 | 备注 |
|------|------|------|
| Pi Zero 2 W | ✓ 推荐 | 512MB RAM 足够运行 |
| Pi 3 | ✓ 支持 | 无需 LPDDR4 |
| Pi 4 | ✓ 支持 | 性能过剩 |
| Pi 5 | ✓ 支持 | 完整功能 |

### 快速安装
```bash
git clone https://github.com/sipeed/picoclaw && cd picoclaw
picoclaw onboard  # 交互式配置向导
```

### 核心优势
1. **超轻量**: <10MB 内存占用
2. **多通道**: 支持 15+ 消息平台 (Telegram/Discord/QQ/飞书/钉钉等)
3. **MCP 协议**: 标准工具集成接口
4. **BM25 搜索**: 智能工具发现
5. **模型路由**: 根据查询复杂度自动选择模型

---

## 4. PiZero-OpenClaw

**GitHub**: https://github.com/sebastianvkl/pizero-openclaw

### 简介
基于 Pi Zero W 和 PiSugar WhisPlay 扩展板的语音交互 AI 助手项目。

### 硬件配置
- **主板**: Raspberry Pi Zero W
- **扩展板**: PiSugar WhisPlay (带麦克风阵列)
- **电源**: PiSugar 电池模块
- **存储**: 16GB+ SD 卡

### 功能特点
| 功能 | 实现方式 |
|------|----------|
| 语音唤醒 | 本地关键词检测 (porcupine) |
| 语音识别 | Whisper.cpp (本地) 或云 API |
| 语音合成 | piper-tts (本地) |
| 自然语言处理 | OpenClaw Agent |
| 便携性 | 电池供电，完全无线 |

### 使用场景
- 便携语音助手
- 智能家居语音控制
- 无需屏幕的交互式查询

---

## 5. PicoClaw-AI-Switch

**GitHub**: https://github.com/nejimonraveendran/picoclaw-ai-switch

### 简介
使用自然语言控制家用电器的智能家居项目，基于 Pi Zero W 和继电器模块。

### 硬件配置
- **主板**: Raspberry Pi Zero W
- **继电器**: 4 路继电器模块
- **电源**: 5V 2A 电源适配器
- **存储**: 8GB+ SD 卡

### 系统架构
```
用户语音 → PicoClaw → 自然语言理解 → 意图识别 → 设备控制 → 继电器 → 电器
                ↓
           技能系统 (skills/)
```

### 示例交互
```
用户: "打开客厅的灯"
Agent: "好的，正在打开客厅灯"
      [继电器 1 闭合]

用户: "把空调调到 26 度"
Agent: "已将空调温度设置为 26°C"
      [红外信号发送]
```

---

## 选型建议

### 按使用场景

| 场景 | 推荐项目 | 理由 |
|------|----------|------|
| **个人日常助手** | PicoClaw | 多通道支持，轻量级，易部署 |
| **复杂任务自动化** | OpenClaw | 功能最完整，生态丰富 |
| **极致资源受限** | ZeroClaw | Rust 实现，最小内存占用 |
| **语音交互硬件** | PiZero-OpenClaw | 专为语音优化的硬件方案 |
| **智能家居控制** | PicoClaw-AI-Switch | 即插即用的继电器控制 |

### 按硬件配置

| 硬件 | 推荐项目 | 可运行模型 |
|------|----------|------------|
| Pi Zero 2 W (512MB) | PicoClaw / ZeroClaw | API 调用 / 超轻量本地模型 |
| Pi 3 (1GB) | PicoClaw / ZeroClaw | 3B 参数本地模型 |
| Pi 4 4GB | OpenClaw / PicoClaw | 7B 参数本地模型 |
| Pi 5 8GB | OpenClaw | 13B+ 参数本地模型 |

---

## 树莓派官方资源

**文章**: [OpenClaw on Raspberry Pi](https://www.raspberrypi.com/news/openclaw-on-raspberry-pi/)

### 官方推荐配置
```
操作系统: Raspberry Pi OS (64-bit)
内存: 4GB+ (推荐 8GB for Pi 5)
存储: 32GB SD 卡 (Class 10)
网络: 以太网或 5GHz WiFi
```

### 官方性能测试数据
| 模型 | 平台 | 推理速度 |
|------|------|----------|
| Llama 3.2 3B | Pi 4 4GB | ~5 tok/s |
| Llama 3.2 1B | Pi 4 4GB | ~15 tok/s |
| Phi-3 Mini | Pi 5 8GB | ~10 tok/s |
| API 调用 | Pi Zero 2W | 取决于网络 |

---

## 总结

树莓派生态中的 "Claw" 项目呈现明显的分层：

1. **PicoClaw** 和 **ZeroClaw** 专注于极致轻量，适合 Zero 系列和旧版 Pi
2. **OpenClaw** 提供完整功能，是 Pi 4/5 的最佳选择
3. **社区项目** (PiZero-OpenClaw, PicoClaw-AI-Switch) 展示了垂直场景的应用潜力

**趋势**: 随着 Pi 5 的普及和本地推理能力的提升，离线 AI Agent 将成为树莓派的主流应用场景之一。

---

*数据来源: GitHub, 官方文档, Raspberry Pi 官方博客*
*调研时间: 2026-03-17*
