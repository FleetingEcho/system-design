# CQRS + Event Sourcing

> 命令查询责任分离（CQRS）和事件溯源（Event Sourcing）：支付、审计、金融场景的标配模式。
> 面试中常以"如何设计一个可审计的订单系统"或"如何实现撤销/回滚"来考察。

---

## 为什么需要 CQRS + Event Sourcing

```
传统 CRUD 的痛点：
  ┌─────────────────────────────────────────┐
  │ UPDATE orders SET status='paid' WHERE id=123 │
  └─────────────────────────────────────────┘
  问题：
  1. 无法知道"状态为什么变成 paid"（谁操作？什么时间？经过哪些中间状态？）
  2. 并发写入冲突（多个进程同时更新同一行）
  3. 读写用同一个模型（复杂查询和快速写入在同一个表，索引冲突）
  4. 无法回放历史（"把系统还原到昨天下午 3 点"做不到）

Event Sourcing 的解法：
  不存储"当前状态"，只存储"发生了什么事件"
  ┌──────────────────────────────────────┐
  │ INSERT events (OrderCreated)          │
  │ INSERT events (PaymentReceived)       │
  │ INSERT events (OrderShipped)          │
  └──────────────────────────────────────┘
  当前状态 = 回放所有事件（状态是事件的"投影"）

CQRS 的解法：
  Command（写）和 Query（读）用不同的模型
  写：Event Store（只追加，高吞吐）
  读：Read Model / Projection（为查询优化的预计算视图）
```

---

## 架构全景

```
                    Command Side（写）
                    ──────────────────
Client ─→ Command ─→ Command Handler ─→ Domain Aggregate
                                              │
                                              │ raise events
                                              ↓
                                       Event Store（只追加）
                                              │
                                              │ publish
                                              ↓
                                       Event Bus（内存或 Kafka）
                                              │
                     ┌────────────────────────┘
                     │ subscribe
                     ↓
              Projector / Read Model Builder
                     │ upsert
                     ↓
              Read Database（PostgreSQL / Redis / Elasticsearch）

                    Query Side（读）
                    ──────────────────
Client ─→ Query ─→ Query Handler ─→ Read Database（直接查，无业务逻辑）
```

---

## Event Store 设计

```typescript
// Event Store：只追加（Append-Only），永不修改历史

// 事件基类
interface DomainEvent {
  eventId: string;
  eventType: string;
  aggregateId: string;
  aggregateType: string;
  version: number;       // 聚合的事件序号（乐观锁用）
  occurredAt: Date;
  payload: Record<string, unknown>;
  metadata: {
    userId?: string;
    correlationId?: string;  // 关联同一业务流程的多个事件
    causationId?: string;    // 触发此事件的 command ID
  };
}

// 订单相关事件
interface OrderCreated extends DomainEvent {
  eventType: 'OrderCreated';
  payload: {
    orderId: string;
    userId: string;
    items: Array<{ productId: string; quantity: number; price: number }>;
    totalAmount: number;
  };
}

interface PaymentReceived extends DomainEvent {
  eventType: 'PaymentReceived';
  payload: {
    orderId: string;
    paymentId: string;
    amount: number;
    method: 'credit_card' | 'bank_transfer';
  };
}

interface OrderCancelled extends DomainEvent {
  eventType: 'OrderCancelled';
  payload: {
    orderId: string;
    reason: string;
    cancelledBy: string;
  };
}

type OrderEvent = OrderCreated | PaymentReceived | OrderCancelled;
```

```typescript
// src/infrastructure/event-store.ts
import { prisma } from '../lib/prisma';

class EventStore {
  async append(
    aggregateId: string,
    events: Omit<DomainEvent, 'eventId' | 'occurredAt'>[],
    expectedVersion: number  // 乐观锁：期望的当前版本号
  ): Promise<void> {
    await prisma.$transaction(async (tx) => {
      // 乐观锁检查：当前版本必须等于 expectedVersion
      const currentVersion = await tx.event.aggregate({
        where: { aggregateId },
        _max: { version: true },
      });

      const currentMax = currentVersion._max.version ?? -1;
      if (currentMax !== expectedVersion) {
        throw new ConcurrencyError(
          `Concurrency conflict: expected version ${expectedVersion}, got ${currentMax}`
        );
      }

      // 批量写入事件
      await tx.event.createMany({
        data: events.map((event, i) => ({
          eventId: randomUUID(),
          eventType: event.eventType,
          aggregateId: event.aggregateId,
          aggregateType: event.aggregateType,
          version: expectedVersion + 1 + i,
          occurredAt: new Date(),
          payload: event.payload as Prisma.JsonObject,
          metadata: event.metadata as Prisma.JsonObject,
        })),
      });
    });

    // 发布事件到 Event Bus（让 Projector 更新 Read Model）
    for (const event of events) {
      await eventBus.publish(event);
    }
  }

  async getEvents(aggregateId: string, fromVersion = 0): Promise<DomainEvent[]> {
    const events = await prisma.event.findMany({
      where: { aggregateId, version: { gte: fromVersion } },
      orderBy: { version: 'asc' },
    });

    return events.map(e => ({
      ...e,
      payload: e.payload as Record<string, unknown>,
      metadata: e.metadata as Record<string, unknown>,
    }));
  }
}
```

---

## Domain Aggregate（聚合）

```typescript
// src/domain/order.aggregate.ts
// 聚合是事件的"生产者"和"状态重建者"

type OrderStatus = 'pending' | 'paid' | 'shipped' | 'delivered' | 'cancelled';

interface OrderState {
  id: string;
  userId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  totalAmount: number;
  status: OrderStatus;
  version: number;
}

class OrderAggregate {
  private state: OrderState | null = null;
  private uncommittedEvents: DomainEvent[] = [];
  private version = -1;  // -1 = 新建（无历史事件）

  // 从事件历史重建状态
  static fromEvents(events: DomainEvent[]): OrderAggregate {
    const aggregate = new OrderAggregate();
    for (const event of events) {
      aggregate.apply(event, false);  // false = 不加入 uncommitted
    }
    return aggregate;
  }

  // 处理 Command：创建订单
  createOrder(command: {
    orderId: string;
    userId: string;
    items: Array<{ productId: string; quantity: number; price: number }>;
  }) {
    if (this.state) throw new Error('Order already exists');

    const totalAmount = command.items.reduce((sum, i) => sum + i.price * i.quantity, 0);

    this.raise({
      eventType: 'OrderCreated',
      aggregateId: command.orderId,
      aggregateType: 'Order',
      version: this.version + 1,
      payload: {
        orderId: command.orderId,
        userId: command.userId,
        items: command.items,
        totalAmount,
      },
      metadata: {},
    } as Omit<OrderCreated, 'eventId' | 'occurredAt'>);
  }

  // 处理 Command：确认支付
  receivePayment(command: { paymentId: string; amount: number; method: string }) {
    if (!this.state) throw new NotFoundError('Order not found');
    if (this.state.status !== 'pending') {
      throw new InvalidStateError(`Cannot receive payment for ${this.state.status} order`);
    }
    if (command.amount !== this.state.totalAmount) {
      throw new ValidationError('Payment amount does not match order total');
    }

    this.raise({
      eventType: 'PaymentReceived',
      aggregateId: this.state.id,
      aggregateType: 'Order',
      version: this.version + 1,
      payload: {
        orderId: this.state.id,
        paymentId: command.paymentId,
        amount: command.amount,
        method: command.method,
      },
      metadata: {},
    } as Omit<PaymentReceived, 'eventId' | 'occurredAt'>);
  }

  // 应用事件（更新内存状态）
  private apply(event: DomainEvent, isNew: boolean) {
    switch (event.eventType) {
      case 'OrderCreated': {
        const p = event.payload as OrderCreated['payload'];
        this.state = {
          id: p.orderId,
          userId: p.userId,
          items: p.items,
          totalAmount: p.totalAmount,
          status: 'pending',
          version: event.version,
        };
        break;
      }
      case 'PaymentReceived': {
        this.state!.status = 'paid';
        this.state!.version = event.version;
        break;
      }
      case 'OrderCancelled': {
        this.state!.status = 'cancelled';
        this.state!.version = event.version;
        break;
      }
    }

    this.version = event.version;
    if (isNew) this.uncommittedEvents.push(event as DomainEvent);
  }

  private raise(event: Omit<DomainEvent, 'eventId' | 'occurredAt'>) {
    this.apply(event as DomainEvent, true);
  }

  getUncommittedEvents() { return [...this.uncommittedEvents]; }
  clearUncommittedEvents() { this.uncommittedEvents = []; }
  getVersion() { return this.version; }
  getState() { return this.state; }
}
```

---

## Command Handler

```typescript
// src/application/order.command-handler.ts
class OrderCommandHandler {
  constructor(
    private eventStore: EventStore,
    private orderRepository: OrderRepository  // 负责加载/保存聚合
  ) {}

  async handle(command: CreateOrderCommand): Promise<string> {
    const aggregate = new OrderAggregate();
    aggregate.createOrder({
      orderId: randomUUID(),
      userId: command.userId,
      items: command.items,
    });

    const events = aggregate.getUncommittedEvents();
    await this.eventStore.append(
      events[0].aggregateId,
      events,
      -1  // 新建：期望版本 -1
    );

    return events[0].aggregateId;  // 返回订单 ID
  }

  async handlePayment(command: ReceivePaymentCommand): Promise<void> {
    // 从 Event Store 重建聚合
    const events = await this.eventStore.getEvents(command.orderId);
    if (!events.length) throw new NotFoundError('Order not found');

    const aggregate = OrderAggregate.fromEvents(events);
    aggregate.receivePayment({
      paymentId: command.paymentId,
      amount: command.amount,
      method: command.method,
    });

    await this.eventStore.append(
      command.orderId,
      aggregate.getUncommittedEvents(),
      aggregate.getVersion() - 1  // 乐观锁：期望版本 = 当前版本 - 1
    );
  }
}
```

---

## Projector（Read Model 构建）

```typescript
// src/infrastructure/order.projector.ts
// Projector 订阅事件，维护为查询优化的 Read Model

class OrderProjector {
  async on(event: DomainEvent): Promise<void> {
    switch (event.eventType) {
      case 'OrderCreated': {
        const p = event.payload as OrderCreated['payload'];
        await prisma.orderReadModel.create({
          data: {
            id: p.orderId,
            userId: p.userId,
            status: 'pending',
            totalAmount: p.totalAmount,
            itemCount: p.items.length,
            createdAt: event.occurredAt,
            updatedAt: event.occurredAt,
          },
        });
        break;
      }

      case 'PaymentReceived': {
        const p = event.payload as PaymentReceived['payload'];
        await prisma.orderReadModel.update({
          where: { id: p.orderId },
          data: { status: 'paid', paidAt: event.occurredAt, updatedAt: event.occurredAt },
        });
        break;
      }

      case 'OrderCancelled': {
        const p = event.payload as OrderCancelled['payload'];
        await prisma.orderReadModel.update({
          where: { id: p.orderId },
          data: {
            status: 'cancelled',
            cancelReason: p.reason,
            updatedAt: event.occurredAt,
          },
        });
        break;
      }
    }
  }

  // 重建所有 Read Model（用于修复投影或添加新字段后重新计算）
  async rebuild(): Promise<void> {
    await prisma.orderReadModel.deleteMany();  // 清空旧数据

    const events = await prisma.event.findMany({
      where: { aggregateType: 'Order' },
      orderBy: [{ aggregateId: 'asc' }, { version: 'asc' }],
    });

    for (const event of events) {
      await this.on(event as unknown as DomainEvent);
    }
  }
}
```

---

## Snapshot（快照优化）

```typescript
// 问题：如果一个订单有 1000 个事件，每次重建都要回放 1000 次，很慢
// 解法：每 N 个事件存一个快照（当前状态），下次从快照开始回放

class SnapshotStore {
  async save(aggregateId: string, state: unknown, version: number): Promise<void> {
    await prisma.snapshot.upsert({
      where: { aggregateId },
      create: { aggregateId, state: state as Prisma.JsonObject, version },
      update: { state: state as Prisma.JsonObject, version },
    });
  }

  async load(aggregateId: string): Promise<{ state: unknown; version: number } | null> {
    const snapshot = await prisma.snapshot.findUnique({ where: { aggregateId } });
    return snapshot ? { state: snapshot.state, version: snapshot.version } : null;
  }
}

// Command Handler 中使用快照
async function loadAggregate(orderId: string): Promise<OrderAggregate> {
  const snapshot = await snapshotStore.load(orderId);

  let events: DomainEvent[];
  let aggregate: OrderAggregate;

  if (snapshot) {
    // 从快照恢复，再回放快照之后的事件
    aggregate = OrderAggregate.fromSnapshot(snapshot.state);
    events = await eventStore.getEvents(orderId, snapshot.version + 1);
  } else {
    events = await eventStore.getEvents(orderId);
    aggregate = new OrderAggregate();
  }

  for (const event of events) {
    aggregate.applyEvent(event);
  }

  // 每 50 个事件存一次快照
  if ((snapshot?.version ?? -1) + 50 <= aggregate.getVersion()) {
    await snapshotStore.save(orderId, aggregate.getState(), aggregate.getVersion());
  }

  return aggregate;
}
```

---

## 面试追问

**Q: CQRS 一定要和 Event Sourcing 一起用吗？**
A: 不一定。CQRS 单独用：读写操作分离到不同的 Service/Repository，写走主库，读走从库/缓存；不引入 Event Sourcing，但能解决读写模型不同的问题。Event Sourcing 单独用：存事件历史，但不分离读写模型（所有查询都回放事件，简单但慢）。两者组合最强大，但复杂度也最高，只在真正需要完整审计、状态回放时引入。

**Q: Event Sourcing 的最大缺点是什么？**
A: ①复杂度高（新增概念：聚合、事件、Projector、Read Model）；②查询能力弱（直接查询当前状态需要 Projection，灵活查询需要多个 Read Model）；③事件 Schema 演进困难（历史事件格式不能轻易改，需要 Upcaster 转换旧格式）；④最终一致性（Command 写入 Event Store 后，Read Model 的更新有延迟）。适用场景：金融交易、审计日志、协作编辑（Undo/Redo）。

**Q: 如何处理 Event Schema 的演进（Upcasting）？**
A: 事件一旦写入就不能修改（不可变），但业务需求变化时字段会改变。解法：版本化事件类型（`OrderCreated_v1`、`OrderCreated_v2`），读取时用 Upcaster 把旧版本转换为最新版本，业务代码只处理最新版本。类似数据库 migration，但操作的是事件流而不是表结构。
