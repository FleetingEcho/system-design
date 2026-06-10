# 系统设计案例：分布式缓存系统

## TL;DR

设计一个类似 Redis Cluster 或 Memcached 的分布式缓存系统。核心难点：**扩缩容时如何最小化数据迁移**（一致性哈希）+ **热点数据处理** + **缓存淘汰策略**。

---

## 需求澄清

**功能需求：**
- 支持 GET / PUT / DELETE 操作（KV 存储）
- 支持 TTL（Key 自动过期）
- 高吞吐读写（每秒百万级操作）

**非功能需求：**
- 高可用（节点故障时缓存继续可用）
- 水平扩展（增加节点提升容量和吞吐）
- 最终一致（允许短暂不一致，优先可用）
- 均匀分布（数据尽量均匀分布在所有节点）

**规模估算：**
```
总缓存大小：1 TB
每个节点内存：32 GB → 需要 32 个节点（加上副本因子 2 = 64 台）
读 QPS：100 万/秒
写 QPS：10 万/秒
平均 Value 大小：1 KB
```

---

## 核心设计：一致性哈希（Consistent Hashing）

### 朴素哈希的问题

```
3 个节点：node0, node1, node2
分配规则：hash(key) % 3

节点 node1 宕机 → 变成 hash(key) % 2
→ 几乎所有 Key 的映射都变了！
→ 缓存全部失效，所有请求打到数据库 → 雪崩！
```

### 一致性哈希

```
原理：
  把哈希空间想象成一个 0 到 2^32-1 的环（Hash Ring）
  节点和 Key 都映射到环上的某个位置

  节点映射：hash("node0") = 位置 100
            hash("node1") = 位置 300
            hash("node2") = 位置 600

  Key 映射：hash("user:123") = 位置 250
  → 顺时针找到第一个节点 = node1 → key 存在 node1

增加节点（node3 插入位置 450）：
  原来 300-600 之间的 key 属于 node2
  现在 300-450 的 key 属于 node3，450-600 的 key 还是 node2
  → 只有 300-450 之间的 key 需要迁移，其他不变！

删除节点（node1 宕机）：
  原来 100-300 之间的 key 属于 node1
  node1 宕机后，这些 key 顺时针找到 node2
  → 只有 node1 的 key 需要迁移
```

### 虚拟节点（Virtual Nodes）

解决节点数量少时的数据不均匀问题：

```
每个物理节点映射成 100 个虚拟节点（vnodes）：
  node0 → node0_1, node0_2, ..., node0_100（均匀分布在环上）
  node1 → node1_1, node1_2, ..., node1_100

效果：
  300 个虚拟节点均匀分布在哈希环上，每个 Key 随机落到一个 vnode
  → 数据分布更均匀
  → 增减节点时迁移更均匀（不是偏向某一侧）
```

---

## 实现：一致性哈希环

```typescript
import crypto from 'crypto';

class ConsistentHashRing {
  private ring: Map<number, string> = new Map(); // position → nodeId
  private sortedPositions: number[] = [];
  private virtualNodeCount = 100;

  addNode(nodeId: string): void {
    for (let i = 0; i < this.virtualNodeCount; i++) {
      const virtualNodeKey = `${nodeId}:vn${i}`;
      const position = this.hash(virtualNodeKey);
      this.ring.set(position, nodeId);
      this.sortedPositions.push(position);
    }
    this.sortedPositions.sort((a, b) => a - b);
  }

  removeNode(nodeId: string): void {
    for (let i = 0; i < this.virtualNodeCount; i++) {
      const virtualNodeKey = `${nodeId}:vn${i}`;
      const position = this.hash(virtualNodeKey);
      this.ring.delete(position);
    }
    this.sortedPositions = this.sortedPositions.filter(p => this.ring.has(p));
  }

  getNode(key: string): string {
    const position = this.hash(key);
    // 找到第一个 >= position 的节点（顺时针）
    const idx = this.binarySearch(position);
    const nodePosition = this.sortedPositions[idx % this.sortedPositions.length];
    return this.ring.get(nodePosition)!;
  }

  // 副本：获取顺时针方向的 N 个不同物理节点
  getNodes(key: string, replicas: number): string[] {
    const nodes: string[] = [];
    const seen = new Set<string>();
    let idx = this.binarySearch(this.hash(key));

    while (nodes.length < replicas && nodes.length < this.getTotalPhysicalNodes()) {
      const pos = this.sortedPositions[idx % this.sortedPositions.length];
      const nodeId = this.ring.get(pos)!;
      if (!seen.has(nodeId)) {
        nodes.push(nodeId);
        seen.add(nodeId);
      }
      idx++;
    }

    return nodes;
  }

  private hash(key: string): number {
    const hash = crypto.createHash('md5').update(key).digest('hex');
    return parseInt(hash.substring(0, 8), 16); // 取前 4 字节作为 32 位整数
  }

  private binarySearch(position: number): number {
    let lo = 0, hi = this.sortedPositions.length - 1;
    while (lo <= hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (this.sortedPositions[mid] < position) lo = mid + 1;
      else hi = mid - 1;
    }
    return lo;
  }

  private getTotalPhysicalNodes(): number {
    return new Set(this.ring.values()).size;
  }
}
```

---

## 高可用：副本策略

```
副本因子 = 2（每份数据存在 2 个节点上）

写入策略（Write Consistency）：
  W=1：写 1 个节点就返回成功（最低延迟，可能数据还没复制就挂了）
  W=2：写 2 个节点才返回成功（更强一致，延迟更高）
  W=all：写所有副本才返回成功（最强，最慢）

读取策略（Read Consistency）：
  R=1：读 1 个节点就返回（可能读到旧数据）
  R=2：读 2 个节点取最新值（可能更慢）

Quorum（法定人数）：
  W + R > 副本数 N → 保证至少读到一个最新副本
  N=3, W=2, R=2 → 2+2>3 ✓ → 强一致（适合对一致性要求高的场景）
  N=3, W=1, R=1 → 不保证一致（高可用，低延迟）

缓存系统通常选 R=1, W=1（或 W=2）：
  读取：读一个节点，快，允许短暂不一致
  写入：同步写主节点，异步复制到副本
```

---

## 缓存淘汰策略

内存满了时，选哪些 Key 删掉：

```
LRU（Least Recently Used）：
  删除最长时间没被访问的 Key
  实现：HashMap + 双向链表
  每次访问：把该节点移到链表头部
  淘汰：删除链表尾部节点
  适合：热点数据有明显的"近期访问"特征

LFU（Least Frequently Used）：
  删除访问频率最低的 Key
  实现：MinHeap + HashMap（Key → 频次）
  适合：某些 Key 长期热点（如商品详情页），LRU 可能错误地淘汰
  缺点：新加入的 Key 频次很低，容易被淘汰（缓存污染）

Redis 的实现（近似 LRU）：
  不维护精确链表（内存开销大），而是随机抽样 5 个 Key，
  淘汰其中访问时间最老的，多次抽样后效果接近精确 LRU
  时间复杂度 O(1)，内存开销低

FIFO：先进先出，实现最简单，效果一般
```

---

## TTL（Key 过期）的实现

```
两种策略：

主动过期（Active Expiration）：
  后台定时任务定期扫描，删除已过期的 Key
  问题：如果 Key 太多，扫描代价大

惰性过期（Lazy Expiration）：
  访问某个 Key 时，先检查是否已过期
  如果过期，立即删除并返回"不存在"
  问题：过期但未访问的 Key 占内存

Redis 的做法：两者结合
  每 100ms 随机抽 20 个设置了 TTL 的 Key，删除其中过期的
  同时任何访问都做惰性过期检查
  → 内存不会积累太多过期 Key，扫描代价也可控
```

---

## 处理热 Key（Hot Key）

某个 Key 访问量极高（如全国首页展示的同一张图的 URL），打满单个缓存节点：

```
方案一：本地缓存（Local Cache）
  应用服务器内存里也缓存一份
  流程：先查本地缓存 → 未命中 → 查 Redis → 写本地缓存
  TTL 很短（1-5 秒），允许短暂不一致
  效果：每台应用服务器各缓存一份，Redis 热 Key 的 QPS 被摊分

方案二：Key 拆分（Sharding the Hot Key）
  把热 Key 复制成多份：
    hot_key → hot_key:shard0, hot_key:shard1, ..., hot_key:shard9
    写：同时写 10 份
    读：随机读一份（hot_key:shard${Math.random()*10 | 0}）
  效果：单 Key 的 QPS 被 10 个节点承载

方案三：动态 Key 发现
  监控 Redis 的访问频率（Redis 4.0+ 的 hotkeys 命令）
  发现热 Key 后自动触发 Key 拆分或本地缓存
```

---

## 节点宕机的处理

```
检测：心跳检测（每 1 秒 Ping 一次，连续 3 次失败 → 标记节点下线）

读取时节点宕机：
  主节点宕机 → 路由到副本节点（Fallback）
  副本也宕机 → 回源数据库（Cache Miss，性能下降但数据不丢失）

写入时节点宕机：
  主节点宕机 → 写入失败（数据不丢失，让业务层处理重试）
  或者：写到副本节点（副本临时升主），原主节点恢复后同步数据

一致性哈希的自动重路由：
  从哈希环上摘除宕机节点
  该节点的 Key 自动路由到下一个节点（副本）
```

---

## 节点扩容的数据迁移

```
添加新节点 node_new：
  一致性哈希确定 node_new 负责的范围（比如原来 node2 的一部分）
  
迁移流程：
  1. 先给 node_new 的范围加上"迁移标记"
  2. 读取请求：先查 node_new → 未命中 → 查 node2（原节点）→ 写回 node_new
  3. 后台异步把 node2 中属于 node_new 的 Key 逐步复制过去
  4. 迁移完成后，移除迁移标记

注意：迁移期间不需要停止服务，用"双读"实现无缝迁移
```

---

## Node.js 类比

如果你用过 Redis，这就是 Redis Cluster 的内部原理：

```typescript
// Redis Cluster 对外表现和单机 Redis 一样
const redis = new Redis.Cluster([
  { host: '127.0.0.1', port: 7000 },
  { host: '127.0.0.1', port: 7001 },
  { host: '127.0.0.1', port: 7002 },
]);

// SET / GET 和单机版一样，Cluster 内部处理路由
await redis.set('user:123', JSON.stringify(user));
const cached = await redis.get('user:123');

// 内部原理：
// 1. hash_slot = crc16('user:123') % 16384
// 2. 找到负责该 hash_slot 的节点
// 3. 连接该节点执行命令
```

---

## 常见陷阱

1. **缓存雪崩时的一致性哈希保护**：加新节点时，如果迁移速度太慢，会有一段时间大量 Cache Miss（Key 在旧节点，但请求路由到新节点）。需要"双读"策略过渡，或者限制单次迁移量

2. **虚拟节点数量的选择**：vnode 数量太少（如 10 个），数据分布仍然不均匀；太多（如 1000 个），哈希环管理（排序、二分查找）的内存和时间开销增大。通常 100-200 个 vnode 是好的折中

3. **跨节点事务**：分布式缓存不支持跨 Key 的原子操作（Redis Cluster 要求同一事务的 Key 在同一 slot）。需要时，用 Hash Tag 强制把相关 Key 路由到同一节点：`{user:123}:profile` 和 `{user:123}:settings` 的 `{user:123}` 部分相同，Hash Slot 相同，路由到同一节点

---

## 面试 Q&A

### 简单

**Q: 一致性哈希解决了什么问题？**

A: 解决了普通取模哈希（`hash(key) % N`）在节点增减时导致几乎全部 Key 重新映射的问题。一致性哈希将节点和 Key 都映射到哈希环上，增减节点时只有相邻范围内的 Key 需要迁移，其他 Key 不受影响。加上虚拟节点，数据分布更均匀。

**Q: LRU 和 LFU 有什么区别，各适合什么场景？**

A: LRU 淘汰最近最少使用的 Key，适合"最近用过的将来还会用"的场景（绝大多数缓存场景）。LFU 淘汰使用频率最低的 Key，适合有长期热点数据的场景（如网站首页的资源，一直被频繁访问不应被淘汰）。LFU 的缺点是新插入的 Key 频次为 0，容易被过早淘汰（可以设置一个初始频次来缓解）。

---

### 中等

**Q: 如何处理缓存节点宕机？**

A: 分两种情况：1）读宕机：立即路由到副本节点（副本因子 ≥ 2 时），如果副本也宕机，Cache Miss，回源数据库，性能下降但数据不丢失；2）写宕机：写入失败，由业务层重试（先写成功再返回），或允许 "eventual consistency"（写到可用的副本，待主节点恢复后同步）。同时更新一致性哈希环，把宕机节点标记为下线，减少路由到宕机节点的概率。

**Q: 如何检测和处理热 Key？**

A: 检测：Redis 4.0+ 的 `redis-cli --hotkeys` 命令可以找出热 Key；或者在客户端统计每个 Key 的访问频次，超过阈值告警。处理：本地缓存（应用层内存缓存，TTL 极短）吸收大部分请求；或者 Key 拆分（`hot_key:shard${random}`），把单 Key 压力分散到多个节点。

---

### 困难

**Q: 如果需要支持 100 万 QPS 的读和 10 万 QPS 的写，如何规划集群大小和架构？**

A: 逐步推导：

**节点数量：** Redis 单核 10 万 QPS（读写合计），100 万 QPS 读需要至少 10 个节点。加上写 10 万 QPS，估算 15 个主节点（读操作放主节点，副本承担只读请求分担一部分）。副本因子 2 → 30 台服务器总计。

**内存规划：** 1 TB 缓存 / 32 GB 每节点 = 32 个主节点（与上面 QPS 要求的 15 个取大值）。最终：32 个主节点 × 2 副本 = 96 台服务器。

**热 Key 处理：** 预期会有一些超热 Key（如首页数据），在客户端加本地缓存（Caffeine，1000 个 Key，TTL 2 秒），大幅降低对 Redis 的压力。

**一致性哈希：** 每个主节点 200 个虚拟节点，总计 6400 个虚拟节点分布在哈希环上。扩容时单次最多扩 2-3 个节点（避免同时迁移太多数据），迁移过程用双读策略，无缝完成。

**监控：** 关键指标：每个节点的 QPS、内存使用率、缓存命中率、慢查询。节点内存 > 80% 或命中率 < 90% 时告警，触发扩容评估。

---

## 关联文档

- [../02_storage/03_cache.md](../02_storage/03_cache.md) — 缓存策略（穿透/击穿/雪崩）
- [../02_storage/02_nosql.md](../02_storage/02_nosql.md) — Redis 数据结构
- [../04_distributed/01_consistency_models.md](../04_distributed/01_consistency_models.md) — Quorum 一致性
