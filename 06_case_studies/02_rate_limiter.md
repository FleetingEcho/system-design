# 系统设计案例：分布式限流系统

## TL;DR

限流（Rate Limiting）防止 API 被滥用或被单个用户打垮。核心难点是在**多台服务器之间共享限流状态**，同时保证低延迟（每个请求都要过限流检查）。

---

## 需求澄清

**功能需求：**
- 对超出频率限制的请求返回 HTTP 429（Too Many Requests）
- 支持不同维度的限流：IP / 用户 ID / API Key / 接口
- 支持不同的时间窗口：每秒、每分钟、每天

**非功能需求：**
- 低延迟（限流检查本身 < 1ms）
- 高可用（限流服务挂了不能影响正常业务）
- 分布式（多台 API Server 共享同一个限流状态）

---

## 四种限流算法

### 1. 固定窗口计数（Fixed Window Counter）

```
每分钟重置一次计数器。
当前计数 < 限制 → 允许，计数 +1
当前计数 >= 限制 → 拒绝

窗口: [0s, 60s)  计数: 0 → 100（达到上限）
窗口: [60s, 120s) 计数重置为 0
```

**问题：边界效应（Boundary Spike）**

```
11:59:59 的最后 1 秒：发了 100 个请求（用完额度）
12:00:00 的第 1 秒：发了 100 个请求（新窗口重置了）

→ 在 2 秒内实际发了 200 个请求，是限制的 2 倍
```

---

### 2. 滑动窗口日志（Sliding Window Log）

```
记录每个请求的时间戳（存在 Redis Sorted Set 里）：
  ZADD timestamps (当前时间戳) (请求唯一ID)
  ZREMRANGEBYSCORE timestamps 0 (当前时间 - 窗口大小)  // 删除过期记录
  count = ZCARD timestamps
  if count > limit → 拒绝
```

**优点：** 精确，没有边界效应
**缺点：** 每个用户存大量时间戳，内存消耗大（高频用户可能每分钟存 1000 条）

---

### 3. 滑动窗口计数（Sliding Window Counter）— 推荐

结合固定窗口的低内存占用和滑动窗口的精确性：

```
当前计数 = 上一个窗口计数 × 重叠比例 + 当前窗口计数

例子（限制每分钟 100 次）：
  上一个窗口（11:00 - 12:00）：90 次
  当前时间 12:00:30（本窗口经过了 30 秒，即 50%）
  当前窗口（12:00 - 13:00）：10 次

  当前估算计数 = 90 × (1 - 0.5) + 10 = 45 + 10 = 55 → 允许
```

**Redis 实现（只需要 2 个 Key）：**

```typescript
async function isAllowed(userId: string, windowSecs: number, limit: number): Promise<boolean> {
  const now = Date.now() / 1000; // 秒
  const currentWindow = Math.floor(now / windowSecs);
  const prevWindow = currentWindow - 1;
  const elapsed = (now % windowSecs) / windowSecs; // 当前窗口已过去的比例

  const currentKey = `rl:${userId}:${currentWindow}`;
  const prevKey = `rl:${userId}:${prevWindow}`;

  const [prevCount, currentCount] = await redis.mget(prevKey, currentKey);
  const estimatedCount = (Number(prevCount) || 0) * (1 - elapsed) + (Number(currentCount) || 0);

  if (estimatedCount >= limit) return false;

  // 原子递增当前窗口
  const pipe = redis.pipeline();
  pipe.incr(currentKey);
  pipe.expire(currentKey, windowSecs * 2); // TTL 设为 2 个窗口，保证前一窗口数据有效
  await pipe.exec();

  return true;
}
```

---

### 4. 令牌桶（Token Bucket）— 推荐（最灵活）

```
桶容量 = 100（burst 上限）
填充速率 = 10 token/秒

每秒往桶里放 10 个 token（不超过桶容量）
每个请求消耗 1 个 token
token 用完 → 拒绝

特点：允许短时突发（有积累的 token），但长期速率受限
```

**Redis 实现：**

```typescript
// 使用 Lua Script 保证原子性
const tokenBucketLua = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])  -- tokens per second
local now = tonumber(ARGV[3])          -- 当前时间戳（秒，精确到小数）
local cost = tonumber(ARGV[4])         -- 本次请求消耗的 token 数

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- 计算自上次填充以来应该补充的 token
local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)

if new_tokens < cost then
  return 0  -- 拒绝
end

redis.call('HMSET', key, 'tokens', new_tokens - cost, 'last_refill', now)
redis.call('EXPIRE', key, 3600)
return 1  -- 允许
`;

async function isAllowed(key: string, capacity: number, refillRate: number): Promise<boolean> {
  const result = await redis.eval(
    tokenBucketLua,
    1,
    `bucket:${key}`,
    capacity.toString(),
    refillRate.toString(),
    (Date.now() / 1000).toString(),
    '1'
  );
  return result === 1;
}
```

---

## 四种算法对比

| 算法 | 内存占用 | 精确度 | 支持突发 | 复杂度 |
|------|---------|--------|---------|--------|
| 固定窗口计数 | 极低（1个计数器）| 低（边界效应）| 否 | 最简单 |
| 滑动窗口日志 | 高（每请求1条记录）| 高 | 否 | 中等 |
| 滑动窗口计数 | 低（2个计数器）| 中高 | 否 | 中等 |
| 令牌桶 | 低（2个字段）| 高 | **是** | 稍高 |
| 漏桶（Leaky Bucket）| 低 | 高 | 否（严格匀速）| 中等 |

**推荐：**
- 简单场景：滑动窗口计数（兼顾精度和实现简单）
- 允许突发流量（如 API 调用）：令牌桶

---

## 分布式限流架构

### 问题：多台服务器不共享状态

```
Service A 看到计数：60（允许）
Service B 看到计数：60（允许）
实际发送：120 → 超限了！
```

### 解决方案：集中式 Redis

```
所有 API Server → Redis 集群（集中存储限流计数）

每次请求：
  1. API Server → Redis（INCR / Lua Script）
  2. Redis 返回当前计数
  3. 计数 > 限制 → 返回 429

延迟预算：
  API Server 到 Redis：~0.1ms（同机房）
  Lua Script 执行：< 0.1ms
  总共：< 1ms  ✓
```

### Redis 宕机怎么办？

```
方案一：降级放行（Fail Open）
  Redis 不可用时，直接放过所有请求，写告警
  适合：宁可被打也不能影响正常用户

方案二：降级拒绝（Fail Closed）
  Redis 不可用时，拒绝所有请求，返回 503
  适合：安全要求高的场景（如金融 API）

方案三：本地限流兜底
  Redis 不可用时，每台服务器用本地内存做粗粒度限流
  适合：折中方案，有一定防护但不精确
```

---

## 限流维度设计

```typescript
type RateLimitKey =
  | { type: 'ip'; ip: string }
  | { type: 'user'; userId: string }
  | { type: 'api_key'; key: string }
  | { type: 'endpoint'; userId: string; endpoint: string };  // 精细化：特定用户的特定接口

function buildRedisKey(key: RateLimitKey, window: number): string {
  const windowBucket = Math.floor(Date.now() / 1000 / window);
  switch (key.type) {
    case 'ip':       return `rl:ip:${key.ip}:${windowBucket}`;
    case 'user':     return `rl:user:${key.userId}:${windowBucket}`;
    case 'api_key':  return `rl:apikey:${key.key}:${windowBucket}`;
    case 'endpoint': return `rl:ep:${key.userId}:${key.endpoint}:${windowBucket}`;
  }
}
```

**多层限流（常见做法）：**

```
Layer 1（IP 层）：每 IP 每秒最多 100 次（防 DDoS）
Layer 2（用户层）：每用户每分钟最多 1000 次
Layer 3（接口层）：每用户每小时最多调用 /send_message 100 次

全部通过 → 放行
任一拒绝 → 返回 429
```

---

## 响应格式

限流拒绝时，应该告诉客户端何时可以重试（HTTP 标准）：

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1733000060

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please retry after 60 seconds.",
  "retry_after": 60
}
```

---

## 系统架构图

```
客户端请求
    ↓
[API Gateway] ← 第一层：IP 级别限流（防 DDoS）
    ↓
[Rate Limiter Middleware]
    ├─→ Redis Cluster（获取/更新计数）
    │      ├─ Key: "rl:user:{id}:minute"
    │      ├─ Key: "rl:ip:{ip}:second"
    │      └─ Key: "rl:apikey:{key}:hour"
    ↓ 通过
[业务服务]
    ↓
响应

超限时：
    ↓ 拒绝
429 + Retry-After Header
```

---

## Node.js 类比

Express 中间件形式：

```typescript
function createRateLimiter(limit: number, windowSecs: number) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userId = req.user?.id || req.ip;
    const allowed = await isAllowed(userId, windowSecs, limit);

    if (!allowed) {
      return res.status(429).json({
        error: 'rate_limit_exceeded',
        retry_after: windowSecs
      });
    }

    next();
  };
}

// 使用
app.use('/api/', createRateLimiter(100, 60));  // 每分钟 100 次
```

---

## 常见陷阱

1. **Lua Script 的重要性**：`INCR` + `GET` 是两步操作，非原子。两个服务器可能同时 GET 到 99，各自 INCR 到 100，实际允许了 2 个本应超限的请求。必须用 Lua Script 保证原子性

2. **TTL 设置**：计数器的 TTL 必须覆盖完整窗口。如果 TTL < 窗口大小，计数器在窗口内就消失了，导致计数被清零

3. **时钟同步**：滑动窗口和令牌桶都依赖时间戳。多台服务器的时钟不同步会导致计数偏差，部署时需要配置 NTP

4. **热 Key 问题**：如果某个 IP 在高频攻击，这个 IP 的限流 Key 会成为 Redis 的热点 Key（单节点打满）。解决：IP 限流用本地内存做第一层，减少对 Redis 的压力

---

## 面试 Q&A

### 简单

**Q: 限流系统如何返回正确的错误码？**

A: 返回 HTTP 429 Too Many Requests，并在响应头中包含：`Retry-After`（告诉客户端多少秒后可以重试）、`X-RateLimit-Limit`（总限制）、`X-RateLimit-Remaining`（剩余次数）、`X-RateLimit-Reset`（下次重置的 Unix 时间戳）。这样客户端可以实现智能退避。

**Q: 令牌桶和漏桶有什么区别？**

A: 令牌桶允许突发流量——桶里积累的 token 可以被一次性消费，短时间内可以超过平均速率；漏桶以固定速率"漏出"请求，不允许任何突发，输出严格匀速。API 限流通常选令牌桶（用户可以偶尔有突发但不能长期高频）；流量整形（如网络设备）用漏桶。

---

### 中等

**Q: 如果 Redis 宕机，限流系统怎么办？**

A: 有两种策略。Fail Open（宽松）：Redis 不可用时直接放过请求，避免影响正常用户，适合可用性优先的场景，但攻击者可以利用这个窗口。Fail Closed（严格）：Redis 不可用时拒绝所有请求，适合安全优先的场景。折中方案：本地内存做兜底限流（每台机器独立限制），Redis 恢复后再切回集中式。

**Q: 如何防止 "边界效应"（边界 spike）？**

A: 用滑动窗口而非固定窗口。滑动窗口计数的核心公式：`estimated = prev_count × (1 - elapsed_fraction) + current_count`，这样在窗口边界处也能平滑计数，避免重置带来的双倍流量问题。实现简单但精度略有牺牲（约 0.1% 的误差），工程上完全可以接受。

---

### 困难

**Q: 设计一个支持每秒处理 100 万个限流请求（全局 1 亿 DAU）的系统，如何保证低延迟和高可用？**

A: 分层设计：

**第一层（网络边缘）：** 在 CDN/负载均衡器层做 IP 级别的粗粒度限流，用内存中的计数器。这一层的目标是挡住明显的 DDoS 攻击，不需要精确，不需要 Redis。

**第二层（API Gateway）：** 用 Redis Cluster 做精细化限流（用户级、API Key 级）。100 万 QPS 的限流请求，Redis Cluster 理论上能支持（Redis 单机 10 万 QPS × 10 节点 = 100 万 QPS）。

**Redis 高可用：** 使用 Redis Cluster（16384 slot 自动分片）+ 主从复制（每个主节点有至少 1 个从节点）。某个主节点宕机，从节点自动升主，50-100ms 内恢复。

**本地缓存兜底：** 每台 API Server 维护本地的 Token Bucket（Caffeine/Node.js Map），Redis 不可达时启用，避免完全不限流。本地限流是"1/N 的限额"（N 是服务器数量），所以比全局限额严格，但保证了基本防护。

**监控：** 限流拒绝率异常升高 → 告警（可能是攻击，也可能是限额设置太低）；Redis 延迟超 5ms → 告警。

---

## 关联文档

- [../04_distributed/04_fault_tolerance.md](../04_distributed/04_fault_tolerance.md) — 令牌桶完整实现
- [../03_communication/01_sync.md](../03_communication/01_sync.md) — API Gateway 与限流集成
- [../02_storage/03_cache.md](../02_storage/03_cache.md) — Redis 缓存和热 Key 处理
