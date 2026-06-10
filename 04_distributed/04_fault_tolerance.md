# 容错：熔断 / 限流 / 幂等性 / 重试

## TL;DR

- **熔断器（Circuit Breaker）**：下游服务故障时快速失败，避免雪崩
- **限流（Rate Limiting）**：保护服务不被流量打垮，令牌桶是最常用算法
- **幂等性（Idempotency）**：操作多次等于一次，是安全重试的前提
- **重试（Retry）**：自动重试失败操作，必须配合幂等和指数退避

这四个机制共同构成微服务容错的基础。

---

## 熔断器（Circuit Breaker）

### 问题：级联失败（Cascading Failure）

```
用户请求 → 服务A → 服务B（响应慢/不可用）

服务B 超时，服务A 等待...
新请求到达，继续等服务B...
服务A 的线程/连接池耗尽
服务A 也开始超时
调用服务A 的服务C 也开始超时
整个系统雪崩
```

没有熔断器，一个下游故障会把整个调用链拖垮。

### 熔断器的三种状态

```
        故障率超过阈值
CLOSED ────────────────→ OPEN
  ↑                        |
  │ 半开测试成功            │ 超过等待时间
  │                        ↓
  └──────────── HALF-OPEN ←┘
                   │
                   │ 测试失败
                   └──→ OPEN
```

**CLOSED（关闭，正常状态）：**
请求正常通过，统计成功/失败率。当失败率超过阈值（如 50%），切换到 OPEN。

**OPEN（打开，熔断状态）：**
所有请求立即失败，不调用下游（快速失败）。等待一段时间（如 30 秒）后切换到 HALF-OPEN。

**HALF-OPEN（半开，探测状态）：**
允许少量请求通过。如果成功，切回 CLOSED；如果失败，回到 OPEN。

### 代码实现示例

```typescript
class CircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private failureCount = 0;
  private lastFailureTime: number | null = null;
  private readonly threshold = 5;        // 5次失败触发熔断
  private readonly resetTimeout = 30000; // 30秒后进入半开状态

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime! > this.resetTimeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');  // 快速失败
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}

// 使用
const breaker = new CircuitBreaker();
const result = await breaker.call(() => externalService.fetchData());
```

### 熔断器的降级策略

熔断后不能只返回错误，要有**降级（Fallback）**方案：

```typescript
async function getUserWithFallback(userId: string): Promise<User> {
  try {
    return await breaker.call(() => userService.getUser(userId));
  } catch (error) {
    // 降级：返回缓存数据
    const cached = await redis.get(`user:${userId}`);
    if (cached) return JSON.parse(cached);
    // 降级：返回默认值
    return { id: userId, name: '用户', avatar: defaultAvatar };
  }
}
```

---

## 限流（Rate Limiting）

限流保护服务不被超过其处理能力的流量打垮。

### 算法 1：固定窗口（Fixed Window）

```
每分钟最多 100 个请求
时间窗口：0:00-1:00、1:00-2:00...

问题（边界效应）：
  0:59 发 100 个请求 → 全部通过（0:00-1:00 窗口）
  1:00 发 100 个请求 → 全部通过（1:00-2:00 新窗口）
→ 在 0:59-1:01 的 2 秒内，发出了 200 个请求
```

### 算法 2：滑动窗口（Sliding Window）

```
记录每个请求的时间戳，用滑动的最近 60 秒窗口统计

每次请求到来：
  统计过去 60 秒内的请求数
  如果 >= 100，拒绝
  如果 < 100，允许并记录时间戳

优点：没有边界效应
缺点：需要存储每个请求的时间戳，内存消耗大
```

### 算法 3：令牌桶（Token Bucket）— 推荐

```
一个桶，容量 100 个令牌
每秒往桶里放 10 个令牌（最多 100 个）
每个请求消耗 1 个令牌
没有令牌 → 拒绝请求

特点：
  允许短时突发：桶里积累了 100 个令牌，可以瞬间处理 100 个请求
  长期速率：平均不超过每秒 10 个请求（令牌生成速率）
```

```typescript
class TokenBucket {
  private tokens: number;
  private lastRefill: number = Date.now();

  constructor(
    private readonly capacity: number,    // 桶容量
    private readonly refillRate: number   // 每秒补充的令牌数
  ) {
    this.tokens = capacity;
  }

  allowRequest(): boolean {
    this.refill();
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }
    return false;
  }

  private refill(): void {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;  // 秒
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.refillRate);
    this.lastRefill = now;
  }
}
```

### 算法 4：漏桶（Leaky Bucket）

```
请求进入桶（队列），以固定速率流出处理
桶满时，新请求被丢弃

vs 令牌桶：漏桶严格控制输出速率（不允许突发），令牌桶允许短时突发
```

适合：需要严格平滑输出的场景（如向第三方 API 发请求，该 API 有严格限流）

### 分布式限流

单机限流用本地变量，分布式限流需要共享计数器：

```typescript
// Redis 实现分布式令牌桶
async function allowRequest(userId: string): Promise<boolean> {
  const key = `ratelimit:${userId}`;
  const script = `
    local tokens = tonumber(redis.call('GET', KEYS[1])) or tonumber(ARGV[1])
    if tokens >= 1 then
      redis.call('SET', KEYS[1], tokens - 1, 'EX', 60)
      return 1
    else
      return 0
    end
  `;
  const result = await redis.eval(script, 1, key, BUCKET_CAPACITY);
  return result === 1;
}
```

---

## 幂等性（Idempotency）

### 什么是幂等

**幂等操作：** 执行一次和执行多次的结果相同。

```
幂等的：
  DELETE /users/1  →  用户删除（第二次调用：用户已不存在，返回 404，但状态没有进一步改变）
  PUT /users/1 {name: 'Alice'}  →  无论调用多少次，用户名都是 Alice

不幂等的：
  POST /orders  →  每次调用创建一个新订单
  POST /accounts/1/deposit {amount: 100}  →  每次调用增加 100 元（重复调用 = 重复存款）
```

### 为什么幂等重要

网络不可靠，客户端可能不知道请求是否成功：

```
客户端 → 发送转账请求
服务器 → 处理成功，但响应在网络中丢失
客户端 → 超时，重试
服务器 → 收到重复请求，再次扣款！
```

如果服务器是幂等的，重试就是安全的。

### 实现幂等：幂等 Key

客户端生成唯一的幂等 Key，服务端检查这个 Key 是否已处理过：

```typescript
// 客户端：每次操作生成唯一 ID
const idempotencyKey = uuidv4();

// 第一次请求
await axios.post('/transfer', { amount: 100, to: 'Bob' }, {
  headers: { 'Idempotency-Key': idempotencyKey }
});

// 网络超时，重试（用同一个 Key）
await axios.post('/transfer', { amount: 100, to: 'Bob' }, {
  headers: { 'Idempotency-Key': idempotencyKey }  // 相同的 Key
});
```

```typescript
// 服务端：检查幂等 Key
async function transfer(amount: number, to: string, idempotencyKey: string): Promise<void> {
  // 查是否已处理
  const existing = await db.idempotencyKeys.findOne({ key: idempotencyKey });
  if (existing) {
    return existing.result;  // 返回之前的结果
  }

  // 在事务中处理 + 记录 Key
  const result = await db.transaction(async (tx) => {
    await tx.accounts.decrement({ userId: currentUser }, amount);
    await tx.accounts.increment({ userId: to }, amount);
    await tx.idempotencyKeys.insert({ key: idempotencyKey, result: 'success' });
    return 'success';
  });

  return result;
}
```

### 天然幂等 vs 需要设计幂等

```
天然幂等（SELECT, GET）：读操作天然幂等
需要设计（写操作）：
  - 用唯一约束防止重复插入
  - 用幂等 Key + 记录
  - 用"状态机"设计（只有在特定状态下才能执行操作）
```

---

## 重试（Retry）

### 不能无脑重试

```
服务崩溃，所有请求都在重试
→ 大量重试流量涌向恢复中的服务
→ 服务刚想恢复，又被打崩（重试风暴）
```

### 指数退避（Exponential Backoff）

每次重试等待时间翻倍，加上随机抖动（Jitter）避免所有客户端同时重试：

```typescript
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;

      const baseDelay = Math.pow(2, attempt) * 1000;  // 1s, 2s, 4s
      const jitter = Math.random() * 1000;             // 随机 0-1s 抖动
      await sleep(baseDelay + jitter);
    }
  }
  throw new Error('Max retries reached');
}

// 使用
const result = await retryWithBackoff(() => callExternalAPI());
```

### 什么情况下应该重试

```
应该重试：
  - 网络超时（请求可能没到达）
  - 5xx 服务器错误（服务暂时不可用）
  - 连接被拒绝（服务刚重启）

不应该重试：
  - 4xx 客户端错误（400 Bad Request，401 Unauthorized, 422 参数错误）
    重试也不会成功，没有意义
  - 业务错误（余额不足）
    重试不会改变结果
```

### 重试 + 熔断 + 幂等的组合

```
请求失败
  ↓
幂等 Key 确保重试安全
  ↓
指数退避重试（最多 3 次）
  ↓
熔断器检测到持续失败（3次都失败）
  ↓
熔断打开，后续请求快速失败（不再重试）
  ↓
等待 30 秒后探测
```

---

## 超时（Timeout）

超时是容错中最基础的机制，但经常被忽视：

```typescript
// 不设超时（危险！）
const result = await axios.get('http://slow-service/api');
// 如果 slow-service 没有响应，这个 await 会永远挂起
// 连接池耗尽，服务崩溃

// 设置超时
const result = await axios.get('http://slow-service/api', { timeout: 3000 });  // 3秒超时
```

**超时的设置原则：**
- **客户端超时 > 服务端超时**：避免服务端已超时取消操作，但客户端还在等
- **不能太长**：超时太长，连接池耗尽，服务崩溃
- **不能太短**：超时太短，正常的慢操作被误判为失败，重试增多

---

## 与 Node.js/TS 生态的类比

```typescript
// 在 Express 中完整的容错链：
import rateLimit from 'express-rate-limit';

// 1. 限流中间件
const limiter = rateLimit({
  windowMs: 60 * 1000,   // 1分钟
  max: 100,               // 每个 IP 最多 100 次
  message: { error: 'Too many requests' }
});
app.use('/api/', limiter);

// 2. 超时
app.use((req, res, next) => {
  req.setTimeout(5000, () => res.status(408).json({ error: 'Timeout' }));
  next();
});

// 3. 熔断器包装外部调用
const paymentBreaker = new CircuitBreaker();
app.post('/order', async (req, res) => {
  try {
    await paymentBreaker.call(() => paymentService.charge(req.body));
    res.json({ success: true });
  } catch (error) {
    res.status(503).json({ error: 'Payment service unavailable' });
  }
});
```

---

## 常见陷阱

1. **重试不幂等的操作**：POST 创建资源的接口没有幂等保护，重试导致重复数据
2. **重试风暴**：大量客户端同时重试，流量是原来的几倍，把恢复中的服务再次打垮
3. **熔断后没有降级**：熔断打开后只返回 500 错误，应该有降级逻辑（返回缓存、默认值）
4. **限流粒度太粗**：按 IP 限流，但后端 API 被服务器端大量调用，限流没有保护到真正需要保护的接口
5. **没有设置超时**：下游服务响应慢，上游连接/线程全部阻塞在等待，最终 OOM

---

## 面试常见问答

### 简单

**Q：什么是熔断器？它的三种状态是什么？**

A：熔断器是一种容错模式，当下游服务故障率超过阈值时，自动切断对该服务的调用，避免错误级联传播。三种状态：CLOSED（正常，请求通过）、OPEN（熔断，请求立即失败，不调用下游）、HALF-OPEN（探测，允许少量请求通过检测服务是否恢复，成功则回 CLOSED，失败则回 OPEN）。

---

**Q：令牌桶和漏桶算法有什么区别？**

A：两者都是限流算法，主要区别在于是否允许突发流量。令牌桶：桶里可以积累令牌，短时间内可以消耗积累的令牌处理突发请求，但长期平均速率受限于令牌生成速率。漏桶：请求进入队列，以固定速率流出处理，不允许突发，输出速率严格平滑。选择：允许短时突发用令牌桶；需要严格平滑输出（如调用有速率限制的第三方 API）用漏桶。

---

### 中等

**Q：什么是幂等 Key？如何在支付接口中实现它？**

A：幂等 Key 是客户端为每次独立操作生成的全局唯一标识符，服务端用它防止重复处理。在支付接口中实现：客户端在发起支付时生成 UUID 作为幂等 Key，放在请求头（Idempotency-Key）中。服务端收到请求时先查幂等记录表，如果该 Key 已存在，直接返回之前的处理结果（无论成功还是失败）；如果不存在，在同一个数据库事务中处理支付并写入幂等记录表。网络超时后客户端使用相同 Key 重试，服务端发现已处理，直接返回之前的结果，不会重复扣款。

---

### 难

**Q：设计一个微服务系统的完整容错策略，包括超时、重试、熔断、降级，它们之间如何协作？**

A：容错策略要分层设计，由内到外：

**第一层：超时**
每个外部调用设置明确超时（DB 查询 1s，内部服务调用 3s，外部 API 5s）。超时时间要经过压测，不能拍脑袋。客户端超时应略大于服务端，避免服务端已超时返回但客户端还在等。

**第二层：幂等性**
所有写操作都要幂等。客户端每次操作生成幂等 Key，服务端用幂等 Key + 原子写保证至多处理一次。这是安全重试的前提。

**第三层：重试（配合幂等）**
只对幂等操作重试，使用指数退避 + 随机抖动。只重试可重试的错误（超时、5xx），不重试 4xx。设置最大重试次数（通常 3 次），超过后抛出错误。

**第四层：熔断器**
包装每个外部依赖。失败率超过阈值（如 50% 或连续 5 次失败）后熔断打开，快速失败 30 秒，再半开探测。这样下游故障时不会拖死上游。

**第五层：降级**
熔断打开时触发降级：返回缓存数据、默认值、或部分数据（"功能暂时不可用，稍后重试"）。降级策略要提前设计，不能等到故障时临时想。

**监控：**
每层都要有监控和告警：重试次数突增（可能下游有问题），熔断器 OPEN 频率（持续故障），降级触发频率（用户体验下降）。

---

## 关联文档

- [../03_communication/01_sync.md](../03_communication/01_sync.md) — API Gateway 层的限流和熔断
- [../02_storage/03_cache.md](../02_storage/03_cache.md) — 降级时的缓存使用
- [02_distributed_tx.md](02_distributed_tx.md) — 幂等性在 Saga 中的应用
- [../06_case_studies/02_rate_limiter.md](../06_case_studies/02_rate_limiter.md) — 分布式限流器的完整设计案例
