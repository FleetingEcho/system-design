# 系统设计：附近的人（Nearby Friends）

## TL;DR

**与附近搜索（Proximity Service）的核心区别**：附近搜索是静态 POI 数据库查询；附近的人是**实时动态位置流**，用户位置每 N 秒更新一次，需要实时推送给朋友。核心技术：WebSocket 长连接 + Redis Geospatial + Pub/Sub。

---

## Nearby Friends vs Proximity Service 深度对比

```mermaid
flowchart LR
    subgraph Proximity Service（附近搜索）
        PS1["数据特征：静态\nPOI 位置不变"] --> PS2["数据库：MySQL+Geohash索引"]
        PS2 --> PS3["查询：用户发起，按需查询"]
        PS3 --> PS4["延迟要求：<100ms即可"]
    end
    
    subgraph Nearby Friends（附近的人）
        NF1["数据特征：高频动态\n位置每4秒更新"] --> NF2["存储：Redis（内存，过期自动清理）"]
        NF2 --> NF3["推送：服务端主动推，WebSocket"]
        NF3 --> NF4["延迟要求：<1s，接近实时"]
    end
```

| 维度 | Proximity Service | Nearby Friends |
|------|------------------|----------------|
| 数据更新频率 | 几乎不变（餐厅位置）| 每 4 秒一次 |
| 数据量 | 静态 DB（2 亿 POI）| 实时在线用户（数百万并发）|
| 查询发起方 | 客户端主动拉 | 服务端主动推 |
| 存储 | MySQL + Geohash 索引 | Redis + WebSocket |
| 关系模型 | 无（任何人都能搜）| 有（只看好友）|

---

## 需求澄清

```
功能需求：
  - 看到 5km 内的在线好友及距离
  - 好友位置实时更新（每 4 秒刷新）
  - 可设置隐私（不让特定好友看到）
  - 历史位置记录（24小时）

非功能需求：
  - 延迟：好友位置更新 < 1 秒到达我的设备
  - 规模：DAU 1 亿，同时在线 10%（1000万）
  - 位置更新 QPS = 1000万 / 4秒 = 250万 QPS（高频写！）
```

---

## 整体架构

```mermaid
flowchart TD
    App["手机 App\nWebSocket 长连接"] -->|"每4秒: 位置更新"| WS["WebSocket 服务器集群\n有状态，维护连接"]
    WS -->|"写位置"| Redis["Redis Cluster\nGEOADD user_locations user_id lat lng\nkey TTL=30s（离线自动过期）"]
    WS -->|"发布位置事件"| MQ["Kafka / Redis Pub/Sub\ntopic: location.{user_id}"]
    MQ --> FanoutSvc["Fanout 服务\n查好友列表，推送给在线好友"]
    FanoutSvc --> Redis2["Redis 好友关系缓存\n{user_id} → Set{friend_ids}"]
    FanoutSvc -->|"推送位置给在线好友"| WS
    
    Redis -->|"GEORADIUS 查询"| FanoutSvc
    
    DB[("MySQL\n用户关系表\nfriendships")] --> Redis2
    
    LocationHistory["位置历史服务\n写 Cassandra（时序）"] --> Kafka2["Kafka"]
    MQ --> Kafka2
```

---

## 核心设计一：位置数据存储

### Redis Geospatial（实时位置）

```
数据结构：Redis GEO（底层是 ZSet，score = 编码后的经纬度）

命令：
  GEOADD user_locations 116.4 39.9 "user:1001"   # 更新位置
  GEORADIUS user_locations 116.4 39.9 5 km        # 查5km内所有人
  GEODIST user_locations user:1001 user:1002 km   # 两人距离
  GEOPOS user_locations user:1001                  # 获取某人位置

设计细节：
  - Key 设计：user_locations（所有用户共一个 ZSet）
    → 问题：单个 ZSet 在 Redis Cluster 无法拆分（GEORADIUS 要求同一节点）
    → 解决：按 Geohash 分区，user_locations:{geohash5} 多个 ZSet
  
  - TTL：每次位置更新重置 TTL=30s
    → 用户关闭 App 后，30秒内位置自动过期
    → 不需要"离线"事件，Redis 自动清理

  - 精度：GEOADD 精度约 0.6m，完全满足"附近"需求
```

### Cassandra（历史位置）

```
表设计（时序数据，按用户+时间分区）：

CREATE TABLE location_history (
  user_id    BIGINT,
  recorded_at TIMESTAMP,
  lat        DOUBLE,
  lng        DOUBLE,
  accuracy   FLOAT,
  PRIMARY KEY (user_id, recorded_at)
) WITH CLUSTERING ORDER BY (recorded_at DESC)
  AND default_time_to_live = 86400;  -- 24小时后自动删除

查询：SELECT * FROM location_history 
      WHERE user_id = 1001 
      AND recorded_at > now() - 1h;
```

---

## 核心设计二：实时推送流程

```mermaid
sequenceDiagram
    participant A as 用户A（发送位置）
    participant WS as WebSocket 服务器
    participant Redis as Redis（位置+好友）
    participant Fanout as Fanout 服务
    participant B as 用户B（好友，在线）
    participant C as 用户C（好友，离线）

    loop 每4秒
        A->>WS: {lat: 39.9, lng: 116.4}
        WS->>Redis: GEOADD user_locations 116.4 39.9 "user:A"
        WS->>Fanout: 位置事件{user_id:A, lat, lng}
        Fanout->>Redis: SMEMBERS friends:A → [B, C, D, ...]
        Fanout->>Redis: 过滤：哪些好友在线？（ws_sessions:{B}存在？）
        Note over Fanout: B在线（有WS连接），C离线
        Fanout->>WS: 推送给B的连接：{friend:A, lat, lng, distance}
        WS->>B: WebSocket推送（实时）
        Note over C: C不在线，跳过（下次打开App时查询快照）
    end
```

---

## 核心设计三：Fanout 的扩展性问题

```mermaid
flowchart TD
    Problem["问题：用户有5000个好友\n每4秒 × 5000次推送 = 1250次/秒（单用户）\n1000万用户同时更新 = 125亿次/秒推送 ❌"]
    
    Solution1["方案1：只推送给 WebSocket 在线的好友\n过滤掉离线好友（节省80%以上）"]
    Solution2["方案2：好友数量限制\n超过1000好友的用户不主动推送\n由客户端定时主动拉取"]
    Solution3["方案3：Pub/Sub\n每个用户订阅自己好友的频道\n发布者发布，各订阅者异步收取"]
    
    Problem --> Solution1 & Solution2 & Solution3
```

**Redis Pub/Sub 方案（推荐）：**

```
每个用户 A 上线时：
  SUB location.channel.friend1
  SUB location.channel.friend2
  ...（订阅所有在线好友的频道）

A 发布位置时：
  PUBLISH location.channel.A "{lat:39.9, lng:116.4}"

好友 B 订阅了 location.channel.A → 收到推送

问题：好友关系变化时要更新订阅
好友下线时 channel 自动无订阅者
```

---

## 核心设计四：隐私控制

```mermaid
flowchart TD
    Update["A 更新位置"] --> Privacy{检查 A 的\n隐私设置}
    Privacy -->|"对所有好友可见"| Fanout["正常推送"]
    Privacy -->|"对部分好友隐藏"| Filter["过滤黑名单好友\nblacklist:{A} → [B, C]"]
    Filter --> Fanout2["推送给未屏蔽的好友"]
    Privacy -->|"完全隐藏"| Skip["不上传位置\n或上传假位置"]
```

**隐私数据结构：**

```
Redis Set：privacy_blacklist:{user_id} → Set{blocked_friend_ids}
在 Fanout 时过滤：好友 B 在黑名单里 → 不推送给 B

模糊化距离（防止精确定位）：
  真实距离 324m → 展示 "300m 以内"
  真实距离 2.1km → 展示 "2km 以内"
  只显示区间，不显示精确坐标
```

---

## 核心设计五：WebSocket 服务横向扩展

```mermaid
flowchart TD
    LB["负载均衡（L4）\nIP Hash（同一用户同一服务器）"] --> WS1["WebSocket 服务器1\n用户A, B, C 的连接"]
    LB --> WS2["WebSocket 服务器2\n用户D, E, F 的连接"]
    LB --> WS3["WebSocket 服务器3"]
    
    WS1 <-->|"Pub/Sub\n跨服务器推送"| Redis["Redis Pub/Sub\n或 Kafka"]
    WS2 <--> Redis
    WS3 <--> Redis
    
    Note1["A在WS1，A的好友D在WS2\nA发布位置 → Redis转发 → WS2推送给D"]
```

**Session 路由：**

```
Redis Hash：ws_server:{user_id} → server_id
  用户A连接到 WS1 → 记录 ws_server:A = "ws1"

Fanout 推送给好友时：
  查 ws_server:{friend_id} → 知道好友在哪台服务器
  通过服务器间消息（Redis Pub/Sub）转发推送
```

---

## 面试追问

**Q: 位置更新 250 万 QPS 怎么扛？**

① Redis Cluster 水平扩展（16+ 分片），每分片扛约 10 万写 QPS  
② WebSocket 服务器集群（无状态化，靠 Redis 共享状态）  
③ 客户端只在位置变化超过阈值（如 50m）才更新，减少无效写入  
④ 静止用户降低更新频率（30秒一次 → 4秒一次只在移动中）

**Q: 用户很多好友，Fanout 怎么优化？**

① 只推给在线好友（过滤离线，节省 80%+）  
② 分批异步推送，避免单次 Fanout 阻塞  
③ 好友超过 1000 的用户：不推送，改为客户端定时拉取（拉 vs 推 混合）

**Q: 和 Proximity Service 对比，为什么这里不用 MySQL + Geohash？**

MySQL 适合静态数据查询，1000 万用户每 4 秒写一次 = 250 万写 QPS，MySQL 根本扛不住。Redis 内存 KV 写入快（50 万+ QPS/节点），Cluster 可线性扩展。另外 MySQL 的 Geohash 索引不适合频繁更新场景（每次更新需要删旧记录插新记录）。
