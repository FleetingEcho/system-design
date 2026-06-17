# 方案比较：如何在面试中有条理地比较两个方案

## TL;DR

面试官问"为什么不用 X"时，不要只说"因为 Y 更好"——要展示**你比较方案的思维框架**：

1. **确认评估维度**：吞吐量、延迟、一致性、运维成本、生态……
2. **给每个维度打分**
3. **结合业务需求加权**
4. **说出你的选择和接受的代价**

---

## 为什么面试官问"为什么不用 X"

这类问题不是挖坑，是在测试你是否：
- 真正理解两个方案的差异（不是只背了结论）
- 有根据业务需求做选择的能力（工程是权衡，不是"哪个更好"）
- 能清晰表达自己的决策过程

---

## 比较框架：5 个维度

### 1. 性能（Performance）

- **吞吐量（Throughput）**：每秒能处理多少请求/消息
- **延迟（Latency）**：单次操作耗时（P50/P99）
- **扩展性（Scalability）**：增加机器能线性提升性能吗？

### 2. 可靠性（Reliability）

- **持久化**：数据会不会丢失
- **容错**：节点故障时系统还能工作吗
- **消息保证**：At-most-once / At-least-once / Exactly-once

### 3. 一致性（Consistency）

- 强一致还是最终一致
- 读写顺序保证

### 4. 运维复杂度（Operational Complexity）

- **部署难度**：多难搭起来
- **维护成本**：日常运维需要多少人
- **云托管**：有没有托管服务（减少运维）

### 5. 生态与团队熟悉度

- 社区支持、文档质量
- 团队已有经验

---

## 比较模板：Kafka vs RabbitMQ

```
问题：你的设计里用了 Kafka，为什么不用 RabbitMQ？

回答结构：

"让我从几个维度来比较：

1. 吞吐量：
   我们的需求是每秒处理 100 万条用户行为事件。
   Kafka 设计目标是高吞吐，
   单 Broker 可以处理 100MB/s ≈ 百万条/秒。
   RabbitMQ 在这个量级会遇到压力，通常在万级 QPS。
   → Kafka 更适合

2. 消息持久化与回放：
   我们需要 T+1 数据分析，分析服务需要回放昨天的数据。
   Kafka 消息默认保留 7 天，消费者可以随时回放历史。
   RabbitMQ 消息消费后就删除，不支持回放。
   → Kafka 更适合

3. 路由灵活性：
   我们的消息路由很简单，按 userId 分区。
   RabbitMQ 的 Exchange 路由功能（Topic/Fanout）
   我们用不上。
   → RabbitMQ 的优势在我们的场景不重要

4. 延迟：
   Kafka 为了提高吞吐，默认批量发送，
   单条消息延迟约 5-15ms（比 RabbitMQ 稍高）。
   我们的用户行为事件不需要实时处理，延迟可以接受。
   → 无关键影响

5. 运维：
   两者都需要专业运维，Kafka 有 Confluent Cloud 托管，
   RabbitMQ 有 CloudAMQP，运维成本差异不大。
   我们团队对 Kafka 更熟悉。
   → 轻微偏向 Kafka

结论：
  基于高吞吐需求 + 回放需求，选择 Kafka。
  代价是：单条消息延迟比 RabbitMQ 稍高（15ms vs 5ms），
  对于我们的场景（异步数据分析）这个代价可以接受。"
```

---

## 常见比较题的核心维度

### 数据库选型

| 比较点 | 关键问题 |
|--------|---------|
| MySQL vs PostgreSQL | 对 JSON、数组、窗口函数的需求？PostgreSQL 更强；MySQL 更广泛部署 |
| MySQL vs Cassandra | 需要 ACID 吗？写 QPS 多高？Cassandra 牺牲事务换高写入 |
| Redis vs Memcached | 需要数据结构（List/Set/ZSet）？持久化？→ Redis；纯缓存性能极致 → Memcached |
| DynamoDB vs Cassandra | 全托管运维便利 → DynamoDB；自建控制力强 → Cassandra |

### 消息队列选型

| 比较点 | 关键问题 |
|--------|---------|
| Kafka vs RabbitMQ | 需要回放？高吞吐？→ Kafka；复杂路由、任务队列 → RabbitMQ |
| Kafka vs SQS | 云托管无运维？→ SQS；需要 Consumer Group、回放 → Kafka |
| Redis Pub/Sub vs Kafka | 简单实时推送、不需要持久化 → Redis；高吞吐、持久化 → Kafka |

### 通信协议选型

| 比较点 | 关键问题 |
|--------|---------|
| REST vs gRPC | 对外公开 API → REST；内部微服务高频调用 → gRPC |
| REST vs GraphQL | 多客户端不同需求 → GraphQL；简单 CRUD → REST |
| HTTP vs WebSocket | 双向实时通信 → WebSocket；单向推送 → SSE 或 Long Polling |

---

## 权衡的表达方式

**不要：**
> "Kafka 比 RabbitMQ 好，所以选 Kafka。"

**要：**
> "对于我们每秒百万级的用户行为日志，Kafka 的顺序写 + 批量发送能满足吞吐需求。我们接受的代价是：单条消息延迟比 RabbitMQ 高约 10ms，以及需要维护 Kafka 集群的运维成本。如果我们的场景是低吞吐但需要灵活路由（比如不同类型的任务分发），我会选 RabbitMQ。"

**关键点：**
1. 说明业务需求（为什么这个维度重要）
2. 说明对比结果（在这个维度上哪个更好）
3. 说明选择的代价（你接受了什么，没有免费的午餐）
4. 说明反例（什么情况下你会选另一个）

---

## 面试中的实战建议

**主动引出比较：**

不要等面试官问，在介绍方案时顺带提及：
> "我这里选了 Kafka，而不是 RabbitMQ，主要是因为我们有数据回放的需求。"

**承认不确定性：**

> "我不熟悉 Pulsar 的具体性能数据，但从我了解的情况来看，
>  Kafka 在我们的用例上已经被充分验证，不确定 Pulsar 有明显优势。"

说不知道比乱说好，但要表现出你有能力评估。

---

## 关联文档

- [../reference/03_patterns.md](../reference/03_patterns.md) — 常见模式和方案
- [../../04_distributed/00_tradeoffs.md](../../04_distributed/00_tradeoffs.md) — 一致性 vs 可用性的权衡图谱
- [../../02_storage/00_choice_framework.md](../../02_storage/00_choice_framework.md) — 数据库选型框架
