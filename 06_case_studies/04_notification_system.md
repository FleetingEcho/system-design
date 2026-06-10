# 系统设计案例：通知系统

## TL;DR

通知系统把"某件事发生了"这个信息，通过多个渠道（App 推送、短信、邮件、站内信）告知用户。核心难点：**保证重要通知不丢失**、**不重复发送**、**跨渠道路由**。

---

## 需求澄清

**支持的通知类型：**
- App Push Notification（iOS APNs / Android FCM）
- 短信（SMS）
- 邮件（Email）
- 站内信（In-App Notification）

**功能需求：**
- 发送通知给单个用户或用户群组
- 用户可以设置偏好（哪些渠道接收哪类通知、Do Not Disturb 时间段）
- 通知有优先级（账号安全类通知必须发，促销类通知可以延迟或跳过）

**非功能需求：**
- 重要通知（安全、交易）不能丢失：at-least-once 语义
- 不能重复发送（At-exactly-once 在端到端很难，但要尽力）
- 高吞吐：如大促销活动给 1 亿用户推送

**规模估算：**
```
日发送量：
  App Push: 1000 万条/天 ≈ 120 条/秒
  SMS: 100 万条/天 ≈ 12 条/秒
  Email: 500 万条/天 ≈ 60 条/秒

峰值（大促销推送）：1 亿条在 1 小时内 ≈ 28,000 条/秒
→ 需要消息队列削峰
```

---

## 系统架构

```
触发源
  ├─ 业务服务（下单成功、登录告警）
  ├─ 营销系统（定时推送活动）
  └─ 管理后台（手动发送）
           ↓
    [通知 API 服务]
    ├─ 验证用户偏好
    ├─ 路由到对应渠道
    └─ 写入消息队列
           ↓
    [Kafka Topic 分渠道]
    ├─ topic: push_notifications
    ├─ topic: sms_notifications
    ├─ topic: email_notifications
    └─ topic: inapp_notifications
           ↓
    [各渠道 Worker 集群]
    ├─ Push Worker → APNs / FCM
    ├─ SMS Worker  → 短信服务商（阿里云、Twilio）
    ├─ Email Worker → 邮件服务商（SendGrid、SES）
    └─ InApp Worker → 写入通知数据库
           ↓
    [通知记录数据库]（写入发送记录，用于去重）
```

---

## 核心设计：数据模型

```sql
-- 通知模板
CREATE TABLE notification_templates (
  id          BIGINT PRIMARY KEY,
  type        VARCHAR(50),        -- 'order_success', 'login_alert', 'promotion'
  title_tpl   VARCHAR(200),       -- "您的订单 {order_id} 已发货"
  body_tpl    TEXT,
  priority    TINYINT,            -- 1=critical, 2=high, 3=normal, 4=low
  channels    JSON                -- ['push', 'email'] 默认发送渠道
);

-- 用户通知偏好
CREATE TABLE user_notification_prefs (
  user_id     BIGINT,
  channel     VARCHAR(20),       -- 'push', 'sms', 'email', 'inapp'
  notif_type  VARCHAR(50),       -- 'order_success', 'promotion', ALL='*'
  enabled     BOOLEAN,
  quiet_start TIME,               -- Do Not Disturb 开始
  quiet_end   TIME,
  device_token VARCHAR(500),      -- Push token（FCM/APNs）
  phone       VARCHAR(20),
  email       VARCHAR(200),
  PRIMARY KEY (user_id, channel, notif_type)
);

-- 发送记录（用于去重和状态跟踪）
CREATE TABLE notification_logs (
  id              BIGINT PRIMARY KEY,
  idempotency_key VARCHAR(100) UNIQUE,   -- 去重用
  user_id         BIGINT,
  channel         VARCHAR(20),
  type            VARCHAR(50),
  status          ENUM('pending','sent','failed','delivered'),
  payload         JSON,                  -- 发送内容快照
  created_at      TIMESTAMP,
  sent_at         TIMESTAMP,
  error_msg       TEXT,

  INDEX idx_user_id (user_id),
  INDEX idx_idempotency (idempotency_key)
);
```

---

## 核心设计：幂等性（防重复发送）

**问题来源：**
```
1. Kafka Consumer 崩溃后重启，同一条消息可能被重新消费
2. 发送服务自动重试（网络超时时发送可能已成功）
3. 用户点击"重新发送"按钮
```

**解决方案：idempotency_key**

```typescript
// 生成幂等键的规则（对相同事件总是生成相同的键）
function buildIdempotencyKey(params: {
  userId: string;
  notifType: string;
  channel: string;
  eventId: string;       // 触发事件的 ID（如订单 ID）
}): string {
  return `${params.notifType}:${params.channel}:${params.userId}:${params.eventId}`;
  // 例：order_success:push:user_123:order_456
}

async function sendWithIdempotency(
  idempotencyKey: string,
  sendFn: () => Promise<void>
): Promise<void> {
  // 检查是否已发送
  const existing = await db.query(
    'SELECT status FROM notification_logs WHERE idempotency_key = ?',
    [idempotencyKey]
  );

  if (existing?.status === 'sent') {
    console.log('Duplicate notification, skipping:', idempotencyKey);
    return;
  }

  // 插入 pending 记录（UNIQUE 约束防止并发重复）
  try {
    await db.query(
      'INSERT INTO notification_logs (idempotency_key, status, ...) VALUES (?, "pending", ...)',
      [idempotencyKey, ...]
    );
  } catch (e) {
    if (isUniqueViolation(e)) {
      // 另一个进程正在发送，跳过
      return;
    }
    throw e;
  }

  // 发送
  try {
    await sendFn();
    await db.query(
      'UPDATE notification_logs SET status = "sent", sent_at = NOW() WHERE idempotency_key = ?',
      [idempotencyKey]
    );
  } catch (e) {
    await db.query(
      'UPDATE notification_logs SET status = "failed", error_msg = ? WHERE idempotency_key = ?',
      [e.message, idempotencyKey]
    );
    throw e;
  }
}
```

---

## 核心设计：用户偏好路由

```typescript
async function routeNotification(params: {
  userId: string;
  notifType: string;
  priority: number;
  payload: object;
}): Promise<string[]> {  // 返回实际要发送的渠道列表

  const prefs = await getUserPrefs(params.userId);
  const template = await getTemplate(params.notifType);

  const channels: string[] = [];

  for (const channel of template.defaultChannels) {
    // 1. 用户是否开启此渠道此类通知
    const pref = prefs.find(p => p.channel === channel &&
      (p.notifType === params.notifType || p.notifType === '*'));

    if (!pref?.enabled) continue;

    // 2. 是否在勿扰时间
    if (params.priority < 2 && isInQuietHours(pref.quietStart, pref.quietEnd)) {
      // 低优先级通知在勿扰时间内延迟
      await scheduleForLater(params, channel, pref.quietEnd);
      continue;
    }

    // 3. 是否有有效的设备信息
    if (channel === 'push' && !pref.deviceToken) continue;
    if (channel === 'sms' && !pref.phone) continue;
    if (channel === 'email' && !pref.email) continue;

    channels.push(channel);
  }

  // Critical 通知至少保证一个渠道
  if (params.priority === 1 && channels.length === 0) {
    channels.push('inapp'); // 站内信作为保底
  }

  return channels;
}
```

---

## 三方服务对接

不同渠道的三方 SDK 差异大，用适配器模式统一接口：

```typescript
interface NotificationSender {
  send(to: string, title: string, body: string, data?: object): Promise<SendResult>;
}

class FCMSender implements NotificationSender {
  async send(deviceToken: string, title: string, body: string): Promise<SendResult> {
    const response = await fcmClient.send({
      token: deviceToken,
      notification: { title, body },
    });
    return { success: true, messageId: response };
  }
}

class APNsSender implements NotificationSender {
  async send(deviceToken: string, title: string, body: string): Promise<SendResult> {
    const notification = new Notification();
    notification.title = title;
    notification.body = body;
    const response = await apnsClient.send(notification, deviceToken);
    return { success: !response.failed.length };
  }
}

class TwilioSmsSender implements NotificationSender {
  async send(phone: string, _title: string, body: string): Promise<SendResult> {
    const message = await twilioClient.messages.create({
      body,
      to: phone,
      from: process.env.TWILIO_FROM_NUMBER
    });
    return { success: true, messageId: message.sid };
  }
}
```

---

## 失败重试策略

```
Push 通知失败原因：
  - 设备 token 失效（用户卸载 App）→ 不重试，标记 token 无效
  - 三方服务临时不可用（5xx）→ 指数退避重试，最多 3 次
  - 网络超时 → 重试

重试队列：
  Worker 消费失败 → 发送到 retry topic
  retry topic 的消息有 delay（5s, 30s, 5min）
  超过 3 次重试 → 发送到 dead_letter_queue（DLQ）
  DLQ 告警 → 人工处理或记录失败

Kafka Consumer 配置：
  - enable.auto.commit = false（手动提交 offset）
  - 发送成功后才 commit，保证 at-least-once
```

---

## Node.js 类比

如果你写过邮件发送功能，这就是它的大规模版本：

```typescript
// 简单版：直接发邮件
await nodemailer.sendMail({ to, subject, html });

// 大规模版：先写队列，Worker 异步处理
await kafka.produce('email_notifications', {
  userId,
  to: userEmail,
  subject,
  html,
  idempotencyKey: `${type}:${userId}:${eventId}`
});

// Worker 进程（独立部署）
kafka.consume('email_notifications', async (msg) => {
  await sendWithIdempotency(msg.idempotencyKey, async () => {
    await sesClient.sendEmail({ ... });
  });
});
```

---

## 常见陷阱

1. **直接在业务逻辑里同步发通知**：下单成功时直接调用 Push SDK，SDK 超时会导致下单接口变慢。必须异步解耦（发消息到 Kafka，Worker 处理）

2. **Push Token 失效未清理**：用户删除 App 后，旧 token 会失效，FCM/APNs 会返回特定错误码。必须捕获这些错误码，从数据库里删除无效 token，否则每次都白发一条请求

3. **大规模推送时的 Kafka 背压**：给 1 亿用户推送时，消息生产速度远超消费速度，Kafka 消息堆积。需要动态扩展 Worker 数量，或者分批推送（每批 100 万，分 100 次批次）

4. **勿扰时间边界的竞争条件**：用户是 UTC+8 时区，勿扰时间 22:00-08:00，但服务器是 UTC。需要存储用户时区，计算时转换。边界时刻（比如刚好 22:00:00 时多条通知并发）要用数据库事务防止重复调度

---

## 面试 Q&A

### 简单

**Q: 为什么通知系统要用消息队列，而不是直接调用 Push SDK？**

A: 三个原因：1）解耦——业务服务不需要关心如何发 Push，只需要发消息到队列；2）削峰——大促销时可能瞬间产生几千万条通知，消息队列可以缓冲，Worker 按能力消费；3）可靠性——如果 Push SDK 暂时不可用，消息留在队列里，恢复后继续消费，不会丢失。

**Q: 如何防止同一条通知发两次？**

A: 每条通知生成一个幂等键（`notif_type:channel:user_id:event_id`），发送前先插入通知日志表（有 UNIQUE 约束），插入成功才发送，发送后更新状态。并发时只有一个进程能插入成功，另一个遇到 Unique Violation 直接跳过。

---

### 中等

**Q: 如何实现"用户设置勿扰时间，勿扰期间的通知延迟到结束后发送"？**

A: 路由时检查当前时间是否在勿扰时间段内（注意用户时区）：
- 如果是：不发送，把通知写入"延迟发送表"，记录 `scheduled_at = 勿扰结束时间`
- 有个定时任务（每分钟执行）扫描"延迟发送表"，把到时间的通知取出来重新路由发送
- Critical 优先级的通知（如账户被盗告警）绕过勿扰限制，立即发送

**Q: 如何处理 Push Token 失效？**

A: FCM 和 APNs 在 token 失效时会返回特定错误：FCM 返回 `UNREGISTERED` 错误码，APNs 返回 410 状态码。Worker 需要捕获这些特定错误，在数据库中标记该 token 为失效（`is_valid = false`），不再向该 token 发送通知。同时，用户重新打开 App 时，前端会从 FCM/APNs 获取新 token 并上报给后端，更新数据库。

---

### 困难

**Q: 设计一个能在 1 小时内给 1 亿用户发送促销推送的系统。**

A: 这是"大规模 Fanout"的典型案例：

**数据准备（提前）：** 促销开始前，运营在管理后台创建推送任务，指定目标用户群（全量用户 or 按标签筛选）和发送时间。系统提前生成用户 ID 列表，存入 S3（分片文件，每份 100 万用户）。

**任务分发：** 到发送时间，任务调度服务读取 S3 的用户列表文件，按分片写入 Kafka（`push_notifications` topic，100 个分区）。每条 Kafka 消息是一个用户的推送请求，每秒写入速率 ≈ 1 亿 / 3600 ≈ 28,000 条/秒。

**Worker 集群：** 100 个 Push Worker 消费 Kafka（每个 Worker 处理一个分区）。每个 Worker 调用 FCM/APNs 批量发送 API（FCM 支持 `sendAll`，每次最多 500 条）。单个 Worker 吞吐约 500 × 每秒 10 批 = 5000 条/秒，100 个 Worker 合计 50 万条/秒，1 亿条 3 分钟内发完（远超 1 小时目标）。

**限流保护：** FCM/APNs 有 API 限额，需要在 Worker 侧做请求速率控制。配置每个 Worker 的并发请求数（如 20 个并发），防止超出服务商配额。

**监控：** 跟踪发送进度（已发 / 总量）、成功率（FCM 无效 token 比例）、延迟分布，大盘一出问题立刻告警。

---

## 关联文档

- [../03_communication/02_async.md](../03_communication/02_async.md) — Kafka 消息队列
- [../04_distributed/04_fault_tolerance.md](../04_distributed/04_fault_tolerance.md) — 幂等性 + 指数退避重试
- [../05_methodology/reference/03_patterns.md](../05_methodology/reference/03_patterns.md) — Fanout 模式
