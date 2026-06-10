# 系统设计案例：广告点击统计系统

## TL;DR

记录用户对广告的点击/曝光事件，实时聚合后给广告主看"我的广告今天被点击了多少次"。核心难点：**极高写入量**（每秒百万级事件）+ **近实时聚合**（分钟级延迟可接受）+ **防作弊**（同一用户短时间内多次点击只算一次）。

---

## 需求澄清

**功能需求：**
- 记录广告点击事件（click）和曝光事件（impression）
- 支持按维度查询聚合数据：
  - 某广告过去 1 小时的点击数
  - 某广告主今天所有广告的总点击数
  - 按地区、设备类型分组
- 数据延迟：允许 1-5 分钟（近实时，不需要毫秒级）
- 数据保留：原始事件保留 90 天，聚合结果永久保留

**非功能需求：**
- 高吞吐写入：每秒 100 万次点击事件
- 准确性：同一用户 1 小时内点击同一广告只计 1 次（防刷）
- 高可用：数据不能丢失（广告主按点击付费，数据直接关系到账单）

**规模估算：**
```
每秒事件量：100 万次（写入极高）
每条事件大小：~200 字节（ad_id, user_id, timestamp, ip, device, geo）
写入带宽：100 万 × 200B = 200 MB/s

聚合维度：
  广告数量：1000 万个
  每个广告每分钟一个聚合桶
  存储：1000 万 × 60 × 24 × 90天 × 50B ≈ 65 TB（聚合结果）
  
原始事件存储（90天）：
  200 MB/s × 86400 × 90 ≈ 1.5 PB（需要压缩，实际约 300 TB）
```

---

## 系统架构

```
[用户浏览器/App]
    ↓ 点击广告（HTTP/像素请求）
[事件收集服务]（无状态，水平扩展）
    ↓ 立即写入（< 1ms 返回客户端）
[Kafka]（原始事件 Topic: ad_events）
    ↓
    ├─────────────────────────────────────────┐
[流处理（Flink）]                      [批处理（Spark）]
  近实时聚合                               离线精确计算
  延迟：1-5 分钟                           延迟：T+1（次日）
  写入 Redis（实时看板）               写入 ClickHouse（历史分析）
    ↓                                         ↓
[查询服务]←──────────────────────────────────→
    ↓
[广告主 Dashboard]
```

---

## 核心设计一：事件收集

写入必须极快，不能因为处理逻辑阻塞客户端：

```typescript
// 事件收集服务：极简，只做三件事：验证、写 Kafka、返回 200
app.post('/track/click', async (req, res) => {
  const { adId, userId, pageUrl } = req.body;
  const ip = req.ip;
  const userAgent = req.headers['user-agent'];

  // 1. 基础验证（< 0.1ms）
  if (!adId || !userId) return res.status(400).send();

  // 2. 写 Kafka（异步，不等 Ack）
  // acks: 1 = 只等 Leader 确认（吞吐高，极小概率丢数据）
  // acks: all = 等所有副本确认（不丢数据，吞吐稍低）
  // 广告数据直接关系账单，用 acks=all
  await kafka.produce('ad_events', {
    type: 'click',
    adId,
    userId,
    ip,
    userAgent,
    geo: geoIpLookup(ip),   // 毫秒级 IP 地理位置查询（本地 MaxMind DB）
    timestamp: Date.now()
  }, { key: adId }); // Partition by adId，保证同广告事件有序

  // 3. 立即返回
  res.status(200).send();
  // 也可以返回 1x1 像素图片（像素追踪方式）
});
```

**为什么不直接写数据库：** 数据库单机 10-50 万 QPS，100 万 QPS 写入需要 20+ 台分片数据库，运维复杂。Kafka 单 Broker 支持百万消息/秒写入，天然适合高吞吐写入场景。

---

## 核心设计二：防作弊去重

同一用户在 1 小时内点击同一广告，只计 1 次有效点击：

```
方案一：Redis Bitmap（极省内存）

用 Bitmap 记录"user X 今天点了广告 Y 没有"：
Key: "click_dedup:{adId}:{1小时窗口}"
Bit: userId 对应的 bit 位

设置：SETBIT click_dedup:ad123:20240101_14 userId 1
检查：GETBIT click_dedup:ad123:20240101_14 userId

内存估算：
  1 亿用户 → 1 亿 bit = 12.5 MB（每个广告每小时只需 12.5 MB！）
  1000 万个广告 × 24小时 × 12.5 MB = 太多了（需要只缓存活跃广告）

方案二：布隆过滤器（Bloom Filter，内存更小）

用布隆过滤器记录 "adId:userId:hour" 的组合是否见过：
  写入：bloomFilter.add(`${adId}:${userId}:${hour}`)
  检查：bloomFilter.has(`${adId}:${userId}:${hour}`)

假阳性率：1%（即 1% 的有效点击被误判为重复，可以接受）
内存：每百万条记录约 1.2 MB（远小于 Bitmap）

Redis 4.0+ 有原生布隆过滤器模块（RedisBloom）：
  BF.ADD click_dedup {adId}:{userId}:{hour}
  BF.EXISTS click_dedup {adId}:{userId}:{hour}

Flink 中集成去重：
  每条事件先查布隆过滤器
  → 已存在（可能重复）→ 丢弃
  → 不存在（新点击）→ 写入布隆过滤器 + 计入聚合
```

---

## 核心设计三：流处理聚合（Flink）

```
Flink 消费 Kafka ad_events Topic：

1. 解析事件，提取维度：
   { adId, userId, geo, device, timestamp }

2. 去重（布隆过滤器）

3. 开窗口聚合（Tumbling Window，滚动窗口）：
   每 1 分钟一个窗口
   在窗口内，按 (adId, geo, device) 分组，COUNT 点击数

4. 窗口关闭时，写出聚合结果：
   写入 Redis（实时 Dashboard 用）：
     HINCRBY ad_stats:{adId}:{minute_bucket} clicks count
   写入 ClickHouse（长期存储）：
     INSERT INTO ad_clicks_1min (ad_id, minute, geo, device, clicks) VALUES (...)

窗口类型：
  滚动窗口（Tumbling Window）：
    每 1 分钟不重叠的独立窗口
    [0:00, 1:00) [1:00, 2:00) [2:00, 3:00)
    适合：每分钟的精确统计
    
  滑动窗口（Sliding Window）：
    "过去 1 小时"这类查询需要滑动窗口
    每 1 分钟滑动一次，窗口大小 60 分钟
    代价更大（60 倍的窗口数量）
    优化：查 ClickHouse 时 SUM 过去 60 个 1 分钟桶
```

---

## 核心设计四：查询层

```sql
-- ClickHouse 表设计（列式存储，OLAP 优化）
CREATE TABLE ad_clicks_1min (
  ad_id       UInt64,
  minute      DateTime,          -- 分钟级时间桶（精确到分钟）
  geo         LowCardinality(String),  -- 'US', 'CN', 'EU'...
  device      LowCardinality(String),  -- 'mobile', 'desktop'
  clicks      UInt32,
  impressions UInt32,
  
  PRIMARY KEY (ad_id, minute)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(minute)    -- 按月分区，便于冷数据归档
ORDER BY (ad_id, minute);

-- 查询：某广告过去 1 小时的点击数
SELECT
  toStartOfMinute(minute) AS t,
  sum(clicks) AS total_clicks,
  sum(impressions) AS total_impressions
FROM ad_clicks_1min
WHERE ad_id = 12345
  AND minute >= now() - INTERVAL 1 HOUR
GROUP BY t
ORDER BY t;

-- ClickHouse 查询这个的速度：< 100ms（列式存储，只读 3 列）
```

---

## Lambda 架构 vs Kappa 架构

这道题天然引出架构范式的讨论：

### Lambda 架构（流 + 批 双路）

```
原始事件（Kafka）
    ├─ 批处理层（Spark）：每天处理完整数据，结果精确
    │    → 批处理结果（ClickHouse）
    └─ 流处理层（Flink）：实时处理，结果近似
         → 流处理结果（Redis）

查询时合并两层结果：
  近期数据（最近 1 天）→ 流处理层（快）
  历史数据（超过 1 天）→ 批处理层（精确）
```

**优点：** 批处理兜底，即使流处理有 Bug，批处理每天重算一次修正
**缺点：** 维护两套代码（流处理逻辑和批处理逻辑），复杂

### Kappa 架构（只用流处理）

```
原始事件（Kafka，保留 90 天）
    └─ 流处理层（Flink）
         → 实时聚合结果（ClickHouse）

如果发现 Bug 需要重新计算：
  从 Kafka 头部重放原始事件
  用修复后的流处理逻辑重新聚合
```

**优点：** 只维护一套代码
**缺点：** 重放计算量大，依赖 Kafka 保留足够久的历史数据

**面试中的推荐选择：** Kappa 架构，更简洁，Kafka 保留 90 天原始数据作为"源头"。

---

## 数据准确性保证

```
问题：Flink 节点崩溃后重启，Kafka 消息从上次 Checkpoint 重放
→ 某些事件被计算两次 → 聚合结果偏高

解决：Flink Exactly-Once 语义

Flink 支持 Exactly-Once Sink：
  每个 Checkpoint 间隔（如 1 分钟），Flink 保存处理进度
  写出到 ClickHouse 时，用 2PC：
    预提交：写到临时表
    Checkpoint 成功 → 提交：从临时表合并到正式表
    Checkpoint 失败 → 回滚：清空临时表

端到端 Exactly-Once：
  Kafka（at-least-once 生产）
  + Flink（Exactly-Once 处理 + 去重）
  + ClickHouse（2PC 写入）
  = 最终 Exactly-Once 效果
```

---

## Node.js 类比

如果你写过 Google Analytics 类似的埋点，这就是它背后的大规模版本：

```typescript
// 前端埋点：用户点击广告
document.querySelector('.ad').addEventListener('click', () => {
  // 像素请求（不阻塞页面）
  new Image().src = `/track/click?ad_id=123&uid=${userId}&t=${Date.now()}`;
  // 或者 navigator.sendBeacon（更可靠）
  navigator.sendBeacon('/track/click', JSON.stringify({ adId: 123, userId }));
});

// 后端统计查询（简单聚合版，适合小规模）
app.get('/stats/:adId', async (req, res) => {
  const clicks = await clickhouse.query(`
    SELECT minute, sum(clicks) as cnt
    FROM ad_clicks_1min
    WHERE ad_id = ${req.params.adId}
      AND minute >= now() - INTERVAL 24 HOUR
    GROUP BY minute
    ORDER BY minute
  `);
  res.json(clicks);
});
```

---

## 常见陷阱

1. **直接写 OLTP 数据库**：100 万 QPS 绕过 Kafka 直写 MySQL/PostgreSQL，单机扛不住，分片又复杂。Kafka 是"写入缓冲区"，必不可少

2. **聚合精度 vs 延迟的取舍**：实时聚合（Flink）因为窗口水印（Watermark）的延迟策略，可能丢弃迟到事件（网络延迟导致事件晚到）。需要设置合理的 allowed lateness（允许迟到多久），或者通过批处理层修正

3. **热点广告的 Flink 算子倾斜**：某个超级热门广告（如双十一主广告）的事件占所有事件的 30%，导致这个广告所在的 Flink partition 处理极慢（数据倾斜）。解决：在 adId 后面加随机后缀拆分（`adId:rand(0-9)`），聚合后再合并

4. **时区问题**：聚合"今天"的数据，"今天"是哪个时区？广告主在不同时区，需要在聚合时存储 UTC，查询时按广告主时区转换

---

## 面试 Q&A

### 简单

**Q: 为什么事件要先写 Kafka 而不是直接写数据库？**

A: 100 万 QPS 的写入速度，任何 OLTP 数据库都撑不住（MySQL 单机上限约 5 万 QPS 写）。Kafka 是面向磁盘的顺序写入，单 Broker 支持百万消息/秒，是天然的"写入缓冲区"。同时 Kafka 持久化保留原始数据，流处理出 Bug 可以重放修正，数据库没有这个能力。

**Q: 什么是 Lambda 架构？**

A: 把数据处理分成两条路：流处理层（低延迟但可能不精确）和批处理层（高延迟但精确）。查询时合并两层结果——近期数据用流处理层（快），历史数据用批处理层（准）。优点是批处理兜底保证准确性，缺点是要维护两套处理逻辑。现在更推荐 Kappa 架构（只用流处理）。

---

### 中等

**Q: 如何防止用户刷广告点击（短时间内多次点击）？**

A: 两道防线：1）事件收集层：同一 IP 每秒超过 N 次点击直接丢弃（IP 级别的速率限制，在 API Gateway 做）；2）流处理层：用 Redis 布隆过滤器记录"adId:userId:小时"的组合，同一组合在 1 小时内只计 1 次有效点击。布隆过滤器有约 1% 的假阳性率（正常点击被误判为重复），这个精度对广告系统是可以接受的。

**Q: 如果 Flink 节点崩溃重启，会不会有数据重复计算？**

A: 不用 Exactly-Once 的话会。Flink 通过 Checkpoint 机制解决：每隔固定时间把处理进度（Offset + 窗口状态）持久化到 HDFS/S3，崩溃重启后从最近的 Checkpoint 恢复。加上对 Sink 的 2PC（两阶段提交），可以实现端到端的 Exactly-Once 语义，保证数据不重不漏。

---

### 困难

**Q: 设计一个支持"任意时间范围查询"的广告统计系统（如查某广告过去 7 天每小时的点击数）。**

A: 分层聚合（Data Cube 思想）：

**原始层：** Flink 每 1 分钟产生一个聚合桶，写入 ClickHouse（`ad_clicks_1min` 表）。

**查询层：** ClickHouse 的列式存储非常适合 SUM 聚合查询，直接 GROUP BY 即可：
```sql
-- 过去 7 天每小时
SELECT toStartOfHour(minute) as hour, sum(clicks)
FROM ad_clicks_1min
WHERE ad_id = ? AND minute >= now() - INTERVAL 7 DAY
GROUP BY hour ORDER BY hour;
```
ClickHouse 对这类查询通常 < 500ms，无需额外预聚合。

**进一步优化（超大规模时）：** 预计算每小时、每天的聚合桶（物化视图）：
```sql
CREATE MATERIALIZED VIEW ad_clicks_1hour
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (ad_id, hour)
AS SELECT ad_id, toStartOfHour(minute) as hour, sum(clicks) as clicks
FROM ad_clicks_1min GROUP BY ad_id, hour;
```
查询 7 天每小时 → 读 168 行（7×24），速度 < 10ms。

---

## 关联文档

- [../03_communication/02_async.md](../03_communication/02_async.md) — Kafka 高吞吐写入
- [../04_distributed/04_fault_tolerance.md](../04_distributed/04_fault_tolerance.md) — 幂等性保证
- [../02_rate_limiter.md](./02_rate_limiter.md) — 防刷限流
