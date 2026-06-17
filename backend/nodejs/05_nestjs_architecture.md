# NestJS 架构深度

> NestJS = Angular 风格的 Node.js 框架：模块化 DI + 装饰器 + 面向切面编程。
> 考察点：模块/Provider 生命周期、Guard/Interceptor/Pipe/Filter 执行顺序、DI 作用域。

---

## 核心概念关系图

```
Module（模块）
  ├── imports    其他 Module（引入其他模块的导出）
  ├── providers  本模块的 Service/Repository（注入到 DI 容器）
  ├── controllers 路由处理
  └── exports    供其他 Module 使用的 Provider

DI 容器
  └── 按 Module 作用域管理 Provider 生命周期
      └── 默认 Singleton：整个应用只有一个实例
```

---

## Module 系统

```typescript
// user.module.ts
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { UserService } from './user.service';
import { UserRepository } from './user.repository';
import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],       // 引入 DatabaseModule 的导出（PrismaService）
  controllers: [UserController],
  providers: [UserService, UserRepository],
  exports: [UserService],          // 允许其他 Module 注入 UserService
})
export class UserModule {}

// app.module.ts（根模块）
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),  // 全局模块，无需每处 import
    DatabaseModule,
    UserModule,
    OrderModule,
  ],
})
export class AppModule {}
```

---

## Provider 生命周期（作用域）

```typescript
import { Injectable, Scope } from '@nestjs/common';

// DEFAULT（Singleton）：整个应用唯一实例（默认，推荐）
@Injectable()
export class UserService {}

// REQUEST：每个 HTTP 请求创建新实例
// 用途：在 Provider 中访问 request 上下文（如当前用户）
@Injectable({ scope: Scope.REQUEST })
export class AuditService {
  constructor(@Inject(REQUEST) private request: Request) {}

  log(action: string) {
    const userId = this.request['user']?.id;
    // 每个请求有独立实例，可以安全存储请求级数据
  }
}

// TRANSIENT：每次注入都创建新实例（少用）
@Injectable({ scope: Scope.TRANSIENT })
export class UniqueIdService {
  private id = Math.random();
}

// 注意：REQUEST scope 的 Provider 会将整条注入链都变为 REQUEST scope
// UserController → UserService(REQUEST) → UserRepository 都会变为请求级
// 有性能开销，谨慎使用
```

---

## 请求处理管道（执行顺序）

```
请求进入
    ↓
Middleware（中间件）      app.use() 注册，最先执行，Express 风格
    ↓
Guards（守卫）           canActivate()，认证/授权，返回 false 则 403
    ↓
Interceptors（before）   intercept() 的 next.handle() 之前
    ↓
Pipes（管道）            transform()，校验和转换参数
    ↓
Controller Handler       实际处理逻辑
    ↓
Interceptors（after）    next.handle() 之后（RxJS Observable map）
    ↓
Exception Filters        捕获异常并格式化响应
    ↓
响应返回

错误路径：任何阶段抛出 → Exception Filters 接管
```

---

## Guards（认证/授权）

```typescript
// guards/jwt-auth.guard.ts
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private reflector: Reflector,  // 读取装饰器元数据
  ) {}

  canActivate(context: ExecutionContext): boolean {
    // 检查是否标记为公开路由（跳过认证）
    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    if (!token) throw new UnauthorizedException();

    try {
      request.user = this.jwtService.verify(token);
      return true;
    } catch {
      throw new UnauthorizedException('Invalid or expired token');
    }
  }

  private extractToken(request: Request): string | null {
    const auth = request.headers['authorization'];
    if (!auth?.startsWith('Bearer ')) return null;
    return auth.slice(7);
  }
}

// 全局注册
app.useGlobalGuards(new JwtAuthGuard(jwtService, reflector));

// 公开路由装饰器
export const Public = () => SetMetadata('isPublic', true);

// RBAC 权限守卫
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user.roles?.includes(role));
  }
}

// 使用
@Get('/admin')
@Roles('admin')
@UseGuards(RolesGuard)
adminOnly() { ... }
```

---

## Interceptors（日志、缓存、响应转换）

```typescript
// interceptors/logging.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, tap } from 'rxjs';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const req = context.switchToHttp().getRequest();
    const start = Date.now();

    // 在 Handler 执行之前记录请求
    logger.info({ method: req.method, path: req.path, requestId: req.requestId }, 'Incoming request');

    return next.handle().pipe(
      // 在 Handler 执行之后记录响应时间
      tap({
        next: () => logger.info({ duration: Date.now() - start }, 'Request completed'),
        error: (err) => logger.error({ duration: Date.now() - start, err }, 'Request failed'),
      }),
    );
  }
}

// interceptors/transform.interceptor.ts（统一响应格式）
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(_ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      map(data => ({ data, timestamp: new Date().toISOString() }))
    );
  }
}

// interceptors/cache.interceptor.ts
@Injectable()
export class CacheInterceptor implements NestInterceptor {
  constructor(private cacheManager: Cache) {}

  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<unknown>> {
    const req = context.switchToHttp().getRequest();
    const cacheKey = `cache:${req.url}`;

    const cached = await this.cacheManager.get(cacheKey);
    if (cached) return of(cached);  // 命中缓存，短路 Handler

    return next.handle().pipe(
      tap(data => this.cacheManager.set(cacheKey, data, 60)),  // 缓存 60 秒
    );
  }
}
```

---

## Pipes（参数校验/转换）

```typescript
// pipes/zod-validation.pipe.ts
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';
import { ZodSchema, ZodError } from 'zod';

@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown) {
    const result = this.schema.safeParse(value);
    if (!result.success) {
      throw new BadRequestException({
        message: 'Validation failed',
        errors: result.error.flatten().fieldErrors,
      });
    }
    return result.data;
  }
}

// Controller 中使用
const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
});

@Post()
async create(
  @Body(new ZodValidationPipe(createUserSchema))
  body: z.infer<typeof createUserSchema>
) {
  return this.userService.create(body);
}

// 全局内置 ValidationPipe（class-validator）
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,        // 剔除 DTO 中未声明的字段
  forbidNonWhitelisted: true,  // 存在未知字段时抛出 400
  transform: true,        // 自动类型转换（string '1' → number 1）
}));
```

---

## Exception Filters（统一异常响应）

```typescript
// filters/http-exception.filter.ts
import {
  ExceptionFilter, Catch, ArgumentsHost,
  HttpException, HttpStatus,
} from '@nestjs/common';

@Catch()  // 捕获所有异常（不指定类型）
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse();
    const req = ctx.getRequest();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';
    let code = 'INTERNAL_ERROR';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const body = exception.getResponse() as string | Record<string, unknown>;
      message = typeof body === 'string' ? body : (body.message as string) ?? message;
      code = typeof body === 'object' ? (body.code as string) ?? code : code;
    } else if (exception instanceof AppError) {
      status = exception.statusCode;
      message = exception.message;
      code = exception.code;
    } else {
      // 编程错误：记录完整错误
      logger.error({ err: exception, path: req.url }, 'Unexpected exception');
    }

    res.status(status).json({
      error: { code, message },
      path: req.url,
      requestId: req.requestId,
    });
  }
}
```

---

## 自定义装饰器

```typescript
// decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: keyof User | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user as User;
    return data ? user?.[data] : user;
  },
);

// 使用
@Get('/profile')
getProfile(@CurrentUser() user: User) {
  return user;
}

@Get('/id-only')
getId(@CurrentUser('id') userId: string) {
  return { userId };
}
```

---

## 面试追问

**Q: Guard 和 Middleware 的区别？**
A: Middleware 是 Express 级别的，在 NestJS 路由系统之前执行，无法访问路由元数据（如 `@Roles` 装饰器）。Guard 是 NestJS 原生的，可以通过 `Reflector` 读取装饰器元数据，且有 `ExecutionContext` 提供更多上下文（支持 HTTP/WebSocket/gRPC）。认证用 Guard，日志/压缩等 HTTP 层面的用 Middleware。

**Q: Interceptor 和 Filter 的区别？**
A: Interceptor 在 Handler 执行前后都能介入（用 RxJS Observable），适合日志、缓存、响应转换。Filter 只在异常发生后才介入，用于统一格式化错误响应。正常流程走 Interceptor，异常流程走 Filter。

**Q: REQUEST scope 的 Provider 有什么性能影响？**
A: 每个请求都新建实例（以及整条依赖链上的所有 Provider），增加 GC 压力，对高 QPS 服务影响明显。一般只在必须访问请求上下文（当前用户、请求 ID）时才用，其余情况用 Singleton + `AsyncLocalStorage` 传递请求上下文（性能更好）。
