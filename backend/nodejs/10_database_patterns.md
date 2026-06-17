# Node.js 数据库模式（Prisma 进阶）

> 软删除、乐观锁、cursor 分页、全文搜索、事务、N+1 解决方案。
> 这些模式在面试中常以"如何实现 X"或"这段代码有什么问题"的形式出现。

---

## 软删除（Soft Delete）

```typescript
// 硬删除的问题：数据不可恢复、外键关联破坏、无法审计
// 软删除：添加 deletedAt 字段，逻辑上删除，物理上保留

// prisma/schema.prisma
model User {
  id        String    @id @default(cuid())
  email     String    @unique
  name      String
  deletedAt DateTime?  // null = 存活，非 null = 已删除
  orders    Order[]

  @@index([deletedAt])  // 查询过滤用
}
```

```typescript
// src/repositories/user.repository.ts
import { Prisma } from '@prisma/client';

// 软删除的核心：所有查询必须加 where: { deletedAt: null }
// 用 Prisma Middleware 自动注入，避免每个查询都手写

export function applySoftDeleteMiddleware(prisma: PrismaClient) {
  prisma.$use(async (params, next) => {
    // 查询时自动过滤已删除记录
    if (params.model === 'User') {
      if (params.action === 'findMany' || params.action === 'findFirst') {
        params.args.where = {
          deletedAt: null,
          ...(params.args.where ?? {}),
        };
      }
      if (params.action === 'findUnique') {
        // findUnique 改为 findFirst 以支持额外过滤
        params.action = 'findFirst';
        params.args.where = { deletedAt: null, ...params.args.where };
      }
      // 将物理 delete 改为软删除
      if (params.action === 'delete') {
        params.action = 'update';
        params.args.data = { deletedAt: new Date() };
      }
      if (params.action === 'deleteMany') {
        params.action = 'updateMany';
        params.args.data = { deletedAt: new Date() };
      }
    }
    return next(params);
  });
}

// 真正需要物理删除时（如 GDPR 数据抹除）绕过 middleware
async function hardDelete(userId: string) {
  await prisma.$queryRaw`DELETE FROM "User" WHERE id = ${userId}`;
}
```

---

## 乐观锁（Optimistic Locking）

```
悲观锁：读取时锁定行（SELECT FOR UPDATE），阻塞其他事务。
        高并发时性能差，但绝对安全。

乐观锁：不加锁，提交时检查版本号。如果版本变了说明被其他事务修改，抛出错误让调用方重试。
        性能好，适合读多写少、冲突概率低的场景。
```

```typescript
// prisma/schema.prisma
model Account {
  id      String @id @default(cuid())
  balance Decimal
  version Int    @default(0)  // 乐观锁版本号
}
```

```typescript
// src/services/account.service.ts
async function transfer(fromId: string, toId: string, amount: Decimal) {
  const MAX_RETRIES = 3;

  for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
    try {
      // 读取当前状态和版本
      const from = await prisma.account.findUniqueOrThrow({ where: { id: fromId } });
      const to = await prisma.account.findUniqueOrThrow({ where: { id: toId } });

      if (from.balance.lessThan(amount)) {
        throw new BadRequestError('Insufficient balance');
      }

      // 更新时检查版本号（CAS: Compare And Swap）
      const [updatedFrom, updatedTo] = await prisma.$transaction([
        prisma.account.updateMany({
          where: {
            id: fromId,
            version: from.version,  // 版本必须匹配
          },
          data: {
            balance: { decrement: amount },
            version: { increment: 1 },
          },
        }),
        prisma.account.updateMany({
          where: {
            id: toId,
            version: to.version,
          },
          data: {
            balance: { increment: amount },
            version: { increment: 1 },
          },
        }),
      ]);

      // updateMany 返回 { count }，count=0 说明版本不匹配（被其他事务修改）
      if (updatedFrom.count === 0 || updatedTo.count === 0) {
        // 乐观锁冲突，重试
        if (attempt < MAX_RETRIES - 1) {
          await new Promise(r => setTimeout(r, Math.random() * 100));  // 随机退避
          continue;
        }
        throw new ConflictError('Transfer failed due to concurrent modification, please retry');
      }

      return { success: true };
    } catch (err) {
      if (err instanceof ConflictError && attempt < MAX_RETRIES - 1) continue;
      throw err;
    }
  }
}
```

---

## Cursor 分页（无限滚动 / 大数据集）

```
Offset 分页的问题：
  SELECT * FROM posts OFFSET 10000 LIMIT 20
  ↑ 数据库必须扫描并丢弃前 10000 行，越翻越慢。
  ↑ 数据新增/删除时页码漂移（翻到第二页发现有重复或漏掉的记录）。

Cursor 分页：
  以上一页最后一条记录的 ID 或时间戳作为游标，查"比游标更新/旧的 N 条"。
  无论翻多深都是 O(log n) 查询（索引直接定位）。
  适合：无限滚动、大数据集。
  不适合：跳页（"跳到第 100 页"这种需求）。
```

```typescript
// src/services/post.service.ts

interface CursorPaginationInput {
  cursor?: string;  // 上一页最后一条的 ID
  limit: number;
  direction?: 'forward' | 'backward';
}

interface CursorPaginationResult<T> {
  items: T[];
  nextCursor: string | null;
  prevCursor: string | null;
  hasNextPage: boolean;
  hasPrevPage: boolean;
}

async function getPosts(input: CursorPaginationInput): Promise<CursorPaginationResult<Post>> {
  const { cursor, limit, direction = 'forward' } = input;
  const take = direction === 'forward' ? limit + 1 : -(limit + 1);

  const posts = await prisma.post.findMany({
    take,
    ...(cursor ? {
      skip: 1,  // 跳过 cursor 本身
      cursor: { id: cursor },
    } : {}),
    orderBy: { createdAt: 'desc' },
    where: { deletedAt: null },
  });

  const hasMore = posts.length > limit;
  if (hasMore) posts.pop();  // 移除多取的那一条

  return {
    items: posts,
    nextCursor: hasMore ? posts[posts.length - 1].id : null,
    prevCursor: cursor ?? null,
    hasNextPage: hasMore,
    hasPrevPage: !!cursor,
  };
}
```

---

## 全文搜索（Prisma + PostgreSQL）

```typescript
// 方案一：PostgreSQL 原生全文搜索（中小规模，够用）
async function searchPosts(query: string) {
  // to_tsquery 支持 AND/OR/NOT，plainto_tsquery 更宽松（普通搜索词）
  const posts = await prisma.$queryRaw<Post[]>`
    SELECT id, title, body, ts_rank(search_vector, query) as rank
    FROM "Post",
         plainto_tsquery('english', ${query}) query
    WHERE search_vector @@ query
    ORDER BY rank DESC
    LIMIT 20
  `;
  return posts;
}

// schema.prisma 中需要添加 GIN 索引
// model Post {
//   searchVector Unsupported("tsvector")?
//   @@index([searchVector], type: Gin)
// }
//
// 用触发器自动更新 search_vector：
// CREATE OR REPLACE FUNCTION update_search_vector()
// RETURNS TRIGGER AS $$
// BEGIN
//   NEW.search_vector := to_tsvector('english', NEW.title || ' ' || NEW.body);
//   RETURN NEW;
// END;
// $$ LANGUAGE plpgsql;

// 方案二：Prisma 原生（较弱，只支持 contains/startsWith）
async function searchPostsSimple(query: string) {
  return prisma.post.findMany({
    where: {
      OR: [
        { title: { contains: query, mode: 'insensitive' } },
        { body: { contains: query, mode: 'insensitive' } },
      ],
    },
    orderBy: { createdAt: 'desc' },
    take: 20,
  });
}
// 注意：contains 走不了索引，大表性能差

// 方案三：Elasticsearch / Meilisearch（大规模，独立搜索引擎）
// 写入时同步到搜索引擎，查询走搜索引擎
```

---

## 事务（Transaction）模式

```typescript
// 模式一：批量操作（最常用，自动提交/回滚）
const [newPost, updatedUser] = await prisma.$transaction([
  prisma.post.create({ data: { title: 'Hello', authorId: userId } }),
  prisma.user.update({ where: { id: userId }, data: { postCount: { increment: 1 } } }),
]);

// 模式二：交互式事务（需要事务内的查询结果来决定下一步）
const result = await prisma.$transaction(async (tx) => {
  const account = await tx.account.findUniqueOrThrow({ where: { id: accountId } });

  if (account.balance < amount) {
    throw new Error('Insufficient funds');  // 自动 rollback
  }

  const updated = await tx.account.update({
    where: { id: accountId },
    data: { balance: { decrement: amount } },
  });

  await tx.transaction.create({
    data: { accountId, amount: -amount, type: 'debit' },
  });

  return updated;
}, {
  maxWait: 5000,   // 等待获取事务连接的最大时间
  timeout: 10000,  // 事务执行超时时间
  isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
});
```

---

## N+1 问题

```typescript
// ❌ N+1：查询 N 个 post，每个 post 再单独查 author（N 次额外查询）
const posts = await prisma.post.findMany({ take: 20 });
for (const post of posts) {
  const author = await prisma.user.findUnique({ where: { id: post.authorId } });
  // 20 个 post = 1 次 + 20 次 = 21 次查询
}

// ✓ 方案一：Prisma include（JOIN，一次查询）
const posts = await prisma.post.findMany({
  take: 20,
  include: { author: true },  // 自动 JOIN
});

// ✓ 方案二：Prisma select（只取需要的字段，减少数据传输）
const posts = await prisma.post.findMany({
  take: 20,
  select: {
    id: true,
    title: true,
    author: {
      select: { id: true, name: true, avatar: true },  // 不取 password 等敏感字段
    },
  },
});

// ✓ 方案三：DataLoader（批量 + 去重，适合 GraphQL 或嵌套查询）
import DataLoader from 'dataloader';

const userLoader = new DataLoader<string, User>(async (userIds) => {
  // 批量查询：多个 loadById('1'), loadById('2') → 合并为一次 findMany
  const users = await prisma.user.findMany({
    where: { id: { in: [...userIds] } },
  });
  // DataLoader 要求返回顺序与 keys 一致
  const userMap = new Map(users.map(u => [u.id, u]));
  return userIds.map(id => userMap.get(id) ?? new Error(`User ${id} not found`));
});

// 即使在循环中调用，DataLoader 会在同一个 tick 内批处理
for (const post of posts) {
  const author = await userLoader.load(post.authorId);  // 不是 N 次查询，而是 1 次
}
```

---

## 连接池配置

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  // 连接池大小：默认 = CPU 核数 * 2 + 1
  // 对于高并发场景，通过 URL 参数调整
  // DATABASE_URL="postgresql://...?connection_limit=20&pool_timeout=30"
});

// 连接池最佳实践：
// - Web 服务器：pool_size = (core_count * 2) + 1
// - 高并发：考虑 PgBouncer（连接池代理）减少 PostgreSQL 连接数
// - Serverless：每个实例连接数限制为 1-2，用 PgBouncer 或 Neon connection pooling
```

---

## 面试追问

**Q: 软删除和硬删除怎么选？**
A: 需要可恢复、有审计要求（金融、医疗）、有外键依赖（删除用户但订单要保留）→ 软删除。纯临时数据、GDPR 要求物理删除 → 硬删除。软删除的代价：所有查询必须过滤 `deletedAt`（遗漏会导致已删数据重新出现）、唯一索引冲突（email 唯一但已软删，新用户不能用同邮箱注册，解决：把 `deletedAt` 纳入唯一索引，或改用 `isDeleted` + 不同处理）。

**Q: Cursor 分页一定比 Offset 好吗？**
A: Cursor 分页在深页码（第 N 页，N 很大）时优势明显；浅页码（前几页）两者差距不大。Cursor 的限制：不能跳页（"跳到第 100 页"），不能在排序条件不稳定时工作（排序字段有重复值需要用复合 cursor）。管理后台常需要跳页 → Offset 更合适；用户无限滚动 → Cursor 更合适。

**Q: Prisma 事务隔离级别用哪个？**
A: 大多数场景用默认（ReadCommitted）够了。跨账户转账、需要防止幻读 → Serializable（性能代价最高）。Serializable 会增加死锁概率，需要在应用层做重试。一般原则：默认 ReadCommitted，有并发冲突问题时具体分析是否需要提升级别，不要一刀切用 Serializable。
