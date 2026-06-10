# Twitter / X 架构演进

> 来源：Twitter Engineering Blog、InfoQ 演讲、@scaled 系列分享。
> 重点不是"背架构图"，而是理解**为什么每一步演进是必然的**。

---

## 系统规模（2022 年峰值）

```
日活用户（DAU）：2.38 亿
每天发推量：5 亿条
读 QPS（Timeline 加载）：约 300 万 QPS
写 QPS（发推）：约 6000 QPS
峰值（世界杯、奥斯卡）：单秒 14.3 万条推文（2014 年世界杯决赛）
```

---

## 核心挑战：读写极度不对称

```
写：一个用户发 1 条推文（1 次写）
读：这条推文被 1000 万粉丝读（1000 万次读）

读写比 ≈ 10,000 : 1

结论：架构必须为读极度优化，写的延迟可以适当牺牲
```

---

## 演进阶段

### 阶段一：早期单体（2006-2008）

```
Rails 应用 + MySQL（单库）

用户看 Timeline 的逻辑：
  SELECT tweets.* FROM tweets
  JOIN follows ON follows.followee_id = tweets.user_id
  WHERE follows.follower_id = :current_user
  ORDER BY tweets.created_at DESC
  LIMIT 20

问题：用户 A 关注了 500 人，每次刷 Timeline 都要 JOIN 500 人的推文，
      随着数据量增长，这个 JOIN 慢到不可用
```

### 阶段二：Timeline 预计算（2008-2012）

**核心思路：用空间换时间，把 Timeline 提前算好**

```
引入 Timeline Service + Redis

写路径（发推 Fanout）：
  用户 A 发推
    → Fanout Service 查出 A 的所有粉丝列表
    → 把推文 ID 写入每个粉丝的 Timeline 缓存（Redis List）
    → 每个粉丝的 Timeline 缓存 = 一个有序的推文 ID 列表

读路径（刷 Timeline）：
  用户 B 刷 Timeline
    → 从 Redis 读 B 的 Timeline 缓存（推文 ID 列表）
    → 批量查 Tweet Store 取推文内容
    → 返回

效果：读变为 O(1) Redis 查询，飞快
代价：写变重——1 条推文要写入所有粉丝的 Timeline 缓存
```

**关键数据结构：**

```
Redis Key：timeline:{userId}
数据结构：Sorted Set（ZSet），score = tweet 时间戳，member = tweetId

写：ZADD timeline:456 1700000000 tweet:789
读：ZREVRANGE timeline:456 0 19（最新 20 条）
保留：只保留最新 800 条（超出的 ZREMRANGEBYRANK 删除）
```

### 阶段三：大 V 问题（2012-2015）

**问题：名人效应打爆 Fanout**

```
Lady Gaga 有 3000 万粉丝
她发一条推文 → Fanout Service 要写 3000 万次 Redis

3000 万 × 每次写约 1ms = 8 小时才能完成！
实测：大 V 发推后粉丝等很久才看到

解决方案：推拉混合（Hybrid Fanout）
```

**推拉混合策略：**

```
普通用户（粉丝 < 1 万）→ 推模式：发推时立即 Fanout 到粉丝 Timeline
大 V（粉丝 > 1 万）→ 拉模式：不提前 Fanout

用户刷 Timeline 时：
  1. 从 Redis 读自己的 Timeline 缓存（来自普通用户的推文）
  2. 实时查询关注的所有大 V 的最新推文（从大 V 推文缓存拉取）
  3. 合并排序，返回最终 Timeline

阈值：Twitter 内部实际阈值约 5 万粉丝（不是 1 万，具体数字未公开）
```

**大 V 推文缓存：**

```
每个用户都有自己的推文缓存（User Tweet Timeline）
大 V 发推 → 只写自己的推文缓存（1 次）
粉丝刷 Timeline → 实时从大 V 缓存拉取

本质：把大 V 的 Fanout 从"写时扇出"变成"读时合并"
      牺牲一点读的性能（需要合并多个大 V 的推文），换取写的可控性
```

### 阶段四：存储层演进（2013-2019）

**Snowflake ID（2010 年推出）**

```
推文 ID 不用自增整数，而是 64 位 Snowflake ID：
  41 位时间戳 + 10 位机器 ID + 12 位序列号

好处：
  - 分布式生成，无需中心节点
  - ID 按时间有序（方便按时间范围查询）
  - 从 ID 本身可以解析出发推时间（不需要查 created_at 字段）

影响：Timeline 排序直接用 ID 排序代替时间戳排序，性能更好
```

**Manhattan（Twitter 自研 KV 存储）**

```
问题：MySQL 撑不住推文存储（数百亿条，持续写入）
解决：自研 Manhattan（类 Cassandra 的宽列存储）

推文存储 Schema：
  row_key = userId + tweetId
  columns = { text, media_ids, reply_to, like_count, ... }

Cassandra 特性天然适合：
  - 写入极快（LSM Tree，追加写）
  - 按 userId 分区，同一用户的推文在同一节点
  - 水平扩展无上限
```

**BlobStore（媒体存储）**

```
图片、视频不存 DB，存专门的对象存储（类 S3）
推文只存媒体的 URL / ID
CDN 加速全球媒体访问
```

---

## 最终架构全景

```
[客户端]
    ↓
[API Gateway + 限流]
    ↓
[Write Path]                          [Read Path]
    ↓                                      ↓
[Tweet Service]                    [Timeline Service]
    |                                      |
    |→ 写推文到 Manhattan（持久化）         |← 读 Redis Timeline 缓存
    |→ 触发 Fanout Service                 |← 合并大 V 的推文
    |                                      |← 批量读推文内容（Manhattan）
[Fanout Service]
    |
    |→ 普通用户：写粉丝 Redis Timeline 缓存
    |→ 大 V：跳过（读时拉取）
    |→ 消息队列（Kafka）异步处理通知、推荐信号
    |
[Search（Earlybird）]← 倒排索引，实时索引新推文（约 15 秒延迟）
[Trends Service]← 实时热词统计（滑动窗口）
[Recommendation]← 基于图和内容的推荐
```

---

## 关键设计决策总结

| 问题 | Twitter 的解决方案 | 为什么 |
|------|-------------------|--------|
| Timeline 读太慢 | 预计算 + Redis 缓存 Timeline | 读写比 10000:1，必须为读优化 |
| 大 V 写 Fanout 太慢 | 推拉混合（大 V 拉模式）| 3000 万粉丝无法实时 Fanout |
| 推文 ID 生成 | Snowflake（时序唯一 ID）| 分布式 + 有序 + 携带时间信息 |
| 推文持久化 | 自研 Manhattan（类 Cassandra）| MySQL 无法支撑数百亿条写密集数据 |
| 全文搜索 | Earlybird（基于 Lucene 的实时索引）| 通用搜索引擎无法满足 Twitter 的实时性要求 |

---

## 面试可借鉴的点

1. **推拉混合是最重要的架构模式**——所有 Feed 类系统都需要考虑大 V / 大 KOL 场景
2. **Snowflake ID 设计**——分布式系统的 ID 生成标准答案（见 `05_id_generation.md`）
3. **Timeline 缓存只存 ID，内容单独查**——数据分层，减少缓存压力
4. **大 V 阈值是动态的**——不是非此即彼，可以按粉丝数连续变化

---

## 关联文档

- [../06_case_studies/05_news_feed.md](../06_case_studies/05_news_feed.md) — News Feed 完整设计
- [../04_distributed/05_id_generation.md](../04_distributed/05_id_generation.md) — Snowflake ID 原理
- [../02_storage/03_cache.md](../02_storage/03_cache.md) — Redis 缓存策略
