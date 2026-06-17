# Node.js 生产级模式

> Graceful Shutdown、结构化日志、Health Check、Config 校验。
> 这四个模式是 Senior 工程师和 Junior 的分水岭。

---

## Graceful Shutdown

```
K8s 滚动部署流程：
  1. K8s 向旧 Pod 发送 SIGTERM
  2. K8s 等待 terminationGracePeriodSeconds（默认 30s）
  3. Pod 处理完 in-flight 请求后正常退出（0）
  4. K8s 如果超时，发送 SIGKILL（强杀）

没有 Graceful Shutdown 的后果：
  - 正在处理的请求被强制中断 → 用户看到 502
  - 数据库事务未提交 → 数据损坏
  - 消息队列任务未完成 → 重复处理
```

```typescript
// server.ts
import http from 'http';
import { app } from './app';
import { prisma } from './lib/prisma';
import { redis } from './lib/redis';
import { logger } from './lib/logger';

const server = http.createServer(app);
let isShuttingDown = false;

// 拒绝新请求（在负载均衡摘除 Pod 之前的缓冲）
app.use((req, res, next) => {
  if (isShuttingDown) {
    res.setHeader('Connection', 'close');  // 通知客户端不要复用连接
    return res.status(503).json({ error: 'Server is shutting down' });
  }
  next();
});

async function shutdown(signal: string) {
  logger.info({ signal }, 'Shutdown signal received');
  isShuttingDown = true;

  // 1. 停止接受新连接
  server.close(async (err) => {
    if (err) logger.error({ err }, 'Error closing HTTP server');

    // 2. 关闭外部连接（等待 in-flight 操作完成）
    await Promise.allSettled([
      prisma.$disconnect(),
      redis.quit(),
      // messageQueue.close(),
    ]);

    logger.info('Shutdown complete');
    process.exit(0);
  });

  // 3. 兜底：超时强制退出（不能超过 K8s terminationGracePeriodSeconds）
  setTimeout(() => {
    logger.error('Graceful shutdown timed out, forcing exit');
    process.exit(1);
  }, 25_000);  // 25s（K8s 默认 30s，留 5s 余量）
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));   // Ctrl+C 本地开发

server.listen(process.env.PORT ?? 3000, () => {
  logger.info({ port: process.env.PORT ?? 3000 }, 'Server started');
});
```

---

## 结构化日志（Pino + AsyncLocalStorage）

```typescript
// lib/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  // 生产：JSON 格式（logstash/datadog 解析）
  // 开发：pino-pretty 格式化（可读）
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  // 标准字段重命名（符合 ECS 规范）
  formatters: {
    level: (label) => ({ level: label }),
  },
  // 每条日志自动带时间戳
  timestamp: pino.stdTimeFunctions.isoTime,
  // 敏感字段脱敏
  redact: ['req.headers.authorization', 'body.password', 'body.token'],
});
```

```typescript
// lib/request-context.ts —— 跨调用链传递 requestId（不用传参）
import { AsyncLocalStorage } from 'async_hooks';

interface RequestContext {
  requestId: string;
  userId?: string;
  traceId?: string;
}

const storage = new AsyncLocalStorage<RequestContext>();

export const RequestContext = {
  run: <T>(ctx: RequestContext, fn: () => T) => storage.run(ctx, fn),
  get: () => storage.getStore(),
  getRequestId: () => storage.getStore()?.requestId ?? 'unknown',
};

// middleware/request-id.middleware.ts
import { randomUUID } from 'crypto';

export const requestIdMiddleware: RequestHandler = (req, res, next) => {
  const requestId = (req.headers['x-request-id'] as string) ?? randomUUID();
  req.requestId = requestId;
  res.setHeader('x-request-id', requestId);

  // 将 requestId 注入 AsyncLocalStorage，Service 层无需传参即可获取
  RequestContext.run(
    { requestId, userId: req.user?.id },
    next
  );
};

// lib/logger.ts —— 子 logger 自动携带 requestId
export const getLogger = () =>
  logger.child({ requestId: RequestContext.getRequestId() });

// service 中使用（不需要把 logger 传来传去）
class UserService {
  async getUser(id: string) {
    const log = getLogger();  // 自动带 requestId
    log.info({ userId: id }, 'Fetching user');
    // ...
  }
}
```

---

## Health Check

```typescript
// routes/health.routes.ts
import { Router } from 'express';
import { prisma } from '../lib/prisma';
import { redis } from '../lib/redis';

const router = Router();

// /health —— Liveness probe：进程是否存活
// K8s 用这个判断是否需要重启容器（失败 → 重启）
router.get('/health', (_req, res) => {
  res.json({
    status: 'ok',
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});

// /ready —— Readiness probe：是否可以接受流量
// K8s 用这个判断是否要把请求路由到该 Pod（失败 → 摘除流量，不重启）
router.get('/ready', async (_req, res) => {
  const checks: Record<string, 'ok' | 'fail'> = {};

  // 检查数据库连接
  try {
    await prisma.$queryRaw`SELECT 1`;
    checks.database = 'ok';
  } catch {
    checks.database = 'fail';
  }

  // 检查 Redis 连接
  try {
    await redis.ping();
    checks.redis = 'ok';
  } catch {
    checks.redis = 'fail';
  }

  const allOk = Object.values(checks).every(v => v === 'ok');

  res.status(allOk ? 200 : 503).json({
    status: allOk ? 'ready' : 'not_ready',
    checks,
    timestamp: new Date().toISOString(),
  });
});

// /metrics —— Prometheus 指标（可选）
// 用 prom-client 暴露 default metrics + 自定义指标
import { register, collectDefaultMetrics } from 'prom-client';
collectDefaultMetrics();

router.get('/metrics', async (_req, res) => {
  res.setHeader('Content-Type', register.contentType);
  res.send(await register.metrics());
});

export default router;
```

---

## Config 管理（Zod 校验）

```typescript
// config.ts —— 启动时校验所有环境变量，失败则立即退出
import { z } from 'zod';

const configSchema = z.object({
  // 服务配置
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  LOG_LEVEL: z.enum(['trace', 'debug', 'info', 'warn', 'error']).default('info'),

  // 数据库
  DATABASE_URL: z.string().url(),
  DATABASE_POOL_MIN: z.coerce.number().default(2),
  DATABASE_POOL_MAX: z.coerce.number().default(10),

  // Redis
  REDIS_URL: z.string().url(),

  // JWT
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_EXPIRES_IN: z.string().default('7d'),

  // 外部服务
  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.coerce.number().optional(),
  AWS_REGION: z.string().optional(),
  AWS_S3_BUCKET: z.string().optional(),
});

// 解析并导出，类型安全
function loadConfig() {
  const result = configSchema.safeParse(process.env);
  if (!result.success) {
    console.error('❌ Invalid configuration:');
    result.error.errors.forEach(e => {
      console.error(`  ${e.path.join('.')}: ${e.message}`);
    });
    process.exit(1);  // 启动失败，而不是运行时 undefined 崩溃
  }
  return result.data;
}

export const config = loadConfig();

// 全类型安全使用
// config.DATABASE_URL    // string
// config.PORT            // number
// config.NODE_ENV        // 'development' | 'test' | 'production'
```

---

## 速率限制（Rate Limiting）

```typescript
// middleware/rate-limit.middleware.ts
import { redis } from '../lib/redis';
import { TooManyRequestsError } from '../errors';

interface RateLimitOptions {
  windowMs: number;   // 时间窗口（毫秒）
  max: number;        // 窗口内最大请求数
  keyFn?: (req: Request) => string;  // 自定义 key（默认用 IP）
}

// 滑动窗口计数器（Redis）
export function rateLimit(options: RateLimitOptions): RequestHandler {
  const { windowMs, max, keyFn } = options;
  const windowSec = Math.ceil(windowMs / 1000);

  return async (req, res, next) => {
    const key = `ratelimit:${keyFn ? keyFn(req) : req.ip}:${req.path}`;

    const current = await redis.incr(key);
    if (current === 1) {
      await redis.expire(key, windowSec);  // 第一次请求时设置过期
    }

    res.setHeader('X-RateLimit-Limit', max);
    res.setHeader('X-RateLimit-Remaining', Math.max(0, max - current));

    if (current > max) {
      const ttl = await redis.ttl(key);
      next(new TooManyRequestsError(ttl));
      return;
    }

    next();
  };
}

// 使用
app.use('/api/auth', rateLimit({ windowMs: 15 * 60 * 1000, max: 5 }));  // 登录接口：15分钟5次
app.use('/api', rateLimit({ windowMs: 60 * 1000, max: 100 }));           // 通用：1分钟100次
```

---

## 面试追问

**Q: Graceful Shutdown 时如何处理还在处理中的请求？**
A: `server.close()` 会停止接受新连接，但不会强制断开已有连接。需要用 `server-destroy` 包或手动跟踪所有活跃 socket：`server.on('connection', socket => sockets.add(socket))`，shutdown 时 `sockets.forEach(s => s.destroy())`。同时设置 `Connection: close` header 提示客户端不要复用连接，让 in-flight 请求自然完成后连接关闭。

**Q: AsyncLocalStorage 是什么，为什么比传参更好？**
A: AsyncLocalStorage 是 Node.js 的异步上下文存储（类似 Thread Local Storage 的异步版本）。每个 async 调用链共享同一个 context，不需要把 `requestId`/`userId` 层层传参。使用 `storage.run(context, callback)` 创建上下文，内部任何深度的 `storage.getStore()` 都能拿到同一个 context。性能影响在 Node.js 16+ 已基本消除。

**Q: /health 和 /ready 的区别，为什么要分开？**
A: Liveness probe（/health）只检查进程是否存活，失败 → K8s 重启容器。Readiness probe（/ready）检查外部依赖（DB、Redis）是否就绪，失败 → K8s 从 Service 摘除该 Pod 流量但不重启。分开的原因：如果 DB 短暂不可用，不应该重启所有 Pod（重启也没用），只需要停止路由流量，等 DB 恢复后自动恢复。
