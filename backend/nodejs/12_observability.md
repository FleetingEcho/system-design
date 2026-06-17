# Node.js 可观测性（Observability）

> OpenTelemetry for Node.js：Traces、Metrics、Logs 三支柱，分布式追踪，生产监控。
> 现有 `06_production_patterns.md` 覆盖了日志和健康检查，本篇专注分布式追踪和指标体系。

---

## 三支柱：Traces / Metrics / Logs

```
Metrics（指标）：
  → 聚合数值（QPS、延迟 P99、错误率、CPU 使用率）
  → 告警触发的数据源
  → 工具：Prometheus + Grafana

Logs（日志）：
  → 具体事件记录（请求详情、错误堆栈）
  → 排查具体 Bug 时看
  → 工具：Pino + ELK / Loki

Traces（追踪）：
  → 跨服务的请求链路（A 调用 B 调用 C，哪一步慢了？）
  → 性能瓶颈定位
  → 工具：OpenTelemetry + Jaeger / Tempo

三者关系：
  Metrics 告诉你"有问题了"
  Logs   告诉你"这个请求发生了什么"
  Traces 告诉你"整条链路在哪卡了"
```

---

## OpenTelemetry 初始化（SDK 配置）

```typescript
// src/lib/telemetry.ts —— 必须在 app 代码之前加载（instrumentation.ts）
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

// 自动 instrumentation：HTTP、Express、Prisma、Redis、pg 等
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: process.env.SERVICE_NAME ?? 'api',
    [SEMRESATTRS_SERVICE_VERSION]: process.env.APP_VERSION ?? '0.0.0',
    'deployment.environment': process.env.NODE_ENV ?? 'development',
  }),

  // Trace 导出到 OTLP 收集器（Jaeger/Tempo）
  traceExporter: new OTLPTraceExporter({
    url: `${process.env.OTEL_EXPORTER_OTLP_ENDPOINT}/v1/traces`,
    headers: {
      Authorization: `Bearer ${process.env.OTEL_API_KEY ?? ''}`,
    },
  }),

  // Metrics 导出到 OTLP（Prometheus 也可）
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: `${process.env.OTEL_EXPORTER_OTLP_ENDPOINT}/v1/metrics`,
    }),
    exportIntervalMillis: 15_000,  // 每 15s 导出
  }),

  // 自动为 HTTP/Express/Prisma/Redis 添加 span
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false },  // 关掉噪音
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingRequestHook: (req) => {
          // 不追踪健康检查（否则 span 数据量爆炸）
          return req.url === '/health' || req.url === '/metrics';
        },
      },
    }),
  ],
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown().catch(console.error);
});
```

```typescript
// 启动时加载（Node.js --require 或 instrumentation.ts）
// package.json scripts:
// "start": "node --require ./dist/lib/telemetry.js dist/server.js"

// Next.js / TSX: 使用 instrumentation.ts（Next.js 13+ 内置支持）
// export async function register() {
//   if (process.env.NEXT_RUNTIME === 'nodejs') {
//     await import('./lib/telemetry');
//   }
// }
```

---

## 自定义 Span（手动埋点）

```typescript
// src/lib/tracing.ts
import { trace, SpanStatusCode, context, propagation } from '@opentelemetry/api';

const tracer = trace.getTracer('api-service', '1.0.0');

// 创建自定义 span（标记关键业务操作）
export async function withSpan<T>(
  name: string,
  attributes: Record<string, string | number | boolean>,
  fn: () => Promise<T>
): Promise<T> {
  return tracer.startActiveSpan(name, { attributes }, async (span) => {
    try {
      const result = await fn();
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (err: any) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      span.recordException(err);
      throw err;
    } finally {
      span.end();
    }
  });
}

// 使用
async function processOrder(orderId: string) {
  return withSpan(
    'order.process',
    { 'order.id': orderId, 'order.source': 'web' },
    async () => {
      const order = await withSpan('db.getOrder', { 'db.table': 'orders' }, () =>
        prisma.order.findUniqueOrThrow({ where: { id: orderId } })
      );

      const payment = await withSpan('payment.charge', { 'payment.amount': order.amount }, () =>
        stripeService.charge(order)
      );

      return { order, payment };
    }
  );
}
```

---

## TraceID 跨服务传播

```typescript
// 服务 A 调用服务 B 时，自动传递 TraceID（W3C Trace Context）
// 自动 instrumentation 已处理 HTTP 请求的传播，手动调用时：

import { context, propagation } from '@opentelemetry/api';

async function callDownstreamService(data: unknown) {
  const headers: Record<string, string> = {};
  // 将当前 span 的 context 注入到 HTTP headers
  propagation.inject(context.active(), headers);
  // headers 现在包含: traceparent: "00-traceId-spanId-flags"

  const response = await fetch('http://order-service/process', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', ...headers },
    body: JSON.stringify(data),
  });
  return response.json();
}

// 服务 B 端：从 headers 提取 context，继续同一条 trace
// 自动 instrumentation 会处理，手动处理：
import { propagation, context } from '@opentelemetry/api';

app.post('/process', (req, res, next) => {
  const parentContext = propagation.extract(context.active(), req.headers);
  context.with(parentContext, () => {
    // 在这个 context 中创建的 span 会自动连接到上游 trace
    next();
  });
});
```

---

## Prometheus 指标（业务指标）

```typescript
// src/lib/metrics.ts
import { MeterProvider, ValueType } from '@opentelemetry/sdk-metrics';
import { metrics } from '@opentelemetry/api';

const meter = metrics.getMeter('api-service');

// 计数器（单调递增：请求总数、错误总数）
const httpRequestCounter = meter.createCounter('http.requests.total', {
  description: 'Total HTTP requests',
  unit: '1',
});

// 直方图（分布：延迟、响应大小）
const httpDurationHistogram = meter.createHistogram('http.request.duration', {
  description: 'HTTP request duration',
  unit: 'ms',
  advice: {
    explicitBucketBoundaries: [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000],
  },
});

// UpDownCounter（可增可减：当前连接数、队列深度）
const activeConnectionsGauge = meter.createUpDownCounter('http.connections.active', {
  description: 'Active HTTP connections',
});

// 业务指标
const ordersProcessed = meter.createCounter('orders.processed.total');
const orderValue = meter.createHistogram('orders.value', { unit: 'USD' });

// Express 中间件：自动记录请求指标
export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const startTime = Date.now();
  activeConnectionsGauge.add(1);

  res.on('finish', () => {
    const duration = Date.now() - startTime;
    const attrs = {
      'http.method': req.method,
      'http.route': req.route?.path ?? req.path,
      'http.status_code': res.statusCode,
    };

    httpRequestCounter.add(1, attrs);
    httpDurationHistogram.record(duration, attrs);
    activeConnectionsGauge.add(-1);
  });

  next();
}
```

---

## 采样策略

```typescript
// 生产环境不能 100% 采样（流量大时数据量太大）
import {
  ParentBasedSampler,
  TraceIdRatioBased,
  AlwaysOnSampler,
} from '@opentelemetry/sdk-trace-base';

// 采样策略：
const sampler = new ParentBasedSampler({
  // 如果上游采样了，跟随上游决定（保持 trace 完整性）
  root: new TraceIdRatioBased(
    process.env.NODE_ENV === 'production' ? 0.1 : 1.0  // 生产 10%，开发 100%
  ),
});

// 高级：对错误请求 100% 采样，正常请求 10% 采样
class ErrorAwareSampler implements Sampler {
  shouldSample(context: Context, traceId: string, spanName: string, spanKind: SpanKind) {
    // 错误路径始终采样
    const parentSpan = trace.getSpan(context);
    if (parentSpan?.isRecording()) {
      // 检查是否有错误标记
    }
    // 默认 10% 采样
    return Math.random() < 0.1
      ? { decision: SamplingDecision.RECORD_AND_SAMPLED }
      : { decision: SamplingDecision.NOT_RECORD };
  }
}
```

---

## 关联 Trace ID 到日志

```typescript
// src/lib/logger.ts —— 在日志中注入 TraceID（关联 Traces 和 Logs）
import pino from 'pino';
import { trace, context } from '@opentelemetry/api';

export function getLogger() {
  return pino({
    mixin() {
      // 从当前 OpenTelemetry context 获取 trace/span ID
      const span = trace.getActiveSpan();
      if (!span) return {};

      const spanContext = span.spanContext();
      return {
        traceId: spanContext.traceId,
        spanId: spanContext.spanId,
      };
    },
    formatters: {
      level(label) { return { level: label }; },
    },
  });
}

// 日志输出：
// {
//   "level": "info",
//   "msg": "User created",
//   "userId": "abc123",
//   "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",  ← 与 Jaeger 中的 trace 关联
//   "spanId": "00f067aa0ba902b7"
// }
// 在 Grafana Loki 中点击 traceId 可直接跳转到 Jaeger 的 trace 详情
```

---

## 告警规则（Prometheus AlertManager）

```yaml
# alerts.yaml
groups:
  - name: api-service
    rules:
      # 高错误率
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% for 2 minutes"

      # P99 延迟过高
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_bucket[5m])) > 1000
        for: 5m
        annotations:
          summary: "P99 latency > 1s"

      # Event Loop 阻塞
      - alert: EventLoopLag
        expr: nodejs_eventloop_lag_seconds > 0.1
        for: 1m
        annotations:
          summary: "Node.js Event Loop lag > 100ms"
```

---

## 面试追问

**Q: OpenTelemetry 和 Datadog/NewRelic 等 APM 有什么区别？**
A: OpenTelemetry 是厂商中立的标准（CNCF 项目），定义了如何生成 Traces/Metrics/Logs，数据可以导出到任何兼容 OTLP 的后端（Jaeger、Tempo、Datadog、NewRelic、Honeycomb 等）。Datadog/NewRelic 是完整的商业 APM 平台，有更多开箱即用功能（异常检测、AI 分析），但厂商锁定。架构上：用 OpenTelemetry SDK 生成数据（标准层），发到自选后端（灵活层）。

**Q: 生产环境 trace 数据量太大怎么办？**
A: 三个方向：①采样（最重要）：正常请求 1-10% 采样，错误/慢请求 100% 采样（Tail-based sampling）；②过滤：健康检查、静态资源请求不采样；③压缩 + TTL：Jaeger/Tempo 存储时设置保留时间（如 7 天），老数据自动删除。生产典型配置：1% 采样 + 错误 100% 采样 + 7 天保留。

**Q: 如何判断是数据库慢还是应用逻辑慢？**
A: 看 Trace 的 span 时间分布。Prisma 自动 instrumentation 会为每条 SQL 创建一个 span，显示 SQL 执行时间；HTTP 客户端 span 显示外部 API 延迟；剩余时间是应用逻辑时间。如果 DB span 占总时间的 80%，问题在 DB；如果 DB span 只占 10% 但总延迟高，问题在应用层（可能有 CPU 密集操作或 N+1）。
