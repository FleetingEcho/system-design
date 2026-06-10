# Discord 架构演进

> 来源：Discord Engineering Blog（极其详细，强烈推荐原文）。
> Discord 是少数把架构决策和踩坑过程**完整公开**的公司，是学习系统设计的极佳案例。

---

## 系统规模（2023 年）

```
注册用户：5.6 亿
日活用户（DAU）：1.5 亿
服务器（Server/Guild）：1900 万个
同时在线用户峰值：约 800 万
消息量：每天 40 亿条
语音通话并发用户：数百万
```

---

## 核心挑战

```
1. 消息存储：40 亿条/天，需要高写入 + 按频道/时间查询
2. 在线状态（Presence）：8000 万用户的在线/离线状态，变更极频繁
3. 语音通话：低延迟（< 150ms）、实时媒体传输
4. 大型服务器（Guild）：单个服务器可以有 80 万成员，消息 Fanout 极难
5. 热点消息：大型服务器的活跃频道，读写都极集中
```

---

## 消息存储的演进：MongoDB → Cassandra → ScyllaDB

### 阶段一：MongoDB（2015-2017）

```
早期选择 MongoDB：
  - 开发快，灵活 Schema
  - 单文档操作原子性

问题（用户增长后暴露）：
  MongoDB 在内存中维护索引
  随着消息量增长（数十亿条），索引无法全部放入内存
  → 索引查询频繁换页（page fault）→ 查询延迟飙升
  → 凌晨低峰期数据压缩（compaction）会导致大幅延迟抖动

2017 年：决定迁移
```

### 阶段二：Cassandra（2017-2022）

**选择 Cassandra 的理由**

```
消息的访问模式：
  写：高频插入（append-only，永远不更新）
  读：按 channel_id + message_id 范围查询（取某个频道的最近 N 条消息）

Cassandra 完美匹配：
  - LSM Tree：写入极快，顺序写磁盘
  - 按 partition key（channel_id）分区存储
  - 按 clustering key（message_id）有序存储
  - 水平扩展无上限

数据模型：
  CREATE TABLE messages (
    channel_id  bigint,
    bucket      int,       -- 时间分桶（每个桶约 10 天）
    message_id  bigint,    -- Snowflake ID（时序有序）
    author_id   bigint,
    content     text,
    ...
    PRIMARY KEY ((channel_id, bucket), message_id)
  ) WITH CLUSTERING ORDER BY (message_id DESC);
  
  为什么加 bucket？
    单个 channel 的消息可以有几千万条，单 partition 太大（Cassandra 的 partition 有大小限制）
    按时间分桶（每桶约 10 天），每个 partition 控制在合理大小
```

**遭遇热点分区（Hot Partition）**

```
问题：某些超大频道（如 Minecraft 官方服务器）的消息量是普通频道的 100 倍
     这些频道的 Cassandra partition 变成热点，单节点 CPU 100%

解决方案：
  1. 限流（Rate Limiting）：超大频道的发消息频率做限制
  2. 读缓存：热点频道的最近消息缓存到 Redis，减少 Cassandra 读压力
  3. 写批处理：多条消息合批写入，降低 Cassandra 写 QPS
```

### 阶段三：迁移到 ScyllaDB（2022-至今）

**为什么从 Cassandra 迁到 ScyllaDB**

```
Cassandra 的问题（Discord 规模下暴露）：
  - JVM GC 暂停：Cassandra 是 Java 写的，GC 停顿会导致延迟毛刺（P99 延迟高）
  - 运维复杂：大集群（数百节点）的 Cassandra 运维成本极高
  - 尾延迟（Tail Latency）：P999 latency 很高，影响用户体验

ScyllaDB：
  - C++ 重写的 Cassandra 替代品，兼容 Cassandra API
  - 无 GC，延迟更稳定
  - Shard-per-core 架构：每个 CPU core 独立处理自己的数据分片，避免锁竞争
  - 声称性能 10x Cassandra

迁移结果（Discord Blog 公开）：
  - 节点数：从 177 个 Cassandra 节点 → 72 个 ScyllaDB 节点（同等容量）
  - P99 延迟：从 40~125ms → 15ms
  - 尾延迟（P999）：大幅改善
  
数据迁移：双写（同时写 Cassandra 和 ScyllaDB）→ 验证一致性 → 切流量 → 下线 Cassandra
```

---

## 在线状态（Presence）系统

**规模：** 1.5 亿用户，每次上下线都要通知所有相关用户

```
朴素方案：用户 A 上线 → 广播给 A 的所有好友
          如果 A 有 5000 个好友，1.5 亿用户同时在上下线...不可行

Discord 的实际做法：

服务器（Guild）级别的 Presence：
  用户加入某个服务器后，该服务器内的成员能看到彼此的在线状态
  在线状态更新只在服务器维度 Fanout，不是全局 Fanout

Presence 存储：
  Redis Hash：presence:{guildId} → { userId: status }
  状态：online / idle / dnd / offline
  
Presence 更新：
  用户 A 上线 → 更新 Redis → 向 A 所在的所有 Guild 广播 Presence 事件
  WebSocket 维持长连接，Presence 变化实时推送
  
优化：
  超大服务器（80 万成员）不实时推送所有成员 Presence
  只推送用户"活跃关注"的 Presence（当前频道的在线成员）
```

---

## 大型服务器（Large Guild）的消息 Fanout

```
普通服务器：100 人，1 条消息 → Fanout 100 个 WebSocket 连接，轻松
大型服务器：80 万成员，1 条消息 → Fanout 80 万个连接？

问题：
  - 即使只有 10% 在线，也是 8 万个 WebSocket 连接
  - 消息 Fanout 的耗时与成员数线性相关
  - 单台服务器撑不住

解决方案：Gateway 分片

架构：
  [消息] → [消息服务] → 发布到 Kafka 的 guild:{guildId} topic
                           ↓
                    [多个 Fanout Worker]
                    每个 Worker 消费一部分 Guild 的消息
                           ↓
                    [WebSocket Gateway 集群]
                    按用户 ID 分片，每台 Gateway 只管理部分用户的连接
                           ↓
                    [具体用户的 WebSocket 连接]

关键：Gateway 节点之间通过 Pub/Sub 协调
  消息到达后，只推送给当前在线且连接在本 Gateway 的用户
```

---

## 语音通话架构（WebRTC + TURN）

```
Discord 语音：延迟要求 < 150ms（人耳对语音延迟非常敏感）

协议选择：WebRTC（浏览器原生支持，低延迟 UDP）
  WebRTC 使用 UDP 传输音视频（不用 TCP）
  丢包时：用 FEC（前向纠错）补偿，而不是重传（重传太慢）

NAT 穿透问题：
  用户通常在 NAT 后面（家用路由器），无法直接 P2P 连接

解决：
  STUN：帮助两端发现自己的公网 IP，尝试 P2P 直连
  TURN（Selective Forwarding Unit）：
    P2P 失败时，媒体流通过 TURN 服务器中继
    Discord 在全球部署大量 TURN 节点（低延迟中继）
    用户 → 最近的 TURN 节点 → 其他用户

音频处理：
  服务器端混音（SFU 模式）：
    每个用户只向服务器发送自己的音频
    服务器把所有人的音频混合后，发给每个参与者
    （N 人通话：N 个发送流，每人接收 1 个混合流，而不是 N-1 个流）
    大幅降低带宽需求
```

---

## 关键设计决策总结

| 问题 | Discord 的解决方案 | 为什么 |
|------|-------------------|--------|
| 消息存储 | Cassandra → ScyllaDB | 写密集 + 范围查询，GC 暂停问题用 C++ 解决 |
| 热点分区 | 限流 + 读缓存 + 写批处理 | Cassandra 单 partition 有瓶颈 |
| 大群 Fanout | Kafka + Gateway 分片 | 单机无法 Fanout 80 万连接 |
| 在线状态 | Redis Hash + Guild 维度 Fanout | 全局 Fanout 不可行 |
| 语音通话 | WebRTC + SFU 混音 | UDP 低延迟；SFU 节省带宽 |

---

## 面试可借鉴的点

1. **Cassandra 数据模型设计**（channel_id + bucket 分桶）——应对超大 partition 的标准做法
2. **ScyllaDB 替代 Cassandra**——了解 JVM GC 是分布式存储延迟抖动的常见原因
3. **SFU 语音架构**——音视频系统的标准架构，比 P2P mesh 可扩展得多
4. **在线状态 Fanout 的范围控制**——不要全局 Fanout，按"用户关心的维度"限制范围

---

## 关联文档

- [../06_case_studies/06_chat_system.md](../06_case_studies/06_chat_system.md) — 聊天系统完整设计
- [../03_communication/03_realtime.md](../03_communication/03_realtime.md) — WebSocket 横向扩展
- [../02_storage/02_nosql.md](../02_storage/02_nosql.md) — Cassandra LSM Tree 原理
