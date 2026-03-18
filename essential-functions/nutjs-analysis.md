# nut.js 技术分析文档

## 概述

**nut.js**（Native UI Toolkit）是一个跨平台的 Node.js 桌面自动化库，允许开发者通过编程方式控制鼠标、键盘和屏幕。它专为 E2E 测试、桌面自动化和构建智能桌面 Agent 而设计。

| 属性 | 信息 |
|------|------|
| 官网 | https://nutjs.dev/ |
| GitHub | https://github.com/nut-tree/nut.js |
| Star 数 | 2.8k+ |
| 许可证 | 开源（Apache-2.0），预编译包需付费 |
| 支持平台 | Windows、macOS、Linux |

---

## 核心功能

### 1. 鼠标控制

```typescript
import { mouse, Button, Point, straightTo } from "@nut-tree/nut-js";

// 移动到指定坐标
await mouse.move(straightTo(new Point(100, 200)));

// 左键点击
await mouse.click(Button.LEFT);

// 右键点击
await mouse.click(Button.RIGHT);

// 双击
await mouse.doubleClick(Button.LEFT);

// 拖拽
await mouse.drag(straightTo(new Point(400, 300)));

// 滚动
await mouse.scrollDown(5);
await mouse.scrollUp(3);

// 获取当前位置
const position = await mouse.getPosition();

// 配置移动速度
mouse.config.mouseSpeed = 1000; // 像素/秒
```

### 2. 键盘控制

```typescript
import { keyboard, Key } from "@nut-tree/nut-js";

// 输入文本
await keyboard.type("Hello, nut.js!");

// 按键组合
await keyboard.pressKey(Key.LeftControl, Key.A);
await keyboard.releaseKey(Key.LeftControl, Key.A);

// 单键
await keyboard.pressKey(Key.Enter);
```

### 3. 屏幕操作

```typescript
import { screen, Region, saveImage, centerOf } from "@nut-tree/nut-js";

// 获取屏幕尺寸
const width = await screen.width();
const height = await screen.height();

// 截图
const screenshot = await screen.capture();
await saveImage(screenshot, "screenshot.png");

// 区域截图
const region = new Region(100, 100, 400, 300);
const partialCapture = await screen.capture(region);

// 获取区域中心点
const center = centerOf(region);
```

### 4. 窗口管理

```typescript
import { getActiveWindow } from "@nut-tree/nut-js";

// 获取活动窗口
const window = await getActiveWindow();

// 获取窗口标题
const title = await window.title;

// 获取窗口区域
const region = await window.region;
```

### 5. 智能元素搜索

```typescript
import { screen, mouse, straightTo, centerOf } from "@nut-tree/nut-js";
import { useElementInspector } from "@nut-tree/element-inspector";

useElementInspector();

const window = await getActiveWindow();

// 通过 UI 结构查找按钮
const submitBtn = await window.find(
  elements.button({ title: "Submit" })
);

await mouse.move(straightTo(centerOf(submitBtn.region)));
await mouse.click(Button.LEFT);
```

---

## 插件生态

nut.js 采用插件架构，核心功能免费开源，高级功能需要额外的插件：

| 插件 | 功能 | 备注 |
|------|------|------|
| `@nut-tree/nut-js` | 核心（鼠标、键盘、屏幕、窗口） | 开源，需自行编译 |
| `@nut-tree/template-matcher` | 图像搜索 | 付费插件 |
| `@nut-tree/plugin-ocr` | 文字识别（OCR） | 付费插件 |
| `@nut-tree/element-inspector` | UI 元素检测 | 付费插件 |
| `@nut-tree/bolt` | 输入监控、窗口搜索 | 付费插件 |
| `@nut-tree/nib` | AI Agent CLI | 付费插件 |

### 图像搜索示例

```typescript
import { screen, mouse, straightTo, centerOf, imageResource } from "@nut-tree/nut-js";

// 在屏幕上查找图像
const buttonLocation = await screen.find(imageResource("button.png"));

// 移动并点击
await mouse.move(straightTo(centerOf(buttonLocation)));
await mouse.click(Button.LEFT);

// 等待图像出现
await screen.waitFor(imageResource("success.png"), 5000);
```

### OCR 文字搜索示例

```typescript
import { screen, singleWord } from "@nut-tree/nut-js";
import { preloadLanguages, Language } from "@nut-tree/plugin-ocr";

// 预加载语言模型
await preloadLanguages([Language.English]);

// 查找文字
const textLocation = await screen.find(singleWord("Submit"));

// 读取屏幕文字
const content = await screen.read();
```

---

## 定价模式

nut.js 采用「开源核心 + 付费预编译」模式：

| 方案 | 价格 | 内容 |
|------|------|------|
| Core | $20/月 | 预编译包、鼠标键盘、屏幕、窗口 |
| Solo | $75/月 | Core + 所有插件（OCR、图像搜索、元素检测等） |
| Team | 定制 | 多用户授权、批量折扣 |

> **注意**：核心功能仍然开源免费，但需要自行从源码编译。付费获得的是预编译的便捷性。

---

## 平台支持

### Windows
- Windows 10 N 需要安装 Media Feature Pack
- 无额外依赖

### macOS
- 需要 Xcode Command Line Tools：`xcode-select --install`
- 需要授予 **Accessibility** 和 **Screen Recording** 权限
- 自动检测并请求权限

### Linux
- 需要 libXtst：`sudo apt-get install libxtst-dev`
- **不支持 Wayland**，仅支持 X11
- 可切换到 XWayland 作为替代

---

## 与其他库对比

| 特性 | nut.js | RobotJS | Puppeteer |
|------|--------|---------|-----------|
| 跨平台 | ✅ | ✅ | ✅ |
| 原生桌面应用 | ✅ | ✅ | ❌ |
| 图像搜索 | ✅ (插件) | ❌ | ❌ |
| OCR 文字识别 | ✅ (插件) | ❌ | ❌ |
| UI 元素检测 | ✅ (插件) | ❌ | ✅ (DOM) |
| TypeScript 支持 | ✅ | ❌ | ✅ |
| 活跃维护 | ✅ | ⚠️ | ✅ |
| 性能 | 高 | 中 | 高 |

---

## 架构设计

```
nut.js
├── core/                    # 核心模块
│   ├── mouse.class.ts       # 鼠标控制
│   ├── keyboard.class.ts    # 键盘控制
│   ├── screen.class.ts      # 屏幕操作
│   └── window.function.ts   # 窗口管理
├── providers/               # 提供者接口
│   ├── image-finder/        # 图像搜索接口
│   ├── ocr/                 # OCR 接口
│   └── log/                 # 日志接口
└── libnut-core/             # 原生 C++ 绑定
```

### 设计特点

1. **插件架构**：核心功能通过接口定义，插件实现具体功能
2. **Provider 模式**：图像搜索、OCR 等功能可替换实现
3. **原生绑定**：通过 libnut-core 调用系统 API
4. **Promise/Async**：全异步 API，易于使用

---

## 实际应用场景

### 1. E2E 桌面应用测试

```typescript
describe("Login Flow", () => {
  it("should login successfully", async () => {
    const window = await getActiveWindow();

    const emailField = await window.find(
      elements.textInput({ id: "email" })
    );
    await mouse.move(straightTo(centerOf(emailField.region)));
    await mouse.click(Button.LEFT);
    await keyboard.type("user@test.com");

    const submitBtn = await window.find(
      elements.button({ title: "Sign In" })
    );
    await mouse.move(straightTo(centerOf(submitBtn.region)));
    await mouse.click(Button.LEFT);
  });
});
```

### 2. AI 桌面 Agent

```typescript
import { screen, mouse, Button, straightTo } from "@nut-tree/nut-js";

async function executeAction(action: Action) {
  // 截取当前屏幕
  const screenshot = await screen.capture();

  // 让 LLM 分析屏幕
  const target = await analyzeWithLLM(screenshot, action.description);

  await mouse.move(straightTo(target.coordinates));
  await mouse.click(Button.LEFT);
}
```

### 3. 自动化工作流

```typescript
// 自动填写表单
await mouse.move(straightTo(new Point(400, 200)));
await mouse.click(Button.LEFT);
await keyboard.type("john.doe@example.com");

// Tab 到下一个字段
await keyboard.pressKey(Key.Tab);
await keyboard.type("password123");

// 点击提交
await mouse.move(straightTo(new Point(400, 350)));
await mouse.click(Button.LEFT);
```

---

## 优缺点分析

### 优点

1. **跨平台支持**：Windows、macOS、Linux 一致 API
2. **TypeScript 原生支持**：类型完整，开发体验好
3. **插件生态**：图像搜索、OCR 等高级功能可选
4. **性能优秀**：原生绑定，比 RobotJS 快 100 倍
5. **活跃维护**：持续更新，社区活跃
6. **AI Agent 友好**：提供 NIB CLI，支持 JSON 协议

### 缺点

1. **付费门槛**：预编译包和高级插件需要订阅
2. **编译复杂**：开源版本需要自行编译原生依赖
3. **Linux 限制**：不支持 Wayland
4. **macOS 权限**：需要手动授权 Accessibility 和 Screen Recording

---

## 与本仓库项目的对比

| 项目 | 类型 | 优势 |
|------|------|------|
| **nut.js** | Node.js 库 | TypeScript 友好、插件丰富、API 设计优雅 |
| **agent-browser** | CLI 工具 | 开箱即用、Rust 高性能、专为 AI Agent 设计 |
| **pinchtab** | HTTP Server | 12MB 轻量、Go 单二进制、HTTP API |

**选择建议**：
- 如果需要 **TypeScript 开发**、**精细控制**：选择 nut.js
- 如果需要 **AI Agent 集成**、**零依赖部署**：选择 agent-browser 或 pinchtab
- 如果需要 **浏览器自动化**：选择 agent-browser（基于 Playwright）

---

## 总结

nut.js 是目前 Node.js 生态中最成熟的桌面自动化库，特别适合：

1. **桌面应用 E2E 测试**
2. **RPA 自动化流程**
3. **AI 桌面 Agent 开发**
4. **跨平台自动化脚本**

其插件架构设计良好，核心功能稳定，是 RobotJS 的优秀替代品。但需要注意开源版本需要自行编译，预编译包需要付费订阅。