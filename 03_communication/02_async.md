# 异步通信：消息队列 / Kafka / RabbitMQ / 语义保证

## TL;DR

- **消息队列**：解耦生产者和消费者，允许异步处理，削峰填谷
- **Kafka**：高吞吐日志流，消息持久化，消费者可以回放历史——适合事件流、日志、大数据管道
- **RabbitMQ**：传统消息队列，灵活路由，消息确认——适合任务队列、业务事件
- **三种语义**：At-most-once / At-least-once / Exactly-once——大多数系统用 At-least-once + 幂等消费

---

## 为什么需要消息队列

### 同步调用的问题

```
支付成功 → 同步调用：
  → 更新订单状态（10ms）
  → 发短信通知（100ms）
  → 更新积分（30ms）
  → 通知仓库备货（50ms）
  → 记录日志（20ms）
总延迟：210ms，且任何一步失败整个链路失败
```

**问题：**
1. **耦合**：支付服务需要知道并直接调用所有下游服务
2. **延迟**：用户等待所有下游处理完才收到支付成功响应
3. **脆弱**：短信服务挂了，支付也失败了

### 消息队列的解决方案

```
支付成功 → 发消息到队列（5ms）→ 立即返回支付成功

队列中的消息被异步消费：
  ← 订单服务消费：更新订单状态
  ← 通知服务消费：发短信
  ← 积分服务消费：更新积分
  ← 仓库服务消费：备货
  ← 日志服务消费：记录日志
```

**优点：**
1. **解耦**：支付服务只管发消息，不知道也不关心谁来消费
2. **低延迟**：用户只等写消息（5ms），其余异步
3. **容错**：短信服务挂了，消息在队列里等着，恢复后继续消费，不影响支付

### 消息队列的其他用途

**削峰填谷（Buffer）：**
```
瞬间流量高峰 → 消息堆积在队列 → 消费者匀速处理
```
双十一秒杀：订单请求打入队列，后端按能力匀速处理，而不是瞬间压垮数据库。

**广播（Fan-out）：**
一条消息被多个消费者各自独立消费（比如：新用户注册消息，被通知服务、推荐服务、分析服务各消费一次）。

---

## Kafka

### 核心模型

Kafka 是一个**分布式日志系统**，数据存储模型类似于日志文件的追加写：

```
Topic: user-events
  ├── Partition 0: [事件1, 事件2, 事件5, 事件8 ...]  → Broker 1
  ├── Partition 1: [事件3, 事件6, 事件9 ...]          → Broker 2
  └── Partition 2: [事件4, 事件7, 事件10 ...]         → Broker 3
```

**关键概念：**
- **Topic**：消息的分类（类比数据库的表）
- **Partition**：Topic 的分片，每个 Partition 是一个有序的日志文件，消息只能追加
- **Offset**：消息在 Partition 内的位置编号（从 0 开始），消费者通过 Offset 追踪进度
- **Consumer Group**：一组消费者共同消费一个 Topic，每个 Partition 只分配给组内一个消费者

### 消费者组（Consumer Group）

```
Topic: orders（3个 Partition）
Consumer Group A（订单服务，3个实例）：
  实例1 → Partition 0
  实例2 → Partition 1
  实例3 → Partition 2
  （每个 Partition 由一个实例独占，并行消费）

Consumer Group B（统计服务，1个实例）：
  实例1 → Partition 0, 1, 2
  （一个实例消费所有 Partition，进度独立于 Group A）
```

同一条消息可以被不同 Consumer Group 各消费一次，但同一个 Consumer Group 内只消费一次。

### Kafka 的特点

**持久化：**
消息写入磁盘，默认保留 7 天（可配置）。消费者可以回放（Replay）历史消息，这是 Kafka 和传统消息队列的核心区别。

**高吞吐：**
顺序写磁盘（比随机写快 100 倍）+ 零拷贝（Zero-Copy，数据不经过用户空间直接从磁盘到网络）+ 批量发送，单 Broker 可以处理每秒百万级消息。

**水平扩展：**
增加 Partition 数量，消费者组自动 Rebalance，消费能力线性增长。

**有序性：**
同一个 Partition 内消息严格有序（先进先出）。跨 Partition 不保证有序。
控制有序性：相同 Key 的消息路由到同一个 Partition（如同一个用户的事件按用户 ID 分区）。

### Kafka 适合什么

- 事件流（用户行为日志、系统事件）
- 日志收集（ELK 架构的数据源）
- 实时数据管道（从一个系统流向另一个系统）
- 需要回放历史数据的场景
- 高吞吐（每秒百万消息级别）

### Kafka 不适合什么

- 需要复杂路由规则（按内容路由消息）
- 任务队列（每条消息只被一个消费者处理一次，Kafka 可以做但不是最优）
- 消息量小、需要快速交付的场景（Kafka 有额外的 broker 开销）

---

## RabbitMQ

### 核心模型

RabbitMQ 是一个传统消息代理（Message Broker），基于 AMQP 协议：

```
生产者 → Exchange（交换机）→ Queue（队列）→ 消费者
```

**Exchange 的路由类型：**

```
Direct Exchange：按 routing key 精确匹配
  消息 routing_key="order.created" → 只发给绑定了该 key 的队列

Topic Exchange：按通配符匹配
  消息 routing_key="order.created.cn" 
  → 匹配 "order.#"（# 匹配多个词）
  → 匹配 "order.*.cn"（* 匹配一个词）

Fanout Exchange：广播给所有绑定的队列

Headers Exchange：按消息 Header 属性匹配
```

### RabbitMQ 的特点

**消息确认（ACK）机制：**
```
消费者收到消息 → 处理中
处理成功 → 发 ACK → RabbitMQ 删除消息
处理失败 → 发 NACK → RabbitMQ 重新入队或进死信队列
```

这是 At-least-once 语义的实现：消息被确认前不会删除，即使消费者崩溃也不会丢消息。

**死信队列（Dead Letter Queue，DLQ）：**
消息处理失败超过重试次数，进入 DLQ：
```
普通队列 → 处理失败（重试3次）→ 死信队列
                                  ↑ 可以人工介入，分析为什么失败
```

**延迟队列（Delayed Queue）：**
消息在指定时间后才投递（基于 TTL + DLQ 实现，或用插件）：
```
用户注册后 24 小时 → 发提醒激活邮件
订单创建后 30 分钟未支付 → 取消订单
```

### RabbitMQ 适合什么

- 任务队列（图片处理、邮件发送等耗时任务）
- 需要复杂路由规则的场景
- 需要消息优先级
- 需要延迟消息
- 消息量中等、对消息 ACK 有强要求

---

## Kafka vs RabbitMQ 对比

| 维度 | Kafka | RabbitMQ |
|------|-------|----------|
| 设计定位 | 分布式日志/事件流 | 传统消息代理/任务队列 |
| 吞吐量 | 极高（百万/秒） | 中等（万级/秒） |
| 消息持久化 | 持久化，可回放 | 消费后删除（默认） |
| 消费模式 | 消费者主动 Pull | Push 给消费者 |
| 消息顺序 | Partition 内有序 | 队列内有序 |
| 路由能力 | 简单（按 Topic/Key） | 丰富（4种 Exchange 类型） |
| 延迟 | 较高（批量发送优化吞吐） | 较低（单消息延迟更小） |
| 消息回放 | 支持 | 不支持 |
| 适用场景 | 事件流、日志管道 | 任务队列、业务事件 |

---

## 消息语义保证

这是面试必考内容，理解清楚三个语义的区别和实现。

### At-most-once（最多一次）

消息发出后不管是否到达，不重试。可能丢消息，但绝不重复。

```
生产者发消息 → 消费者收到（或没收到）
消费者不发 ACK → 消息立即删除
如果消费者崩溃 → 消息丢失，不重试
```

**适合：** 允许丢失的日志、监控指标（丢几个数据点无所谓）

### At-least-once（至少一次）

消息一定会被投递，但可能重复投递。不丢消息，但可能重复。

```
生产者发消息
消费者处理成功 → 发 ACK → 消息删除
消费者处理中崩溃（没发 ACK）→ 消息重新投递 → 消费者重新处理（重复！）
```

**适合：** 大多数业务场景，配合**幂等消费**（见下文）使用

### Exactly-once（恰好一次）

消息被且只被处理一次。最强保证，也最难实现。

**Kafka 的 Exactly-once 实现：**
```
生产者 + 事务 API：
  producer.beginTransaction()
  producer.send(records)
  consumer.commitOffset()  // 把 offset 和消息生产放在同一个事务里
  producer.commitTransaction()
```

代价：性能下降 20-30%，实现复杂。

**实际工程中更常用的方式：At-least-once + 幂等消费**

### 幂等消费（Idempotent Consumer）

与其追求 Exactly-once，不如让消费者能安全地处理重复消息：

```typescript
async function processOrder(message: OrderMessage): Promise<void> {
  const messageId = message.id;

  // 检查是否已处理（幂等检查）
  const processed = await db.processedMessages.findOne({ id: messageId });
  if (processed) {
    return;  // 已处理，跳过（幂等）
  }

  // 在事务中处理业务 + 记录已处理
  await db.transaction(async (tx) => {
    await tx.orders.update({ status: 'completed', id: message.orderId });
    await tx.processedMessages.insert({ id: messageId, processedAt: new Date() });
  });
}
```

这样，同一条消息被重复投递时，第二次处理时发现 `processedMessages` 里已有记录，直接跳过。

---

## 与 Node.js/TS 生态的类比

```typescript
// 使用 bullmq（基于 Redis 的任务队列）
import { Queue, Worker } from 'bullmq';

// 生产者：添加任务
const emailQueue = new Queue('emails', { connection: redis });
await emailQueue.add('send-welcome', { userId: 1, email: 'alice@example.com' });

// 消费者：处理任务（At-least-once，bullmq 内置重试）
const worker = new Worker('emails', async (job) => {
  await sendEmail(job.data.email, 'Welcome!');
}, { connection: redis });

// bullmq 会在 job 处理失败时自动重试
// 如果 sendEmail 是幂等的（相同邮件内容+收件人不会重复发），就是安全的
```

---

## 常见陷阱

1. **没有处理消费者失败**：消费者崩溃，消息既不 ACK 也不 NACK，队列里消息越积越多
2. **消费者不幂等**：At-least-once 系统中，消费者处理重复消息导致重复扣款、重复发邮件
3. **Kafka Partition 数设置过少**：Partition 数是消费者并行度的上限，后期增加 Partition 会导致同一 Key 的消息重新分布到不同 Partition，破坏有序性
4. **消息过大**：Kafka 默认消息大小限制 1MB，发送大文件应该只发消息 ID，数据存对象存储，消费者再取
5. **消息堆积不告警**：消息消费跟不上生产速度，队列越堆越深，应该监控消费延迟（Consumer Lag）

---

## 面试常见问答

### 简单

**Q：消息队列的主要作用是什么？**

A：三个核心作用：
1. **解耦**：生产者和消费者不需要直接知道对方，通过消息传递，服务可以独立部署和扩展
2. **异步**：耗时操作（发邮件、发短信）不阻塞主流程，用户体验更好
3. **削峰填谷**：流量高峰时消息堆积在队列，消费者按自己的处理能力匀速消费，保护下游系统

---

**Q：At-least-once 和 Exactly-once 有什么区别？实际工程中通常选哪个？**

A：At-least-once 保证消息不丢失，但可能被投递多次（消费者未 ACK 时重试）。Exactly-once 保证消息被且只被处理一次，需要分布式事务支持，实现复杂且性能代价大。实际工程中通常选 **At-least-once + 幂等消费者**：允许重复投递，但消费端设计成幂等（相同消息处理多次结果相同），效果等同于 Exactly-once，且实现简单得多。

---

### 中等

**Q：Kafka 和 RabbitMQ 的核心区别是什么？如何选择？**

A：核心区别在于设计定位：Kafka 是分布式日志系统，消息持久化到磁盘，消费者可以随时回放历史消息，适合高吞吐的事件流和数据管道；RabbitMQ 是传统消息代理，消费后删除，路由规则灵活，有完善的消息确认和死信机制，适合任务队列和业务事件。选型建议：需要回放历史 / 高吞吐 / 事件驱动数据管道 → Kafka；需要复杂路由 / 延迟消息 / 死信处理 / 任务队列 → RabbitMQ。

---

**Q：Kafka 如何保证同一个 Key 的消息有序消费？**

A：Kafka 在 Partition 级别保证有序。生产者发消息时指定 Key，Kafka 用 `hash(Key) % PartitionCount` 把相同 Key 的消息路由到同一个 Partition。同一个 Partition 内消息严格按写入顺序排列。同一个 Consumer Group 内，一个 Partition 只分配给一个消费者实例，所以同一个 Key 的消息由同一个消费者实例按顺序处理。前提：Partition 数量不能中途修改（改了 hash 值变了，同一 Key 可能跑到不同 Partition）。

---

### 难

**Q：用消息队列实现分布式事务（Saga 模式），如何保证最终一致性？**

A：Saga 模式把跨服务的事务拆成一系列本地事务，通过消息队列传递事件，每步失败触发补偿。

以下单流程为例：

```
1. 订单服务：创建订单（待支付） → 发消息"订单已创建"
2. 库存服务：消费消息，扣减库存 → 发消息"库存已扣减"
            失败 → 发消息"库存扣减失败"
3. 支付服务：消费"库存已扣减"，发起扣款 → 发消息"支付成功"
            失败 → 发消息"支付失败"
4. 订单服务：消费"支付成功"，更新订单为已完成
```

补偿流程（任何步骤失败时反向触发）：
```
支付失败 → 订单服务消费"支付失败"，取消订单
库存扣减失败 → 支付服务已扣款？→ 发消息触发退款
```

关键点：
- 每个步骤都要幂等（消息重试不会产生副作用）
- 每个本地事务和发消息操作要用**事务性发件箱（Transactional Outbox）**：把消息先写入数据库（和业务数据在同一个事务），再异步发出，避免"DB 提交了但消息没发出"的问题
- 补偿操作必须设计为幂等且不可失败

---

## 关联文档

- [01_sync.md](01_sync.md) — 同步 vs 异步通信的权衡
- [../04_distributed/02_distributed_tx.md](../04_distributed/02_distributed_tx.md) — Saga 模式详解
- [../04_distributed/04_fault_tolerance.md](../04_distributed/04_fault_tolerance.md) — 消息消费的重试和幂等性
- [../06_case_studies/04_notification_system.md](../06_case_studies/04_notification_system.md) — 消息队列在通知系统中的应用
