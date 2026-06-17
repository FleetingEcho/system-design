# Node.js 服务分层架构

> 如何组织一个可维护、可测试的 TypeScript Node.js 服务。
> 分层原则：每一层只依赖下一层的接口，不感知实现细节。

---

## 分层模型

```
HTTP 请求
    ↓
┌─────────────────────────────────────────────────────┐
│  Routes          路由定义：URL + HTTP 方法 → Handler │
├─────────────────────────────────────────────────────┤
│  Middleware      认证、日志、请求 ID、错误处理        │
├─────────────────────────────────────────────────────┤
│  Controllers     解析请求、调用 Service、格式化响应   │
│                  （薄层，不含业务逻辑）               │
├─────────────────────────────────────────────────────┤
│  Services        业务逻辑（不依赖 HTTP，可单测）      │
├─────────────────────────────────────────────────────┤
│  Repositories    数据访问（隔离 ORM/DB 细节）         │
├─────────────────────────────────────────────────────┤
│  Database / External APIs                            │
└─────────────────────────────────────────────────────┘
```

---

## 项目结构

```
src/
├── app.ts               Express/Fastify 实例创建和中间件注册
├── server.ts            HTTP server 启动和 graceful shutdown
├── routes/
│   ├── index.ts         汇总所有路由
│   ├── user.routes.ts
│   └── order.routes.ts
├── controllers/
│   ├── user.controller.ts
│   └── order.controller.ts
├── services/
│   ├── user.service.ts
│   └── order.service.ts
├── repositories/
│   ├── user.repository.ts
│   └── order.repository.ts
├── middleware/
│   ├── auth.middleware.ts
│   ├── request-id.middleware.ts
│   └── error.middleware.ts
├── lib/
│   ├── prisma.ts        Prisma client 单例
│   ├── redis.ts         Redis client 单例
│   └── logger.ts        Pino logger 实例
├── types/
│   ├── express.d.ts     扩展 Express Request 类型
│   └── api.ts           请求/响应 DTO 类型
├── config.ts            环境变量校验（Zod）
└── errors.ts            自定义 Error 类层次
```

---

## Repository 层（数据访问隔离）

```typescript
// types/api.ts
export interface User {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
}

export interface CreateUserInput {
  email: string;
  name: string;
  password: string;
}

// repositories/user.repository.ts
import { prisma } from '../lib/prisma';
import type { User, CreateUserInput } from '../types/api';

// 接口：Service 依赖接口而非具体实现（便于 mock 测试）
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(input: CreateUserInput): Promise<User>;
  update(id: string, data: Partial<User>): Promise<User>;
  delete(id: string): Promise<void>;
}

export class UserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    return prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return prisma.user.findUnique({ where: { email } });
  }

  async create(input: CreateUserInput): Promise<User> {
    const { password, ...rest } = input;
    const hashedPassword = await bcrypt.hash(password, 12);
    return prisma.user.create({
      data: { ...rest, password: hashedPassword },
    });
  }

  async update(id: string, data: Partial<User>): Promise<User> {
    return prisma.user.update({ where: { id }, data });
  }

  async delete(id: string): Promise<void> {
    await prisma.user.delete({ where: { id } });
  }
}
```

---

## Service 层（业务逻辑）

```typescript
// services/user.service.ts
import { NotFoundError, ConflictError } from '../errors';
import type { IUserRepository } from '../repositories/user.repository';
import type { CreateUserInput, User } from '../types/api';

// Service 只依赖 Repository 接口，不知道数据库实现
export class UserService {
  constructor(private userRepo: IUserRepository) {}

  async getUserById(id: string): Promise<User> {
    const user = await this.userRepo.findById(id);
    if (!user) throw new NotFoundError('User', id);
    return user;
  }

  async createUser(input: CreateUserInput): Promise<User> {
    const existing = await this.userRepo.findByEmail(input.email);
    if (existing) {
      throw new ConflictError(`Email ${input.email} is already registered`);
    }
    return this.userRepo.create(input);
  }

  async updateProfile(
    requesterId: string,
    targetId: string,
    data: Pick<User, 'name'>
  ): Promise<User> {
    // 业务规则：只能修改自己的 profile
    if (requesterId !== targetId) {
      throw new ForbiddenError('Cannot modify another user\'s profile');
    }
    await this.getUserById(targetId);  // 确认存在
    return this.userRepo.update(targetId, data);
  }
}
```

---

## Controller 层（HTTP 适配器）

```typescript
// controllers/user.controller.ts
import { RequestHandler } from 'express';
import { z } from 'zod';
import { UserService } from '../services/user.service';

// 请求校验 schema
const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  password: z.string().min(8),
});

const updateProfileSchema = z.object({
  name: z.string().min(1).max(100),
});

export class UserController {
  constructor(private userService: UserService) {}

  getUser: RequestHandler = async (req, res) => {
    // Controller 只做：解析请求 + 调用 Service + 格式化响应
    const user = await this.userService.getUserById(req.params.id);
    res.json({ data: user });
  };

  createUser: RequestHandler = async (req, res) => {
    const input = createUserSchema.parse(req.body);  // 校验，失败自动抛出
    const user = await this.userService.createUser(input);
    res.status(201).json({ data: user });
  };

  updateProfile: RequestHandler = async (req, res) => {
    const data = updateProfileSchema.parse(req.body);
    const user = await this.userService.updateProfile(
      req.user!.id,    // 来自 auth middleware
      req.params.id,
      data
    );
    res.json({ data: user });
  };
}
```

---

## 依赖注入（不用框架）

```typescript
// lib/container.ts —— 手动组装依赖（工厂函数 DI）
import { prisma } from './prisma';
import { UserRepository } from '../repositories/user.repository';
import { UserService } from '../services/user.service';
import { UserController } from '../controllers/user.controller';
import { OrderRepository } from '../repositories/order.repository';
import { OrderService } from '../services/order.service';
import { OrderController } from '../controllers/order.controller';

// 单例：每个实例只创建一次
const userRepo = new UserRepository();
const userService = new UserService(userRepo);
export const userController = new UserController(userService);

const orderRepo = new OrderRepository();
const orderService = new OrderService(orderRepo, userRepo);  // 跨 repo 依赖
export const orderController = new OrderController(orderService);

// 测试时：替换 Repository 实现为 mock
// const mockUserRepo: IUserRepository = { findById: jest.fn(), ... };
// const userService = new UserService(mockUserRepo);
```

---

## Routes 层

```typescript
// routes/user.routes.ts
import { Router } from 'express';
import { userController } from '../lib/container';
import { authMiddleware } from '../middleware/auth.middleware';
import { asyncHandler } from '../middleware/async-handler';

const router = Router();

// asyncHandler 包装 async 函数，捕获 reject 传给 next(err)
router.get('/:id', asyncHandler(userController.getUser));
router.post('/', asyncHandler(userController.createUser));
router.patch(
  '/:id',
  authMiddleware,
  asyncHandler(userController.updateProfile)
);

export default router;

// middleware/async-handler.ts
import { RequestHandler } from 'express';

export const asyncHandler =
  (fn: RequestHandler): RequestHandler =>
  (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
```

---

## 测试策略

```typescript
// services/__tests__/user.service.test.ts
import { UserService } from '../user.service';
import { NotFoundError, ConflictError } from '../../errors';
import type { IUserRepository } from '../../repositories/user.repository';

// Mock Repository（不需要数据库）
const mockRepo: jest.Mocked<IUserRepository> = {
  findById: jest.fn(),
  findByEmail: jest.fn(),
  create: jest.fn(),
  update: jest.fn(),
  delete: jest.fn(),
};

describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    jest.clearAllMocks();
    service = new UserService(mockRepo);
  });

  describe('getUserById', () => {
    it('throws NotFoundError when user does not exist', async () => {
      mockRepo.findById.mockResolvedValue(null);
      await expect(service.getUserById('non-existent'))
        .rejects.toThrow(NotFoundError);
    });

    it('returns user when found', async () => {
      const user = { id: '1', email: 'a@b.com', name: 'Alice', createdAt: new Date() };
      mockRepo.findById.mockResolvedValue(user);
      await expect(service.getUserById('1')).resolves.toEqual(user);
    });
  });

  describe('createUser', () => {
    it('throws ConflictError when email already exists', async () => {
      mockRepo.findByEmail.mockResolvedValue({ id: '1' } as any);
      await expect(service.createUser({ email: 'a@b.com', name: 'Alice', password: '12345678' }))
        .rejects.toThrow(ConflictError);
    });
  });
});
```

---

## 面试追问

**Q: Controller 和 Service 的边界在哪里？**
A: Controller 是 HTTP 适配器：解析 `req.params/body/query`，调用 Service，格式化 `res.json`。业务规则（"用户只能改自己的 profile"、"库存不足时不能下单"）全在 Service。这样 Service 可以被 CLI、消息队列消费者、定时任务复用，不依赖 HTTP 上下文。判断标准：如果单测 Service 需要 mock `req/res`，说明 Service 承担了 Controller 的职责。

**Q: 为什么 Repository 要写接口？**
A: 让 Service 依赖接口而不是具体的 Prisma 实现，有两个好处：①单测 Service 时用 mock 替换，不需要真实数据库；②如果将来从 Prisma 换成其他 ORM，只需要换 Repository 实现，Service 代码不变（依赖倒置原则）。

**Q: 不用 NestJS 如何做依赖注入？**
A: 简单场景用工厂函数 + 单例手动组装（`lib/container.ts`）。复杂场景可用 `tsyringe`（微软出的轻量 DI 容器，用 TypeScript 装饰器）或 `awilix`（不需要装饰器，用注册表模式）。NestJS 的 DI 容器本质也是这个原理，只是加了模块系统和生命周期管理。
