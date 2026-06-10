# 数据库选型框架

## TL;DR

面试被问"你的系统用什么数据库，为什么"时，不要直接说答案，先问 4 个问题：
1. **数据有关系吗？** → 有复杂 JOIN、外键 → 倾向 RDBMS
2. **需要 ACID 事务吗？** → 金融/订单 → 倾向 RDBMS
3. **读写比例和 QPS 是多少？** → 极高写入 → 倾向 NoSQL
4. **数据量和增长率？** → 需要水平扩展 → 倾向 NoSQL

没有"最好的数据库"，只有"最适合这个场景的数据库"。

---

## 决策框架：4 个问题

### 问题 1：数据模型是什么

```
数据有明确的关系结构（JOIN、外键）？
  ├── 是 → 优先考虑 RDBMS（MySQL/PostgreSQL）
  └── 否 → 继续看数据的形态
             ├── 文档（嵌套 JSON，Schema 灵活）→ 文档型（MongoDB）
             ├── 键值对（简单查找，无复杂查询）→ KV（Redis/DynamoDB）
             ├── 时序/写多读少/按列分析 → 列存储（Cassandra/ClickHouse）
             └── 节点和关系（社交图谱）→ 图数据库（Neo4j）
```

### 问题 2：一致性要求

```
操作失败时，数据不一致的代价有多高？
  ├── 极高（金融、库存）→ 需要 ACID 事务 → RDBMS 或 NewSQL
  └── 可以接受短暂不一致 → BASE / 最终一致性 → NoSQL
```

### 问题 3：读写模式

| 场景 | 特征 | 建议 |
|------|------|------|
| 读多写少 | 大量查询，数据相对稳定 | RDBMS + 读副本 + 缓存 |
| 写多读少 | 大量写入（日志、事件流） | Cassandra / HBase |
| 读写均衡 | 常见业务系统 | RDBMS，垂直扩展先够用 |
| 极高 QPS（10万+） | 热点数据频繁读 | Redis 缓存层 |

### 问题 4：规模和扩展性

```
数据量和 QPS 预期：
  ├── 中等规模（< 数 TB，< 1万 QPS）→ 单机 RDBMS 通常够用
  ├── 大规模，需要水平扩展 → NoSQL（天然分布式）或 RDBMS 分片
  └── 全球多地域部署 → NewSQL（CockroachDB/Spanner）或 NoSQL
```

---

## 选型速查表

| 数据库类型 | 代表产品 | 选它的理由 | 不选它的理由 |
|-----------|---------|-----------|------------|
| **关系型（RDBMS）** | MySQL, PostgreSQL | 复杂查询、JOIN、ACID 事务、数据完整性约束 | 水平扩展困难，Schema 变更麻烦 |
| **文档型** | MongoDB | 灵活 Schema，嵌套文档，JSON 友好 | 跨文档事务弱，不适合复杂关联查询 |
| **KV 型** | Redis, DynamoDB | 极低延迟，简单查找，水平扩展 | 不支持复杂查询，Redis 数据量受内存限制 |
| **列存储（宽列）** | Cassandra, HBase | 极高写入吞吐，时序数据，大数据量 | 不支持 JOIN，查询模式固定，运维复杂 |
| **列存储（分析型）** | ClickHouse, BigQuery | OLAP 分析查询，聚合统计 | 不适合 OLTP（高频小事务） |
| **图数据库** | Neo4j, Amazon Neptune | 关系密集（社交、推荐、知识图谱） | 不适合普通业务数据 |
| **NewSQL** | CockroachDB, TiDB, Spanner | 分布式 ACID，水平扩展，SQL 兼容 | 相对复杂，成本高 |
| **时序数据库** | InfluxDB, TimescaleDB | 时间序列数据（监控指标、IoT） | 只适合时序场景 |
| **搜索引擎** | Elasticsearch, Solr | 全文搜索，复杂查询，倒排索引 | 不是主存储，数据需要同步，最终一致 |

---

## 常见系统的选型组合

真实系统通常**多种数据库混用**，每种数据库只做它最擅长的事：

### 电商系统
```
MySQL          → 订单、用户、库存（ACID，强一致）
Redis          → 商品价格缓存、Session、限流计数器
Elasticsearch  → 商品搜索（全文检索）
HBase/Cassandra → 用户行为日志（大量写入）
ClickHouse     → 销售数据分析（OLAP 查询）
```

### 社交平台
```
PostgreSQL     → 用户资料、关系（有 JOIN 需求）
Cassandra      → 消息、Feed 流（高写入，按时间查询）
Redis          → 在线状态、计数器（点赞数、粉丝数）
Neo4j          → 好友关系图（推荐、共同好友）
```

### SaaS 业务系统
```
PostgreSQL     → 核心业务数据（多租户、复杂查询）
Redis          → 缓存、队列
S3             → 文件存储
```

---

## 面试中如何回答选型问题

面试官问："这个系统用什么数据库？"

**错误答法**（直接给结论）：
> "用 MySQL。"

**正确答法**（展示决策过程）：
> "在选择之前，我想先确认几个点。首先，这个系统的核心数据是用户订单，有强一致性要求，不能超卖，所以需要 ACID 事务——这倾向于用关系型数据库。其次，数据之间有明确的关联，订单关联用户、商品，需要 JOIN 查询。基于这两点，我会选 MySQL 或 PostgreSQL 作为主存储。同时，商品列表查询量很大，我会加一层 Redis 缓存，把热点数据的读请求挡在数据库前面。搜索功能需要全文索引，会单独用 Elasticsearch，通过 Binlog 异步同步数据。"

---

## 与 Node.js/TS 生态的类比

你用 Prisma 连 PostgreSQL，用 ioredis 连 Redis，这两个库已经代表了两种截然不同的数据模型：

```typescript
// Prisma (RDBMS) — 有 Schema，有关系，有事务
const order = await prisma.order.create({
  data: {
    userId: 1,
    items: { create: [...] }  // JOIN 关系
  }
});

// ioredis (KV) — 无 Schema，极简操作，极低延迟
await redis.setex(`session:${userId}`, 3600, JSON.stringify(session));
const session = await redis.get(`session:${userId}`);
```

你在同一个项目里用两种不同数据库，这就是多数据库混用架构。

---

## 常见陷阱

1. **过早优化**：流量没到就上 Cassandra，把简单问题复杂化。大多数系统一台 PostgreSQL 能撑很久
2. **MongoDB 万能论**：MongoDB 适合灵活 Schema，不适合有复杂关联的业务数据
3. **Redis 当主数据库**：Redis 是内存数据库，重启默认丢数据，只适合缓存或可重建的数据
4. **忽略运维成本**：Cassandra 性能很好，但运维极其复杂，小团队慎用
5. **搜索引擎当主存储**：Elasticsearch 不是主数据库，写入性能弱，不支持事务，必须和主数据库配合使用

---

## 面试常见问答

### 简单

**Q：什么时候选 NoSQL 而不是关系型数据库？**

A：选 NoSQL 通常有以下原因：数据模型不是关系型（文档、图、时序）；需要极高的写入吞吐（Cassandra 擅长）；需要水平扩展且数据量超过单机 RDBMS 的处理能力；Schema 不固定，需要灵活性（MongoDB）。但不应该因为"NoSQL 更现代"就选 NoSQL——如果数据有关系、需要 ACID，RDBMS 仍然是最好的选择。

---

**Q：Redis 可以作为主数据库吗？**

A：通常不行。Redis 是内存数据库，有以下限制：数据量受内存限制（内存贵）；默认持久化策略会有数据丢失风险（RDB 是快照，AOF 可以配置，但有性能代价）；不支持复杂查询和 JOIN。Redis 最适合作为缓存层、Session 存储、计数器、分布式锁、消息队列，而不是主存储。如果需要持久化的 KV 存储，可以考虑 DynamoDB 或 RocksDB。

---

### 中等

**Q：你的系统从 MySQL 迁移到 Cassandra，需要考虑哪些问题？**

A：这是 ACID → BASE 的转变，需要考虑：
- **查询模式改变**：Cassandra 不支持任意 WHERE 查询和 JOIN，查询模式必须在建表时就设计好（按主键和分区键查询）
- **事务缺失**：原来的 MySQL 事务需要改成应用层补偿（Saga）或接受最终一致性
- **数据模型重设计**：关系型的范式设计要改成反范式（冗余存储换查询性能）
- **数据迁移**：双写过渡期，同时写 MySQL 和 Cassandra，验证数据一致后切流量
- **运维技能**：Cassandra 的运维（节点扩缩容、Repair、Compaction）和 MySQL 完全不同

---

**Q：面试题：设计一个用户系统，存储用户资料、好友关系、消息记录，你用什么数据库，为什么？**

A：三个功能用三种存储：
- **用户资料（PostgreSQL）**：结构化数据，有唯一性约束，偶尔需要按条件过滤，RDBMS 合适
- **好友关系（图数据库或 RDBMS）**：如果只需要"是否是好友"和"共同好友"，PostgreSQL + 索引足够；如果需要多跳查询（朋友的朋友的朋友）、推荐算法，用 Neo4j 更高效
- **消息记录（Cassandra）**：写多读少，数据量大，按用户 ID + 时间戳查询，Cassandra 的分区键设计完美契合这个场景

---

### 难

**Q：NewSQL（CockroachDB/TiDB）和 NoSQL 都能水平扩展，什么时候选 NewSQL？**

A：NewSQL 的核心价值是"分布式 + ACID"，选它的条件：
- 业务需要强一致性和 ACID 事务，但数据量超过了单机 RDBMS 的上限
- 需要 SQL 兼容（现有代码迁移成本低）
- 多地域写入（Spanner 的强项）

不选 NewSQL 的情况：
- 数据量还没超单机上限（过早优化，NewSQL 运维成本高）
- 业务可以接受最终一致性（NoSQL 更简单、成本更低）
- 查询模式固定、写入量极大（Cassandra 仍然更高效）

NewSQL 是"鱼和熊掌都想要"时的方案，代价是系统复杂度和成本更高。

---

## 关联文档

- [01_rdbms.md](01_rdbms.md) — 关系型数据库原理深入
- [02_nosql.md](02_nosql.md) — NoSQL 各类型详解
- [../04_distributed/00_tradeoffs.md](../04_distributed/00_tradeoffs.md) — 一致性 vs 可用性 vs 延迟权衡图谱
- [../05_methodology/reference/03_patterns.md](../05_methodology/reference/03_patterns.md) — 读写分离、CQRS 等常见存储设计模式
