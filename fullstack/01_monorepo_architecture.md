# 全栈 Monorepo 架构

> TypeScript 全栈的核心竞争力：前后端共享类型，端到端类型安全，单仓库统一工具链。
> 技术栈：Turborepo + Next.js + Node.js（或 Next.js API）+ tRPC + Zod + Prisma

---

## 为什么 Monorepo

```
Polyrepo（多仓库）的痛点：
  - 前端调用 /api/users 返回 { user_name }，后端改成 { username }，前端运行时才发现
  - 共享类型要发 npm 包，每次改接口要发版、前后端分别 npm install
  - 工具配置（ESLint、TypeScript、Prettier）多份维护
  - 跨仓库重构困难（一个类型改名影响 5 个仓库）

Monorepo 的解法：
  - 前后端共享一个 types 包，TypeScript 编译期发现类型不匹配
  - 一个仓库一套工具链
  - 原子提交：前后端改动在同一个 PR 中，不存在"接口未更新"问题
```

---

## 目录结构

```
my-app/
├── apps/
│   ├── web/                    Next.js 前端
│   │   ├── app/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── api/                    Node.js/Fastify 后端（可选，如果不用 Next.js API）
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── admin/                  另一个前端应用
│
├── packages/
│   ├── database/               Prisma schema + 生成的类型 + Repository
│   │   ├── prisma/schema.prisma
│   │   ├── src/index.ts        导出 prisma client + 类型
│   │   └── package.json
│   ├── api-types/              tRPC router 类型（前后端共享）
│   │   ├── src/index.ts
│   │   └── package.json
│   ├── validators/             Zod schemas（前后端共享）
│   │   ├── src/
│   │   │   ├── user.schema.ts
│   │   │   └── order.schema.ts
│   │   └── package.json
│   └── ui/                     共享 React 组件库
│       ├── src/
│       └── package.json
│
├── turbo.json                  Turborepo 配置
├── package.json                根 package.json（workspace 定义）
└── tsconfig.base.json          共享 TypeScript 配置
```

---

## Turborepo 配置

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],  // 先构建依赖的 packages
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,           // dev 不缓存
      "persistent": true        // 长期运行进程
    },
    "lint": {
      "outputs": []
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "db:generate": {
      "cache": false
    }
  }
}
```

```json
// package.json（根）
{
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "turbo lint",
    "test": "turbo test",
    "db:push": "turbo db:push",
    "db:generate": "turbo db:generate"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## Zod 作为单一真相来源

```typescript
// packages/validators/src/user.schema.ts
import { z } from 'zod';

// 基础 schema（数据库字段）
export const userBaseSchema = z.object({
  id: z.string().cuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(['user', 'admin']),
  createdAt: z.date(),
});

// 创建用户（POST body）：只包含必要字段
export const createUserSchema = userBaseSchema
  .pick({ email: true, name: true })
  .extend({
    password: z.string().min(8).max(100),
  });

// 更新用户（PATCH body）：所有字段可选
export const updateUserSchema = userBaseSchema
  .pick({ name: true })
  .partial();

// 查询参数
export const userQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  search: z.string().optional(),
  role: userBaseSchema.shape.role.optional(),
});

// 从 schema 推导 TypeScript 类型（不重复定义）
export type User = z.infer<typeof userBaseSchema>;
export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UpdateUserInput = z.infer<typeof updateUserSchema>;
export type UserQuery = z.infer<typeof userQuerySchema>;
```

```typescript
// 后端使用：路由校验
import { createUserSchema } from '@my-app/validators';

app.post('/users', async (req, res) => {
  const input = createUserSchema.parse(req.body);  // 类型安全，校验失败抛出
  const user = await userService.create(input);
  res.json(user);
});

// 前端使用：表单校验
import { createUserSchema } from '@my-app/validators';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

function CreateUserForm() {
  const form = useForm({
    resolver: zodResolver(createUserSchema),  // 复用同一个 schema！
  });
  // 表单字段类型自动推导，与后端完全一致
}
```

---

## tRPC：端到端类型安全

```typescript
// packages/api-types/src/router.ts（或 apps/api/src/router.ts）
import { initTRPC, TRPCError } from '@trpc/server';
import { z } from 'zod';
import { createUserSchema, userQuerySchema } from '@my-app/validators';

// 初始化 tRPC
const t = initTRPC.context<{ userId?: string }>().create();

export const router = t.router;
export const publicProcedure = t.procedure;
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.userId) throw new TRPCError({ code: 'UNAUTHORIZED' });
  return next({ ctx: { ...ctx, userId: ctx.userId } });  // userId 类型变为 string
});

// 定义路由
export const userRouter = router({
  list: publicProcedure
    .input(userQuerySchema)
    .query(async ({ input }) => {
      return userService.list(input);
    }),

  getById: publicProcedure
    .input(z.object({ id: z.string().cuid() }))
    .query(async ({ input }) => {
      const user = await userService.getById(input.id);
      if (!user) throw new TRPCError({ code: 'NOT_FOUND' });
      return user;
    }),

  create: protectedProcedure
    .input(createUserSchema)
    .mutation(async ({ input }) => {
      return userService.create(input);
    }),
});

export const appRouter = router({
  user: userRouter,
  order: orderRouter,
});

// 导出 Router 类型（供前端使用）
export type AppRouter = typeof appRouter;
```

```typescript
// apps/web/src/lib/trpc.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@my-app/api-types';

// AppRouter 类型从后端导入，前端自动获得完整类型
export const trpc = createTRPCReact<AppRouter>();

// apps/web/src/app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { trpc } from '../lib/trpc';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());
  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [httpBatchLink({ url: '/api/trpc' })],
    })
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

```typescript
// 前端使用：完全类型安全，无需手写 fetch
function UserList() {
  // 自动推导返回类型（来自后端 userService.list 的返回值）
  const { data, isLoading } = trpc.user.list.useQuery({ page: 1, limit: 20 });
  //                                                    ↑ 类型安全，IDE 自动补全

  const createMutation = trpc.user.create.useMutation({
    onSuccess: () => utils.user.list.invalidate(),
  });

  return (
    <div>
      {data?.users.map(user => (  // user 类型自动推导
        <div key={user.id}>{user.name}</div>
      ))}
      <button onClick={() => createMutation.mutate({ email: 'a@b.com', name: 'Alice', password: '12345678' })}>
        Create
      </button>
    </div>
  );
}
```

---

## Prisma：数据库类型层

```prisma
// packages/database/prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  role      Role     @default(user)
  password  String
  orders    Order[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

enum Role {
  user
  admin
}
```

```typescript
// packages/database/src/index.ts
import { PrismaClient } from '@prisma/client';

// 单例（避免 hot reload 时创建多个连接）
const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'warn', 'error'] : ['error'],
});

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;

// 导出 Prisma 生成的类型（其他包可以用）
export type { User, Order, Role } from '@prisma/client';
export { Prisma } from '@prisma/client';
```

---

## 共享 UI 组件包

```typescript
// packages/ui/src/button.tsx
// 这是一个 Server Component 安全的 UI 组件
import { ButtonHTMLAttributes } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md font-medium transition-colors',
  {
    variants: {
      variant: {
        default: 'bg-primary text-white hover:bg-primary/90',
        outline: 'border border-input hover:bg-accent',
        ghost: 'hover:bg-accent',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4',
        lg: 'h-12 px-6 text-lg',
      },
    },
    defaultVariants: { variant: 'default', size: 'md' },
  }
);

interface ButtonProps
  extends ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

export function Button({ variant, size, className, ...props }: ButtonProps) {
  return (
    <button className={buttonVariants({ variant, size, className })} {...props} />
  );
}

// packages/ui/package.json
// {
//   "exports": {
//     "./button": "./src/button.tsx"  ← 源码直接导出（Turborepo 不需要构建）
//   }
// }
```

---

## 类型安全的全链路数据流

```
数据库 Schema（Prisma）
    ↓  自动生成
Prisma 类型（User, Order...）
    ↓  手动对齐或 import
Zod Schema（validators 包）
    ├─→ 后端：tRPC input 校验 + ORM 类型约束
    ├─→ 前端：表单校验（react-hook-form + zodResolver）
    └─→ 共享：TypeScript 类型推导（z.infer<typeof schema>）
            ↓
         tRPC Router（从输入到输出全类型）
            ↓
         前端 tRPC hooks（自动推导返回类型）
            ↓
         UI 组件（编译时知道 user.name 是 string）
```

---

## 面试追问

**Q: Monorepo 和 Microservices 是矛盾的吗？**
A: 不矛盾。Monorepo 是代码组织方式（一个仓库），Microservices 是部署架构。可以在 Monorepo 里有多个独立部署的服务（`apps/api`、`apps/worker`、`apps/web`），共享 `packages/` 里的公共代码，但各自独立打 Docker 镜像部署。好处是代码共享方便，坏处是仓库规模增大、CI 需要增量构建（Turborepo 的缓存解决这个问题）。

**Q: tRPC 和 REST API / GraphQL 怎么选？**
A: tRPC 适合前后端同一个 Monorepo 且都是 TypeScript（类型安全零成本，不需要 codegen）；REST 适合需要对外开放 API（第三方调用、OpenAPI 文档）或多语言客户端；GraphQL 适合多个客户端（移动端、Web、第三方）需要灵活查询字段、或前端团队独立于后端。tRPC 不是 REST 的替代，是"内部 RPC"。

**Q: Turborepo 的缓存是如何工作的？**
A: 对每个任务（build/test/lint），Turborepo 计算输入文件的哈希值（源码、package.json、tsconfig、环境变量），如果哈希未变，直接复用上次的输出（`dist/`、`coverage/`）而不重新执行。可以用远程缓存（Vercel Remote Cache 或自建）跨 CI 机器共享缓存，第一次构建慢，之后几乎瞬间。
