# Node.js Redis 实战（ioredis）

> Cache-aside 实现、Redis 数据结构选型、缓存失效策略、Lua 原子操作、Pipeline 批量处理。
> 理论见 `backend/system_design/02_storage/03_cache.md`，本篇专注 Node.js 代码实现。

---

## ioredis 客户端配置

```typescript
// src/lib/redis.ts
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST ?? 'localhost',
  port: Number(process.env.REDIS_PORT ?? 6379),
  password: process.env.REDIS_PASSWORD,
  db: 0,
  // 连接池（ioredis 默认单连接，Cluster 模式内置多连接）
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  // 断线重连：指数退避，最多 10s
  retryStrategy: (times) => Math.min(times * 200, 10_000),
  // 命令超时（防止 Redis 卡死阻塞 Event Loop）
  commandTimeout: 5000,
  lazyConnect: true,
});

redis.on('error', (err) => {
  logger.error({ err }, 'Redis error');
});
redis.on('connect', () => {
  logger.info('Redis connected');
});

export { redis };

// Graceful shutdown（见 06_production_patterns.md）
// await redis.quit();
```

---

## Cache-Aside 模式（最常用）

```typescript
// src/lib/cache.ts
// 通用 Cache-Aside 包装器：先查缓存，miss 时查 DB 并写缓存

export async function withCache<T>(
  key: string,
  ttlSeconds: number,
  fetcher: () => Promise<T>
): Promise<T> {
  // 1. 查缓存
  const cached = await redis.get(key);
  if (cached !== null) {
    return JSON.parse(cached) as T;
  }

  // 2. Cache Miss → 查数据源
  const data = await fetcher();

  // 3. 写缓存（SET key value EX ttl）
  await redis.setex(key, ttlSeconds, JSON.stringify(data));

  return data;
}

// 使用
const user = await withCache(
  `user:${userId}`,
  300,  // 5 分钟
  () => prisma.user.findUniqueOrThrow({ where: { id: userId } })
);

// 主动失效（写入时）
async function updateUser(userId: string, data: UpdateUserInput) {
  const updated = await prisma.user.update({ where: { id: userId }, data });
  await redis.del(`user:${userId}`);  // 删除缓存，下次请求重建
  return updated;
}
```

---

## Cache Stampede 防护（防缓存击穿）

```typescript
// 问题：热点 key 过期瞬间，大量请求同时 miss → 同时打到 DB
// 解法：分布式锁 + 单飞（Single-flight）

const pendingRequests = new Map<string, Promise<unknown>>();

export async function withCacheSingleFlight<T>(
  key: string,
  ttlSeconds: number,
  fetcher: () => Promise<T>
): Promise<T> {
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached) as T;

  // 如果已经有相同 key 的请求在飞，复用同一个 Promise
  if (pendingRequests.has(key)) {
    return pendingRequests.get(key) as Promise<T>;
  }

  const promise = (async () => {
    // 再查一次（防止等待锁期间已经被其他进程填入）
    const cached2 = await redis.get(key);
    if (cached2 !== null) return JSON.parse(cached2) as T;

    const data = await fetcher();
    await redis.setex(key, ttlSeconds, JSON.stringify(data));
    return data;
  })().finally(() => {
    pendingRequests.delete(key);
  });

  pendingRequests.set(key, promise);
  return promise;
}

// 多进程场景：用 Redis 分布式锁实现真正的单飞
export async function withCacheLock<T>(
  key: string,
  ttlSeconds: number,
  fetcher: () => Promise<T>
): Promise<T> {
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached) as T;

  const lockKey = `lock:${key}`;
  const lockValue = randomUUID();
  const lockAcquired = await redis.set(lockKey, lockValue, 'PX', 5000, 'NX');

  if (!lockAcquired) {
    // 没有拿到锁，等待后重试（其他进程正在重建缓存）
    await new Promise(r => setTimeout(r, 100));
    return withCacheLock(key, ttlSeconds, fetcher);  // 递归重试
  }

  try {
    const data = await fetcher();
    await redis.setex(key, ttlSeconds, JSON.stringify(data));
    return data;
  } finally {
    // 释放锁（Lua 保证原子性：只删除自己的锁）
    await redis.eval(
      `if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`,
      1,
      lockKey,
      lockValue
    );
  }
}
```

---

## Redis 数据结构选型

### Hash：对象存储（用户 Session、配置）

```typescript
// Hash 比 JSON string 更节省内存，且可以单独更新某个字段
// HSET user:1 name "Alice" email "a@b.com" role "admin"

// 存储 Session
async function saveSession(sessionId: string, data: Record<string, string>, ttlSec: number) {
  await redis.hset(`session:${sessionId}`, data);
  await redis.expire(`session:${sessionId}`, ttlSec);
}

async function getSession(sessionId: string): Promise<Record<string, string> | null> {
  const data = await redis.hgetall(`session:${sessionId}`);
  return Object.keys(data).length ? data : null;
}

// 原子更新单个字段（不需要先 get 再 set）
await redis.hset(`session:${sessionId}`, 'lastActiveAt', Date.now().toString());
```

### Sorted Set：排行榜 / 限时有序集合

```typescript
// ZADD leaderboard 1000 "user:1"
// ZREVRANK leaderboard "user:1"  → 排名（0-based）
// ZREVRANGE leaderboard 0 9 WITHSCORES → 前 10 名

const LEADERBOARD_KEY = 'game:leaderboard:weekly';

async function updateScore(userId: string, newScore: number) {
  // ZADD ... GT：只有 newScore 大于现有 score 才更新
  await redis.zadd(LEADERBOARD_KEY, 'GT', newScore, userId);
}

async function getTopN(n: number): Promise<Array<{ userId: string; score: number }>> {
  const results = await redis.zrevrange(LEADERBOARD_KEY, 0, n - 1, 'WITHSCORES');
  // results: ['user:1', '1000', 'user:2', '900', ...]
  const top: Array<{ userId: string; score: number }> = [];
  for (let i = 0; i < results.length; i += 2) {
    top.push({ userId: results[i], score: Number(results[i + 1]) });
  }
  return top;
}

async function getRank(userId: string): Promise<number | null> {
  const rank = await redis.zrevrank(LEADERBOARD_KEY, userId);
  return rank !== null ? rank + 1 : null;  // 转为 1-based
}

// 自动过期成员（限时活动）
async function addWithExpiry(userId: string, score: number, expiryAt: Date) {
  await redis.zadd(LEADERBOARD_KEY, score, userId);
  // 用另一个 sorted set 按过期时间排序，定时任务清理
  await redis.zadd('leaderboard:expiry', expiryAt.getTime(), userId);
}
```

### List：最近历史 / 消息队列

```typescript
// List 用于：最近浏览、消息队列（LPUSH + BRPOP）

const MAX_HISTORY = 50;

// 维护最近 N 条浏览记录（LPUSH + LTRIM）
async function addToHistory(userId: string, productId: string) {
  const key = `user:${userId}:history`;
  await redis
    .pipeline()
    .lrem(key, 0, productId)       // 先删除已有的（去重）
    .lpush(key, productId)         // 插到头部
    .ltrim(key, 0, MAX_HISTORY - 1) // 只保留最近 N 条
    .expire(key, 7 * 86400)        // 7 天过期
    .exec();
}

async function getHistory(userId: string): Promise<string[]> {
  return redis.lrange(`user:${userId}:history`, 0, -1);
}

// 简单消息队列（生产）
async function enqueue(queueName: string, job: unknown) {
  await redis.lpush(`queue:${queueName}`, JSON.stringify(job));
}

// 消费（阻塞等待，适合 Worker 进程）
async function consume(queueName: string): Promise<unknown> {
  const result = await redis.brpop(`queue:${queueName}`, 30);  // 最多等 30s
  if (!result) return null;
  return JSON.parse(result[1]);
}
```

### Pub/Sub：实时通知

```typescript
// Pub/Sub 适合一对多广播，不需要持久化
// 注意：ioredis 的 subscribe 模式下连接只能用于订阅，不能执行其他命令
// → 需要两个 Redis 连接（一个 pub，一个 sub）

const pubClient = new Redis(redisConfig);
const subClient = new Redis(redisConfig);

// 订阅
subClient.subscribe('notifications:new_order', (err, count) => {
  if (err) logger.error({ err }, 'Subscribe error');
  logger.info({ channels: count }, 'Subscribed');
});

subClient.on('message', (channel, message) => {
  const data = JSON.parse(message);
  // 广播给 WebSocket 客户端
  io.to(`user:${data.userId}`).emit('notification', data);
});

// 发布
async function publishNotification(userId: string, notification: unknown) {
  await pubClient.publish('notifications:new_order', JSON.stringify({ userId, ...notification }));
}

// Redis Streams（比 Pub/Sub 更强：持久化、消费者组、重新投递）
// 适合需要保证消费的场景，类似 Kafka 的轻量版
async function publishEvent(stream: string, event: Record<string, string>) {
  // XADD stream * field1 val1 field2 val2
  await redis.xadd(stream, '*', ...Object.entries(event).flat());
}
```

---

## Lua 脚本：原子操作

```typescript
// 场景：需要多个 Redis 命令原子执行（中间不能被其他命令插入）
// Lua 脚本在 Redis 单线程中原子执行

// 原子性递增计数器（带最大值限制）
const incrementWithMaxScript = `
  local current = redis.call('GET', KEYS[1])
  local max = tonumber(ARGV[1])
  local incr = tonumber(ARGV[2])
  
  if current == false then
    current = 0
  else
    current = tonumber(current)
  end
  
  if current + incr > max then
    return -1  -- 超过限制，返回 -1
  end
  
  local new_value = redis.call('INCRBY', KEYS[1], incr)
  redis.call('EXPIRE', KEYS[1], tonumber(ARGV[3]))
  return new_value
`;

async function reserveInventory(productId: string, quantity: number): Promise<boolean> {
  const result = await redis.eval(
    incrementWithMaxScript,
    1,
    `inventory:reserved:${productId}`,  // KEYS[1]
    '1000',   // ARGV[1]: max reserved
    String(quantity),  // ARGV[2]: increment
    '3600'    // ARGV[3]: TTL
  );
  return result !== -1;  // -1 表示超过限制
}

// 分布式限流（滑动窗口，比 INCR+EXPIRE 精确）
const slidingWindowScript = `
  local key = KEYS[1]
  local now = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local limit = tonumber(ARGV[3])
  
  -- 删除窗口外的旧请求
  redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
  
  -- 统计窗口内请求数
  local count = redis.call('ZCARD', key)
  
  if count < limit then
    -- 记录当前请求
    redis.call('ZADD', key, now, now .. math.random())
    redis.call('PEXPIRE', key, window)
    return 1  -- 允许
  end
  
  return 0  -- 拒绝
`;

async function checkRateLimit(identifier: string, windowMs: number, limit: number): Promise<boolean> {
  const result = await redis.eval(
    slidingWindowScript,
    1,
    `ratelimit:${identifier}`,
    Date.now(),
    windowMs,
    limit
  );
  return result === 1;
}
```

---

## Pipeline：批量操作减少 RTT

```typescript
// 问题：10 次 redis 命令 = 10 次网络往返（即使每次 < 1ms，累计也显著）
// Pipeline：将命令打包一次性发送，一次往返拿到所有结果

// ❌ 串行（10 次网络往返）
for (const userId of userIds) {
  const data = await redis.get(`user:${userId}`);
}

// ✓ Pipeline（1 次网络往返）
async function batchGetUsers(userIds: string[]): Promise<Array<string | null>> {
  const pipeline = redis.pipeline();
  for (const userId of userIds) {
    pipeline.get(`user:${userId}`);
  }
  const results = await pipeline.exec();
  // results: [[null, value], [null, value], ...]  → [err, value] pairs
  return results?.map(([err, value]) => {
    if (err) throw err;
    return value as string | null;
  }) ?? [];
}

// 批量写入（例如：缓存预热）
async function batchSetUsers(users: Array<{ id: string; data: unknown }>, ttlSec: number) {
  const pipeline = redis.pipeline();
  for (const { id, data } of users) {
    pipeline.setex(`user:${id}`, ttlSec, JSON.stringify(data));
  }
  await pipeline.exec();
}

// MULTI/EXEC（事务：保证顺序执行，但不回滚）
async function atomicTransfer(fromKey: string, toKey: string, amount: number) {
  const multi = redis.multi();
  multi.decrby(fromKey, amount);
  multi.incrby(toKey, amount);
  const results = await multi.exec();
  // 如果某条命令执行失败（如类型错误），其他命令仍然执行
  // Redis 事务不支持回滚，要真正原子性用 Lua 脚本
  return results;
}
```

---

## 缓存分层策略

```typescript
// L1：进程内 LRU 缓存（极快，< 1μs，但不跨进程）
// L2：Redis（快，< 1ms，跨进程共享）
// L3：数据库（慢，10-100ms）

import { LRUCache } from 'lru-cache';

const l1Cache = new LRUCache<string, unknown>({
  max: 1000,         // 最多 1000 条
  ttl: 30 * 1000,    // 30s
});

export async function withLayeredCache<T>(
  key: string,
  redisTtlSec: number,
  fetcher: () => Promise<T>
): Promise<T> {
  // L1 查询
  const l1 = l1Cache.get(key);
  if (l1 !== undefined) return l1 as T;

  // L2 查询
  const l2 = await redis.get(key);
  if (l2 !== null) {
    const data = JSON.parse(l2) as T;
    l1Cache.set(key, data);  // 回填 L1
    return data;
  }

  // L3 查询
  const data = await fetcher();
  await redis.setex(key, redisTtlSec, JSON.stringify(data));
  l1Cache.set(key, data);
  return data;
}
```

---

## 面试追问

**Q: Redis INCR 和 Lua 脚本保证原子性，为什么还会有 ABA 问题？**
A: INCR 是单命令原子，Lua 脚本是多命令原子——都没有 ABA 问题，因为 Redis 是单线程，脚本执行期间不会有其他命令插入。ABA 问题出现在"读-改-写"跨多个命令的场景（比如用 GET 读出值、客户端计算、再 SET 写回），这中间可能被其他客户端修改。解法：用 Lua 脚本把读改写合并为原子操作，或用 WATCH + MULTI/EXEC（乐观锁：如果 WATCH 的 key 被修改，EXEC 返回 nil）。

**Q: Redis Pub/Sub 和 Redis Streams 怎么选？**
A: Pub/Sub 是即时广播，不持久化，订阅者离线期间的消息会丢失，适合实时通知（在线用户推送）；Streams 持久化、支持消费者组、支持从任意位置重新消费、支持 Pending 消息重新投递，适合可靠事件传递（订单事件、审计日志）。如果需要"至少消费一次"语义 → Streams；只需要当前在线用户实时推送 → Pub/Sub 够了。

**Q: Redis 缓存和数据库不一致怎么处理？**
A: 三种策略：① Cache-Aside + 写时删缓存（最常用，简单但有短暂不一致窗口）；② Write-Through（写 DB 同时写缓存，强一致但写放大）；③ 消息队列异步更新（DB 变更 → 发消息 → Consumer 更新缓存，最终一致，复杂度高）。实践中大多用 Cache-Aside + 删缓存，容忍 TTL 内的短暂不一致，对一致性要求极高的数据（如余额）不走缓存，直接查 DB。
