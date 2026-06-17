# OOD：中间件管道（Middleware Pipeline）

> 设计 Express/Koa 风格的中间件系统：洋葱模型、compose、错误中间件。
> 面试考察：函数式组合、责任链模式、异步控制流、错误传播机制。

---

## 需求分析

```
功能需求：
  1. 中间件链：按注册顺序执行多个中间件
  2. next() 控制流：调用 next() 进入下一个中间件，不调用则停止
  3. 洋葱模型：中间件可以在 next() 前后各做逻辑（before/after）
  4. 错误传播：中间件抛出错误 → 跳过后续中间件 → 进入错误处理中间件
  5. 异步支持：中间件可以是 async 函数

核心概念：
  洋葱模型（Koa 风格）：
    middleware1 前 → middleware2 前 → handler → middleware2 后 → middleware1 后
  vs Express 风格（单向，next() 只往后走）
```

---

## 核心接口

```typescript
// 中间件上下文
interface Context {
  request: {
    method: string;
    path: string;
    headers: Record<string, string>;
    body: unknown;
    params: Record<string, string>;
    query: Record<string, string>;
  };
  response: {
    statusCode: number;
    headers: Record<string, string>;
    body: unknown;
  };
  state: Record<string, unknown>;  // 中间件间传递数据
}

// next 函数类型
type Next = () => Promise<void>;

// 中间件类型
type Middleware = (ctx: Context, next: Next) => Promise<void>;

// 错误中间件类型
type ErrorMiddleware = (err: Error, ctx: Context, next: Next) => Promise<void>;
```

---

## compose 实现（洋葱模型核心）

```typescript
// src/middleware/compose.ts

/**
 * 将中间件数组组合为单个函数（Koa 的 koa-compose 实现）
 *
 * 执行顺序（3个中间件）：
 *   fn1 before → fn2 before → fn3 before
 *   fn3 after  → fn2 after  → fn1 after
 */
export function compose(middlewares: Middleware[]): Middleware {
  return async function composed(ctx: Context, next: Next): Promise<void> {
    let index = -1;

    async function dispatch(i: number): Promise<void> {
      if (i <= index) {
        throw new Error('next() called multiple times in the same middleware');
      }
      index = i;

      // 取第 i 个中间件，如果超出范围则执行最终的 next
      const fn = i === middlewares.length ? next : middlewares[i];
      if (!fn) return;

      // 调用中间件，将 dispatch(i+1) 作为 next 传入
      await fn(ctx, dispatch.bind(null, i + 1));
    }

    await dispatch(0);
  };
}
```

---

## Application 类（整体框架）

```typescript
// src/middleware/application.ts

class Application {
  private middlewares: Middleware[] = [];
  private errorMiddlewares: ErrorMiddleware[] = [];

  // 注册普通中间件
  use(fn: Middleware): this {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be a function');
    this.middlewares.push(fn);
    return this;  // 支持链式调用
  }

  // 注册错误中间件（区分：4个参数 vs 2个参数）
  useError(fn: ErrorMiddleware): this {
    this.errorMiddlewares.push(fn);
    return this;
  }

  // 处理请求的入口
  async handle(ctx: Context): Promise<void> {
    const composed = compose(this.middlewares);

    try {
      await composed(ctx, async () => {});  // 空的最终 next
    } catch (err) {
      await this.handleError(err as Error, ctx);
    }
  }

  private async handleError(err: Error, ctx: Context): Promise<void> {
    if (!this.errorMiddlewares.length) {
      // 没有错误中间件：设置 500
      ctx.response.statusCode = err instanceof AppError ? (err as AppError).statusCode : 500;
      ctx.response.body = { error: err.message };
      return;
    }

    // 依次尝试错误中间件
    let index = -1;

    const dispatch = async (i: number): Promise<void> => {
      if (i >= this.errorMiddlewares.length) return;
      if (i <= index) throw new Error('next() called multiple times');
      index = i;

      const fn = this.errorMiddlewares[i];
      await fn(err, ctx, dispatch.bind(null, i + 1));
    };

    await dispatch(0);
  }

  // 转换为 Express 兼容的 handler
  callback() {
    return (req: Request, res: Response) => {
      const ctx = this.createContext(req, res);
      this.handle(ctx).then(() => {
        this.respond(ctx, res);
      });
    };
  }

  private createContext(req: Request, res: Response): Context {
    return {
      request: {
        method: req.method,
        path: req.url,
        headers: req.headers as Record<string, string>,
        body: (req as any).body,
        params: {},
        query: Object.fromEntries(new URL(req.url, 'http://x').searchParams),
      },
      response: { statusCode: 200, headers: {}, body: null },
      state: {},
    };
  }

  private respond(ctx: Context, res: Response) {
    for (const [key, value] of Object.entries(ctx.response.headers)) {
      res.setHeader(key, value);
    }
    res.statusCode = ctx.response.statusCode;
    res.json(ctx.response.body);
  }
}
```

---

## 内置中间件实现

```typescript
// 日志中间件（洋葱模型：记录请求开始和结束）
function logger(): Middleware {
  return async (ctx, next) => {
    const start = Date.now();
    console.log(`→ ${ctx.request.method} ${ctx.request.path}`);

    await next();  // 等待后续所有中间件执行完毕

    const duration = Date.now() - start;
    console.log(`← ${ctx.response.statusCode} ${duration}ms`);
  };
}

// 鉴权中间件
function auth(secret: string): Middleware {
  return async (ctx, next) => {
    const token = ctx.request.headers['authorization']?.split(' ')[1];

    if (!token) {
      ctx.response.statusCode = 401;
      ctx.response.body = { error: 'Missing token' };
      return;  // 不调用 next()，停止管道
    }

    try {
      const payload = jwt.verify(token, secret) as { sub: string; role: string };
      ctx.state.user = payload;  // 存入 state，后续中间件可以使用
      await next();
    } catch {
      ctx.response.statusCode = 401;
      ctx.response.body = { error: 'Invalid token' };
    }
  };
}

// 速率限制中间件
function rateLimit(options: { limit: number; windowMs: number }): Middleware {
  const counts = new Map<string, { count: number; resetAt: number }>();

  return async (ctx, next) => {
    const ip = ctx.request.headers['x-forwarded-for'] ?? 'unknown';
    const now = Date.now();
    const record = counts.get(ip);

    if (!record || now > record.resetAt) {
      counts.set(ip, { count: 1, resetAt: now + options.windowMs });
      await next();
      return;
    }

    if (record.count >= options.limit) {
      ctx.response.statusCode = 429;
      ctx.response.body = { error: 'Too many requests' };
      ctx.response.headers['Retry-After'] = String(Math.ceil((record.resetAt - now) / 1000));
      return;
    }

    record.count++;
    await next();
  };
}

// Body 解析中间件
function bodyParser(): Middleware {
  return async (ctx, next) => {
    // 已经由框架解析，这里只做类型确保
    if (!ctx.request.body) {
      ctx.request.body = {};
    }
    await next();
  };
}

// 错误处理中间件
function errorHandler(): ErrorMiddleware {
  return async (err, ctx, next) => {
    if (err instanceof ValidationError) {
      ctx.response.statusCode = 422;
      ctx.response.body = { error: err.message, details: err.details };
    } else if (err instanceof NotFoundError) {
      ctx.response.statusCode = 404;
      ctx.response.body = { error: err.message };
    } else if (err instanceof UnauthorizedError) {
      ctx.response.statusCode = 401;
      ctx.response.body = { error: err.message };
    } else {
      console.error('Unhandled error:', err);
      ctx.response.statusCode = 500;
      ctx.response.body = { error: 'Internal server error' };
    }
    // 不调用 next()，停止错误链
  };
}
```

---

## 组合使用

```typescript
const app = new Application();

// 全局中间件（按顺序执行）
app
  .use(logger())
  .use(bodyParser())
  .use(rateLimit({ limit: 100, windowMs: 60_000 }))
  .useError(errorHandler());

// 路由级中间件（只对特定路由生效）
const protectedRoutes = compose([
  auth(process.env.JWT_SECRET!),
  requireRole('admin'),
]);

app.use(async (ctx, next) => {
  if (ctx.request.path.startsWith('/admin')) {
    // 对 /admin 路由额外运行鉴权中间件
    await protectedRoutes(ctx, next);
  } else {
    await next();
  }
});

// 测试执行顺序
const testApp = new Application();
const log: string[] = [];

testApp
  .use(async (ctx, next) => {
    log.push('middleware1 before');
    await next();
    log.push('middleware1 after');
  })
  .use(async (ctx, next) => {
    log.push('middleware2 before');
    await next();
    log.push('middleware2 after');
  })
  .use(async (ctx, next) => {
    log.push('handler');
    ctx.response.body = { message: 'ok' };
  });

const ctx = createTestContext();
await testApp.handle(ctx);
console.log(log);
// ['middleware1 before', 'middleware2 before', 'handler', 'middleware2 after', 'middleware1 after']
```

---

## 面试追问

**Q: 洋葱模型和 Express 中间件有什么区别？**
A: Express 是单向链（next 只往后，不能在后续中间件完成后执行逻辑）；Koa 是双向洋葱（await next() 后可以继续执行后续代码）。Express 实现计时日志需要 hack（`res.on('finish', ...)`），Koa 中 `await next(); const duration = Date.now() - start` 天然支持。现代框架（Hono、Fastify）都采用洋葱模型或类似方式。

**Q: compose 为什么要检查 `i <= index`？**
A: 防止中间件多次调用 `next()`。如果不检查，`await next(); await next();` 会导致同一个中间件被触发两次，造成重复处理（响应被发送两次）。检查 `i <= index` 确保每个中间件只能前进，不能重复或后退。这是 koa-compose 的核心安全机制。

**Q: 如何实现路由匹配？**
A: 在中间件内部匹配：检查 `ctx.request.path` 和 `ctx.request.method`，符合则处理，不符合则 `next()`。更完整的实现是 Router 类，维护 `[method, pathPattern, ...middlewares]` 数组，path 转正则（`:id` → `([^/]+)`），匹配成功则提取 params 存入 `ctx.request.params`，再执行路由级中间件链。
