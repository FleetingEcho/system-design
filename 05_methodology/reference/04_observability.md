# 可观测性（Observability）

## TL;DR

可观测性回答一个问题：**系统出问题时，你怎么知道？出在哪里？为什么？**

三支柱：
- **Metrics（指标）**：系统健康状态的数字快照，用于告警和趋势分析
- **Logs（日志）**：事件的详细文字记录，用于问题定位
- **Traces（链路追踪）**：一次请求跨多个服务的完整调用链，用于性能瓶颈定位

面试中主动提出可观测性设计，是区分"会做系统"和"只会画架构图"的重要信号。

---

## 为什么需要可观测性

```
没有可观测性的系统：
  用户投诉："你们网站挂了"
  工程师："我看看……哦，数据库连接池满了，已经修了"
  时间：30 分钟

有可观测性的系统：
  告警在用户发现之前触发："DB 连接池使用率 95%，P99 延迟突增到 2s"
  工程师查 Trace：哪个接口、哪条 SQL、慢在哪一步
  时间：5 分钟
```

可观测性的目标不是"收集所有数据"，而是**在正确的时间告诉你正确的事**。

---

## 支柱一：Metrics（指标）

### 什么是 Metrics

Metrics 是对系统状态的**数值化描述**，随时间变化形成时间序列：

```
http_requests_total{method="GET", status="200"} → 每秒请求数
http_request_duration_seconds{p99} → P99 延迟
db_connections_active → 当前活跃连接数
kafka_consumer_lag → 消费者积压消息数
```

### 四类黄金指标（Google SRE Book）

面试必背，适用于任何系统：

| 指标 | 含义 | 示例 |
|------|------|------|
| **Latency（延迟）** | 请求处理时间（P50/P99/P999）| API P99 < 200ms |
| **Traffic（流量）** | 系统负载（QPS/TPS/带宽）| 当前 5000 QPS |
| **Errors（错误率）** | 失败请求比例 | 5xx 错误率 < 0.1% |
| **Saturation（饱和度）** | 关键资源使用率 | CPU 75%，连接池 60% |

### RED 方法（微服务场景更常用）

每个服务追踪三个指标：

```
Rate     — 每秒请求数（这个服务处理了多少流量？）
Errors   — 错误率（这个服务有多少请求失败？）
Duration — 延迟分布（这个服务慢不慢？）
```

RED 是针对服务的，四黄金指标是针对整个系统的，两者配合使用。

### Metrics 的技术栈

```
采集：
  Prometheus（开源，Pull 模式，应用暴露 /metrics 端点）
  StatsD（Push 模式，UDP 发送，低开销）
  CloudWatch / Datadog（云托管，无需自建）

存储：
  Prometheus TSDB（时间序列数据库，本地存储）
  InfluxDB（专门的时序数据库）
  Thanos / VictoriaMetrics（Prometheus 的分布式扩展）

可视化：
  Grafana（Dashboard，配合 Prometheus 使用）
  Datadog Dashboard

告警：
  Alertmanager（Prometheus 配套，支持路由/静默/去重）
  PagerDuty / OpsGenie（On-call 轮值和升级）
```

### Node.js 代码示例

```typescript
import { Counter, Histogram, Registry } from 'prom-client';

const registry = new Registry();

// 请求计数器
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status'],
  registers: [registry],
});

// 延迟直方图（自动计算 P50/P95/P99）
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'path'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [registry],
});

// Express 中间件
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer({
    method: req.method,
    path: req.route?.path ?? req.path,
  });

  res.on('finish', () => {
    httpRequestsTotal.inc({
      method: req.method,
      path: req.route?.path ?? req.path,
      status: res.statusCode.toString(),
    });
    end(); // 记录耗时
  });

  next();
});

// Prometheus 抓取端点
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', registry.contentType);
  res.send(await registry.metrics());
});
```

---

## 支柱二：Logs（日志）

### 结构化日志 vs 非结构化日志

```
❌ 非结构化（难以机器解析）：
  [2024-01-01 12:00:01] User 123 failed to login from IP 1.2.3.4

✅ 结构化（JSON，可以按字段过滤）：
  {
    "timestamp": "2024-01-01T12:00:01Z",
    "level": "warn",
    "event": "login_failed",
    "userId": 123,
    "ip": "1.2.3.4",
    "reason": "invalid_password",
    "traceId": "abc-123-xyz"   ← 关联到 Trace
  }
```

结构化日志能直接被 Elasticsearch 索引，支持 `event=login_failed AND userId=123` 这类精确查询。

### 日志级别的正确使用

```
ERROR：需要立刻处理的问题（数据库连不上、外部 API 一直失败）
WARN： 异常但不需要立刻处理（重试成功、降级触发、配置不推荐项）
INFO： 重要的业务事件（用户注册、订单创建、支付成功）
DEBUG：开发调试信息（函数入参、中间状态），生产环境默认关闭

原则：ERROR 触发告警，WARN 触发周报，INFO 业务审计，DEBUG 只在需要时打开
```

### 日志技术栈（ELK Stack）

```
Logstash / Filebeat
  ↓ 采集（从应用日志文件或标准输出读取）
Elasticsearch
  ↓ 存储 + 索引（按字段建倒排索引，支持全文检索）
Kibana
  ↓ 查询 + 可视化（Discover 页面搜日志，Dashboard 看趋势）

云托管替代：
  AWS CloudWatch Logs
  Datadog Logs
  Loki（Grafana 生态，比 ELK 轻量，按 Label 而非全文索引）
```

### 日志采样（高流量场景）

```
问题：每秒 10 万请求，每条日志 1 KB → 100 MB/s 日志写入
      存储和查询成本极高

解决：Head-based Sampling（头部采样）
  只记录 1% 的请求（随机抽样），节省 99% 存储
  
  但正常 1% 请求的日志用处不大 —— 我们更关心慢请求和错误

Tail-based Sampling（尾部采样，更智能）：
  请求完成后，根据结果决定是否保留日志：
    有错误（5xx）→ 必须保留
    延迟 > 1s → 必须保留
    正常快速请求 → 1% 概率保留
  
  效果：异常请求 100% 保留，正常请求只保留 1%，
        存储省了 95%，但关键信息不丢失
```

---

## 支柱三：Traces（链路追踪）

### 为什么需要 Traces

```
用户请求慢，Metrics 告诉你"P99 = 3s"，但慢在哪里？

微服务调用链：
  API Gateway → User Service → Order Service → Payment Service
                                     ↓
                              → Inventory Service
  
  哪一步慢？User Service 的 DB 查询？Payment Service 的外部 API？

Trace 告诉你完整调用链的每一步耗时：
  API Gateway: 5ms
  User Service: 10ms (DB查询: 8ms)
  Order Service: 2800ms  ← 这里！
    └── Inventory Service: 2750ms  ← 更具体：这里！
  Payment Service: 50ms
  总计: 2865ms
```

### Span 和 Trace

```
Trace：一次完整请求的全部调用记录（由多个 Span 组成）
Span：调用链中的一个操作单元（一次 RPC 调用、一次 DB 查询）

Span 数据结构：
{
  traceId: "abc-123",          // 整条链路的唯一 ID
  spanId: "def-456",           // 本 Span 的 ID
  parentSpanId: "ghi-789",     // 父 Span（谁调用了我）
  operationName: "SELECT users",
  startTime: 1700000000100,
  duration: 8,                 // ms
  tags: {
    "db.type": "mysql",
    "db.statement": "SELECT * FROM users WHERE id=?",
    "error": false
  }
}
```

### Trace 传播（Propagation）

```
请求在服务间传递时，需要把 traceId 带过去：

Service A 调用 Service B：
  HTTP Header: traceparent: 00-abc123-def456-01
                              ↑      ↑      ↑
                           版本  traceId  spanId

Service B 收到后：
  解析 traceId（用来关联到同一条链路）
  创建新的 Span（parentSpanId = 上游的 spanId）

标准：W3C Trace Context（现代标准，OpenTelemetry 使用）
旧标准：B3 Propagation（Zipkin）、X-B3-TraceId（Uber Jaeger）
```

### Traces 技术栈

```
SDK（埋点）：
  OpenTelemetry（现代标准，厂商中立，支持自动埋点）
  Jaeger Client / Zipkin Client（老标准）

收集器：
  OpenTelemetry Collector（接收、处理、转发）

存储 + 查询：
  Jaeger（开源，Uber 开发，支持 Elasticsearch/Cassandra 存储）
  Zipkin（开源，Twitter 开发）
  Tempo（Grafana 生态，轻量）
  Datadog APM / AWS X-Ray（云托管）
```

---

## 三支柱的关联

三者相互补充，排查问题时通常这样走：

```
1. Metrics 触发告警：
   "Order Service 错误率突然从 0.1% 升到 5%"
        ↓
2. Logs 定位具体错误：
   搜索 level=ERROR AND service=order-service
   发现：大量 "Connection timeout to inventory-service"
        ↓
3. Traces 找到根因：
   找一条慢请求的 Trace
   发现：Inventory Service 的某个 DB 查询从平时 5ms 变成 3000ms
   → DBA 一看：那张表的索引昨晚被误删了

Metrics → 发现问题（What）
Logs    → 定位问题（Where）
Traces  → 分析根因（Why）
```

**关联的关键：traceId**

```
每条 Log 里都带上 traceId：
  { "level": "error", "msg": "DB timeout", "traceId": "abc-123" }

从 Metrics 告警 → 找到告警时间窗口 → 搜 Logs（该时间段的 ERROR）
→ 拿到 traceId → 在 Jaeger 里查完整 Trace
```

---

## 告警设计

### 告警的核心原则

**只为需要人工介入的事情告警。**

```
好告警：
  "Order Service 错误率 > 1% 持续 5 分钟" → On-call 工程师需要看
  "主数据库磁盘使用率 > 85%" → 需要扩容

坏告警（告警疲劳，工程师开始忽略告警）：
  "CPU 偶发超过 80% 1 秒" → 可能是正常峰值
  "某个健康检查失败一次后立即恢复" → 抖动，无需人工介入
```

### SLO 驱动的告警（最佳实践）

不要对"CPU > 80%"这种底层指标告警，而是对**用户体验**告警：

```
SLO 示例：
  "99.9% 的请求在 500ms 内返回，错误率 < 0.1%，统计周期 30 天"

Error Budget（错误预算）：
  30 天内允许的"坏请求"配额 = 0.1% × 30天 × 86400秒/天 × QPS
  
  Error Budget 消耗速率告警：
  "当前 1 小时的错误率，按此速率，会在 2 天内耗尽 30 天的 Error Budget"
  → 这才是真正需要介入的信号（Burn Rate Alert）

Burn Rate = 实际错误率 / SLO 允许错误率
  Burn Rate = 1：刚好按预算消耗（不告警）
  Burn Rate = 10：10 倍速消耗，30 天预算 3 天耗尽（告警！）
  Burn Rate = 100：100 倍速，30 天预算 7 小时耗尽（紧急告警！）
```

### 告警分级

```
P1（立刻响应，15 分钟内）：
  用户直接受影响（系统完全不可用、数据丢失风险）
  通知方式：电话 + PagerDuty + Slack

P2（1 小时内）：
  部分功能受损（某个非核心功能失败）
  通知方式：PagerDuty + Slack

P3（工作时间内）：
  性能降级、资源接近上限（但未超）
  通知方式：Slack 消息 + 周报
```

---

## 面试中如何讨论可观测性

设计完架构后，主动补充：

```
"这个系统的可观测性设计：

Metrics：
  对每个服务暴露 RED 指标（Rate/Error/Duration）
  关键业务指标：订单创建成功率、支付成功率
  资源指标：数据库连接池使用率、Kafka 消费者 Lag
  用 Prometheus + Grafana 做 Dashboard

Logs：
  结构化 JSON 日志，每条带 traceId
  ERROR/WARN 日志实时聚合到 ELK，支持告警
  日志采样：错误请求 100% 保留，正常请求 1% 采样

Traces：
  OpenTelemetry 自动埋点，覆盖所有服务间调用和 DB 查询
  Jaeger 存储，保留 7 天
  与 Logs 通过 traceId 关联

告警：
  P1：SLO Burn Rate > 10（2 天内耗尽错误预算），电话 On-call
  P2：Kafka Lag 持续增长超过 10 分钟，Slack 通知
  P3：磁盘使用率 > 80%，写入 Ticket"
```

---

## 常见陷阱

1. **只监控基础设施，不监控业务指标**：CPU/内存全绿，但"支付成功率"悄悄从 99.5% 掉到 95%，因为没有业务指标告警，没人发现。必须同时监控系统指标和业务指标

2. **告警太多导致疲劳**：每天几十条告警，On-call 开始忽略。告警要定期 review，关掉没有 Action 的告警。一个好指标：On-call 接到告警后，95% 的情况需要做什么操作，5% 可以忽略

3. **日志没有 traceId**：排查问题时，日志和 Trace 无法关联，只能靠时间戳猜。每条日志必须带 traceId，框架层面强制注入（不能依赖开发者手动添加）

4. **Metrics 粒度太粗**：只记录 `http_requests_total`，不带 `path` label，无法区分是哪个接口慢。但粒度太细（如每个 userId 一个 label）会导致 label cardinality 爆炸，Prometheus 内存撑不住。原则：label 的取值集合要有上限（接口路径 OK，用户 ID 不行）

5. **只看平均延迟**：平均延迟 100ms 看起来很好，但 P99 可能是 3 秒。必须关注百分位数（P99、P999），平均值会掩盖长尾问题

---

## 面试 Q&A

### 简单

**Q: 可观测性的三个支柱是什么，各自解决什么问题？**

A: Metrics（指标）：数值化的系统状态，随时间变化，用于告警和趋势分析，回答"系统整体健不健康"；Logs（日志）：事件的详细文字记录，用于问题定位，回答"具体发生了什么事"；Traces（链路追踪）：一次请求跨多个服务的完整调用链，用于性能瓶颈定位，回答"慢在哪一步"。三者配合：Metrics 发现问题，Logs 定位错误，Traces 找到根因。

**Q: 为什么要关注 P99 延迟而不是平均延迟？**

A: 平均延迟会被大量正常请求"稀释"，掩盖长尾慢请求。比如 10000 次请求，9900 次 10ms，100 次 5000ms，平均延迟 = (9900×10 + 100×5000)/10000 = 59ms，看起来很好，但实际上 1% 的用户体验了 5 秒的等待。P99 = 5000ms 直接暴露了这个问题。SLO 通常定义在 P99 上，因为它反映了"最差的正常用户体验"。

---

### 中等

**Q: 什么是 Error Budget，如何用它驱动告警？**

A: Error Budget 是 SLO 允许的"失败额度"。比如 SLO 是"99.9% 的请求成功"，则 30 天内允许 0.1% 的请求失败。如果当前错误率超过 SLO，就是在"消耗 Error Budget"。Burn Rate 告警不对"当前错误率"告警，而是对"按当前速率，Error Budget 会在多久内耗尽"告警：Burn Rate = 10 意味着 10 倍速消耗，3 天内耗尽 30 天预算，这才触发告警。这样避免了对短暂抖动告警，只在真正影响 SLO 达成时介入。

**Q: 微服务里的链路追踪如何实现跨服务的 traceId 传递？**

A: 每个服务在发出 HTTP/gRPC 请求时，把 traceId 和当前 spanId 注入请求头（标准是 W3C Trace Context 的 `traceparent` 头）。下游服务收到请求后，从 Header 里解析出 traceId，创建新的 Span（parentSpanId 设为上游的 spanId），继续传递。OpenTelemetry 的 SDK 会自动处理这个过程（自动埋点），不需要开发者手动传递。最终所有 Span 汇聚到 Jaeger/Zipkin，按 traceId 重组成完整的调用树。

---

### 困难

**Q: 设计一个日处理 10 亿请求系统的可观测性方案，在控制成本的同时不丢失关键信息。**

A: 分层策略：

**Metrics（全量采集，成本低）：** Prometheus 的时间序列数据压缩率高，10 亿请求 × RED 指标 × 1 分钟粒度 ≈ 每天几 GB，完全可以接受。用 VictoriaMetrics 替代 Prometheus 单机，支持水平扩展，存储效率更高。

**Logs（采样，控制量）：** 正常请求按 0.1% 采样（1000 亿 × 0.001 = 1 亿条/天），ERROR 和慢请求（>P99）100% 保留。用 Loki 替代 ELK，Loki 按 Label 索引而不是全文索引，存储成本低 10 倍以上。日志保留 7 天（只需热查询），冷归档到 S3（Glacier，成本极低）。

**Traces（尾部采样，只保留有价值的）：** 使用 OpenTelemetry Collector 的 Tail Sampling Processor：正常快速请求 1% 采样，有错误/延迟 > P99/有特定业务标记的请求 100% 保留。Trace 数据量减少 95%，但 100% 覆盖了需要排查的场景。存储在 Tempo，只保留 3 天（问题通常当天发现当天查）。

**成本估算（粗略）：** Metrics: ~$500/月（VictoriaMetrics 托管）；Logs: ~$2000/月（Loki + S3）；Traces: ~$1000/月（Tempo + 采样后数据量可控）。总计 ~$3500/月，对 10 亿请求/天的系统来说是合理的。

---

## 关联文档

- [../design_process/01_framework.md](../design_process/01_framework.md) — 面试框架（可观测性是加分项）
- [../../01_fundamentals/01_metrics.md](../../01_fundamentals/01_metrics.md) — SLA/SLO/SLI 定义
- [../../04_distributed/04_fault_tolerance.md](../../04_distributed/04_fault_tolerance.md) — 熔断器（告警触发后的自动响应）
