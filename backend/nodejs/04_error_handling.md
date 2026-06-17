# Node.js 错误处理模式

> 结构化错误处理是生产服务的必备能力。
> 目标：错误可分类、可追踪、不泄露内部实现、能优雅降级。

---

## 错误分类

```
操作性错误（Operational）      编程错误（Programmer）
—————————————————————————     ————————————————————
可预期的运行时错误               代码 Bug
用户输入非法                    TypeError / undefined.foo
资源不存在（404）               逻辑错误、越界访问
权限不足（401/403）             应该修复代码，而不是处理
第三方服务超时                  不要捕获，让进程 crash 然后重启
数据库连接失败                  用 process.uncaughtException 兜底
——→ 捕获并返回友好错误          ——→ 记录日志并重启进程
```

---

## 自定义 Error 类层次

```typescript
// errors.ts

// 基类：所有业务错误的父类
export class AppError extends Error {
  constructor(
    public readonly message: string,
    public readonly statusCode: number,
    public readonly code: string,          // 机器可读的错误码（客户端用）
    public readonly isOperational = true,  // 区分操作性错误 vs 编程错误
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

// HTTP 4xx 错误
export class ValidationError extends AppError {
  constructor(message: string, public readonly fields?: Record<string, string[]>) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Insufficient permissions') {
    super(message, 403, 'FORBIDDEN');
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    const msg = id ? `${resource} with id '${id}' not found` : `${resource} not found`;
    super(msg, 404, 'NOT_FOUND');
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, 'CONFLICT');
  }
}

export class TooManyRequestsError extends AppError {
  constructor(retryAfter?: number) {
    super('Rate limit exceeded', 429, 'RATE_LIMIT_EXCEEDED');
    if (retryAfter) this.retryAfter = retryAfter;
  }
  retryAfter?: number;
}

// HTTP 5xx 错误
export class ServiceUnavailableError extends AppError {
  constructor(service: string) {
    super(`${service} is currently unavailable`, 503, 'SERVICE_UNAVAILABLE');
  }
}

// 判断是否是操作性错误
export function isOperationalError(err: unknown): err is AppError {
  return err instanceof AppError && err.isOperational;
}
```

---

## Express 全局错误处理中间件

```typescript
// middleware/error.middleware.ts
import { ErrorRequestHandler } from 'express';
import { ZodError } from 'zod';
import { AppError, ValidationError, isOperationalError } from '../errors';
import { logger } from '../lib/logger';

// Express 错误处理中间件必须有 4 个参数（err, req, res, next）
export const globalErrorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  const requestId = req.headers['x-request-id'] as string;

  // 1. Zod 校验错误 → 转换为 ValidationError
  if (err instanceof ZodError) {
    const fields: Record<string, string[]> = {};
    err.errors.forEach(e => {
      const path = e.path.join('.');
      fields[path] = [...(fields[path] ?? []), e.message];
    });
    const validationErr = new ValidationError('Validation failed', fields);

    return res.status(400).json({
      error: {
        code: validationErr.code,
        message: validationErr.message,
        fields,
      },
      requestId,
    });
  }

  // 2. 业务错误（操作性错误）→ 返回对应 HTTP 状态码
  if (err instanceof AppError) {
    logger.warn({
      requestId,
      err: { message: err.message, code: err.code, statusCode: err.statusCode },
    }, 'Operational error');

    const body: Record<string, unknown> = {
      error: { code: err.code, message: err.message },
      requestId,
    };
    if ('fields' in err) body.error = { ...body.error as object, fields: (err as ValidationError).fields };
    if ('retryAfter' in err) {
      res.setHeader('Retry-After', String((err as TooManyRequestsError).retryAfter));
    }

    return res.status(err.statusCode).json(body);
  }

  // 3. Prisma 错误 → 映射为业务错误
  if (err?.code === 'P2002') {  // Unique constraint violation
    return res.status(409).json({
      error: { code: 'CONFLICT', message: 'Resource already exists' },
      requestId,
    });
  }
  if (err?.code === 'P2025') {  // Record not found
    return res.status(404).json({
      error: { code: 'NOT_FOUND', message: 'Resource not found' },
      requestId,
    });
  }

  // 4. 未知错误（编程错误）→ 记录完整 stack，返回 500
  logger.error({ requestId, err }, 'Unexpected error');

  // 不暴露内部错误详情给客户端（安全）
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
    },
    requestId,
  });
};

// app.ts 中注册（必须在所有路由之后）
app.use(globalErrorHandler);
```

---

## async 路由的错误传递

```typescript
// 问题：Express 不自动捕获 async 函数的 reject
app.get('/user/:id', async (req, res) => {
  const user = await userService.getUser(req.params.id);  // 如果抛错，Express 不知道
  res.json(user);
});

// 方案 1：手动 try/catch（冗余）
app.get('/user/:id', async (req, res, next) => {
  try {
    const user = await userService.getUser(req.params.id);
    res.json(user);
  } catch (err) {
    next(err);  // 传给错误处理中间件
  }
});

// 方案 2：asyncHandler 包装器（推荐）
const asyncHandler =
  (fn: RequestHandler): RequestHandler =>
  (req, res, next) =>
    Promise.resolve(fn(req, res, next)).catch(next);

app.get('/user/:id', asyncHandler(async (req, res) => {
  const user = await userService.getUser(req.params.id);
  res.json(user);
}));

// 方案 3：express-async-errors 包（零配置，monkey-patch）
import 'express-async-errors';  // 只需在 app.ts 顶部 import，之后 async 路由自动捕获

// 方案 4：Express 5（正式版）原生支持 async 路由（无需 next(err)）
```

---

## 进程级错误处理

```typescript
// server.ts

// 未捕获的同步异常（编程错误）
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception — process will exit');
  // 记录日志后必须退出：进程状态未知，继续运行不安全
  process.exit(1);
  // PM2/K8s 会自动重启
});

// 未处理的 Promise rejection
process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'Unhandled rejection — process will exit');
  // Node.js 17+ 默认就会 exit，但显式处理更清晰
  process.exit(1);
});

// SIGTERM（K8s 停止 Pod 时发出）→ Graceful Shutdown
process.on('SIGTERM', () => {
  logger.info('Received SIGTERM, starting graceful shutdown');
  shutdown();
});
```

---

## 结构化错误响应格式

```typescript
// 统一响应格式（客户端始终可以解析）
interface SuccessResponse<T> {
  data: T;
  requestId: string;
}

interface ErrorResponse {
  error: {
    code: string;          // 机器可读，客户端 switch/case
    message: string;       // 人类可读，可直接显示
    fields?: Record<string, string[]>;  // 字段级校验错误
  };
  requestId: string;       // 用于日志关联
}

// 客户端使用：
// switch (error.code) {
//   case 'UNAUTHORIZED': redirect('/login'); break;
//   case 'VALIDATION_ERROR': showFieldErrors(error.fields); break;
//   case 'RATE_LIMIT_EXCEEDED': showRetryMessage(headers['retry-after']); break;
//   default: showGenericError(error.message);
// }
```

---

## 面试追问

**Q: 操作性错误和编程错误为什么要区别对待？**
A: 操作性错误（用户输入非法、资源不存在）是正常业务流程的一部分，应该捕获并返回友好的 HTTP 响应；编程错误（TypeError、逻辑 bug）说明程序处于不可预期状态，继续运行可能产生数据不一致，应该记录完整 stack trace 后重启进程（让 PM2/K8s 恢复到干净状态）。混淆两者的后果：要么吞掉了 bug（把编程错误当操作性错误处理），要么把内部错误暴露给用户。

**Q: 为什么错误响应不应该返回 stack trace？**
A: 安全问题：stack trace 暴露文件路径、框架版本、代码结构，给攻击者提供信息。生产环境错误响应只返回 `code` 和 `message`，stack trace 写入日志系统（只有团队内部可查）。

**Q: Zod 校验错误如何优雅地转化为字段错误？**
A: 在全局错误处理中间件 `instanceof ZodError` 检测，遍历 `err.errors` 数组，用 `e.path.join('.')` 生成字段路径（如 `address.city`），聚合同字段多条错误信息，构造 `fields: Record<string, string[]>` 返回给前端，前端可以直接映射到表单字段的 error 提示。
