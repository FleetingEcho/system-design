# 分布式 ID 生成：UUID / Snowflake / 号段

## TL;DR

- **UUID**：全局唯一，无需协调，但随机性导致 B+ Tree 索引碎片化，太长（128位）
- **数据库自增 ID**：简单，天然有序，但分布式场景下是单点，无法水平扩展
- **Snowflake**：Twitter 开源，64位，趋势递增，高性能——**分布式系统的首选**
- **号段模式**：批量预申请 ID，减少数据库压力——适合需要连续数字 ID 的场景

---

## 为什么分布式 ID 难

**需求：**
1. **全局唯一**：不同机器生成的 ID 绝不重复
2. **趋势递增**：ID 越新越大，利于 B+ Tree 索引顺序插入（减少页分裂）
3. **高性能**：每秒能生成百万级 ID，不成为瓶颈
4. **无需协调**：不依赖中心节点（中心节点是单点）

**矛盾：** 无需协调（去中心化）和全局唯一（需要某种协调）天然冲突。不同方案的权衡不同。

---

## UUID（Universally Unique Identifier）

### 结构（UUID v4）

```
550e8400-e29b-41d4-a716-446655440000
 ↑         ↑    ↑                ↑
时间戳   随机  版本          随机
128 位，36 个字符（含 4 个横线）
```

UUID v4 基本上是随机的（122 位随机数），重复概率极低（10^36 分之一）。

### 优点

- **完全去中心化**：任何机器任何时间都能生成，不依赖外部服务
- **实现极简**：`uuid()` 一行代码
- **全球唯一**：不只是系统内唯一，而是全球范围唯一（适合分布式跨系统场景）

### 缺点

**索引性能差（核心问题）：**

```
数据库表用 UUID 作为主键：
  插入 ID: 550e8400...  → B+ Tree 位置：中间某处
  插入 ID: a97f11bb...  → B+ Tree 位置：末尾
  插入 ID: 3c9a8b12...  → B+ Tree 位置：开头附近

每次插入都在 B+ Tree 的随机位置，导致：
  1. 大量页分裂（Page Split）：新数据插入已满的页，要分裂为两个页
  2. 索引碎片化：数据不是顺序存储，磁盘随机 I/O 增多
  3. 缓冲池命中率低：随机位置导致每次都要加载新的页
```

**太长：**
36 个字符 vs 数字 ID 的 8 字节，存储成本高，联合索引时占用更多空间。

**不可读：**
`550e8400-e29b-41d4-a716-446655440000` vs `10086`，日志追踪不友好。

**UUID v7（2022 年标准化）** 解决了顺序问题，前 48 位是时间戳，趋势递增，同时保留随机后缀保证唯一性。

---

## 数据库自增 ID

### 工作方式

```sql
CREATE TABLE users (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100)
);
-- 每次 INSERT 后，id 自动从上次的最大值 +1
```

### 优点

- 简单，天然有序
- 人类可读
- B+ Tree 顺序插入，性能最好

### 缺点

**分布式场景的单点问题：**

```
分片数据库：
  分片1：id 1, 3, 5... （奇数）
  分片2：id 2, 4, 6... （偶数）
  靠人为配置步长来避免重复，维护麻烦

水平扩展：
  如果分片数量变化，步长配置要重新规划
```

**暴露业务信息：**
用户 ID=10086 说明系统里只有 1 万多个用户，或者注册顺序可被猜测（竞争对手知道你的增速）。

---

## Snowflake（雪花算法）

### 结构（64 位 Long）

```
| 1位 | 41位        | 10位      | 12位       |
|-----|------------|----------|------------|
| 0   | 时间戳(ms)  | 机器ID   | 序列号     |
```

- **1位**：符号位，永远是 0（保证正数）
- **41位时间戳**：毫秒级，从某个自定义起始时间算起，可以用约 69 年
- **10位机器 ID**：支持 1024 台机器（可以拆分为 5 位数据中心 + 5 位机器）
- **12位序列号**：同一毫秒内同一台机器最多生成 4096 个 ID

**每秒可生成 ID 数：** 1000ms × 4096 = **4,096,000 个/秒/机器**

### ID 生成逻辑

```typescript
class SnowflakeIdGenerator {
  private readonly EPOCH = 1609459200000n;      // 2021-01-01 00:00:00 UTC
  private readonly MACHINE_ID_BITS = 10n;
  private readonly SEQUENCE_BITS = 12n;
  private readonly MAX_SEQUENCE = (1n << this.SEQUENCE_BITS) - 1n;  // 4095

  private sequence = 0n;
  private lastTimestamp = -1n;

  constructor(private readonly machineId: bigint) {}

  nextId(): bigint {
    let timestamp = BigInt(Date.now()) - this.EPOCH;

    if (timestamp === this.lastTimestamp) {
      this.sequence = (this.sequence + 1n) & this.MAX_SEQUENCE;
      if (this.sequence === 0n) {
        // 同一毫秒内序列号用完，等到下一毫秒
        while (timestamp <= this.lastTimestamp) {
          timestamp = BigInt(Date.now()) - this.EPOCH;
        }
      }
    } else {
      this.sequence = 0n;
    }

    this.lastTimestamp = timestamp;

    return (timestamp << (this.MACHINE_ID_BITS + this.SEQUENCE_BITS))
      | (this.machineId << this.SEQUENCE_BITS)
      | this.sequence;
  }
}
```

### Snowflake 的优点

- **趋势递增**：时间戳在高位，新 ID 总是大于旧 ID（同一毫秒内按序列号排序）
- **高性能**：纯本地计算，无网络调用，每秒数百万 ID
- **64 位 Long**：比 UUID 小，存储高效
- **包含时间信息**：可以从 ID 反推生成时间

### Snowflake 的问题

**时钟回拨（Clock Drift）：**

```
服务器时钟被 NTP 校正，往回调了 1 秒
此时生成的 ID 时间戳 < 之前生成的 ID 时间戳
→ 新 ID < 旧 ID，破坏了趋势递增
→ 更严重：可能和之前的 ID 重复！
```

**解决方案：**
- 检测到时钟回拨时，拒绝生成 ID，等待时钟追上来（影响可用性）
- 使用额外的序列号补偿（复杂）
- Seata 的 TiD、百度 UidGenerator 对此做了增强

**机器 ID 分配：**
10 位机器 ID 需要集中管理，通常用 ZooKeeper 或 etcd 分配，机器 ID 一旦分配就固定。

---

## 号段模式（Segment Mode）

### 工作方式

```
数据库中维护一个号段分配表：
  id_table: { biz_tag, max_id, step, update_time }

服务启动时从数据库预申请一段 ID（号段）：
  UPDATE id_table SET max_id = max_id + step WHERE biz_tag = 'order'
  → 拿到 [1000, 2000) 号段（step=1000）

本地按顺序分配这 1000 个 ID，用完后再申请下一段
```

### 双缓冲优化

```
当前号段用到 50% 时，异步申请下一个号段（Buffer）
用完当前号段时，直接切换到 Buffer（无需等待 DB）

好处：用 DB 的速率 = 1次 DB 请求 / 1000个 ID
     号段用完等 DB 申请的等待消除了
```

### 号段模式的优缺点

**优点：**
- ID 趋势递增（连续的整数）
- 数据库压力极低（批量申请）
- 实现简单

**缺点：**
- 服务重启时，未用完的号段浪费（ID 不连续）
- 依赖数据库（但只是低频访问，容错性好）
- 不适合需要精确连续 ID 的场景（如发票号码，要求绝对连续，需要更复杂机制）

---

## 各方案对比

| 方案 | 有序性 | 性能 | 依赖 | 适用场景 |
|------|--------|------|------|---------|
| UUID v4 | 无序（随机） | 高（本地） | 无 | 跨系统唯一标识，对 DB 性能不敏感 |
| UUID v7 | 趋势递增 | 高（本地） | 无 | 需要全球唯一且趋势递增 |
| 数据库自增 | 严格递增 | 中（DB 写） | DB | 单库场景 |
| Snowflake | 趋势递增 | 极高（本地） | 机器 ID 分配 | 分布式系统高性能 ID |
| 号段模式 | 趋势递增 | 高（批量）| DB（低频） | 需要整数 ID，可以接受不连续 |

---

## 实际工程选择

```
大多数分布式业务系统：Snowflake 变体（趋势递增 + 高性能）
  例如：订单 ID、用户 ID、消息 ID

需要全球唯一（跨系统）：UUID v7（新项目）或 UUID v4（兼容性好）
  例如：API Key、Token、文件 Key

需要连续整数：号段模式
  例如：序列号、流水号

传统小规模单库：数据库自增
```

---

## 与 Node.js/TS 生态的类比

```typescript
// UUID（最常用）
import { v4 as uuidv4, v7 as uuidv7 } from 'uuid';
const id = uuidv4();  // 随机
const id7 = uuidv7(); // 趋势递增（UUID v7）

// 如果用 PostgreSQL，可以用数据库原生 UUID：
// CREATE TABLE users (id UUID PRIMARY KEY DEFAULT gen_random_uuid())

// Snowflake 实现（多个 npm 包，如 @dutu/snowflake-id）
import SnowflakeId from '@dutu/snowflake-id';
const snowflake = new SnowflakeId({ mid: 1, offset: 0 });
const id = snowflake.generate();  // 返回字符串，因为 JS BigInt 超过 Number.MAX_SAFE_INTEGER
```

---

## 常见陷阱

1. **JavaScript 中 Snowflake ID 溢出**：64 位整数超过 JS 的 Number.MAX_SAFE_INTEGER（2^53），必须用 BigInt 或字符串，前端 JSON 序列化时要特别处理
2. **时钟回拨导致 ID 重复**：生产环境必须处理时钟回拨，简单拒绝服务、等待时钟追上是常见做法
3. **UUID 做主键的 B+ Tree 性能问题**：大表用 UUID 作主键，插入性能下降，考虑用 UUID v7 或 Snowflake
4. **机器 ID 重复**：两台机器分配了相同的机器 ID，生成的 Snowflake ID 可能重复，机器 ID 分配需要中心化管理
5. **号段步长太小**：步长太小（如 10），每 10 个 ID 就要访问一次数据库，失去了号段模式的意义

---

## 面试常见问答

### 简单

**Q：为什么 UUID 作为数据库主键性能不好？**

A：UUID 是随机生成的，插入数据库时 B+ Tree 要在随机位置插入新节点，而不是顺序追加到末尾。这导致：频繁的页分裂（已满的叶子节点需要分裂）、索引碎片化（数据不连续存储，磁盘随机 I/O 增多）、缓冲池命中率低（每次访问的页都不同）。对比自增 ID：新记录总是插入 B+ Tree 的最右边，顺序追加，无页分裂，性能最好。UUID v7 通过在高位加时间戳解决了这个问题。

---

**Q：Snowflake 算法生成的 ID 由哪几部分组成？**

A：64 位 ID 分三部分：最高 1 位永远为 0（保证正数）；其次 41 位是毫秒级时间戳（从自定义起始时间算起，可用约 69 年）；再次 10 位是机器 ID（支持 1024 台机器）；最后 12 位是序列号（同一毫秒内同一机器最多生成 4096 个 ID）。时间戳在高位保证了 ID 的趋势递增性，机器 ID 和序列号保证了同一毫秒内的唯一性。

---

### 中等

**Q：Snowflake 的时钟回拨问题如何解决？**

A：时钟回拨是指服务器时钟被 NTP 同步往回调整，导致当前时间戳小于上一个 ID 的时间戳，如果继续生成 ID 可能重复。常见解决方案：
1. **拒绝服务**：检测到回拨时抛出异常，等待时钟追上（简单但影响可用性，适合回拨时间短的情况）
2. **缓冲等待**：如果回拨时间极短（< 10ms），原地自旋等待时钟追上
3. **扩展序列号位**：发生回拨时，使用一个额外的 bit 标记"回拨态"，在不影响唯一性的前提下继续生成
4. **使用单调时钟**：不用系统时钟，用单调递增的逻辑时钟（但需要持久化最后的时间戳，重启时恢复）

美团的 Leaf 和百度的 UidGenerator 都对这个问题做了工程化处理。

---

### 难

**Q：设计一个分布式 ID 生成服务，要求高可用、无单点、每秒 100 万 ID，ID 趋势递增。**

A：基于 Snowflake 的改进方案：

**机器 ID 分配（ZooKeeper/etcd）：**
服务启动时在 etcd 创建临时节点，获取唯一机器 ID（0-1023）。服务下线时临时节点删除，ID 被回收供其他实例使用。机器 ID 是会话绑定的，不持久化。

**高可用部署：**
多台机器各自独立生成 ID（机器 ID 不同），前面挂 Load Balancer。不需要主从复制，完全无状态（除了机器 ID）。任何一台宕机，其他机器继续工作，可用性 > 99.99%。

**时钟回拨处理：**
本地记录上一次生成 ID 的时间戳（持久化到文件或 etcd），启动时恢复。如果检测到时钟回拨 < 10ms，等待时钟追上；如果 > 10ms，拒绝启动，告警运维检查时钟同步问题。

**性能：**
纯本地计算，单台每秒 4,096,000 个 ID，100 万需求 1 台机器即可。部署 3 台提供冗余。

**JavaScript 注意：**
ID 超过 Number.MAX_SAFE_INTEGER，返回字符串类型，客户端不做 JSON.parse 的数字解析（或配置 JSON 大数解析）。

---

## 关联文档

- [../02_storage/01_rdbms.md](../02_storage/01_rdbms.md) — B+ Tree 索引与 ID 有序性的关系
- [06_distributed_lock.md](06_distributed_lock.md) — ZooKeeper/etcd 用于机器 ID 分配
