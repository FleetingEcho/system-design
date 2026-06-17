# API 设计深度专题

> 系统设计面试中，"API 怎么设计"往往是第一个被追问的细节。
> 本文覆盖：分页策略、版本管理、幂等性、tRPC 类型安全 API。

---

## 分页设计

### 三种分页方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **Offset 分页** | `LIMIT n OFFSET m` | 实现简单、支持跳页 | 深分页慢、数据漂移 | 后台管理、总量小 |
| **Cursor 分页** | 基于上一页最后一条记录的标识 | 性能稳定、无数据漂移 | 不支持跳页 | Feed 流、无限滚动 |
| **Keyset 分页** | 基于排序字段的值范围查询 | 利用索引、性能最优 | 依赖排序字段唯一性 | 大数据集、时序数据 |

---

### Offset 分页的问题

```sql
-- 第 1 页：快（扫描 10 行）
SELECT * FROM posts ORDER BY created_at DESC LIMIT 10 OFFSET 0;

-- 第 1000 页：慢（扫描并丢弃 10000 行）
SELECT * FROM posts ORDER BY created_at DESC LIMIT 10 OFFSET 10000;
```

**数据漂移**：用户看第 2 页时，如果有新数据插入，原第 1 页最后一条会"掉入"第 2 页，导致重复。

---

### Cursor 分页（推荐用于 Feed 流）

**原理**：用上一页最后一条数据的 ID（或时间戳）作为下一页的起点。

```typescript
// API 设计
// GET /api/posts?limit=10&cursor=<opaque_cursor>
// 响应：
interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    hasNextPage: boolean;
    endCursor: string | null;    // 下一次请求带这个
    hasPreviousPage: boolean;
    startCursor: string | null;
  };
}
```

**服务端实现**：

```typescript
// cursor 是 base64 编码的 JSON，对客户端不透明
function encodeCursor(data: { id: string; createdAt: number }): string {
  return Buffer.from(JSON.stringify(data)).toString('base64url');
}

function decodeCursor(cursor: string): { id: string; createdAt: number } {
  return JSON.parse(Buffer.from(cursor, 'base64url').toString());
}

async function getPosts(limit: number, cursor?: string) {
  let whereClause = {};

  if (cursor) {
    const { id, createdAt } = decodeCursor(cursor);
    whereClause = {
      OR: [
        { createdAt: { lt: createdAt } },
        // 同一时间戳时按 ID 排序（确保唯一性）
        { createdAt: createdAt, id: { lt: id } },
      ],
    };
  }

  // 多取 1 条，用来判断 hasNextPage
  const posts = await db.post.findMany({
    where: whereClause,
    orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
    take: limit + 1,
  });

  const hasNextPage = posts.length > limit;
  const data = hasNextPage ? posts.slice(0, limit) : posts;

  const endCursor = data.length > 0
    ? encodeCursor({ id: data.at(-1)!.id, createdAt: data.at(-1)!.createdAt.getTime() })
    : null;

  return { data, pagination: { hasNextPage, endCursor } };
}
```

---

### Keyset 分页（最高性能）

**原理**：直接用索引字段作为 WHERE 条件，避免 OFFSET 的全表扫描。

```sql
-- 第 1 页
SELECT * FROM posts
WHERE created_at <= '2024-01-10'
ORDER BY created_at DESC
LIMIT 10;

-- 第 2 页（用上一页最后一条的 created_at）
SELECT * FROM posts
WHERE created_at < '2024-01-08'   -- 严格小于，不含上一页最后一条
ORDER BY created_at DESC
LIMIT 10;
```

**对比**：
- Cursor：游标对客户端不透明，可以包含复合字段
- Keyset：直接暴露排序字段值（如果有安全顾虑用 Cursor 包装一层）

---

## API 版本管理

### 四种版本策略

| 策略 | 示例 | 优点 | 缺点 |
|------|------|------|------|
| **URL 路径版本** | `/v1/users`, `/v2/users` | 直观、可缓存 | URL 变化大，难以渐进迁移 |
| **请求头版本** | `Accept: application/vnd.api+json;version=2` | URL 稳定 | 不易测试（curl 需加 header）|
| **查询参数** | `/users?version=2` | 简单 | 容易忘记、可缓存性差 |
| **内容协商** | `Accept: application/vnd.myapi.v2+json` | RESTful 标准 | 实现复杂 |

**面试推荐方案：URL 路径版本**（简单、直观、工具链友好）。

### 版本共存策略

```typescript
// router/index.ts
app.use('/v1', v1Router);
app.use('/v2', v2Router);

// v2 不是从零开始写，而是在 v1 基础上修改
// v2Router 内部可以复用 v1 的 service 层逻辑
```

### 破坏性变更（Breaking Change）vs 非破坏性变更

```
非破坏性变更（不需要新版本）：
  ✓ 新增响应字段（老客户端忽略新字段）
  ✓ 新增可选请求参数
  ✓ 新增端点

破坏性变更（需要新版本）：
  ✗ 删除/重命名字段
  ✗ 改变字段类型（string → number）
  ✗ 改变语义（status 从 "active"/"inactive" 变成数字码）
  ✗ 改变 HTTP 方法
  ✗ 改变错误格式
```

### 版本废弃流程

```
1. 发布 v2
2. 在 v1 响应头加 Sunset 提示
   Sunset: Sat, 31 Dec 2024 23:59:59 GMT
   Deprecation: Tue, 01 Jan 2024 00:00:00 GMT
   Link: <https://api.example.com/v2/users>; rel="successor-version"
3. 90 天通知期
4. 监控 v1 流量降至 0
5. 关闭 v1
```

---

## 幂等性设计

### 什么是幂等性

**定义**：多次执行同一操作，结果和执行一次相同。

```
幂等操作：
  GET    — 天然幂等（读操作）
  PUT    — 幂等（全量更新）
  DELETE — 幂等（删了还删，状态不变）

非幂等操作：
  POST   — 非幂等（每次创建新资源）
  PATCH  — 可能非幂等（如 +1 操作）
```

### 幂等键（Idempotency Key）

让 POST 变幂等的标准方案：

```typescript
// 客户端生成唯一 key，重试时带相同 key
// POST /api/payments
// Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

// 服务端实现
async function createPayment(req: Request) {
  const idempotencyKey = req.headers.get('Idempotency-Key');

  if (!idempotencyKey) {
    return Response.json({ error: 'Idempotency-Key header required' }, { status: 400 });
  }

  // 检查是否已处理过
  const cached = await redis.get(`idem:${idempotencyKey}`);
  if (cached) {
    // 返回之前的结果（不重复执行）
    return Response.json(JSON.parse(cached), { status: 200 });
  }

  // 首次处理，加锁防止并发重复执行
  const acquired = await redis.set(
    `idem:${idempotencyKey}:lock`,
    '1',
    'NX',   // 仅当不存在时设置
    'EX', 30  // 30s 超时
  );

  if (!acquired) {
    // 另一个请求正在处理，返回 409
    return Response.json({ error: 'Request in progress' }, { status: 409 });
  }

  try {
    const result = await processPayment(req.body);

    // 缓存结果（TTL 24h，客户端重试窗口内有效）
    await redis.setex(`idem:${idempotencyKey}`, 86400, JSON.stringify(result));

    return Response.json(result, { status: 201 });
  } finally {
    await redis.del(`idem:${idempotencyKey}:lock`);
  }
}
```

---

## RESTful API 设计规范

### URL 设计

```
资源名用名词，复数：
  ✓ GET    /users
  ✓ POST   /users
  ✓ GET    /users/:id
  ✓ PUT    /users/:id
  ✓ DELETE /users/:id

嵌套资源（表达从属关系，最多 2 层）：
  ✓ GET /users/:userId/orders
  ✓ GET /users/:userId/orders/:orderId
  ✗ GET /users/:userId/orders/:orderId/items/:itemId/reviews  （太深）

操作用动词（无法用 REST 表达时）：
  POST /users/:id/activate
  POST /orders/:id/cancel
  POST /payments/:id/refund
```

### HTTP 状态码使用

```
2xx 成功：
  200 OK            — 通用成功（GET/PUT/PATCH）
  201 Created       — 创建成功（POST），Location 头指向新资源
  204 No Content    — 成功但无响应体（DELETE）
  202 Accepted      — 异步处理已接受（任务进队列）

4xx 客户端错误：
  400 Bad Request   — 请求格式错误、参数校验失败
  401 Unauthorized  — 未认证（token 缺失或过期）
  403 Forbidden     — 已认证但无权限
  404 Not Found     — 资源不存在
  409 Conflict      — 资源冲突（重复创建、版本冲突）
  422 Unprocessable — 格式正确但业务逻辑验证失败
  429 Too Many Req  — 限速

5xx 服务端错误：
  500 Internal      — 未预期的服务端异常
  502 Bad Gateway   — 上游服务不可用
  503 Unavailable   — 服务暂时不可用（过载/维护）
  504 Gateway Timeout — 上游超时
```

### 统一错误响应格式

```typescript
// 错误响应结构（参考 RFC 7807 Problem Details）
interface ApiError {
  type: string;         // 错误类型 URI（文档链接）
  title: string;        // 人类可读的错误标题
  status: number;       // HTTP 状态码
  detail: string;       // 具体错误描述
  instance?: string;    // 触发错误的资源路径
  errors?: FieldError[]; // 字段级错误（表单校验）
}

interface FieldError {
  field: string;
  message: string;
}

// 示例响应
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Request body contains invalid fields",
  "instance": "/api/users",
  "errors": [
    { "field": "email", "message": "Invalid email format" },
    { "field": "age", "message": "Must be at least 18" }
  ]
}
```

---

## API 安全设计

### 请求签名（防重放攻击）

```typescript
// 客户端签名
function signRequest(method: string, path: string, body: string, secretKey: string): {
  signature: string;
  timestamp: string;
  nonce: string;
} {
  const timestamp = Date.now().toString();
  const nonce = crypto.randomUUID();
  const payload = `${method}\n${path}\n${timestamp}\n${nonce}\n${body}`;
  const signature = createHmac('sha256', secretKey).update(payload).digest('hex');
  return { signature, timestamp, nonce };
}

// 服务端验证
async function verifySignature(req: Request): Promise<boolean> {
  const timestamp = req.headers.get('X-Timestamp');
  const nonce = req.headers.get('X-Nonce');
  const signature = req.headers.get('X-Signature');

  // 1. 时间戳检查（5 分钟窗口，防重放）
  if (Math.abs(Date.now() - parseInt(timestamp!)) > 5 * 60 * 1000) return false;

  // 2. Nonce 去重（Redis 检查，TTL = 时间窗口）
  const used = await redis.set(`nonce:${nonce}`, '1', 'NX', 'EX', 300);
  if (!used) return false;  // Nonce 已使用，重放攻击

  // 3. 重新计算签名对比
  const expectedSig = computeSignature(req, timestamp!, nonce!);
  return timingSafeEqual(Buffer.from(signature!), Buffer.from(expectedSig));
}
```

### 速率限制响应头

```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1704067200    (Unix 时间戳，限制重置时间)
Retry-After: 60                  (秒，客户端等待时间)
```

---

## API 文档即代码

### OpenAPI 3.0（TypeScript 类型生成）

```typescript
// 用 zod 定义 Schema，同时生成运行时校验和 OpenAPI 文档
import { z } from 'zod';
import { generateSchema } from '@anatine/zod-openapi';

const CreateUserSchema = z.object({
  name: z.string().min(1).max(100).openapi({ example: 'John Doe' }),
  email: z.string().email().openapi({ example: 'john@example.com' }),
  age: z.number().int().min(18).max(120),
});

// 同一个 Schema 既做运行时校验，又生成 OpenAPI 文档
// 避免文档和实现不一致的问题
export type CreateUserDto = z.infer<typeof CreateUserSchema>;
```

---

## 面试常见追问

**Q: Cursor 分页为什么不支持跳页？**
A: Cursor 分页依赖上一页的最后一条记录作为起点，无法直接跳到"第 N 页"。如果业务需要跳页（如搜索结果分页），用 Offset；如果是无限滚动 Feed，用 Cursor。

**Q: 幂等键 Redis 崩溃了怎么办？**
A: 两个选择：①降级（Redis 不可用时，不做幂等检查，业务层通过数据库唯一约束兜底）；②持久化幂等键到 DB（性能略差但更可靠）。对于支付等关键操作，DB 持久化是必选项。

**Q: API 版本维护多久？**
A: 业界惯例 12-18 个月。具体取决于：客户端更新速度（移动 App 强制更新难）、B2B（更长）vs B2C（更短）、合同 SLA 承诺。

**Q: GraphQL 和 REST 谁更好？**
A: 没有绝对优劣，看场景。REST 的优势：工具链成熟、CDN 缓存友好、简单场景实现快。GraphQL 的优势：客户端按需取数据、减少 Overfetching、Schema 即文档、适合多客户端。混合策略：对内（BFF→微服务）用 REST/gRPC，对外（客户端→BFF）用 GraphQL。
