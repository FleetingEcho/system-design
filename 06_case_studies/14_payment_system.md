# 系统设计案例：支付系统

## TL;DR

支付系统处理"钱的转移"——用户下单，从账户 A 扣款，向账户 B 打款。核心难点：**钱绝对不能丢、不能多扣、不能少扣**。这要求比普通系统更严格的幂等性、一致性保证，以及面对网络故障时的正确处理策略。

---

## 需求澄清

**功能需求：**
- 用户向商家付款（扣用户余额，加商家余额）
- 支持第三方支付（支付宝、微信支付、Stripe）
- 支付结果通知商家（Webhook 回调）
- 支付记录查询（用户账单、商家对账）
- 退款

**非功能需求：**
- 精确性：不能多扣、不能少扣、不能重复扣（Exactly-Once）
- 高可用：支付服务不可用时，不能丢失正在处理的订单
- 一致性：扣款和加款必须原子完成（要么都成功，要么都失败）
- 审计：所有资金变动必须有完整记录，可追溯

**规模估算：**
```
DAU: 1000 万活跃付款用户
高峰 QPS（双十一）：10 万笔/秒
平均 QPS：1000 笔/秒
每笔事务：平均 2 次账户操作（扣款 + 加款）

存储（5 年）：
  每笔交易记录约 500 字节
  1000 QPS × 86400 × 365 × 5 年 ≈ 1500 亿条
  → 75 TB（需要分片）
```

---

## 核心挑战：支付的"双花"问题

```
场景：用户余额 100 元，同时发起两笔 80 元的支付

并发读写问题：
  事务 A：读取余额 = 100 → 计算 100-80=20 → 写入余额 20
  事务 B：读取余额 = 100 → 计算 100-80=20 → 写入余额 20

两笔都成功了！用户花了 160 元，但余额只扣了 80 元

解决：数据库行锁 + 乐观锁
  UPDATE accounts SET balance = balance - 80
  WHERE user_id = 1 AND balance >= 80
  → 原子操作，数据库保证只有一个事务成功
```

---

## 系统架构

```
[用户/商家客户端]
    ↓ POST /payments
[API Gateway]（鉴权、限流、幂等键校验）
    ↓
[支付服务]
    ├─ 幂等性校验（Redis + DB）
    ├─ 账户余额检查
    ├─ 写 payment_ledger（资金流水，不可变）
    ├─ 原子扣款/加款（MySQL 事务）
    └─ 发 Kafka 事件（异步通知下游）
         ↓
    [通知服务] → Webhook 回调商家
    [对账服务] → 每日与第三方账单核对
    [风控服务] → 异常交易检测

[第三方支付渠道]（支付宝、Stripe）
    ↓ 异步回调
[支付网关服务]（处理三方回调，更新支付状态）
```

---

## 核心设计一：幂等性（防重复扣款）

支付请求的幂等性是**最重要的保证**：

```
场景：
  用户点了"支付"按钮 → 请求发出 → 网络超时
  用户不知道是否成功 → 再点一次"支付"
  
  没有幂等性：两次扣款！
  有幂等性：第二次请求识别为重复，返回第一次的结果

实现：客户端生成 idempotency_key（UUID），每次请求携带
```

```typescript
// 服务端幂等性处理
async function processPayment(
  idempotencyKey: string,
  params: PaymentParams
): Promise<PaymentResult> {

  // Step 1: 查 Redis（快速路径）
  const cached = await redis.get(`idem:${idempotencyKey}`);
  if (cached) return JSON.parse(cached); // 直接返回上次结果

  // Step 2: 查 DB（防 Redis 宕机）
  const existing = await db.payments.findOne({ idempotencyKey });
  if (existing) {
    await redis.setex(`idem:${idempotencyKey}`, 86400, JSON.stringify(existing));
    return existing;
  }

  // Step 3: 插入"处理中"记录（UNIQUE 约束防并发重复）
  try {
    await db.payments.insert({
      idempotencyKey,
      status: 'processing',
      ...params,
      createdAt: new Date()
    });
  } catch (e) {
    if (isUniqueViolation(e)) {
      // 并发请求，等待另一个请求完成后重新查
      await sleep(100);
      return processPayment(idempotencyKey, params);
    }
    throw e;
  }

  // Step 4: 执行实际扣款
  const result = await executePayment(params);

  // Step 5: 更新状态 + 写入 Redis 缓存
  await db.payments.update({ idempotencyKey }, { status: result.status, ...result });
  await redis.setex(`idem:${idempotencyKey}`, 86400, JSON.stringify(result));

  return result;
}
```

---

## 核心设计二：资金账本（Double-Entry Ledger）

普通系统更新余额字段：`UPDATE accounts SET balance = balance - 100`

支付系统用**复式记账法（Double-Entry Bookkeeping）**，这是银行系统的基础：

```
原则：每笔资金变动都记两条相反的记录，借贷必须平衡

用户向商家支付 100 元：
  借（Debit）：用户账户   -100 元（资产减少）
  贷（Credit）：商家账户  +100 元（负债增加）

账本表（ledger_entries，只能追加，不能修改）：
  id  | payment_id | account_id | amount | type   | created_at
  1   | pay_001    | user_A     | -100   | debit  | 2024-01-01
  2   | pay_001    | merchant_B | +100   | credit | 2024-01-01

账户余额 = SUM(amount) WHERE account_id = ?
（不存储余额字段，实时计算 or 用物化视图/缓存）

优点：
  完整审计轨迹（每笔资金流向都有记录）
  不可篡改（只追加，不修改删除）
  可以重放历史计算任意时间点的余额
  错误可以用反向记账（对冲）纠正，而不是直接修改
```

**账户余额的性能优化：**

```sql
-- 方案一：每次查询实时 SUM（小账户 OK，大账户慢）
SELECT SUM(amount) FROM ledger_entries WHERE account_id = 'user_A';

-- 方案二：物化余额（Balance Snapshot）
-- 单独维护一张 balances 表，每次 ledger 写入时原子更新
BEGIN TRANSACTION;
  INSERT INTO ledger_entries (...) VALUES (...);  -- 写流水
  UPDATE balances SET amount = amount - 100       -- 更新快照
    WHERE account_id = 'user_A' AND amount >= 100; -- 乐观锁
  -- 受影响行数 = 0 → 余额不足，ROLLBACK
COMMIT;

-- 方案三：定期生成 Checkpoint（折中）
-- 每天凌晨计算所有账户余额，存为 checkpoint
-- 查询余额 = checkpoint + SUM(amount WHERE created_at > checkpoint_time)
```

---

## 核心设计三：对接第三方支付

支付宝/Stripe 是异步的，支付结果通过 Webhook 回调：

```
[用户发起支付]

1. 我们的系统 → 调用 Stripe API，创建 PaymentIntent
   POST https://api.stripe.com/v1/payment_intents
   → 返回 { id: "pi_xxx", client_secret: "..." }

2. 本地记录：
   payment_id: "pay_001"
   external_id: "pi_xxx"
   status: "pending"

3. 前端用 client_secret 完成支付（用户输入卡号，Stripe 处理）

[Stripe 异步回调]

4. 支付成功/失败后，Stripe 发 Webhook 到我们的服务器：
   POST https://ourserver.com/webhooks/stripe
   { type: "payment_intent.succeeded", data: { id: "pi_xxx", ... } }

5. 我们的服务：
   a. 验证签名（防伪造：Stripe-Signature Header + 我们的 webhook secret）
   b. 查 payment_id（by external_id = "pi_xxx"）
   c. 幂等处理（同一个 webhook 可能重发多次）
   d. 更新支付状态 → 触发扣款/加款逻辑
   e. 通知商家（再发一个 Webhook）
   f. 立即返回 200（否则 Stripe 会重试）
```

**Webhook 处理的关键点：**

```typescript
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  // 1. 验证签名（必须，防止伪造请求）
  let event;
  try {
    event = stripe.webhooks.constructEvent(
      req.body,                                    // 原始 body（不能被 JSON.parse 处理过）
      req.headers['stripe-signature'],
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (e) {
    return res.status(400).send(`Webhook Error: ${e.message}`);
  }

  // 2. 立即返回 200（处理逻辑放异步，不阻塞 Stripe 的重试计时器）
  res.status(200).send();

  // 3. 异步处理（发 Kafka 消息，Worker 处理）
  await kafka.produce('stripe_events', {
    eventId: event.id,   // 用作幂等键，防止 Stripe 重发同一事件被重复处理
    type: event.type,
    data: event.data
  });
});
```

---

## 核心设计四：对账（Reconciliation）

支付系统必须能发现和修复数据不一致：

```
对账类型：

内部对账（每小时）：
  所有账户余额之和应该 = 初始总资金 + 充值 - 提现
  发现不平衡 → 立刻告警

外部对账（每天）：
  我们系统里的交易记录 vs 第三方（Stripe）的账单
  
  Stripe 每天发对账文件（CSV）：
    pay_001, success, $100, 2024-01-01
    pay_002, success, $50, 2024-01-01
    ...
  
  我们系统的记录：
    pay_001, success, $100 ✓
    pay_002, failed, $50  ← 不一致！Stripe 显示成功，我们显示失败
    
  差异处理：
    Stripe 成功，我们失败 → 可能是 Webhook 漏收，需要补处理（补扣款）
    我们成功，Stripe 失败 → 可能是重复扣款，需要退款
    金额不匹配 → 立刻人工介入
```

---

## 退款流程

```
退款不是简单地"反向操作"，需要完整的状态追踪：

退款流程：
  1. 商家/用户发起退款申请
  2. 创建 refund 记录（status=pending）
  3. 调用 Stripe Refund API：
     POST https://api.stripe.com/v1/refunds
     { payment_intent: "pi_xxx", amount: 100 }
  4. Stripe 异步处理（通常 5-10 个工作日）
  5. Stripe Webhook 回调退款结果
  6. 更新退款状态，写 ledger（+100 给用户，-100 从商家）

账本记录（Double-Entry）：
  借：商家账户  -100  （退回给用户）
  贷：用户账户  +100  （退款收到）
```

---

## 数据模型

```sql
-- 支付记录
CREATE TABLE payments (
  id              BIGINT PRIMARY KEY,     -- Snowflake ID
  idempotency_key VARCHAR(100) UNIQUE,
  user_id         BIGINT NOT NULL,
  merchant_id     BIGINT NOT NULL,
  amount          DECIMAL(18, 2) NOT NULL, -- 使用 DECIMAL，不用 FLOAT（浮点精度问题）
  currency        CHAR(3) NOT NULL,        -- 'USD', 'CNY'
  status          ENUM('pending','processing','succeeded','failed','refunded'),
  external_id     VARCHAR(200),            -- Stripe/支付宝的交易 ID
  description     VARCHAR(500),
  metadata        JSON,
  created_at      TIMESTAMP NOT NULL,
  updated_at      TIMESTAMP NOT NULL,

  INDEX idx_user (user_id, created_at),
  INDEX idx_merchant (merchant_id, created_at),
  INDEX idx_external (external_id)
);

-- 账本流水（只追加，不修改）
CREATE TABLE ledger_entries (
  id          BIGINT PRIMARY KEY,
  payment_id  BIGINT NOT NULL,
  account_id  BIGINT NOT NULL,
  amount      DECIMAL(18, 2) NOT NULL,  -- 负数=扣款，正数=加款
  entry_type  ENUM('debit','credit'),
  created_at  TIMESTAMP NOT NULL,

  INDEX idx_account_time (account_id, created_at)
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED;

-- 账户余额快照（物化视图，提升查询性能）
CREATE TABLE account_balances (
  account_id  BIGINT PRIMARY KEY,
  balance     DECIMAL(18, 2) NOT NULL DEFAULT 0,
  version     BIGINT NOT NULL DEFAULT 0, -- 乐观锁版本号
  updated_at  TIMESTAMP NOT NULL
);
```

---

## Node.js 类比

如果你写过电商结账，这就是它加了金融级保证的版本：

```typescript
// 普通电商扣款（不够健壮）
async function checkout(userId: string, amount: number) {
  const user = await db.users.findById(userId);
  if (user.balance < amount) throw new Error('Insufficient balance');
  await db.users.update(userId, { balance: user.balance - amount }); // 非原子！
}

// 支付系统（幂等 + 原子 + 审计）
async function checkout(idempotencyKey: string, userId: string, amount: number) {
  return await db.transaction(async (trx) => {
    // 原子扣款（行锁 + 余额检查）
    const updated = await trx.raw(`
      UPDATE account_balances
      SET balance = balance - ?, version = version + 1, updated_at = NOW()
      WHERE account_id = ? AND balance >= ?
    `, [amount, userId, amount]);

    if (updated.affectedRows === 0) throw new Error('Insufficient balance');

    // 写不可变账本
    await trx('ledger_entries').insert({
      paymentId, accountId: userId, amount: -amount, entryType: 'debit'
    });

    return { success: true };
  });
}
```

---

## 常见陷阱

1. **用浮点数存金额**：`0.1 + 0.2 = 0.30000000000000004`，浮点精度误差在金融场景是灾难。必须用 `DECIMAL(18,2)` 或以分为单位存整数（100 元存为 10000 分）

2. **直接 UPDATE 余额而不检查**：`UPDATE SET balance = balance - 100` 没有 `WHERE balance >= 100`，余额可能变成负数。必须在 WHERE 里加余额检查，通过 `affectedRows == 0` 来判断余额不足

3. **Webhook 没有幂等处理**：第三方支付可能因为超时重发同一个 Webhook 多次，必须用 event_id 做幂等去重，否则会重复扣款或重复通知

4. **退款成功但账本没有对应记录**：退款也要写 ledger，补全资金流向。缺少退款的 ledger 会导致对账不平

5. **跨货币没有汇率锁定**：用户下单时显示 100 CNY，支付时汇率变了变成 99 CNY，差异谁承担？支付时必须锁定汇率（记录下单时的汇率），不能用实时汇率重新计算

---

## 面试 Q&A

### 简单

**Q: 为什么支付金额不能用浮点数（float/double）存储？**

A: 浮点数在二进制表示中无法精确表示大多数十进制小数（如 0.1 在 IEEE 754 中是无限循环小数），计算会产生微小误差。对普通应用无感，但在金融场景中，几千万笔交易累积下来误差不可接受。正确做法：用 `DECIMAL(18,2)` 类型（数据库精确十进制），或在应用层以"分"为单位存整数（100.00 元 → 10000 分），避免小数运算。

**Q: 什么是幂等性，支付系统为什么必须保证幂等？**

A: 幂等性是指同一操作执行多次的结果与执行一次相同。支付场景里，用户网络超时后重试、Webhook 重发、内部服务重试，都可能导致同一请求被处理多次。没有幂等性，会出现重复扣款（用户多付钱）或重复入账（商家多收钱）。实现方式：客户端生成唯一 idempotency_key，服务器用 UNIQUE 约束防止重复处理，相同 key 的请求直接返回上次结果。

---

### 中等

**Q: 用户同时发起两笔超出余额的支付，如何防止双花（double spending）？**

A: 用数据库原子操作加行锁：

```sql
UPDATE account_balances
SET balance = balance - 100, version = version + 1
WHERE account_id = ? AND balance >= 100
```

两个并发事务竞争同一行的行锁，只有一个能先获取锁执行扣款，另一个等待。第一个成功后余额变为 0，第二个获取锁后检查 `balance >= 100` 失败，`affectedRows = 0`，返回余额不足错误。乐观锁（version 字段）可以额外检测并发冲突，进一步保证数据正确性。

**Q: 如何设计每日对账系统，发现并修复支付数据不一致？**

A: 每天凌晨执行两种对账：1）内部对账——从 ledger_entries SUM 所有账户的资金变动，验证借贷平衡（所有借方总额 = 贷方总额），发现不平衡立刻告警；2）外部对账——下载第三方（Stripe/支付宝）的账单文件（CSV），与我们数据库的 payments 表逐条比对，发现差异分类处理：三方显示成功但我们是 pending → 补处理（可能是 Webhook 漏收）；我们成功但三方失败 → 触发退款流程；金额不匹配 → 人工介入。所有差异记录到 reconciliation_issues 表，附上自动处理结果或人工处理备注。

---

### 困难

**Q: 设计一个支持双十一 10 万笔/秒支付峰值的系统，同时保证资金安全。**

A: 分层架构应对峰值：

**流量层：** API Gateway 做幂等键校验（Redis 快速拦截重复请求），减少到达支付服务的重复流量。限流：每用户每秒最多 1 笔支付，防止脚本刷单。

**异步化：** 支付请求不同步等待第三方（Stripe 调用可能 200ms+），而是写入本地数据库（status=pending）立即返回支付页面，后台 Worker 异步调用 Stripe，通过 Webhook 更新状态。这样系统吞吐量由本地 DB 写入速度决定，而非第三方 API 延迟。

**数据库分片：** 10 万 QPS × 2 次账户操作 = 20 万 TPS，单机 MySQL 极限约 5 万 TPS。按 account_id 哈希分 8 个 MySQL 分片（每片 2.5 万 TPS），同一用户的扣款/加款路由到同一分片，保证跨分片事务降到最低（不同用户的转账涉及 2 个分片，用 Saga 模式处理）。

**对账兜底：** 即使实时处理出错，每小时对账能发现并修复差异。关键路径：Kafka 保存所有支付事件（保留 7 天），任何异常都可以从 Kafka 重放修复，账本（ledger）是最终真相。

**监控告警：** 核心指标：支付成功率（< 99.9% 立即告警）、P99 延迟、账户余额负数（绝对不允许）、对账差异率。支付成功率是最重要的业务指标，比技术指标更直接反映用户体验。

---

## 关联文档

- [../04_distributed/02_distributed_tx.md](../04_distributed/02_distributed_tx.md) — 分布式事务（Saga 模式处理跨分片转账）
- [../04_distributed/04_fault_tolerance.md](../04_distributed/04_fault_tolerance.md) — 幂等性实现细节
- [../01_fundamentals/05_auth_security.md](../01_fundamentals/05_auth_security.md) — API 安全（支付接口鉴权）
