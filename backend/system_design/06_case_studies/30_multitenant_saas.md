# 多租户 SaaS 架构

> 数据隔离三种模型、租户路由、按租户功能开关、计费模型、安全边界。
> 面试场景：设计一个类似 Slack / Notion / Linear 的多租户 B2B SaaS 平台。

---

## 什么是多租户

```
单租户：每个客户独立部署一套系统（代码、DB、基础设施）
        优点：强隔离、定制化  缺点：成本高、运维复杂

多租户：所有客户共享同一套系统，逻辑上隔离
        优点：成本低、快速迭代  缺点：隔离复杂、性能隔离难

SaaS 场景的"租户"就是一个组织/公司（Team / Workspace / Organization）
用户属于租户，数据归属于租户，配置按租户隔离
```

---

## 数据隔离三种模型

```
模型一：共享数据库 + 行级隔离（Shared Database, Shared Schema）
  所有租户的数据在同一个表，通过 tenant_id 列区分
  
  user 表：id | tenant_id | email | name
  post 表：id | tenant_id | title | content
  
  优点：成本最低、维护最简单
  缺点：隔离性最弱（一个 SQL 漏掉 WHERE tenant_id = ? 就会泄露数据）
  适合：小型 SaaS、早期阶段、租户数量多但单个租户数据量小

模型二：共享数据库 + 独立 Schema（Shared Database, Separate Schema）
  每个租户有自己的 PostgreSQL Schema（命名空间）
  tenant_abc.users, tenant_abc.posts
  tenant_xyz.users, tenant_xyz.posts
  
  优点：隔离性好、SQL 误操作不会跨租户
  缺点：Schema 数量多时管理复杂（migration 需要逐 Schema 执行）
  适合：中型 SaaS、对隔离有一定要求

模型三：独立数据库（Separate Database）
  每个租户有独立的数据库实例
  
  优点：最强隔离、可以为大客户单独扩容/备份
  缺点：成本最高、连接池复杂（N 个租户 = N 个连接池）
  适合：大企业客户（Enterprise）、对合规有要求（GDPR 数据本地化）
```

---

## 模型一实现：行级隔离 + RLS

```typescript
// Prisma Schema：每张业务表加 tenantId
model Post {
  id        String   @id @default(cuid())
  tenantId  String   // 租户 ID，每张表必有
  title     String
  content   String
  authorId  String
  createdAt DateTime @default(now())

  @@index([tenantId])  // 每张表在 tenantId 上建索引
}

// ❌ 危险：忘记 WHERE tenantId 导致数据泄露
const posts = await prisma.post.findMany();

// ✓ 安全：始终带 tenantId
const posts = await prisma.post.findMany({
  where: { tenantId: ctx.tenantId },
});
```

```typescript
// 更安全的方式：Prisma Middleware 自动注入 tenantId
// 任何 findMany/findFirst 都自动加上 tenantId 过滤

function applyTenantIsolation(prisma: PrismaClient, tenantId: string): PrismaClient {
  const tenantPrisma = prisma.$extends({
    query: {
      $allModels: {
        async $allOperations({ operation, model, args, query }) {
          // 跳过不需要隔离的模型（如 Tenant 本身）
          if (model === 'Tenant') return query(args);

          // 读操作：注入 tenantId 过滤
          if (['findMany', 'findFirst', 'findUnique', 'count', 'aggregate'].includes(operation)) {
            args.where = { tenantId, ...(args.where ?? {}) };
          }
          // 写操作：注入 tenantId
          if (['create'].includes(operation)) {
            args.data = { tenantId, ...(args.data ?? {}) };
          }
          // 批量写：注入 tenantId
          if (['createMany'].includes(operation)) {
            args.data = (args.data as any[]).map(d => ({ tenantId, ...d }));
          }
          // 更新/删除：确保只操作本租户数据
          if (['update', 'delete'].includes(operation)) {
            args.where = { tenantId, ...(args.where ?? {}) };
          }

          return query(args);
        },
      },
    },
  });

  return tenantPrisma as unknown as PrismaClient;
}

// 请求中间件：构建租户专属的 Prisma client
export function tenantMiddleware(req: Request, res: Response, next: NextFunction) {
  const tenantId = req.user?.tenantId;
  if (!tenantId) throw new UnauthorizedError('No tenant context');

  // 挂载租户隔离的 prisma 到 request
  req.db = applyTenantIsolation(prisma, tenantId);
  next();
}
```

---

## PostgreSQL Row Level Security（RLS）

```sql
-- 更强的隔离：在数据库层面强制隔离，即使应用层忘记过滤也安全

-- 开启 RLS
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- 创建策略：只能看到自己租户的数据
CREATE POLICY tenant_isolation ON posts
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- 应用层：每次查询前设置当前租户
SET LOCAL app.current_tenant_id = 'tenant-abc-123';
```

```typescript
// 在 Prisma 中使用 RLS
async function withTenant<T>(tenantId: string, fn: () => Promise<T>): Promise<T> {
  return prisma.$transaction(async (tx) => {
    // 在事务级别设置租户 ID（SET LOCAL 只在当前事务有效）
    await tx.$executeRaw`SET LOCAL app.current_tenant_id = ${tenantId}`;
    return fn();
  });
}

// 使用（不需要任何 WHERE tenantId = ? 过滤，DB 自动处理）
const posts = await withTenant(tenantId, () =>
  prisma.post.findMany({ take: 20 })
);
```

---

## 租户路由（Tenant Resolution）

```typescript
// 三种租户识别方式：

// 1. 子域名：tenant.app.com
function resolveFromSubdomain(req: Request): string | null {
  const host = req.headers.host ?? '';
  const parts = host.split('.');
  // parts: ['tenant-slug', 'app', 'com']
  if (parts.length < 3) return null;
  return parts[0];  // 'tenant-slug'
}

// 2. 路径前缀：app.com/t/tenant-slug/...
function resolveFromPath(req: Request): string | null {
  const match = req.path.match(/^\/t\/([^/]+)/);
  return match ? match[1] : null;
}

// 3. JWT Payload（最常用，用户登录后 token 里含 tenantId）
function resolveFromToken(req: Request): string | null {
  const payload = req.user;  // 鉴权中间件已解析
  return payload?.tenantId ?? null;
}

// 中间件：解析租户
export async function resolveTenant(req: Request, res: Response, next: NextFunction) {
  const slug =
    resolveFromSubdomain(req) ??
    resolveFromPath(req) ??
    req.user?.tenantSlug;

  if (!slug) return next(new UnauthorizedError('Cannot determine tenant'));

  const tenant = await prisma.tenant.findUnique({
    where: { slug },
    select: { id: true, slug: true, plan: true, features: true },
  });

  if (!tenant) return next(new NotFoundError('Tenant not found'));

  req.tenant = tenant;
  next();
}
```

---

## 功能开关（Feature Flags by Tenant）

```typescript
// 不同 Plan 对应不同功能集
const PLAN_FEATURES: Record<string, string[]> = {
  free:       ['basic_analytics', 'up_to_5_members'],
  pro:        ['basic_analytics', 'advanced_analytics', 'up_to_50_members', 'custom_domain'],
  enterprise: ['*'],  // 所有功能
};

// 按租户的自定义 Feature Flag（Enterprise 客户专属功能）
interface Tenant {
  id: string;
  slug: string;
  plan: 'free' | 'pro' | 'enterprise';
  features: string[];  // 手动开启的额外功能（覆盖 plan 默认）
}

function hasFeature(tenant: Tenant, feature: string): boolean {
  // Enterprise 有所有功能
  if (tenant.plan === 'enterprise') return true;

  // 手动开启的功能
  if (tenant.features.includes(feature)) return true;

  // Plan 默认功能
  const planFeatures = PLAN_FEATURES[tenant.plan] ?? [];
  return planFeatures.includes(feature);
}

// 中间件：检查功能权限
export function requireFeature(feature: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.tenant) return next(new UnauthorizedError());
    if (!hasFeature(req.tenant, feature)) {
      return next(new ForbiddenError(`Feature "${feature}" not available on ${req.tenant.plan} plan`));
    }
    next();
  };
}

// 路由使用
router.post(
  '/analytics/export',
  authenticate,
  resolveTenant,
  requireFeature('advanced_analytics'),
  exportAnalyticsHandler
);
```

---

## 配额限制（Quota / Rate Limits per Tenant）

```typescript
// 每个租户有配额（成员数、存储空间、API 调用次数等）

const PLAN_QUOTAS: Record<string, { maxMembers: number; maxStorageGB: number; apiRateLimit: number }> = {
  free:       { maxMembers: 5,   maxStorageGB: 1,   apiRateLimit: 100  },
  pro:        { maxMembers: 50,  maxStorageGB: 100,  apiRateLimit: 1000 },
  enterprise: { maxMembers: Infinity, maxStorageGB: Infinity, apiRateLimit: 10000 },
};

async function checkMemberQuota(tenant: Tenant): Promise<void> {
  const quota = PLAN_QUOTAS[tenant.plan];
  const memberCount = await prisma.user.count({ where: { tenantId: tenant.id } });

  if (memberCount >= quota.maxMembers) {
    throw new ForbiddenError(
      `Member limit reached (${memberCount}/${quota.maxMembers}). Upgrade to add more members.`
    );
  }
}

// 按租户的 API 速率限制（Redis 实现）
async function tenantRateLimit(tenant: Tenant): Promise<void> {
  const quota = PLAN_QUOTAS[tenant.plan];
  const key = `ratelimit:tenant:${tenant.id}`;
  const count = await redis.incr(key);

  if (count === 1) await redis.expire(key, 3600);  // 1 小时窗口

  if (count > quota.apiRateLimit) {
    throw new TooManyRequestsError('API rate limit exceeded for your plan');
  }
}
```

---

## 数据库迁移策略（多租户）

```typescript
// 挑战：模型一（共享表）migration 简单，就是普通 Prisma migrate
// 模型二（独立 Schema）migration 需要逐 Schema 执行

// 模型二 migration 脚本
async function runMigrationForAllTenants() {
  const tenants = await prisma.tenant.findMany({ select: { id: true, dbSchema: true } });

  for (const tenant of tenants) {
    console.log(`Migrating tenant: ${tenant.dbSchema}`);

    // 切换到租户 Schema
    await prisma.$executeRawUnsafe(`SET search_path TO ${tenant.dbSchema}`);

    // 执行 migration（或者用 Prisma 的 --schema 参数）
    execSync(`DATABASE_URL="${buildTenantUrl(tenant)}" npx prisma migrate deploy`);
  }
}
```

---

## 面试追问

**Q: 三种隔离模型如何根据客户规模选择？**
A: 早期/中小客户 → 行级隔离（模型一），低成本快速迭代；进入规模化 → 可以按客户规模动态分配：Free/Pro 用共享 Schema，Enterprise 用独立 DB。这叫"混合多租户"，Salesforce / Notion 都这样做。判断标准：客户是否对隔离有强合规要求（HIPAA、SOC2）、是否需要单独备份恢复、是否有性能 SLA 要求 → 独立 DB。

**Q: 如何防止租户 A 的数据泄露给租户 B（除了 WHERE tenantId）？**
A: 纵深防御：① 应用层：Prisma 中间件自动注入 tenantId；② 数据库层：PostgreSQL RLS（最后一道防线，即使代码 bug 也能防）；③ 测试层：集成测试专门测试跨租户数据泄露（创建两个租户，验证 A 查不到 B 的数据）；④ 审计日志：记录所有数据访问（tenantId + userId + 操作），定期审查异常。

**Q: 多租户如何处理"噪音邻居"问题（一个租户大量请求影响其他租户）？**
A: 三层控制：① API 层：按 tenantId 做速率限制（Redis）；② 数据库层：查询超时（`statement_timeout`）+ 连接池按 tenant plan 分配权重；③ 应用层：后台任务（报表生成、数据导出）放入 BullMQ 队列，按 plan 优先级排队，避免大客户的报表任务占满 Worker。极端情况（Enterprise 客户）给独立 Worker Pod。
