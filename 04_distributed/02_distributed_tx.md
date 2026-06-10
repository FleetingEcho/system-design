# 分布式事务：2PC / Saga / TCC

## TL;DR

- **分布式事务**：跨多个数据库或服务的事务，无法用单一 ACID 事务覆盖
- **2PC（两阶段提交）**：强一致，但性能差、有阻塞问题——不推荐在微服务中使用
- **Saga**：最终一致，通过补偿操作回滚——是微服务架构中的主流方案
- **TCC（Try-Confirm-Cancel）**：业务层的"显式锁"，比 Saga 更强的一致性，代价是侵入业务

---

## 为什么需要分布式事务

单数据库事务解决了"一台机器上多条 SQL 的原子性"。但微服务架构下，一个业务操作需要跨多个服务：

```
用户下单：
  1. 订单服务（自己的 MySQL）：创建订单
  2. 库存服务（自己的 MySQL）：扣减库存
  3. 支付服务（自己的 MySQL）：扣款

这三步用的是三个独立数据库，不能用一个 BEGIN TRANSACTION 覆盖
```

**如果不处理：**
```
步骤1成功，步骤2成功，步骤3失败（余额不足）
→ 订单已创建，库存已扣减，但没有收到钱
→ 数据不一致
```

---

## 2PC（Two-Phase Commit，两阶段提交）

### 流程

2PC 引入一个**协调者（Coordinator）**协调所有参与者（Participant）：

**阶段一：Prepare（准备）**
```
协调者 → 参与者1（订单服务）："你能提交吗？"
协调者 → 参与者2（库存服务）："你能提交吗？"
协调者 → 参与者3（支付服务）："你能提交吗？"

参与者1 → 锁定资源，写 Undo/Redo Log → "可以提交"（YES）
参与者2 → 锁定资源，写 Undo/Redo Log → "可以提交"（YES）
参与者3 → 锁定资源，写 Undo/Redo Log → "不能提交"（NO，余额不足）
```

**阶段二：Commit 或 Abort（提交或中止）**
```
任何参与者回答 NO → 协调者广播 ABORT
所有参与者回答 YES → 协调者广播 COMMIT

收到 ABORT：
  参与者1 → 用 Undo Log 回滚
  参与者2 → 用 Undo Log 回滚
  参与者3 → 已经拒绝，无需操作

收到 COMMIT：
  所有参与者提交本地事务，释放锁
```

### 2PC 的问题

**阻塞问题（Blocking）：**
```
协调者在等待所有参与者 Prepare 响应
参与者3 网络故障，迟迟没有响应
→ 参与者1 和 2 锁住资源，无限等待
→ 这段时间内，其他请求无法访问这些资源
```

**协调者单点故障：**
```
所有参与者 Prepare 完成，等待协调者的 COMMIT 指令
协调者在发出 COMMIT 前崩溃了
→ 参与者不知道该提交还是回滚
→ 无限阻塞，等待协调者恢复
```

**性能差：**
- 每次事务需要 2 次网络往返
- Prepare 阶段期间资源一直锁定
- 任何参与者慢，整个事务就慢

**结论：2PC 在微服务中通常不推荐，只在严格需要强一致的场景（如数据库内部）中使用。**

---

## Saga 模式

### 核心思想

把跨服务的事务**拆成一系列本地事务**，每步成功发出一个事件或消息，触发下一步。某步失败时，执行**补偿操作（Compensating Transaction）**逆向撤销已完成的步骤。

```
正向流程：
  T1: 创建订单 → 发事件"订单已创建"
  T2: 扣减库存 → 发事件"库存已扣减"
  T3: 扣款     → 发事件"支付成功"
  T4: 更新订单为已完成

补偿流程（T3 失败时）：
  C3: 退款（无需，T3 已失败）
  C2: 恢复库存
  C1: 取消订单
```

### Saga 的两种实现

**1. 编排式（Choreography）：** 各服务通过事件驱动，自主响应

```
订单服务 → 发"订单已创建"到 Kafka
  ← 库存服务监听"订单已创建"，扣减库存，发"库存已扣减"
     ← 支付服务监听"库存已扣减"，发起扣款，发"支付成功"
        ← 订单服务监听"支付成功"，更新订单状态

失败时：
支付服务扣款失败 → 发"支付失败"
  ← 库存服务监听"支付失败"，恢复库存
     ← 订单服务监听库存恢复或超时，取消订单
```

优点：去中心化，服务间松耦合，没有协调者单点
缺点：流程分散在各服务中，难以追踪和调试；随着步骤增多，事件关系变复杂

**2. 编排式（Orchestration）：** 一个 Saga 协调者驱动所有步骤

```
Saga 协调者（可以是状态机）：
  1. 调用订单服务：创建订单 → 成功
  2. 调用库存服务：扣减库存 → 成功
  3. 调用支付服务：扣款 → 失败
  4. 触发补偿：
     - 调用库存服务：恢复库存
     - 调用订单服务：取消订单
```

优点：流程集中，容易追踪和调试；失败处理逻辑清晰
缺点：协调者需要高可用，本身是个有状态的服务

**实践推荐：Orchestration（编排式）在工程上更容易维护。**

### 事务性发件箱（Transactional Outbox）

Saga 中最难的问题：**本地事务提交了，但消息还没发出去，服务崩溃**

```
订单服务：
  BEGIN TRANSACTION
    INSERT INTO orders (status='pending')
    // 崩溃！消息还没发出去
    PUBLISH event to Kafka
  COMMIT
→ 订单创建了，但库存没扣，消息丢了
```

**解决方案：Outbox 模式**

```
订单服务：
  BEGIN TRANSACTION
    INSERT INTO orders (status='pending')
    INSERT INTO outbox (event='ORDER_CREATED', data=...)  // 消息写入同一个 DB
  COMMIT

// 独立的 Outbox 处理器：
  轮询 outbox 表，把未发送的消息发到 Kafka
  发送成功后，删除 outbox 记录（或标记为已发送）
```

消息写入和业务数据在同一个数据库事务里，要么都成功要么都失败。Outbox 处理器保证消息最终发出（At-least-once）。

---

## TCC（Try-Confirm-Cancel）

### 核心思想

TCC 是在业务层实现的"显式锁"，分三个阶段：

- **Try**：尝试操作，预留资源（不真正执行，只是锁定）
- **Confirm**：确认执行，真正提交
- **Cancel**：取消，释放预留的资源

```
下单 TCC 示例：

Try 阶段：
  订单服务：创建"预留订单"（状态=RESERVED，非 CONFIRMED）
  库存服务：冻结 1 件库存（frozen_count += 1，available_count -= 1）
  支付服务：预扣款（冻结余额，不真正扣减）

Confirm 阶段（所有 Try 成功后）：
  订单服务：预留订单 → 确认订单（status=CONFIRMED）
  库存服务：冻结库存 → 扣减库存（frozen_count -= 1，sold_count += 1）
  支付服务：冻结余额 → 真正扣款

Cancel 阶段（任何 Try 失败时）：
  订单服务：删除预留订单
  库存服务：释放冻结库存（frozen_count -= 1，available_count += 1）
  支付服务：解冻余额
```

### TCC vs Saga

| 维度 | Saga | TCC |
|------|------|-----|
| 一致性 | 最终一致（补偿可能有延迟） | 更强（Try 阶段资源锁定，不会被并发） |
| 业务侵入 | 低（补偿是独立逻辑） | **高**（每个接口都要实现 Try/Confirm/Cancel 三个方法） |
| 并发控制 | 无（可能两个事务同时修改同一数据） | 有（Try 阶段锁定资源，防并发冲突） |
| 实现复杂度 | 中 | 高 |
| 适用场景 | 大多数微服务事务 | 需要防并发抢占的场景（预约、预留） |

**TCC 适合：**
- 机票/酒店预订（座位/房间需要先"锁定"，等确认支付后才真正扣减）
- 库存抢占（高并发秒杀时，Try 阶段锁定库存，防止超卖）

---

## 最佳实践总结

**选型建议：**

```
跨服务事务？
  ├── 可以接受最终一致性？
  │   ├── 需要防并发资源抢占？→ TCC
  │   └── 不需要 → Saga（优先编排式）
  │
  └── 必须强一致？
      ├── 少数极关键操作 → 考虑重新设计（合并服务？）
      └── 确实需要 → 2PC（仅限数据库层，如 XA 事务）
```

**无论用哪种方案，都要保证：**

1. **每个步骤幂等**：消息可能被重复投递，操作多次等于操作一次
2. **补偿操作不可失败**：补偿本身是关键路径，必须有重试机制
3. **状态可查**：Saga/TCC 的每一步状态要持久化，服务重启后能恢复
4. **最大努力交付**：消息可能丢，用 At-least-once + 幂等，而不是追求 Exactly-once

---

## 与 Node.js/TS 生态的类比

简单的 Saga 实现（编排式，没有框架时手写）：

```typescript
class OrderSaga {
  async execute(orderData: OrderData): Promise<void> {
    let orderId: string | null = null;
    let stockReserved = false;

    try {
      // Step 1: 创建订单
      orderId = await orderService.createOrder(orderData);

      // Step 2: 扣减库存
      await inventoryService.deductStock(orderData.itemId, 1);
      stockReserved = true;

      // Step 3: 扣款
      await paymentService.charge(orderData.userId, orderData.amount);

      // Step 4: 确认订单
      await orderService.confirmOrder(orderId);

    } catch (error) {
      // 触发补偿
      if (stockReserved) {
        await inventoryService.restoreStock(orderData.itemId, 1);
      }
      if (orderId) {
        await orderService.cancelOrder(orderId);
      }
      throw error;
    }
  }
}
```

生产环境通常用专门的 Saga 框架（如 Conductor、Temporal）管理状态和重试。

---

## 常见陷阱

1. **补偿操作没有幂等**：网络超时后重试补偿，重复退款或重复恢复库存
2. **补偿操作可以失败**：补偿失败了怎么办？需要人工介入或告警，不能无声地失去
3. **Saga 中间状态用户可见**：步骤1-2完成，步骤3还没完成时，用户可能看到"库存已扣但未支付"的中间状态，需要在产品层面屏蔽
4. **用了 2PC 但协调者没做高可用**：协调者宕机，所有参与者永久阻塞
5. **没有 Outbox**：直接 commit DB 然后 publish 消息，进程崩溃导致消息丢失，数据不一致

---

## 面试常见问答

### 简单

**Q：什么是分布式事务？为什么需要它？**

A：分布式事务是跨多个数据库或服务的原子操作，需要保证所有参与方要么全部成功，要么全部回滚。微服务架构下，一个业务操作（如下单）需要同时操作订单库、库存库、账户库，它们不在同一个数据库中，无法用单个 BEGIN TRANSACTION 覆盖。如果不处理，可能出现"钱扣了但库存没减"或"库存减了但订单没建"的数据不一致问题。

---

**Q：2PC 有什么缺点？**

A：2PC 的主要缺点：
1. **阻塞**：Prepare 阶段锁定资源，如果协调者或某个参与者故障，资源会被无限锁住
2. **协调者单点**：协调者故障后，参与者不知道该提交还是回滚，形成不确定状态
3. **性能差**：两次网络往返 + 资源锁定期间其他请求阻塞
适合数据库层面（XA 协议），不适合微服务间。

---

### 中等

**Q：Saga 模式中"补偿事务"和"回滚"有什么区别？**

A：数据库回滚（ROLLBACK）是把事务中的所有操作撤销，就像它从未发生过一样。Saga 的补偿事务是一个新的正向操作，它的语义是"撤销之前的操作效果"，但不能"假装它没发生过"——中间状态可能已经被其他系统看到了（比如库存服务已经发了"库存已扣减"的事件）。因此补偿操作是业务补偿（退款、恢复库存、取消订单），而不是数据库级别的撤销。这就是为什么 Saga 是最终一致性而非强一致性。

---

### 难

**Q：用 Saga 模式实现一个订单系统，如何处理支付超时的情况？**

A：支付超时是 Saga 中最复杂的场景之一，因为"超时"意味着不知道支付是否成功。

**设计方案：**

1. **支付操作加唯一幂等 Key**：`paymentService.charge(userId, amount, idempotencyKey=orderId)`。这样即使因为超时重试，支付服务保证同一个 orderId 只扣款一次。

2. **状态轮询**：超时后不直接触发补偿，而是主动查询支付结果：
```typescript
const result = await paymentService.queryStatus(orderId);
if (result === 'SUCCESS') {
  // 继续后续步骤
} else if (result === 'FAILED') {
  // 触发补偿
} else {
  // 还在处理中，继续等待或重试
}
```

3. **超时补偿（最终手段）**：如果轮询超过最大等待时间（如 30 分钟），触发补偿（取消订单，支付服务侧会通过对账发现这笔扣款并退回）。

4. **对账机制**：每天定期对比订单系统和支付系统的数据，发现不一致自动告警，人工或自动修复。

关键原则：**超时 ≠ 失败**，不能立即触发补偿（可能支付已成功但网络超时了），要先查询状态，查询超时后再补偿，再配合对账兜底。

---

## 关联文档

- [00_tradeoffs.md](00_tradeoffs.md) — 何时选强一致 vs 最终一致
- [01_consistency_models.md](01_consistency_models.md) — Saga 对应的最终一致性级别
- [../03_communication/02_async.md](../03_communication/02_async.md) — 消息队列在 Saga 中的作用（Outbox 模式）
- [04_fault_tolerance.md](04_fault_tolerance.md) — 幂等性的实现方式
