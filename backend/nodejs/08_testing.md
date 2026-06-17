# Node.js 测试体系

> 测试策略、工具链、测试替身模式。Node.js 架构师必须能设计测试体系，不只是写测试用例。
> 技术栈：Vitest + Supertest + TestContainers + MSW v2

---

## 测试金字塔

```
        /\
       /E2E\         少：慢、昂贵、维护成本高（Playwright）
      /──────\
     /Integration\   中：真实 DB/HTTP，验证组件协作（Supertest + TestContainers）
    /────────────\
   /  Unit Tests  \  多：快、独立、纯函数逻辑（Vitest）
  /────────────────\
```

| 层级 | 工具 | 覆盖内容 | 运行时机 |
|------|------|---------|---------|
| Unit | Vitest | Service 业务逻辑、工具函数 | 每次 commit，< 10s |
| Integration | Vitest + Supertest + TestContainers | API 路由 + 真实 DB | PR CI，~30-60s |
| E2E | Playwright | 完整用户流程 | 部署前 / 定时 |

---

## Vitest 配置

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      exclude: ['**/*.test.ts', 'src/test/**', 'src/types/**'],
      thresholds: { lines: 80, functions: 80 },
    },
    // 单测和集成测试分开运行
    workspace: [
      {
        test: {
          name: 'unit',
          include: ['src/**/*.unit.test.ts'],
          environment: 'node',
        },
      },
      {
        test: {
          name: 'integration',
          include: ['src/**/*.integration.test.ts'],
          globalSetup: './src/test/global-setup.ts',
          poolOptions: { threads: { singleThread: true } },  // 避免 DB 并发冲突
        },
      },
    ],
  },
});
```

---

## 测试替身（Test Doubles）

```
四种替身的区别：

Stub   → 固定返回值，不关心调用方式。最简单。
Mock   → 预设期望，验证调用次数和参数。侧重"验证行为"。
Spy    → 监视真实实现，记录调用但不改变行为。
Fake   → 完整的内存轻量实现（内存数据库、内存邮件服务）。最真实。
```

```typescript
// --- Stub：让依赖返回固定值 ---
const userRepo = {
  findById: vi.fn().mockResolvedValue({ id: '1', name: 'Alice', role: 'user' }),
};

// --- Mock：验证调用行为 ---
const emailService = {
  sendWelcome: vi.fn(),
};
// 业务逻辑执行后验证
expect(emailService.sendWelcome).toHaveBeenCalledOnce();
expect(emailService.sendWelcome).toHaveBeenCalledWith('alice@example.com', 'Alice');

// --- Spy：监视真实实现 ---
import * as bcrypt from 'bcryptjs';
const hashSpy = vi.spyOn(bcrypt, 'hash');
// bcrypt.hash 真实执行，但我们能验证它被调用了
await userService.createUser({ email: 'a@b.com', password: 'pass123' });
expect(hashSpy).toHaveBeenCalledWith('pass123', 10);

// --- Fake：内存实现（最推荐用于 Repository）---
class InMemoryUserRepository implements IUserRepository {
  private store = new Map<string, User>();

  async findById(id: string): Promise<User | null> {
    return this.store.get(id) ?? null;
  }
  async findByEmail(email: string): Promise<User | null> {
    return [...this.store.values()].find(u => u.email === email) ?? null;
  }
  async create(data: CreateUserInput): Promise<User> {
    const user: User = { id: randomUUID(), ...data, createdAt: new Date() };
    this.store.set(user.id, user);
    return user;
  }
  async update(id: string, data: Partial<User>): Promise<User> {
    const user = this.store.get(id)!;
    const updated = { ...user, ...data };
    this.store.set(id, updated);
    return updated;
  }
  async delete(id: string): Promise<void> {
    this.store.delete(id);
  }
  // 测试辅助：清空所有数据
  clear() { this.store.clear(); }
}
```

---

## Unit 测试：Service 层

```typescript
// src/services/user.service.unit.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { UserService } from './user.service';
import { InMemoryUserRepository } from '../test/fakes/user.repository.fake';
import { ConflictError, NotFoundError } from '../lib/errors';

describe('UserService', () => {
  let service: UserService;
  let repo: InMemoryUserRepository;

  beforeEach(() => {
    repo = new InMemoryUserRepository();
    service = new UserService(repo);
  });

  describe('createUser', () => {
    it('creates user and hashes password', async () => {
      const user = await service.createUser({
        email: 'alice@example.com',
        name: 'Alice',
        password: 'password123',
      });

      expect(user.email).toBe('alice@example.com');
      // 验证密码被哈希（不是明文存储）
      const stored = await repo.findByEmail('alice@example.com');
      expect(stored!.password).not.toBe('password123');
      expect(stored!.password).toMatch(/^\$2[ab]\$/);  // bcrypt 格式
    });

    it('throws ConflictError when email already exists', async () => {
      await service.createUser({ email: 'alice@example.com', name: 'Alice', password: 'pass' });

      await expect(
        service.createUser({ email: 'alice@example.com', name: 'Bob', password: 'pass' })
      ).rejects.toThrow(ConflictError);
    });
  });

  describe('updateUser', () => {
    it('throws NotFoundError when user does not exist', async () => {
      await expect(
        service.updateUser('nonexistent-id', { name: 'New Name' })
      ).rejects.toThrow(NotFoundError);
    });
  });
});
```

---

## Integration 测试：TestContainers + Supertest

```typescript
// src/test/global-setup.ts —— 所有集成测试前运行一次
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { execSync } from 'child_process';

let container: StartedPostgreSqlContainer;

export async function setup() {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('testdb')
    .withUsername('test')
    .withPassword('test')
    .withStartupTimeout(60_000)
    .start();

  process.env.DATABASE_URL = container.getConnectionUri();

  // 运行 Prisma migration（不是 push，保证与生产一致）
  execSync('npx prisma migrate deploy', {
    env: { ...process.env },
    stdio: 'inherit',
  });

  (global as any).__TEST_CONTAINER__ = container;
}

export async function teardown() {
  await (global as any).__TEST_CONTAINER__?.stop();
}
```

```typescript
// src/routes/users.integration.test.ts
import { describe, it, expect, beforeEach, afterAll } from 'vitest';
import request from 'supertest';
import { app } from '../app';
import { prisma } from '../lib/prisma';

describe('Users API', () => {
  beforeEach(async () => {
    // 每个测试清空表（比事务回滚简单，速度可接受）
    await prisma.user.deleteMany();
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  describe('POST /users', () => {
    it('201: creates user', async () => {
      const res = await request(app)
        .post('/users')
        .send({ email: 'alice@example.com', name: 'Alice', password: 'password123' })
        .expect(201);

      expect(res.body.data.email).toBe('alice@example.com');
      expect(res.body.data.password).toBeUndefined();

      // 验证真实写入 DB
      const dbUser = await prisma.user.findUnique({ where: { email: 'alice@example.com' } });
      expect(dbUser).not.toBeNull();
    });

    it('409: duplicate email', async () => {
      await prisma.user.create({
        data: { email: 'alice@example.com', name: 'Alice', password: 'hashed' },
      });

      await request(app)
        .post('/users')
        .send({ email: 'alice@example.com', name: 'Bob', password: 'pass123' })
        .expect(409);
    });

    it('422: invalid email format', async () => {
      const res = await request(app)
        .post('/users')
        .send({ email: 'not-an-email', name: 'Alice', password: 'pass123' })
        .expect(422);

      expect(res.body.code).toBe('VALIDATION_ERROR');
      expect(res.body.details.email).toBeDefined();
    });
  });

  describe('GET /users/:id', () => {
    it('200: returns user', async () => {
      const created = await prisma.user.create({
        data: { email: 'bob@example.com', name: 'Bob', password: 'hashed' },
      });

      const res = await request(app).get(`/users/${created.id}`).expect(200);
      expect(res.body.data.id).toBe(created.id);
    });

    it('404: non-existent user', async () => {
      await request(app).get('/users/nonexistent-id').expect(404);
    });
  });
});
```

---

## MSW v2：拦截外部 HTTP（不 Mock 模块）

```typescript
// src/test/msw-handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  // 拦截 Stripe 支付
  http.post('https://api.stripe.com/v1/payment_intents', () => {
    return HttpResponse.json({
      id: 'pi_test_123',
      status: 'succeeded',
      amount: 2000,
    });
  }),

  // 拦截发送邮件（SendGrid）
  http.post('https://api.sendgrid.com/v3/mail/send', () => {
    return new HttpResponse(null, { status: 202 });
  }),
];

// src/test/setup.ts
import { setupServer } from 'msw/node';
import { handlers } from './msw-handlers';

export const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

```typescript
// 测试中覆盖 handler（模拟错误场景）
import { http, HttpResponse } from 'msw';
import { server } from '../test/setup';

it('handles payment failure gracefully', async () => {
  server.use(
    http.post('https://api.stripe.com/v1/payment_intents', () => {
      return HttpResponse.json(
        { error: { code: 'card_declined', message: 'Your card was declined.' } },
        { status: 402 }
      );
    })
  );

  const res = await request(app)
    .post('/orders')
    .send({ productId: '1', amount: 2000 })
    .expect(402);

  expect(res.body.code).toBe('PAYMENT_FAILED');
});
```

---

## 测试数据工厂

```typescript
// src/test/factories/user.factory.ts
import { faker } from '@faker-js/faker';
import type { User, CreateUserInput } from '../../types';

export const UserFactory = {
  build(overrides: Partial<User> = {}): User {
    return {
      id: faker.string.uuid(),
      email: faker.internet.email(),
      name: faker.person.fullName(),
      role: 'user' as const,
      createdAt: new Date(),
      updatedAt: new Date(),
      ...overrides,
    };
  },

  buildInput(overrides: Partial<CreateUserInput> = {}): CreateUserInput {
    return {
      email: faker.internet.email(),
      name: faker.person.fullName(),
      password: faker.internet.password({ length: 12 }),
      ...overrides,
    };
  },

  // 常用预设
  admin: (overrides: Partial<User> = {}) =>
    UserFactory.build({ role: 'admin', ...overrides }),
};

// 使用
const user = UserFactory.build({ role: 'admin' });
const adminUser = UserFactory.admin({ name: 'Super Admin' });
```

---

## 什么该 Mock、什么不该 Mock

```
✓ 该用 Test Double 的：
  - 外部 HTTP（Stripe/SendGrid/GitHub OAuth）→ MSW v2 在网络层拦截
  - 时间（Date.now / new Date）→ vi.useFakeTimers()
  - 随机值（Math.random / randomUUID）→ vi.fn().mockReturnValue(...)
  - 文件系统（Unit 测试中）→ vi.mock('node:fs/promises')

✗ 不该 Mock 的：
  - 数据库 → 用 TestContainers 跑真实 DB
    原因：Mock Prisma 不会检测 SQL 语法、约束违反、事务行为
  - 你自己代码的内部模块
    原因：内部 Mock 让重构后测试误报通过，测试变成实现细节的镜子
  - Redis（集成测试）→ 用 ioredis-mock 或启动真实 Redis 容器
```

---

## 面试追问

**Q: 单测 mock 了 Repository，集成测试又用真实 DB，不是重复了吗？**
A: 不重复，测的层级不同。单测验证 Service 业务逻辑（分支、错误处理、转换），用 Fake/Mock 是为了速度和隔离；集成测试验证路由→Service→DB 完整链路，检查 Prisma 查询语法、DB 约束（唯一索引）、HTTP 响应格式。单测快（毫秒级），集成测试慢（秒级）但更接近真实行为。

**Q: TestContainers 启动太慢怎么办？**
A: 全局只启动一次容器（`globalSetup`），所有测试共享，容器启动约 5-10s。测试间清数据用 `deleteMany`（比重启容器快 100 倍）或事务回滚（每个测试用 `$transaction` 包裹后 `rollback`）。CI 可以用 Docker layer cache 加速镜像拉取。

**Q: 测试覆盖率要达到多少？**
A: 覆盖率是手段不是目标。关键业务（支付、权限、核心数据写入）要接近 100%；工具函数、类型文件不需要强求。80% 覆盖率但覆盖了所有错误分支，强过 95% 覆盖率但只测了 happy path。PR 门控建议：lines > 80%，但同时要求关键模块 branches > 90%。
