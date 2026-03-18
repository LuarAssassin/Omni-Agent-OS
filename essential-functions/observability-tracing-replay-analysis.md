# Observability / Tracing / Replay 架构分析

> **覆盖范围**: 15个AI Agent项目 (8个Personal Assistants + 7个Programming Agents)
> **分析维度**: 日志框架、指标收集、分布式追踪、事件流、会话重放、遥测后端
> **最后更新**: 2026-03-18

---

## 目录

1. [执行摘要](#执行摘要)
2. [可观测性成熟度分级](#可观测性成熟度分级)
3. [日志框架与结构化日志](#日志框架与结构化日志)
4. [指标收集体系](#指标收集体系)
5. [分布式追踪实现](#分布式追踪实现)
6. [事件流与Wire日志](#事件流与wire日志)
7. [会话重放与检查点](#会话重放与检查点)
8. [遥测后端对比](#遥测后端对比)
9. [对比矩阵](#对比矩阵)
10. [推荐统一架构](#推荐统一架构)

---

## 执行摘要

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         可观测性架构全景图                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│   │    Logs     │    │   Metrics   │    │   Traces    │    │   Replay    │ │
│   │             │    │             │    │             │    │             │ │
│   │ • zerolog   │    │ • Token Use │    │ • OpenTel   │    │ • D-Mail    │ │
│   │ • tracing   │    │ • Latency   │    │ • Trace ID  │    │ • SQLite    │ │
│   │ • loguru    │    │ • Success   │    │ • Spans     │    │ • JSONL     │ │
│   │ • zerolog   │    │ • Cost      │    │ • Context   │    │ • Checkpt   │ │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘ │
│          │                  │                  │                  │        │
│          └──────────────────┴──────────────────┘                  │        │
│                             │                                      │        │
│                             ▼                                      ▼        │
│                  ┌─────────────────────┐    ┌─────────────────────┐         │
│                  │   Event Stream      │    │   Session Store     │         │
│                  │   (MessageBus)      │    │   (Checkpoint)      │         │
│                  └─────────────────────┘    └─────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 核心发现

| 维度 | 关键洞察 |
|------|----------|
| **日志框架** | Rust项目普遍使用`tracing`，Go项目使用`zerolog`，Python使用`loguru`或标准库 |
| **结构化日志** | 高性能项目(PicoClaw, ZeroClaw)采用JSON/JSONL格式，支持机器解析 |
| **指标收集** | Token使用、延迟、成功率是三大核心指标；Gemini CLI实现反向Token预算 |
| **分布式追踪** | 仅Gemini CLI和ZeroClaw完整支持OpenTelemetry；多数项目使用自定义追踪 |
| **事件流** | MessageBus模式被广泛采用(Nanobot, Gemini CLI, OpenCode) |
| **会话重放** | Kimi-CLI的D-Mail、Codex的SQLite快照、Qwen Code的ChatRecording |
| **遥测后端** | 零依赖项目使用SQLite/JSONL；云原生项目使用Google Cloud Trace |

---

## 可观测性成熟度分级

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        可观测性成熟度金字塔                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                              ┌─────────────┐                                │
│                              │   Level 3   │  全链路追踪 + 自动回放           │
│                              │  (Advanced) │  Gemini CLI, IronClaw          │
│                              └──────┬──────┘                                │
│                                     │                                       │
│                         ┌───────────┴───────────┐                           │
│                         │       Level 2         │  结构化日志 + 指标 + 事件流 │
│                         │    (Intermediate)     │  Codex, ZeroClaw, Nanobot   │
│                         └───────────┬───────────┘                           │
│                                     │                                       │
│                    ┌────────────────┴────────────────┐                      │
│                    │           Level 1               │  基础日志 + 会话存储    │
│                    │         (Basic)                 │  Kimi-CLI, OpenCode     │
│                    │                                 │  PicoClaw, NanoClaw     │
│                    └────────────────┬────────────────┘                      │
│                                     │                                       │
│         ┌───────────────────────────┴───────────────────────────┐           │
│         │                     Level 0                           │           │
│         │                   (Minimal)                           │           │
│         │              简单控制台日志                            │           │
│         │              Aider, Qwen Code, Pi-Mono                │           │
│         └───────────────────────────────────────────────────────┘           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

| 等级 | 项目 | 特征 |
|------|------|------|
| **Level 3** | Gemini CLI, IronClaw | OpenTelemetry全链路追踪、Google Cloud集成、安全审计日志、Token预算追踪 |
| **Level 2** | Codex, ZeroClaw, Nanobot, OpenClaw | 结构化日志、SQLite/JSONL持久化、MessageBus事件流、可插拔遥测后端 |
| **Level 1** | Kimi-CLI, OpenCode, PicoClaw, NanoClaw | 会话状态持久化、基础指标、文件/数据库日志 |
| **Level 0** | Aider, Qwen Code, Pi-Mono, Mimiclaw | 控制台日志、最小化可观测性、依赖外部工具 |

---

## 日志框架与结构化日志

### 技术栈分布

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      日志框架技术栈分布                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Rust                    Go                 Python              TypeScript │
│  ─────                   ───                ──────              ────────── │
│                                                                             │
│  ┌─────────┐            ┌─────────┐        ┌─────────┐        ┌─────────┐  │
│  │ tracing │            │ zerolog │        │ loguru  │        │ OpenTel │  │
│  │         │            │         │        │         │        │ SDK     │  │
│  │•Codex   │            │•PicoClaw│        │•KimiCLI │        │•Gemini  │  │
│  │•ZeroClaw│            │         │        │         │        │         │  │
│  │•IronClaw│            │ JSONL   │        │ + rich  │        │ Event   │  │
│  └─────────┘            └─────────┘        │•CoPaw   │        │ Bus     │  │
│       │                                    │ 标准库  │        │•OpenCode│  │
│       │                                    └─────────┘        │•OpenClaw│  │
│       │                                                       └─────────┘  │
│       │                                                                    │
│       ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    结构化日志格式                                     │   │
│  │                                                                       │   │
│  │   JSON:     {"level":"info","time":"2026-03-18T10:00:00Z","msg":"..."}│   │
│  │   logfmt:   level=info time=2026-03-18T10:00:00Z msg="..."            │   │
│  │   JSONL:    {"event":"msg","ts":...}\n{"event":"reply","ts":...}        │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. Rust生态 (tracing)

#### Codex
```rust
// core/src/lib.rs - 严格禁止直接输出
#![deny(clippy::print_stdout, clippy::print_stderr)]

// 使用tracing进行结构化日志
tracing::info!(target: "codex", session_id = %id, "Session started");

// 输出路径分流
// 1. Library Code → tracing (结构化日志)
// 2. Library Code → EQ (Event Queue) → UI显示
// 3. Library Code → 结构化输出 (App Server JSON-RPC)
```

#### ZeroClaw
```rust
// src/observability/log.rs
pub struct Logger;

impl Logger {
    pub fn init(level: LogLevel) {
        // 特征门控后端支持
        tracing_subscriber::fmt()
            .with_env_filter(EnvFilter::from_default_env())
            .json() // 结构化JSON输出
            .init();
    }
}

// Cargo.toml 特征配置
[features]
observability-otel = ["opentelemetry"]
observability-prometheus = ["prometheus"]
```

#### IronClaw
```rust
// src/tracing_fmt.rs - 自定义tracing格式化器
pub struct IronClawFormatter;

impl<F> FormatEvent<F> for IronClawFormatter {
    fn format_event(
        &self,
        ctx: &F,
        writer: &mut dyn Write,
        event: &Event,
    ) -> std::fmt::Result {
        // 自定义字段提取和格式化
        // 支持上下文传播和Trace ID注入
    }
}
```

### 2. Go生态 (zerolog)

#### PicoClaw
```go
// pkg/logger/logger.go
package logger

import (
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func Init(level string) {
    // JSON结构化日志
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnixMs
    log.Logger = log.With().
        Timestamp().
        Str("service", "picoclaw").
        Logger()

    // 日志级别配置
    switch level {
    case "debug":
        zerolog.SetGlobalLevel(zerolog.DebugLevel)
    case "info":
        zerolog.SetGlobalLevel(zerolog.InfoLevel)
    }
}

// 使用示例
log.Info().
    Str("channel", "telegram").
    Str("user_id", userID).
    Msg("Message received")
```

**会话存储**: JSONL (JSON Lines) 格式
```jsonl
{"ts":1710748800000,"type":"inbound","channel":"telegram","content":"Hello"}
{"ts":1710748805000,"type":"outbound","channel":"telegram","content":"Hi there!"}
```

### 3. Python生态

#### Kimi-CLI (loguru + rich)
```python
# pyproject.toml 依赖
loguru = "*"   # 日志框架
rich = "*"     # 终端UI输出

# 会话状态持久化 (session_state.py)
import json
from pathlib import Path

class SessionState:
    """D-Mail机制: 会话状态持久化用于时间旅行调试"""

    def save(self, path: Path):
        state = {
            "session_id": self.session_id,
            "messages": self.messages,
            "sub_agents": self.sub_agents,
            "checkpoint": self.checkpoint,
            "timestamp": datetime.utcnow().isoformat()
        }
        path.write_text(json.dumps(state, indent=2))

    @classmethod
    def load(cls, path: Path) -> "SessionState":
        data = json.loads(path.read_text())
        return cls(**data)
```

#### CoPaw (标准库 + AgentScope)
```python
# 环境变量配置
COPAW_LOG_LEVEL=INFO  # DEBUG, INFO, WARN, ERROR

# FastAPI结构化日志
import logging
from fastapi import Request

logger = logging.getLogger("copaw")

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start

    logger.info(
        "Request completed",
        extra={
            "method": request.method,
            "path": request.url.path,
            "status": response.status_code,
            "duration_ms": duration * 1000,
            "agent_id": request.headers.get("X-Agent-ID")
        }
    )
    return response
```

### 4. TypeScript生态

#### Gemini CLI (OpenTelemetry)
```typescript
// OpenTelemetry SDK集成
import { NodeSDK } from '@opentelemetry/sdk-node';
import { CloudTraceExporter } from '@google-cloud/opentelemetry-cloud-trace-exporter';
import { CloudMonitoringExporter } from '@google-cloud/opentelemetry-cloud-monitoring-exporter';

const sdk = new NodeSDK({
  traceExporter: new CloudTraceExporter(),
  metricReader: new CloudMonitoringExporter(),
});

sdk.start();

// 手动Span创建
const span = tracer.startSpan('process_message');
span.setAttribute('message.length', content.length);
span.setAttribute('model', modelName);
span.end();
```

#### OpenCode (Event Bus)
```typescript
// packages/opencode/src/bus/bus.ts
export class Bus {
  private static listeners: Map<string, Set<Handler>> = new Map();

  static publish(event: string, data: any): void {
    const handlers = this.listeners.get(event);
    if (handlers) {
      handlers.forEach(h => {
        try {
          h(data);
        } catch (err) {
          logger.error({ event, error: err }, 'Event handler failed');
        }
      });
    }

    // 自动记录所有事件用于回放
    EventRecorder.record({ type: event, data, timestamp: Date.now() });
  }

  static subscribe(event: string, handler: Handler): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);

    return () => this.listeners.get(event)?.delete(handler);
  }
}

// 使用示例
Bus.publish(File.Event.Edited, { file: filepath, agent: agentId });
Bus.subscribe(File.Event.Edited, (data) => {
  metrics.increment('file.edit', { agent: data.agent });
});
```

### 5. 嵌入式C (ESP32)

#### Mimiclaw
```c
// bus/message_bus.c
#include "message_bus.h"

static QueueHandle_t inbound_queue;
static QueueHandle_t outbound_queue;

esp_err_t message_bus_init(void) {
    // 双队列分离: 入站/出站
    inbound_queue = xQueueCreate(32, MESSAGE_MAX_LEN);
    outbound_queue = xQueueCreate(32, MESSAGE_MAX_LEN);

    ESP_LOGI(TAG, "MessageBus initialized");
    return ESP_OK;
}

esp_err_t message_bus_push_inbound(const char *msg) {
    return xQueueSend(inbound_queue, msg, portMAX_DELAY);
}

esp_err_t message_bus_pop_inbound(char *buf, size_t len, uint32_t timeout_ms) {
    return xQueueReceive(inbound_queue, buf, pdMS_TO_TICKS(timeout_ms));
}
```

---

## 指标收集体系

### 核心指标矩阵

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         指标收集体系架构                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│   │   Token Metrics │  │ Performance     │  │   Business      │             │
│   │                 │  │   Metrics       │  │   Metrics       │             │
│   │ • Input tokens  │  │ • Latency       │  │ • Success rate  │             │
│   │ • Output tokens │  │ • Throughput    │  │ • Error rate    │             │
│   │ • Cost tracking │  │ • Queue depth   │  │ • User sessions │             │
│   │ • Budget usage  │  │ • Memory usage  │  │ • Tool usage    │             │
│   └────────┬────────┘  └────────┬────────┘  └────────┬────────┘             │
│            │                    │                    │                      │
│            └────────────────────┼────────────────────┘                      │
│                                 │                                           │
│                                 ▼                                           │
│            ┌─────────────────────────────────────┐                          │
│            │         Metrics Storage             │                          │
│            │                                     │                          │
│            │  ┌──────────┐ ┌──────────┐ ┌──────┐ │                          │
│            │  │ In-Memory│ │ SQLite   │ │Prom. │ │                          │
│            │  │ (Gauge)  │ │ (Events) │ │(Pull) │ │                          │
│            │  └──────────┘ └──────────┘ └──────┘ │                          │
│            └─────────────────────────────────────┘                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. Token使用追踪

#### Gemini CLI (反向Token预算)
```typescript
// ADR-011: Chat Compression Strategy - Reverse Token Budget
interface TokenBudget {
  systemInstruction: 0.1;   // 10% 系统提示
  recentMessages: 0.5;      // 50% 最近消息
  olderMessages: 0.3;       // 30% 历史消息
  functionResponses: 0.1;   // 10% 工具响应
}

class TokenTracker {
  private usage: Map<string, TokenMetrics> = new Map();

  record(model: string, input: number, output: number) {
    const key = `${model}:${new Date().toISOString().slice(0, 7)}`; // 按月聚合
    const current = this.usage.get(key) || { input: 0, output: 0, cost: 0 };

    current.input += input;
    current.output += output;
    current.cost += this.calculateCost(model, input, output);

    this.usage.set(key, current);

    // 导出到Cloud Monitoring
    this.exportMetric('tokens/input', input, { model });
    this.exportMetric('tokens/output', output, { model });
    this.exportMetric('tokens/cost', current.cost, { model });
  }
}
```

#### Codex (SQLite事件表)
```rust
// 事件表结构 (ADR-005)
const EVENTS_SCHEMA: &str = r#"
CREATE TABLE events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    type TEXT NOT NULL,           -- 'token_usage', 'latency', 'error'
    data JSON,                    -- 结构化指标数据
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Token使用查询
SELECT
    session_id,
    json_extract(data, '$.input_tokens') as input,
    json_extract(data, '$.output_tokens') as output,
    json_extract(data, '$.model') as model
FROM events
WHERE type = 'token_usage'
  AND timestamp > datetime('now', '-1 day');
"#;
```

### 2. 性能指标

#### ZeroClaw (可插拔指标)
```rust
// src/observability/metrics.rs
pub trait MetricsBackend: Send + Sync {
    fn record_counter(&self, name: &str, value: u64, labels: &[(&str, &str)]);
    fn record_gauge(&self, name: &str, value: f64, labels: &[(&str, &str)]);
    fn record_histogram(&self, name: &str, value: f64, labels: &[(&str, &str)]);
}

// Prometheus实现
#[cfg(feature = "observability-prometheus")]
pub struct PrometheusBackend {
    registry: Registry,
}

impl MetricsBackend for PrometheusBackend {
    fn record_histogram(&self, name: &str, value: f64, labels: &[(&str, &str)]) {
        let histogram = self.registry.get_histogram(name, labels);
        histogram.observe(value);
    }
}

// 使用示例
METRICS.record_histogram("llm.latency", duration.as_millis() as f64, &[
    ("model", model_name),
    ("provider", provider_id),
]);
```

#### IronClaw (安全指标)
```rust
// src/safety/policy.rs
pub struct SafetyMetrics {
    pub wasm_sandbox_hits: Counter,
    pub allowlist_blocks: Counter,
    pub leak_detections: Counter,
    pub audit_events: Counter,
}

impl SafetyMetrics {
    pub fn record_policy_violation(&self, rule: &PolicyRule, action: &str) {
        self.audit_events.inc(&[
            ("rule_id", &rule.id),
            ("severity", &format!("{:?}", rule.severity)),
            ("action", action),
        ]);

        // 高严重性立即告警
        if rule.severity == Severity::Critical {
            tracing::error!(
                rule_id = %rule.id,
                action = %action,
                "Critical safety policy violation"
            );
        }
    }
}
```

### 3. 成功率追踪

#### NanoClaw (双游标追踪)
```typescript
// src/queue/GroupQueue.ts
interface QueueMetrics {
  lastTimestamp: number;        // 最后处理消息
  lastAgentTimestamp: number;   // 最后Agent响应
  processedCount: number;
  errorCount: number;
  avgLatency: number;           // EMA计算
}

class GroupQueue {
  private metrics: Map<string, QueueMetrics> = new Map();

  async processMessage(msg: Message): Promise<void> {
    const start = Date.now();
    const groupId = msg.groupId;

    try {
      await this.agent.process(msg);

      const latency = Date.now() - start;
      this.updateMetrics(groupId, {
        lastTimestamp: msg.timestamp,
        lastAgentTimestamp: Date.now(),
        processedCount: (m) => m + 1,
        avgLatency: (m) => m * 0.9 + latency * 0.1, // EMA
      });
    } catch (err) {
      this.metrics.get(groupId)!.errorCount++;
      throw err;
    }
  }

  getSuccessRate(groupId: string): number {
    const m = this.metrics.get(groupId);
    if (!m) return 0;
    return m.processedCount / (m.processedCount + m.errorCount);
  }
}
```

---

## 分布式追踪实现

### OpenTelemetry集成对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      分布式追踪架构对比                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Gemini CLI (Full OTel)              ZeroClaw (Feature-Gated)               │
│  ───────────────────────             ───────────────────────                │
│                                                                             │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │   CLI App    │                    │  ZeroClaw    │                       │
│  │  ┌────────┐  │                    │  ┌────────┐  │                       │
│  │  │OpenTel │  │                    │  │tracing │  │                       │
│  │  │  SDK   │──┼──→ Cloud Trace     │  │        │  │                       │
│  │  └────────┘  │     (自动导出)      │  └───┬────┘  │                       │
│  └──────────────┘                    │      │       │                       │
│                                      │   [observability-otel]              │
│                                      │      │       │                       │
│                                      │      ▼       │                       │
│                                      │  ┌────────┐  │                       │
│                                      │  │OpenTel │──┼──→ OTLP Endpoint      │
│                                      │  │Exporter│  │    (条件编译)         │
│                                      │  └────────┘  │                       │
│                                      └──────────────┘                       │
│                                                                             │
│  Codex (SQ/EQ Protocol)              IronClaw (Custom)                      │
│  ───────────────────────             ─────────────────                      │
│                                                                             │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │   UI Layer   │                    │  IronClaw    │                       │
│  │  ┌────────┐  │                    │  ┌────────┐  │                       │
│  │  │   SQ   │──┼──→ Op ──→ Core    │  │ Custom │  │                       │
│  │  │(Submit)│  │                    │  │ Trace  │  │                       │
│  │  └────────┘  │                    │  │  fmt   │──┼──→ File/DB            │
│  └──────────────┘                    │  └────────┘  │                       │
│         │                            └──────────────┘                       │
│         ▼                                                                   │
│  ┌──────────────┐                                                           │
│  │  Core Layer  │                                                           │
│  │  ┌────────┐  │                                                           │
│  │  │   EQ   │──┼──→ EventMsg ──→ UI                                        │
│  │  │(Event) │  │     (Trace上下文传播)                                       │
│  │  └────────┘  │                                                           │
│  └──────────────┘                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. Trace ID生成与传播

#### OpenCode (会话追踪)
```typescript
// packages/opencode/src/session/session.ts
export class Session {
  readonly traceId: string;
  readonly parentSpan?: Span;

  constructor(config: SessionConfig) {
    // 生成Trace ID
    this.traceId = generateTraceId();

    // 创建根Span
    this.parentSpan = tracer.startSpan('session', {
      attributes: {
        'session.id': this.id,
        'session.type': config.type,
        'session.workspace': config.workspace,
      }
    });
  }

  async executeTool(name: string, args: any): Promise<ToolResult> {
    // 创建子Span
    const span = tracer.startSpan(`tool.${name}`, {
      parent: this.parentSpan,
      attributes: {
        'tool.name': name,
        'tool.args': JSON.stringify(args),
      }
    });

    try {
      const result = await this.toolRegistry.execute(name, args);
      span.setAttribute('tool.success', true);
      span.setAttribute('tool.result_size', JSON.stringify(result).length);
      return result;
    } catch (err) {
      span.setAttribute('tool.success', false);
      span.setAttribute('tool.error', err.message);
      throw err;
    } finally {
      span.end();
    }
  }
}
```

#### Kimi-CLI (Wire追踪)
```python
# wire_tracing.py
import uuid
from contextvars import ContextVar

# Trace上下文
current_trace_id: ContextVar[str] = ContextVar('trace_id')

class WireTracer:
    """WebSocket通信追踪"""

    def start_trace(self) -> str:
        trace_id = str(uuid.uuid4())[:8]
        current_trace_id.set(trace_id)
        return trace_id

    def log_websocket_event(self, direction: str, event_type: str, payload: dict):
        trace_id = current_trace_id.get(None)
        logger.info(
            f"Wire {direction}",
            extra={
                "trace_id": trace_id,
                "direction": direction,  # 'in' | 'out'
                "event_type": event_type,
                "payload_size": len(json.dumps(payload)),
                "timestamp": time.time_ns()
            }
        )
```

### 2. Span层级结构

#### Gemini CLI (完整Span层级)
```
Trace: gemini_cli_session_abc123
├── Span: process_input (500ms)
│   ├── Span: parse_command (10ms)
│   ├── Span: load_context (100ms)
│   └── Span: call_model (350ms)
│       ├── Span: prepare_prompt (50ms)
│       ├── Span: api_request (250ms)
│       │   ├── Span: network (200ms)
│       │   └── Span: parse_response (50ms)
│       └── Span: process_response (50ms)
├── Span: execute_tools (200ms)
│   ├── Span: tool.file_read (50ms)
│   └── Span: tool.shell_exec (150ms)
└── Span: render_output (50ms)
```

#### Codex (跨进程追踪)
```rust
// ADR-001: SQ/EQ Protocol支持UI和Core在不同进程
core/src/protocol.rs

pub struct TracedOperation {
    pub trace_id: String,
    pub span_id: String,
    pub parent_span_id: Option<String>,
    pub operation: Operation,
}

impl TracedOperation {
    pub fn new(op: Operation) -> Self {
        Self {
            trace_id: generate_trace_id(),
            span_id: generate_span_id(),
            parent_span_id: None,
            operation: op,
        }
    }

    pub fn child(&self, op: Operation) -> Self {
        Self {
            trace_id: self.trace_id.clone(),
            span_id: generate_span_id(),
            parent_span_id: Some(self.span_id.clone()),
            operation: op,
        }
    }
}

// 序列化后通过SQ/EQ传递，保持追踪上下文
```

---

## 事件流与Wire日志

### MessageBus架构对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      MessageBus架构模式对比                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Nanobot (Pub/Sub)                   Gemini CLI (Event-Driven)              │
│  ──────────────────                  ─────────────────────────              │
│                                                                             │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │   Agent      │                    │   Scheduler  │                       │
│  └──────┬───────┘                    └──────┬───────┘                       │
│         │ publish                           │                               │
│         ▼                                   ▼                               │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │  MessageBus  │◄───────────────────│ MessageBus   │                       │
│  │  (Typed)     │    subscribe       │ (Deep Module)│                       │
│  └──────┬───────┘                    └──────┬───────┘                       │
│         │                                   │                               │
│    ┌────┴────┐                         ┌───┴───┐                           │
│    │         │                         │       │                           │
│    ▼         ▼                         ▼       ▼                           │
│ ┌────┐   ┌────┐                   ┌────┐   ┌────────┐                      │
│ │UI  │   │Tool│                   │UI  │   │Policy  │                      │
│ └────┘   └────┘                   └────┘   └────────┘                      │
│                                                                             │
│  OpenCode (Bus.ts)                   OpenClaw (Gateway)                     │
│  ─────────────────                   ──────────────────                     │
│                                                                             │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │   Bus.publish│                    │  WebSocket   │                       │
│  │   (Event)    │───────────────────►│  Gateway     │                       │
│  └──────────────┘                    │  Event Bus   │                       │
│         │                            └──────┬───────┘                       │
│         │                                   │                               │
│         ▼                                   ▼                               │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │ EventRecorder│                    │  Clients     │                       │
│  │ (Persistence)│                    │  (Broadcast) │                       │
│  └──────────────┘                    └──────────────┘                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. Nanobot (Pub/Sub + JSONL)

```typescript
// ADR-005: MessageBus Pub/Sub Pattern
// ADR-002: JSONL for append-only writes

class MessageBus<T extends Event> {
  private listeners: Map<EventType, Set<Handler<T>>> = new Map();
  private logFile: WriteStream;

  constructor(sessionKey: string) {
    // 会话键格式: slack:{chat_id}:{thread_ts}
    const logPath = `sessions/${sessionKey}.jsonl`;
    this.logFile = createWriteStream(logPath, { flags: 'a' });
  }

  publish(event: T): void {
    // 1. 持久化到JSONL
    const line = JSON.stringify({
      ...event,
      _meta: {
        timestamp: Date.now(),
        sequence: this.getNextSequence()
      }
    });
    this.logFile.write(line + '\n');

    // 2. 广播给订阅者
    const handlers = this.listeners.get(event.type);
    if (handlers) {
      handlers.forEach(h => h(event));
    }
  }

  subscribe(type: EventType, handler: Handler<T>): () => void {
    if (!this.listeners.has(type)) {
      this.listeners.set(type, new Set());
    }
    this.listeners.get(type)!.add(handler);

    return () => this.listeners.get(type)?.delete(handler);
  }

  // 重放功能
  async *replay(fromSequence?: number): AsyncGenerator<T> {
    const fileStream = createReadStream(this.logPath);
    const rl = createInterface(fileStream);

    for await (const line of rl) {
      const event = JSON.parse(line);
      if (fromSequence === undefined || event._meta.sequence >= fromSequence) {
        yield event as T;
      }
    }
  }
}
```

### 2. Gemini CLI (Deep Module)

```typescript
// ADR-002: Event-Driven Architecture
// Deep Module设计: 简单接口，复杂内部

interface MessageBus {
  // 极简接口 (2个方法)
  publish(event: Event): void;
  subscribe<T extends Event>(
    type: string,
    handler: (event: T) => void | Promise<void>
  ): Unsubscribe;
}

class GeminiMessageBus implements MessageBus {
  private subscriptions: Map<string, Set<Handler>> = new Map();
  private timeouts: Map<string, NodeJS.Timeout> = new Map();

  publish(event: Event): void {
    // 内部处理复杂性:
    // 1. Policy检查
    if (this.policyEngine.shouldBlock(event)) {
      this.emitBlocked(event);
      return;
    }

    // 2. 超时管理
    if (event.timeout) {
      this.timeouts.set(event.id, setTimeout(() => {
        this.emitTimeout(event);
      }, event.timeout));
    }

    // 3. 子Agent派生
    if (event.requiresSubAgent) {
      this.spawnSubAgent(event);
    }

    // 4. 请求-响应模式匹配
    if (event.correlationId) {
      this.resolvePending(event);
    }

    // 5. 实际分发
    this.dispatch(event);

    // 6. 遥测记录
    this.telemetry.record(event);
  }

  subscribe<T>(type: string, handler: Handler<T>): Unsubscribe {
    // 内部处理:
    // - 自动重连
    // - 背压处理
    // - 错误隔离
    return this.doSubscribe(type, handler);
  }
}
```

### 3. Codex (Wire Protocol)

```rust
// ADR-012: App Server JSON-RPC Protocol
// Wire日志用于调试跨进程通信

pub struct WireLogger {
    log_file: Option<File>,
}

impl WireLogger {
    pub fn log_request(&mut self, req: &JsonRpcRequest) {
        let entry = WireLogEntry {
            direction: Direction::Out,
            timestamp: Instant::now(),
            message: WireMessage::Request(req.clone()),
        };
        self.write(&entry);
    }

    pub fn log_response(&mut self, res: &JsonRpcResponse) {
        let entry = WireLogEntry {
            direction: Direction::In,
            timestamp: Instant::now(),
            message: WireMessage::Response(res.clone()),
        };
        self.write(&entry);
    }

    fn write(&mut self, entry: &WireLogEntry) {
        if let Some(ref mut file) = self.log_file {
            let json = serde_json::to_string(entry).unwrap();
            writeln!(file, "{}", json).ok();
        }
    }
}

// Wire日志格式
{
  "timestamp": "2026-03-18T10:00:00.000Z",
  "direction": "out",
  "message": {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "agent/action",
    "params": { ... }
  }
}
```

---

## 会话重放与检查点

### 重放机制对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      会话重放机制架构对比                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Kimi-CLI (D-Mail)                   Codex (SQLite)                         │
│  ─────────────────                   ───────────────                        │
│                                                                             │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │  SessionState│                    │  SQLite DB   │                       │
│  │  (JSON)      │                    │              │                       │
│  │              │                    │  sessions    │                       │
│  │ • messages[] │                    │  threads     │                       │
│  │ • checkpoint │                    │  events      │                       │
│  │ • sub_agents │                    │  approvals   │                       │
│  └──────┬───────┘                    └──────┬───────┘                       │
│         │                                   │                               │
│         │ save()                            │ checkpoint()                  │
│         ▼                                   ▼                               │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │  ~/.kimi/    │                    │  ~/.codex/   │                       │
│  │  sessions/   │                    │  state.db    │                       │
│  │  *.json      │                    │              │                       │
│  └──────────────┘                    └──────────────┘                       │
│                                                                             │
│  NanoClaw (Container)                Qwen Code (Recording)                  │
│  ─────────────────────               ─────────────────────                  │
│                                                                             │
│  ┌──────────────┐                    ┌──────────────┐                       │
│  │  Container   │                    │  ChatRecording│                      │
│  │  IPC Files   │                    │  Service     │                       │
│  │              │                    │              │                       │
│  │ /tmp/ipc/    │                    │ • logPrompts │                       │
│  │ • state.db   │                    │ • outfile    │                       │
│  │ • cursor.pos │                    │ • playback   │                       │
│  └──────────────┘                    └──────────────┘                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. Kimi-CLI D-Mail (时间旅行调试)

```python
# session_state.py - D-Mail机制
from dataclasses import dataclass, asdict
from datetime import datetime
from pathlib import Path
import json

@dataclass
class Checkpoint:
    """检查点: 可恢复的状态快照"""
    timestamp: datetime
    messages: List[Message]
    context_window: ContextWindow
    agent_states: Dict[str, AgentState]

@dataclass
class SessionState:
    """D-Mail: 会话状态完整持久化"""
    session_id: str
    messages: List[Message]
    sub_agents: Dict[str, SubAgent]
    checkpoints: List[Checkpoint]
    current_checkpoint: Optional[int]

    def save(self, path: Path):
        """保存到JSON文件"""
        state = {
            "session_id": self.session_id,
            "messages": [m.to_dict() for m in self.messages],
            "sub_agents": {k: v.to_dict() for k, v in self.sub_agents.items()},
            "checkpoints": [
                {
                    "timestamp": c.timestamp.isoformat(),
                    "message_count": len(c.messages),
                    "context_tokens": c.context_window.token_count
                }
                for c in self.checkpoints
            ],
            "current_checkpoint": self.current_checkpoint,
            "saved_at": datetime.utcnow().isoformat()
        }
        path.write_text(json.dumps(state, indent=2))

    def load(self, path: Path) -> "SessionState":
        """从JSON文件恢复"""
        data = json.loads(path.read_text())
        return SessionState(
            session_id=data["session_id"],
            messages=[Message.from_dict(m) for m in data["messages"]],
            sub_agents={k: SubAgent.from_dict(v) for k, v in data["sub_agents"].items()},
            checkpoints=[],  # 检查点按需加载
            current_checkpoint=data.get("current_checkpoint")
        )

    def time_travel(self, checkpoint_index: int):
        """时间旅行: 恢复到指定检查点"""
        if checkpoint_index >= len(self.checkpoints):
            raise ValueError(f"Invalid checkpoint: {checkpoint_index}")

        checkpoint = self.checkpoints[checkpoint_index]
        self.messages = checkpoint.messages.copy()
        self.current_checkpoint = checkpoint_index

        # 重置所有子Agent到该状态
        for agent in self.sub_agents.values():
            agent.reset_to(checkpoint.agent_states.get(agent.id))
```

### 2. Codex SQLite快照

```rust
// ADR-005: SQLite状态存储
pub struct StateStore {
    conn: Connection,
}

impl StateStore {
    pub fn new(path: &Path) -> Result<Self> {
        let conn = Connection::open(path)?;

        // 初始化表结构
        conn.execute_batch(r#"
            CREATE TABLE IF NOT EXISTS sessions (
                id TEXT PRIMARY KEY,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                config JSON NOT NULL
            );

            CREATE TABLE IF NOT EXISTS threads (
                id TEXT PRIMARY KEY,
                session_id TEXT NOT NULL,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (session_id) REFERENCES sessions(id)
            );

            CREATE TABLE IF NOT EXISTS events (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                thread_id TEXT NOT NULL,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                type TEXT NOT NULL,
                data JSON,
                FOREIGN KEY (thread_id) REFERENCES threads(id)
            );

            CREATE TABLE IF NOT EXISTS checkpoints (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                session_id TEXT NOT NULL,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                event_id INTEGER NOT NULL,
                state_snapshot JSON,
                FOREIGN KEY (session_id) REFERENCES sessions(id),
                FOREIGN KEY (event_id) REFERENCES events(id)
            );

            CREATE INDEX IF NOT EXISTS idx_events_thread
                ON events(thread_id, timestamp);
        "#)?;

        Ok(Self { conn })
    }

    pub fn create_checkpoint(&self, session_id: &str) -> Result<u64> {
        let snapshot = self.capture_state(session_id)?;

        self.conn.execute(
            "INSERT INTO checkpoints (session_id, event_id, state_snapshot)
             VALUES (?1, (SELECT MAX(id) FROM events WHERE thread_id IN
                (SELECT id FROM threads WHERE session_id = ?1)), ?2)",
            params![session_id, snapshot]
        )?;

        Ok(self.conn.last_insert_rowid() as u64)
    }

    pub fn restore_checkpoint(&self, checkpoint_id: u64) -> Result<StateSnapshot> {
        let snapshot: String = self.conn.query_row(
            "SELECT state_snapshot FROM checkpoints WHERE id = ?1",
            params![checkpoint_id],
            |row| row.get(0)
        )?;

        Ok(serde_json::from_str(&snapshot)?)
    }

    // 事件溯源重放
    pub fn replay_events(
        &self,
        thread_id: &str,
        from_checkpoint: Option<u64>
    ) -> Result<Vec<Event>> {
        let from_id = match from_checkpoint {
            Some(cp) => self.conn.query_row(
                "SELECT event_id FROM checkpoints WHERE id = ?1",
                params![cp],
                |row| row.get::<_, i64>(0)
            )?,
            None => 0
        };

        let mut stmt = self.conn.prepare(
            "SELECT type, data, timestamp FROM events
             WHERE thread_id = ?1 AND id > ?2
             ORDER BY id"
        )?;

        let events = stmt.query_map(params![thread_id, from_id], |row| {
            Ok(Event {
                event_type: row.get(0)?,
                data: row.get(1)?,
                timestamp: row.get(2)?
            })
        })?.collect::<Result<Vec<_>>>()?;

        Ok(events)
    }
}
```

### 3. Qwen Code (ChatRecording)

```typescript
// ChatRecordingService for replay
interface RecordingSession {
  id: string;
  startTime: number;
  messages: RecordedMessage[];
  toolCalls: RecordedToolCall[];
}

interface TelemetrySettings {
  enabled?: boolean;
  logPrompts?: boolean;      // 记录提示词
  outfile?: string;          // 录制输出文件
  useCollector?: boolean;    // 使用收集器服务
}

class ChatRecordingService {
  private currentSession?: RecordingSession;
  private settings: TelemetrySettings;

  startSession(): string {
    const sessionId = generateUUID();
    this.currentSession = {
      id: sessionId,
      startTime: Date.now(),
      messages: [],
      toolCalls: []
    };
    return sessionId;
  }

  recordMessage(role: 'user' | 'assistant' | 'system', content: string) {
    if (!this.currentSession || !this.settings.logPrompts) return;

    this.currentSession.messages.push({
      timestamp: Date.now(),
      role,
      content,
      tokenCount: estimateTokens(content)
    });
  }

  recordToolCall(name: string, args: any, result: any) {
    if (!this.currentSession) return;

    this.currentSession.toolCalls.push({
      timestamp: Date.now(),
      name,
      arguments: args,
      result,
      duration: 0 // 由调用者设置
    });
  }

  async saveRecording(): Promise<void> {
    if (!this.currentSession || !this.settings.outfile) return;

    const recording: ChatRecording = {
      version: '1.0',
      session: this.currentSession,
      metadata: {
        tool: 'qwen-code',
        version: getVersion(),
        platform: process.platform
      }
    };

    await fs.writeFile(
      this.settings.outfile,
      JSON.stringify(recording, null, 2)
    );
  }

  // 回放功能
  async replay(recordingPath: string, target: Agent): Promise<void> {
    const recording: ChatRecording = JSON.parse(
      await fs.readFile(recordingPath, 'utf-8')
    );

    for (const msg of recording.session.messages) {
      if (msg.role === 'user') {
        // 模拟用户输入
        await target.processInput(msg.content);
      }
      // 等待Agent响应完成
      await waitForAgentIdle(target);
    }
  }
}
```

---

## 遥测后端对比

### 后端类型矩阵

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        遥测后端架构选择                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐     │
│   │    OpenTelemetry │    │   Prometheus     │    │   Custom/Local   │     │
│   │                  │    │                  │    │                  │     │
│   │ • Cloud Trace    │    │ • Pull metrics   │    │ • SQLite         │     │
│   │ • Jaeger         │    │ • Time series    │    │ • JSONL          │     │
│   │ • OTLP           │    │ • Alerting       │    │ • Files          │     │
│   │                  │    │                  │    │                  │     │
│   │ Gemini CLI       │    │ ZeroClaw         │    │ Codex, NanoClaw  │     │
│   │ ZeroClaw (opt)   │    │ (feature-gated)  │    │ PicoClaw, Nanobot│     │
│   └──────────────────┘    └──────────────────┘    └──────────────────┘     │
│                                                                             │
│   选择指南:                                                                 │
│   • 云原生/GCP部署 → OpenTelemetry + Cloud Trace                          │
│   • 自托管/K8s    → Prometheus + Grafana                                  │
│   • 边缘/嵌入式   → SQLite/JSONL本地存储                                   │
│   • 零依赖要求    → 自定义文件格式                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. OpenTelemetry (云原生)

#### Gemini CLI (完整集成)
```typescript
// OpenTelemetry配置
import { NodeSDK } from '@opentelemetry/sdk-node';
import { CloudTraceExporter } from '@google-cloud/opentelemetry-cloud-trace-exporter';
import { CloudMonitoringExporter } from '@google-cloud/opentelemetry-cloud-monitoring-exporter';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { MeterProvider } from '@opentelemetry/sdk-metrics';

class TelemetryBackend {
  private sdk: NodeSDK;

  constructor(config: TelemetryConfig) {
    const traceExporter = new CloudTraceExporter({
      projectId: config.gcpProjectId,
    });

    const metricExporter = new CloudMonitoringExporter({
      projectId: config.gcpProjectId,
    });

    this.sdk = new NodeSDK({
      traceExporter,
      metricReader: metricExporter,
      instrumentations: [
        new HttpInstrumentation(),
        new FsInstrumentation(),
      ],
    });
  }

  start(): void {
    this.sdk.start();
  }

  shutdown(): Promise<void> {
    return this.sdk.shutdown();
  }
}
```

#### ZeroClaw (条件编译)
```rust
// Cargo.toml
[features]
default = []
observability-otel = ["opentelemetry", "opentelemetry-otlp"]
observability-prometheus = ["prometheus"]

[dependencies]
opentelemetry = { version = "0.21", optional = true }
opentelemetry-otlp = { version = "0.14", optional = true }
prometheus = { version = "0.13", optional = true }

// src/observability/mod.rs
#[cfg(feature = "observability-otel")]
pub mod otel;

#[cfg(feature = "observability-prometheus")]
pub mod prometheus;

// src/observability/otel.rs
#[cfg(feature = "observability-otel")]
pub struct OtelBackend {
    tracer: Tracer,
    meter: Meter,
}

#[cfg(feature = "observability-otel")]
impl OtelBackend {
    pub fn new(endpoint: &str) -> Result<Self> {
        let exporter = OtlpExporter::new()
            .with_endpoint(endpoint)
            .with_protocol(Protocol::Grpc)
            .build()?;

        let provider = TracerProvider::builder()
            .with_batch_exporter(exporter)
            .build();

        Ok(Self {
            tracer: provider.tracer("zeroclaw"),
            meter: global::meter("zeroclaw"),
        })
    }
}
```

### 2. Prometheus (自托管)

```rust
// ZeroClaw Prometheus实现
#[cfg(feature = "observability-prometheus")]
pub struct PrometheusBackend {
    registry: Registry,
    counters: HashMap<String, Counter>,
    histograms: HashMap<String, Histogram>,
    gauges: HashMap<String, Gauge>,
}

#[cfg(feature = "observability-prometheus")]
impl PrometheusBackend {
    pub fn new() -> Self {
        Self {
            registry: Registry::new(),
            counters: HashMap::new(),
            histograms: HashMap::new(),
            gauges: HashMap::new(),
        }
    }

    pub fn serve(addr: SocketAddr) -> Result<Server> {
        let encoder = TextEncoder::new();

        let server = Server::bind(&addr).serve(make_service_fn(|_| {
            let registry = GLOBAL_REGISTRY.clone();
            async move {
                Ok::<_, Infallible>(service_fn(move |_req| {
                    let metric_families = registry.gather();
                    let mut buffer = vec![];
                    encoder.encode(&metric_families, &mut buffer).unwrap();

                    async move {
                        Ok::<_, Infallible>(
                            Response::builder()
                                .status(200)
                                .header("Content-Type", encoder.format_type())
                                .body(Body::from(buffer))
                                .unwrap()
                        )
                    }
                }))
            }
        }));

        Ok(server)
    }
}
```

### 3. 本地存储 (SQLite/JSONL)

#### Nanobot (双层内存)
```markdown
# ADR: Two-Layer Memory System

## 结构
1. MEMORY.md - 工作记忆 (热数据)
2. HISTORY.md - 压缩历史 (冷数据)

## 整合流程
```
新消息 → MEMORY.md → (Phase 2异步) → HISTORY.md
```

## 优势
- 快速访问热数据
- 低成本存储历史
- 自然分层老化
```

#### PicoClaw (JSONL)
```go
// JSONL会话存储
func (s *SessionStore) AppendEvent(sessionID string, event Event) error {
    path := filepath.Join(s.baseDir, sessionID+".jsonl")
    file, err := os.OpenFile(path, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer file.Close()

    line, err := json.Marshal(event)
    if err != nil {
        return err
    }

    if _, err := file.Write(line); err != nil {
        return err
    }
    if _, err := file.WriteString("\n"); err != nil {
        return err
    }

    return nil
}

// 流式读取用于重放
func (s *SessionStore) ReplaySession(sessionID string, handler func(Event)) error {
    path := filepath.Join(s.baseDir, sessionID+".jsonl")
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        var event Event
        if err := json.Unmarshal(scanner.Bytes(), &event); err != nil {
            continue // 跳过损坏的行
        }
        handler(event)
    }

    return scanner.Err()
}
```

---

## 对比矩阵

### 1. 可观测性功能矩阵

| 项目 | 结构化日志 | 指标收集 | 分布式追踪 | 事件流 | 会话重放 | 遥测后端 |
|------|-----------|----------|-----------|--------|----------|----------|
| **CoPaw** | ✓ (标准库) | ✓ (基础) | ✗ | ✗ | ✗ | 文件 |
| **IronClaw** | ✓ (tracing) | ✓ (安全指标) | ✗ | ✗ | ✗ | 文件/DB |
| **NanoClaw** | ✓ (基础) | ✓ (双游标) | ✗ | ✗ | ✓ (SQLite) | SQLite |
| **OpenClaw** | ✓ (EventBus) | ✓ (Token) | ✗ | ✓ (Gateway) | ✗ | 文件 |
| **PicoClaw** | ✓ (zerolog) | ✓ (JSONL) | ✗ | ✗ | ✓ (JSONL) | JSONL |
| **ZeroClaw** | ✓ (tracing) | ✓ (可插拔) | △ (OTel opt) | ✗ | ✓ (多后端) | OTel/Prom |
| **Mimiclaw** | ✓ (ESP-IDF) | ✗ | ✗ | ✓ (MessageBus) | ✗ | Flash |
| **Nanobot** | ✓ (JSONL) | ✓ (基础) | ✗ | ✓ (Pub/Sub) | ✓ (JSONL) | JSONL |
| **Aider** | △ (基础) | ✓ (Token) | ✗ | ✗ | ✗ | 文件 |
| **Codex** | ✓ (tracing) | ✓ (SQLite) | △ (SQ/EQ) | ✓ (JSON-RPC) | ✓ (SQLite) | SQLite |
| **Kimi-CLI** | ✓ (loguru) | ✓ (基础) | ✗ | ✓ (WebSocket) | ✓ (D-Mail) | JSON |
| **OpenCode** | ✓ (EventBus) | ✓ (基础) | △ (Session) | ✓ (Bus.ts) | ✓ (Snapshot) | 文件 |
| **Qwen Code** | ✓ (基础) | ✓ (Token) | ✗ | ✗ | ✓ (Recording) | 文件 |
| **Gemini CLI** | ✓ (OTel) | ✓ (预算) | ✓ (OTel) | ✓ (MessageBus) | △ | Cloud Trace |
| **Pi-Mono** | ✗ | ✗ | ✗ | ✗ | ✗ | 控制台 |

**图例**: ✓ 完整支持 | △ 部分支持 | ✗ 不支持

### 2. 日志框架对比

| 项目 | 语言 | 框架 | 格式 | 轮转 | 级别控制 |
|------|------|------|------|------|----------|
| CoPaw | Python | 标准库 | 文本 | ✗ | 环境变量 |
| IronClaw | Rust | tracing | 文本/JSON | ✓ | Config |
| NanoClaw | Python | 标准库 | 文本 | ✗ | 硬编码 |
| OpenClaw | TypeScript | console | 文本 | ✗ | 环境变量 |
| PicoClaw | Go | zerolog | JSON | ✓ | 环境变量 |
| ZeroClaw | Rust | tracing | JSON | ✓ | TOML配置 |
| Mimiclaw | C | ESP-IDF | 文本 | ✗ | 编译时 |
| Nanobot | TypeScript | console | JSONL | ✓ | 配置 |
| Aider | Python | 标准库 | 文本 | ✗ | 参数 |
| Codex | Rust | tracing | 结构化 | ✓ | 环境变量 |
| Kimi-CLI | Python | loguru | 文本/JSON | ✓ | 配置 |
| OpenCode | TypeScript | Bus/Event | 对象 | ✗ | 代码 |
| Qwen Code | TypeScript | 标准 | 文本 | ✗ | 参数 |
| Gemini CLI | TypeScript | OpenTel | OTel | ✓ | OTel配置 |
| Pi-Mono | Python | 标准库 | 文本 | ✗ | 无 |

### 3. 会话存储格式对比

| 项目 | 格式 | 位置 | 原子性 | 压缩 | 查询能力 |
|------|------|------|--------|------|----------|
| Kimi-CLI | JSON | ~/.kimi/ | ✗ | ✗ | 无 |
| Codex | SQLite | ~/.codex/ | ✓ | ✗ | SQL |
| NanoClaw | SQLite | ~/.nanoclaw/ | ✓ | ✗ | SQL |
| PicoClaw | JSONL | ~/.picoclaw/ | ✓ | ✗ | 流式 |
| Nanobot | JSONL | ./sessions/ | ✓ | ✗ | 流式 |
| ZeroClaw | 多后端 | 配置决定 | ✓ | ✗ | 取决于后端 |
| Qwen Code | JSON | 配置路径 | ✗ | ✗ | 无 |
| OpenCode | Snapshot | 内存+文件 | △ | ✓ | 无 |

### 4. 重放机制对比

| 项目 | 机制 | 粒度 | 速度 | 依赖 |
|------|------|------|------|------|
| Kimi-CLI | D-Mail | 检查点 | 快 | 状态快照 |
| Codex | 事件溯源 | 事件级 | 中 | SQLite |
| NanoClaw | 双游标 | 消息级 | 快 | SQLite |
| PicoClaw | JSONL流 | 事件级 | 快 | 文件系统 |
| Nanobot | JSONL流 | 事件级 | 快 | 文件系统 |
| Qwen Code | Recording | 会话级 | 慢 | 重执行 |
| OpenCode | Snapshot | 状态级 | 快 | 内存 |

### 5. 遥测后端对比

| 后端 | 项目 | 部署复杂度 | 存储成本 | 查询能力 | 可视化 |
|------|------|-----------|----------|----------|--------|
| OpenTelemetry | Gemini CLI, ZeroClaw | 高 | 中 | 强 | Cloud Trace |
| Prometheus | ZeroClaw | 中 | 低 | 中 | Grafana |
| SQLite | Codex, NanoClaw, ZeroClaw | 低 | 极低 | SQL | 无 |
| JSONL | PicoClaw, Nanobot, Kimi-CLI | 极低 | 极低 | 流式 | 无 |
| 文件日志 | CoPaw, IronClaw, OpenClaw | 极低 | 低 | 文本 | 无 |
| Cloud Trace | Gemini CLI | 高 | 高 | 极强 | GCP Console |

---

## 推荐统一架构

### 分层可观测性架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    推荐统一可观测性架构                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        采集层 (Collection)                             │  │
│  │                                                                       │  │
│  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │  │
│  │   │   Logger    │  │  Metrics    │  │   Tracer    │  │   Events    │ │  │
│  │   │             │  │             │  │             │  │             │ │  │
│  │   │ • structured│  │ • counters  │  │ • spans     │  │ • bus       │ │  │
│  │   │ • levels    │  │ • gauges    │  │ • context   │  │ • pub/sub   │ │  │
│  │   │ • fields    │  │ • histograms│  │ • baggage   │  │ • replay    │ │  │
│  │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │  │
│  │          └─────────────────┴─────────────────┴─────────────────┘       │  │
│  │                              │                                         │  │
│  └──────────────────────────────┼─────────────────────────────────────────┘  │
│                                 │                                           │
│                                 ▼                                           │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        处理层 (Processing)                             │  │
│  │                                                                       │  │
│  │   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐   │  │
│  │   │   Enrichment    │    │   Sampling      │    │   Aggregation     │   │  │
│  │   │                 │    │                 │    │                 │   │  │
│  │   │ • add metadata  │    │ • head-based    │    │ • rollups       │   │  │
│  │   │ • attach context│    │ • tail-based    │    │ • windows       │   │  │
│  │   │ • normalize     │    │ • error bias    │    │ • cardinality   │   │  │
│  │   └─────────────────┘    └─────────────────┘    └─────────────────┘   │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                 │                                           │
│                                 ▼                                           │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        存储层 (Storage)                                │  │
│  │                                                                       │  │
│  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │  │
│  │   │   Hot       │  │   Warm      │  │   Cold      │  │   Archive   │ │  │
│  │   │             │  │             │  │             │  │             │ │  │
│  │   │ • Memory    │  │ • SQLite    │  │ • JSONL     │  │ • Object    │ │  │
│  │   │ • 1 hour    │  │ • 7 days    │  │ • 90 days   │  │ • 1 year+   │ │  │
│  │   │ • Real-time │  │ • Queryable │  │ • Batch     │  │ • Glacier   │ │  │
│  │   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                 │                                           │
│                                 ▼                                           │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        导出层 (Export)                                 │  │
│  │                                                                       │  │
│  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │  │
│  │   │   OTel      │  │ Prometheus  │  │   Cloud     │  │   Custom    │ │  │
│  │   │   Exporter  │  │   Endpoint  │  │   Provider  │  │   Webhook   │ │  │
│  │   │             │  │             │  │             │  │             │ │  │
│  │   │ • gRPC      │  │ • /metrics  │  │ • GCP       │  │ • HTTP      │ │  │
│  │   │ • HTTP      │  │ • Pull      │  │ • AWS       │  │ • Queue     │ │  │
│  │   │ • Batch     │  │ • Scrape    │  │ • Azure     │  │ • Stream    │ │  │
│  │   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 配置建议

```yaml
# unified-observability.yaml
observability:
  # 日志配置
  logging:
    level: info  # debug, info, warn, error
    format: json  # json, logfmt, text
    output: stdout  # stdout, file, both
    file:
      path: /var/log/agent/app.log
      rotation: daily  # hourly, daily, size
      max_size: 100MB
      max_files: 7
    fields:
      - timestamp
      - level
      - message
      - trace_id
      - span_id
      - agent_id
      - session_id

  # 指标配置
  metrics:
    enabled: true
    collection_interval: 15s
    categories:
      - token_usage
      - latency
      - throughput
      - errors
      - tool_calls
    labels:
      - agent_id
      - model
      - provider
      - channel

  # 追踪配置
  tracing:
    enabled: true
    sampler:
      type: probabilistic  # always, never, probabilistic, rate_limiting
      probability: 0.1  # 10%采样
    propagation:
      - tracecontext  # W3C标准
      - baggage
    span_limits:
      max_events: 128
      max_links: 32
      max_attributes: 128

  # 事件流配置
  events:
    bus_type: memory  # memory, redis, nats
    persistence:
      enabled: true
      format: jsonl  # jsonl, sqlite
      path: /var/lib/agent/events/
    replay:
      enabled: true
      buffer_size: 1000

  # 会话重放配置
  replay:
    checkpoints:
      enabled: true
      interval: 5m  # 每5分钟自动检查点
      max_checkpoints: 12  # 保留最近12个
    storage:
      type: sqlite  # sqlite, json, jsonl
      path: /var/lib/agent/sessions/
    compression:
      enabled: true
      algorithm: zstd

  # 遥测后端配置
  backends:
    # 本地存储 (默认启用)
    local:
      enabled: true
      type: sqlite
      path: /var/lib/agent/telemetry.db

    # OpenTelemetry (可选)
    otel:
      enabled: false
      endpoint: http://localhost:4317
      protocol: grpc  # grpc, http
      headers:
        api-key: ${OTEL_API_KEY}
      export_interval: 10s

    # Prometheus (可选)
    prometheus:
      enabled: false
      port: 9090
      path: /metrics

    # 云提供商 (可选)
    cloud:
      provider: gcp  # gcp, aws, azure
      project_id: ${GCP_PROJECT_ID}
      credentials: /etc/agent/gcp-key.json

  # 告警配置
  alerting:
    enabled: true
    rules:
      - name: high_error_rate
        condition: error_rate > 0.05  # 5%错误率
        duration: 5m
        severity: warning
        channels: [slack, email]

      - name: high_latency
        condition: p99_latency > 5000  # 5秒
        duration: 2m
        severity: critical
        channels: [pagerduty, slack]

      - name: token_budget_exceeded
        condition: daily_tokens > 1000000
        duration: 0s
        severity: warning
        channels: [email]
```

### 实现推荐

| 场景 | 推荐配置 | 理由 |
|------|----------|------|
| **开发调试** | `level: debug`, `format: text`, `otel: false` | 可读性优先，低开销 |
| **生产云原生** | `otel: true`, `prometheus: true`, `sqlite: true` | 全栈可观测，本地兜底 |
| **边缘/嵌入式** | `sqlite: true`或`jsonl: true`, 其他`false` | 资源受限，本地存储 |
| **高吞吐量** | `sampling: 0.01`, `batch: true`, `compression: true` | 采样降低开销，批处理优化 |
| **安全敏感** | `audit: true`, `encryption: true`, `retention: 90d` | 完整审计，加密存储 |

### 迁移路径

```
当前状态 → 目标架构

Level 0 (控制台日志)
    ↓
添加结构化日志框架 (tracing/zerolog/loguru)
    ↓
Level 1 (结构化日志 + 文件存储)
    ↓
添加指标收集 (Token/延迟/成功率)
添加SQLite/JSONL持久化
    ↓
Level 2 (指标 + 事件流 + 本地存储)
    ↓
添加分布式追踪 (OpenTelemetry)
添加MessageBus事件流
    ↓
Level 3 (全链路追踪 + 多后端导出)
```

---

## 附录: 快速参考

### 日志级别使用指南

| 级别 | 使用场景 | 示例 |
|------|----------|------|
| DEBUG | 开发调试，详细信息 | 函数进入/退出，变量值 |
| INFO | 正常操作记录 | 会话开始，工具调用，完成 |
| WARN | 非致命问题 | 重试，降级，配置缺失 |
| ERROR | 操作失败 | API错误，工具执行失败 |
| FATAL | 系统级错误 | 初始化失败，数据损坏 |

### 追踪Span命名规范

```
<component>.<operation>

示例:
- agent.process_message
- llm.chat_completion
- tool.file_read
- db.query_execute
- channel.send_message
```

### 指标命名规范

```
<namespace>_<entity>_<metric>_<unit>

示例:
- agent_messages_total
- llm_tokens_input_count
- tool_execution_duration_ms
- db_connection_pool_size
- channel_webhook_latency_ms
```

### 事件类型命名规范

```
<domain>.<action>.<status>

示例:
- message.received
- message.processed
- tool.called
- tool.completed
- tool.failed
- session.created
- session.ended
```

---

*文档结束*
