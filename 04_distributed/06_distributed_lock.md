# 分布式锁：Redis / ZooKeeper

## TL;DR

- **分布式锁**：保证分布式环境下某段代码同一时刻只被一个进程执行
- **Redis 分布式锁（Redlock）**：基于 SET NX EX，实现简单性能高——适合大多数场景
- **ZooKeeper 分布式锁**：基于临时有序节点，更强的一致性——适合对正确性要求极高的场景
- **关键要素**：唯一标识（防误删）+ 超时（防死锁）+ 原子操作（防竞争）

---

## 为什么需要分布式锁

单机环境下，用操作系统的互斥锁（Mutex）或线程同步原语保护临界区：

```typescript
// 单机
const mutex = new Mutex();
await mutex.lock();
try {
  // 临界区：只有一个线程执行
  await doSomething();
} finally {
  mutex.unlock();
}
```

分布式环境下，同一个服务部署了多个实例，进程级别的锁无法跨进程生效：

```
服务A 实例1 和 实例2 同时收到请求
同时进入临界区（扣减库存）
→ 超卖！
```

需要一个**所有实例都能访问的共享锁机制**。

---

## Redis 分布式锁

### 基础实现：SET NX EX

```typescript
// 加锁
const lockKey = 'lock:order:12345';
const lockValue = uuidv4();  // 唯一标识，用于释放时校验

const acquired = await redis.set(
  lockKey,
  lockValue,
  'NX',   // Not eXists：只在 Key 不存在时设置
  'EX',   // EXpire
  30      // 30 秒后自动过期（防止进程崩溃导致死锁）
);

if (!acquired) {
  throw new Error('获取锁失败，请稍后重试');
}

try {
  // 临界区
  await processOrder();
} finally {
  // 释放锁（必须验证是自己的锁，防止误删）
  await releaseLock(lockKey, lockValue);
}
```

**为什么要唯一的 lockValue？**

```
进程A 加锁，lockValue = "abc"
进程A 执行超时，锁自动过期被释放
进程B 加锁（同一个 key），lockValue = "xyz"
进程A 执行完，想释放锁：redis.del(lockKey)
→ 误删了进程B 的锁！
```

释放锁时必须验证 value 是否是自己的：

```typescript
// 释放锁：用 Lua 脚本保证"检查 + 删除"的原子性
async function releaseLock(key: string, value: string): Promise<void> {
  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;
  await redis.eval(script, 1, key, value);
}
```

**为什么用 Lua 脚本？**

```
// 非原子操作（有竞争条件）：
const val = await redis.get(lockKey);        // 步骤1
if (val === lockValue) {
  await redis.del(lockKey);                   // 步骤2
}
// 步骤1 和 步骤2 之间，锁可能已经过期被别人拿走了
// 步骤2 删除的是别人的锁！
```

Lua 脚本在 Redis 中是原子执行的，不会被打断。

### 锁超时的设置

超时时间设置是个权衡：
- **太短**：业务还没执行完锁就过期，其他进程拿到锁，进入临界区，两个进程并发执行
- **太长**：进程崩溃后，锁要很久才会释放，其他进程长时间等待

**看门狗（Watchdog）机制：**

```typescript
// 在执行期间定期续期
async function acquireLockWithWatchdog(key: string, value: string): Promise<void> {
  await redis.set(key, value, 'NX', 'EX', 30);

  // 每 10 秒续期一次（锁超时 30 秒的 1/3 时开始续期）
  const watchdog = setInterval(async () => {
    const current = await redis.get(key);
    if (current === value) {
      await redis.expire(key, 30);  // 续期
    } else {
      clearInterval(watchdog);  // 锁已被释放，停止续期
    }
  }, 10000);

  // 执行完后清理
  return () => clearInterval(watchdog);
}
```

### Redlock（多节点 Redis 分布式锁）

单节点 Redis 的问题：Redis 宕机后，所有锁信息丢失，其他进程可以拿到"失效的锁"。

**Redlock 算法（官方推荐，5 个独立 Redis 节点）：**

```
要求：N 个完全独立的 Redis 实例（不是主从，不是集群，是完全独立的）

加锁：
1. 获取当前时间 t1
2. 向所有 N（如 5）个 Redis 实例发送 SET NX EX 请求
3. 如果在超时时间内（通常 < 10ms）收到 N/2+1（如 3）个成功响应
   且 t2 - t1 < 锁超时时间
   → 加锁成功
4. 否则，向所有实例发送解锁请求，加锁失败

解锁：
向所有 N 个 Redis 实例发送解锁请求（Lua 脚本删除对应 value 的 key）
```

Redlock 的争议：Martin Kleppmann（DDIA 作者）认为 Redlock 在某些极端情况下不安全（时钟漂移 + GC 停顿），Antirez（Redis 作者）对此有反驳。实际工程中对安全性要求极高的场景（如分布式事务协调），建议用 ZooKeeper 而不是 Redlock。

---

## ZooKeeper 分布式锁

### ZooKeeper 的基础知识

ZooKeeper 是一个分布式协调服务，基于 ZAB 协议（类 Paxos），提供强一致性。

**关键特性：**
- **临时节点（Ephemeral Node）**：客户端断开连接后自动删除，天然防死锁
- **有序节点（Sequential Node）**：`/lock/node-0000000001`，后缀自动递增
- **Watcher**：监听节点变化，节点创建/删除时通知

### ZooKeeper 锁的实现

```
加锁流程：
1. 在 /locks/order_12345/ 下创建临时有序节点
   例如：/locks/order_12345/node-0000000003

2. 获取 /locks/order_12345/ 下所有子节点，排序

3. 如果我的节点是最小的 → 加锁成功

4. 如果我的节点不是最小的：
   监听（Watch）比我小一位的节点
   等待那个节点删除

5. 那个节点删除时（持有锁的进程完成或崩溃）
   → Watcher 触发 → 重新检查我是否最小 → 加锁成功
```

**示例（4 个竞争者）：**

```
节点列表（按序号排）：
  /locks/order/node-0001  ← 进程A（序号最小，加锁成功）
  /locks/order/node-0002  ← 进程B（Watch node-0001）
  /locks/order/node-0003  ← 进程C（Watch node-0002）
  /locks/order/node-0004  ← 进程D（Watch node-0003）

进程A 执行完毕，删除 node-0001
→ 进程B 的 Watcher 触发
→ 进程B 发现自己是最小节点
→ 进程B 加锁成功
```

**为什么监听前一个节点，而不是监听根节点？**

如果所有进程都监听根节点，一个节点删除时，所有进程同时被唤醒（**羊群效应 Herd Effect**），大量 ZooKeeper 请求同时涌入。监听前一个节点，每次只有一个进程被唤醒，更高效。

### ZooKeeper 锁的优点

- **基于强一致的共识（ZAB 协议）**，比 Redis 更可靠
- **临时节点自动清理**：进程崩溃后，ZooKeeper Session 过期，临时节点自动删除，锁自动释放
- **公平锁**：按序号排队，保证 FIFO

### ZooKeeper 锁的缺点

- **性能不如 Redis**：每次操作都要经过 ZAB 协议，延迟通常在 10-100ms
- **运维复杂**：ZooKeeper 集群本身需要维护
- **会话超时问题**：GC 停顿或网络抖动可能导致 Session 超时，临时节点被意外删除

---

## Redis vs ZooKeeper 分布式锁对比

| 维度 | Redis | ZooKeeper |
|------|-------|-----------|
| 性能 | 极高（< 1ms） | 中（10-100ms） |
| 一致性 | 依赖配置（Redlock 才强） | 强一致（ZAB 协议） |
| 可靠性 | 一般（单节点有 SPOF） | 高（多节点强一致） |
| 死锁防护 | TTL 到期自动释放 | 临时节点 + Session 过期 |
| 公平性 | 否（非公平锁） | 是（有序节点保证 FIFO） |
| 适用场景 | 高并发，对偶发误差可容忍 | 对正确性要求极高 |

**选型建议：**
- 秒杀限流、短期互斥 → Redis
- 分布式任务调度、Leader 选举、高安全业务 → ZooKeeper 或 etcd

---

## 实践中的注意点

### 1. 锁粒度要细

```
// 粒度太粗（整个服务锁）：
await redis.set('lock:order_service', ...)  // 只有一个实例能处理任何订单
// 吞吐量极低

// 粒度合适（锁单个资源）：
await redis.set(`lock:order:${orderId}`, ...)  // 不同订单可以并发处理
```

### 2. 非阻塞 vs 阻塞加锁

```typescript
// 非阻塞（拿不到直接失败）：
const lock = await tryAcquireLock(key, 30);
if (!lock) {
  return { error: 'Resource is busy, please retry' };
}

// 阻塞（等待，但要有超时）：
const lock = await acquireLockWithTimeout(key, 30, waitTimeout: 5000);
if (!lock) {
  return { error: 'Timeout waiting for lock' };
}
```

### 3. 最长持有时间与看门狗

关键业务中，加锁时间要有上限，同时要配合看门狗续期，避免"执行到一半锁过期"的情况。

---

## 与 Node.js/TS 生态的类比

```typescript
// 使用 redlock npm 包（官方推荐的 Redlock 实现）
import Redlock from 'redlock';
import { createClient } from 'redis';

const client = createClient();
const redlock = new Redlock([client]);

async function processOrderSafely(orderId: string): Promise<void> {
  // 尝试获取锁（最多重试 3 次，每次等 200ms）
  const lock = await redlock.acquire([`locks:order:${orderId}`], 30000);

  try {
    await processOrder(orderId);
  } finally {
    await lock.release();
  }
}
```

---

## 常见陷阱

1. **锁超时但业务还在执行**：没有看门狗，锁过期后其他进程进入临界区，两个进程并发执行
2. **锁释放时没有校验 value**：直接 `DEL lockKey`，可能删除别人的锁
3. **Redis 主从不一致导致锁失效**：写主库成功，主库宕机，从库提升为主库（旧锁信息未同步），新进程能再次加锁
4. **ZooKeeper Session 超时误删节点**：GC 停顿 > Session 超时时间，持有锁的进程 Session 超时，临时节点被删，其他进程误以为锁已释放
5. **分布式锁防不住所有并发**：锁只是排他访问的机制，如果业务逻辑本身不是事务性的，仍然可能有数据不一致

---

## 面试常见问答

### 简单

**Q：为什么 Redis 的 SET NX EX 要用一个原子命令而不是分开两步（SETNX + EXPIRE）？**

A：`SETNX key value` 后还没来得及 `EXPIRE key 30`，进程崩溃了，那这个 key 会永久存在，变成死锁（任何进程都无法获取锁）。原子 `SET key value NX EX 30` 保证设置值和过期时间是原子操作，不存在执行到一半崩溃的情况。这个 bug 在早期 Redis 版本很常见，直到 Redis 2.6.12 才引入原子参数组合。

---

**Q：ZooKeeper 分布式锁为什么要监听前一个节点而不是根节点？**

A：避免羊群效应（Herd Effect）。如果所有等待者都监听根节点，每当一个节点删除，所有等待的进程同时被唤醒，都向 ZooKeeper 查询最新节点列表，产生大量并发请求。实际上每次只有一个进程能获得锁，其他 N-1 个进程的唤醒都是无效的。用有序节点 + 监听前一个节点，每次只有一个进程被唤醒，ZooKeeper 压力从 O(N²) 降到 O(N)。

---

### 中等

**Q：Redis 分布式锁在主从复制场景下可能有什么安全问题？**

A：Redis 主从复制是异步的，存在时间窗口：进程A 在主库设置锁，主库还没把数据同步给从库，主库宕机，从库升级为主库（新主库上没有这个锁）。进程B 在新主库上设置锁成功，此时进程A 和进程B 同时持有锁，互斥性被破坏。Redlock 算法通过要求在多个独立 Redis 实例上同时加锁（多数派）来解决这个问题，但代价是性能下降和实现复杂性增加。

---

### 难

**Q：一个持有 Redis 分布式锁的进程，在执行业务逻辑期间发生了 GC 停顿，锁超时过期了，另一个进程获取了锁，然后第一个进程 GC 结束继续执行，这时候怎么办？**

A：这是分布式锁最难的问题，称为"锁的安全性"（Lock Safety）。

**问题分析：**
进程A 加锁 → 业务执行中 → GC 停顿（例如 Stop-the-World 30 秒）→ 锁超时删除 → 进程B 加锁 → 进程B 执行 → 进程A GC 结束继续执行 → 两者并发

**缓解方案1：看门狗**
进程A 定期续期，GC 期间无法续期，锁过期。GC 恢复后进程A 发现锁已不属于自己，中止操作（通过检查 lockValue 是否还存在）。

**缓解方案2：Fencing Token（围栏令牌）**
获取锁时同时获取一个单调递增的 Token（如 ZooKeeper 的序列号）。对所有受保护资源的写操作都携带这个 Token，资源服务拒绝 Token 值小于当前已见到的最大 Token 的写请求：
```
进程A 获取锁，Token=100
进程A GC，锁过期
进程B 获取锁，Token=101
进程B 写资源，携带 Token=101，资源接受
进程A GC 结束，写资源，携带 Token=100，资源拒绝（100 < 101）
```
这是最可靠的方案，但需要资源服务端支持 Token 校验。

**结论：** Redis 分布式锁无法在所有情况下提供完美的互斥性，对正确性要求极高的场景（如金融操作），应配合业务层幂等性（唯一约束、乐观锁）双重保护，或者改用基于共识的系统（ZooKeeper/etcd）。

---

## 关联文档

- [03_consensus.md](03_consensus.md) — ZooKeeper 底层 ZAB 协议（类 Paxos）
- [../02_storage/03_cache.md](../02_storage/03_cache.md) — Redis 基础特性
- [04_fault_tolerance.md](04_fault_tolerance.md) — 幂等性与锁的配合使用
- [../06_case_studies/02_rate_limiter.md](../06_case_studies/02_rate_limiter.md) — Redis 在限流中的应用
