# NanoClaw 依赖分析

> 核心依赖与运行时要求详解

---

## 核心依赖清单

NanoClaw 采用极简依赖策略，核心功能仅依赖 6 个生产依赖包。

| 包名 | 版本 | 用途 | 体积 |
|------|------|------|------|
| `better-sqlite3` | ^11.8.1 | SQLite 数据库（原生绑定） | ~5MB |
| `cron-parser` | ^5.5.0 | Cron 表达式解析 | ~150KB |
| `pino` | ^9.6.0 | 结构化日志 | ~200KB |
| `pino-pretty` | ^13.0.0 | 日志美化输出 | ~100KB |
| `yaml` | ^2.8.2 | YAML 解析（技能配置） | ~300KB |
| `zod` | ^4.3.6 | 运行时类型校验 | ~100KB |

**总生产依赖体积**: ~6MB（包含 better-sqlite3 原生模块）

---

## 依赖深度分析

### 1. better-sqlite3

**类型**: 原生绑定模块（C++）

**用途**:
- 消息持久化存储
- 群组元数据管理
- 定时任务调度
- Session 状态存储

**设计考量**:
```typescript
// 直接 SQL，无 ORM 抽象泄漏
db.prepare('INSERT INTO messages (chat_jid, content) VALUES (?, ?)')
  .run(chatJid, content);
```

遵循 Ousterhout 原则：**避免抽象泄漏**。直接使用 SQL 而非 ORM，调用者完全理解底层操作。

**安全注意**:
- 数据库文件位于 `store/database.sqlite`
- 容器内代理无权直接访问（通过 IPC 代理）

---

### 2. cron-parser

**类型**: 纯 JavaScript

**用途**:
- 解析 Cron 表达式（如 `0 9 * * *`）
- 计算下次执行时间
- 支持复杂表达式（L、W、# 等特殊字符）

**代码示例**:
```typescript
import { CronExpressionParser } from 'cron-parser';

const interval = CronExpressionParser.parse('0 9 * * *', {
  tz: 'Asia/Shanghai'
});
const nextRun = interval.next().toDate();
```

---

### 3. pino / pino-pretty

**类型**: 纯 JavaScript

**用途**:
- 结构化 JSON 日志（便于机器解析）
- 开发时美化输出（pino-pretty）
- 支持日志级别动态调整

**设计考量**:
```typescript
// 深模块：简单接口，内部处理复杂格式化
logger.info({ chatJid, count: 5 }, 'New messages');
// 输出: {"level":30,"time":...,"chatJid":"...","count":5,"msg":"New messages"}
```

**优势**:
- 零配置即可用
- 高性能（比 Winston 快 5-10 倍）
- 结构化日志便于集中收集

---

### 4. yaml

**类型**: 纯 JavaScript

**用途**:
- 解析技能配置文件（`.claude/skills/*.yaml`）
- 群组配置持久化

---

### 5. zod

**类型**: 纯 JavaScript

**用途**:
- 运行时类型校验（替代 TypeScript 仅在编译时检查）
- 外部输入验证（消息内容、配置等）

**使用场景**:
```typescript
const TaskSchema = z.object({
  id: z.string(),
  prompt: z.string(),
  schedule_type: z.enum(['cron', 'interval', 'once']),
  schedule_value: z.string(),
});

// 运行时验证
const task = TaskSchema.parse(rawData);
```

---

## 开发依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `@types/better-sqlite3` | ^7.6.12 | TypeScript 类型定义 |
| `@types/node` | ^22.10.0 | Node.js API 类型 |
| `@vitest/coverage-v8` | ^4.0.18 | 测试覆盖率 |
| `husky` | ^9.1.7 | Git hooks 管理 |
| `prettier` | ^3.8.1 | 代码格式化 |
| `tsx` | ^4.19.0 | TypeScript 直接运行（开发） |
| `typescript` | ^5.7.0 | 编译器 |
| `vitest` | ^4.0.18 | 测试框架 |

---

## 运行时要求

### Node.js

- **最低版本**: Node.js 20
- **推荐版本**: Node.js 20 LTS 或更高
- **原因**: 使用 `import.meta.url`、原生 `fetch`（实验性）、V8 性能优化

### 容器运行时

| 平台 | 运行时 | 备注 |
|------|--------|------|
| macOS | Apple Container Framework | Apple Silicon 原生 |
| Linux | Docker + BuildKit | 标准容器运行时 |
| Windows | WSL2 + Docker | 需启用 WSL2 |

### 系统依赖

- **git**: 技能系统依赖（分支合并）
- **sqlite3**: better-sqlite3 编译需要（已预编译二进制）

---

## 频道依赖（可选）

频道通过独立包或 git 分支引入，不属于核心依赖：

| 频道 | 依赖方式 | 主要依赖 |
|------|----------|----------|
| WhatsApp | Git 分支 | `@whiskeysockets/baileys` |
| Telegram | Git 分支 | `node-telegram-bot-api` |
| Discord | Git 分支 | `discord.js` |
| Slack | Git 分支 | `@slack/bolt` |
| Gmail | Git 分支 | `googleapis` |

**设计理念**: 零配置自注册。频道工厂返回 `null` 时自动跳过，不阻塞启动。

---

## 依赖关系图

```
nanoclaw/
├── better-sqlite3 (数据层)
│   └── SQLite3 (系统库)
├── cron-parser (调度)
├── pino (日志)
│   └── pino-pretty (开发)
├── yaml (配置)
└── zod (校验)

optional/
├── whatsapp (分支)
│   └── baileys
├── telegram (分支)
│   └── node-telegram-bot-api
├── discord (分支)
│   └── discord.js
├── slack (分支)
│   └── @slack/bolt
└── gmail (分支)
    └── googleapis
```

---

## 安全考量

### 依赖攻击面

| 依赖 | 攻击向量 | 缓解措施 |
|------|----------|----------|
| better-sqlite3 | 原生代码漏洞 | 容器隔离（Agent 无直接访问） |
| cron-parser | ReDoS | 内部使用，不暴露用户输入 |
| yaml | YAML 解析漏洞 | 仅解析可信配置文件 |
| zod | 无 | 纯校验库，无副作用 |

### 最小化原则

NanoClaw 刻意避免以下常见依赖：

| 常见依赖 | 为何不用 | 替代方案 |
|----------|----------|----------|
| Express/Fastify | 无需 HTTP 服务 | 仅 IPC 通信 |
| Lodash | 现代 JS 已足够 | 原生方法 |
| Moment.js | 体积大 | 原生 `Date` + `Intl` |
| Axios | Node 18+ 有 fetch | 原生 `fetch` |
| ORM (Prisma/TypeORM) | 抽象泄漏 | 直接 SQL |

---

## 更新建议

### 自动更新安全

```bash
# 仅更新 patch/minor 版本
npm update

# 检查过时依赖
npm outdated
```

### 手动审查

| 依赖 | 审查频率 | 关注点 |
|------|----------|--------|
| better-sqlite3 | 每 major 版本 | 原生模块兼容性 |
| zod | 每 major 版本 | API 变更 |
| 其他 | 每季度 | 安全公告 |

### 锁定策略

- `package-lock.json` 必须提交
- CI 使用 `npm ci` 保证可重复构建
- 容器构建使用 `--immutable` 模式

---

## 依赖体积优化

### 生产环境

```bash
# 仅安装生产依赖
npm ci --production

# 结果
node_modules/  ~15MB
  ├── better-sqlite3/  ~5MB
  ├── pino/            ~1MB
  └── 其他/            ~9MB
```

### 容器镜像

依赖安装在主机，容器内仅复制 `dist/` 和必要资源：

```dockerfile
# 主机构建
COPY dist/ /app/
COPY container/skills/ /app/skills/
# 无需 node_modules（主机 Node.js 运行）
```

---

## 总结

NanoClaw 的依赖策略体现 **"Small Enough to Understand"** 原则：

1. **极简依赖**: 仅 6 个生产依赖
2. **明确用途**: 每个依赖解决特定问题，无功能重叠
3. **安全第一**: 原生代码通过容器隔离，解析库处理可信输入
4. **可选扩展**: 频道通过外部分支引入，不污染核心依赖

```
依赖数量对比:
├── NanoClaw:      6  核心依赖
├── Express 应用:  30+ 典型依赖
├── Next.js 项目:  50+ 典型依赖
└── Electron 应用: 100+ 典型依赖
```

---

*依赖分析完成。详见主分析文档了解架构设计。*
