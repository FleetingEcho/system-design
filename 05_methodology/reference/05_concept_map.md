# 系统设计概念关系网

> 用法：遇到某个问题，顺着箭头找到对应的概念和技术；看到某个技术，顺着反向箭头找到它解决的问题。

---

## 图一：全局视图（问题 → 方向）

遇到什么问题，往哪个方向找答案。

```mermaid
graph LR
    subgraph 问题
        P1[读QPS高]
        P2[写QPS高]
        P3[数据量大]
        P4[节点会故障]
        P5[网络会分区]
        P6[跨服务要事务]
        P7[下游会崩溃]
        P8[请求会重复]
    end

    subgraph 方向
        D1[缓存 / 读扩展]
        D2[异步 / 写扩展]
        D3[分片 / 冷热分离]
        D4[复制 / 共识]
        D5[一致性权衡 CAP]
        D6[分布式事务]
        D7[容错 / 降级]
        D8[幂等性]
    end

    P1 --> D1
    P1 --> D4
    P2 --> D2
    P2 --> D3
    P3 --> D3
    P4 --> D4
    P5 --> D5
    P6 --> D6
    P7 --> D7
    P8 --> D8

    D5 --> D6
    D6 --> D8
    D7 --> D8
```

---

## 图二：一致性权衡全景（核心中的核心）

CAP 定理是所有分布式决策的起点。

```mermaid
graph TD
    CAP[CAP 定理\n分区时只能选 C 或 A]

    CAP -->|强一致优先| CP[CP 系统]
    CAP -->|可用性优先| AP[AP 系统]

    CP --> RAFT[Raft / Paxos\n共识算法]
    CP --> DISTLOCK[分布式锁]
    CP --> 2PC[两阶段提交 2PC]

    AP --> EVENTUAL[最终一致性]
    AP --> CASSANDRA[(Cassandra\nTunable Consistency)]

    RAFT --> LEADER[Leader 选举]
    RAFT --> LOGREP[日志复制]
    RAFT --> ETCD[etcd / ZooKeeper]

    DISTLOCK --> REDISNX[Redis SET NX EX\n+ Lua 脚本]
    DISTLOCK --> ZKNODE[ZK 临时顺序节点]
    DISTLOCK --> FENCING[Fencing Token\n防脑裂]

    2PC -->|阻塞 + 单点| 2PC_PROB[⚠️ 协调者故障\n系统卡死]
    2PC_PROB -->|改用| SAGA

    EVENTUAL --> SAGA[Saga 补偿事务]
    EVENTUAL --> CRDT[CRDT 自动合并]
    EVENTUAL --> QUORUM[Quorum 读写\nW + R > N]

    QUORUM --> CASSANDRA

    SAGA --> CHOREOGRAPHY[编排式\n事件驱动]
    SAGA --> ORCHESTRATION[协调式\n中心 Saga 控制器]
    SAGA --> OUTBOX[Outbox 模式\n保证消息不丢]
    SAGA --> IDEM[幂等键\n补偿操作安全重试]
```

---

## 图三：存储选型决策

```mermaid
graph TD
    START{需要什么?}

    START -->|结构化数据 + 事务| RDBMS
    START -->|高写吞吐 + 时序| CASSANDRA
    START -->|缓存 + 低延迟读| REDIS
    START -->|全文搜索| ES
    START -->|文件 / 大对象| S3
    START -->|图关系| NEO4J
    START -->|半结构化文档| MONGO

    subgraph RDBMS [MySQL / PostgreSQL]
        BTREE[B+ Tree 索引\n精确/范围查询]
        MVCC[MVCC 并发控制]
        REPLICA[主从复制\n读写分离]
        SHARDING[水平分片]
    end

    SHARDING --> CONSIST_HASH[一致性哈希\n+ 虚拟节点]

    subgraph CASSANDRA [Cassandra]
        LSM[LSM Tree\n顺序写 极快]
        WIDE[宽列模型\nPartition Key 设计]
        AP_MODE[AP 可调一致性]
    end

    subgraph REDIS [Redis]
        DS[5 种数据结构\nString Hash List Set ZSet]
        CACHE_USE[缓存 / 计数器 / 锁]
        GEORADIUS[GEO 地理查询]
        PUBSUB_R[Pub/Sub 广播]
    end

    subgraph ES [Elasticsearch]
        INVERTED[倒排索引]
        BM25[BM25 相关性排序]
        ES_SHARD[分片 + 副本]
    end

    REPLICA --> RDBMS
    SHARDING --> RDBMS
```

---

## 图四：缓存体系与三大故障

```mermaid
graph LR
    subgraph 缓存写入模式
        ASIDE[Cache-Aside\n读时懒加载\n写时删缓存]
        WT[Write-Through\n写时同步更新]
        WB[Write-Back\n写时只写缓存\n异步刷盘]
    end

    subgraph 三大故障
        PEN[缓存穿透\n查根本不存在的 Key\n每次打穿到 DB]
        BD[缓存击穿\n热 Key 过期瞬间\n并发打到 DB]
        AV[缓存雪崩\n大量 Key 同时过期\n或 Redis 宕机]
    end

    subgraph 解决方案
        BLOOM[布隆过滤器\n一定不存在 → 拦截]
        NULL[空值缓存\n短 TTL 占位]
        MUTEX[互斥锁\n只让一个线程回源]
        LOGICAL[逻辑过期\n异步刷新不设 TTL]
        JITTER[TTL 随机抖动\n错开过期时间]
        WARMUP[缓存预热\n启动时主动加载]
        CB_CACHE[熔断器\nRedis 不可用时降级]
    end

    PEN --> BLOOM
    PEN --> NULL
    BD --> MUTEX
    BD --> LOGICAL
    MUTEX --> DISTLOCK2[分布式锁]
    AV --> JITTER
    AV --> WARMUP
    AV --> CB_CACHE
    WB -->|宕机风险| WB_RISK[⚠️ 数据可能丢失]
```

---

## 图五：异步消息与事件驱动

```mermaid
graph TD
    MQ[消息队列\n异步解耦 + 削峰]

    MQ --> KAFKA[Kafka]
    MQ --> RABBIT[RabbitMQ]

    subgraph Kafka 核心概念
        TOPIC[Topic / Partition\n并行消费]
        CG[Consumer Group\n水平扩展]
        OFFSET[Offset 可回放\n支持重新计算]
        EXACTLY_ONCE[Exactly-Once\n幂等生产者 + 事务]
    end

    subgraph RabbitMQ 核心概念
        EXCHANGE[Exchange 路由\nDirect / Topic / Fanout]
        DLQ[死信队列 DLQ\n处理失败消息]
    end

    KAFKA --> TOPIC
    KAFKA --> CG
    KAFKA --> OFFSET
    KAFKA --> EXACTLY_ONCE
    RABBIT --> EXCHANGE
    RABBIT --> DLQ

    EXACTLY_ONCE --> OUTBOX2[Outbox 模式\n业务 + 消息同一事务]

    MQ --> PATTERNS[消息模式]

    PATTERNS --> FANOUT2[Fanout 扇出]
    PATTERNS --> EVENTSRC[Event Sourcing]

    FANOUT2 --> PUSH[推模式\n发布时写粉丝 Timeline]
    FANOUT2 --> PULL[拉模式\n读时聚合]
    FANOUT2 --> HYBRID[推拉结合\n大V拉 普通用户推]

    EVENTSRC --> CQRS[CQRS\n读写模型分离]
    CQRS --> READ_MODEL[读模型\nElasticsearch / Redis 反范式]
    CQRS --> WRITE_MODEL[写模型\nMySQL 事务 规范化]
    OFFSET --> EVENTSRC
```

---

## 图六：容错与弹性

```mermaid
graph TD
    FAULT[分布式故障模式]

    FAULT --> TO[超时 / 网络慢]
    FAULT --> CRASH[服务崩溃]
    FAULT --> OVERLOAD[过载]
    FAULT --> DUP[重复请求]

    TO --> RETRY[重试]
    RETRY --> BACKOFF[指数退避 + 随机抖动\n防重试风暴]
    RETRY --> IDEM2[幂等键\n重试安全]

    CRASH --> CB2[熔断器]
    CB2 --> CLOSED[闭合\n正常放行]
    CB2 --> OPEN[断开\n快速失败]
    CB2 --> HALFOPEN[半开\n小流量探测]
    CB2 --> FALLBACK[降级\n返回默认 / 缓存值]

    OVERLOAD --> RL2[限流]
    RL2 --> TB[令牌桶\n允许突发]
    RL2 --> LEAKY[漏桶\n严格匀速]
    RL2 --> SLIDE[滑动窗口计数\n防边界效应]
    RL2 --> REDIS_LUA[Redis Lua 脚本\n分布式原子限流]

    DUP --> IDEM2
    IDEM2 --> REDIS_IDEM[Redis 快速查重]
    IDEM2 --> DB_UNIQUE[DB UNIQUE 约束\n最终防线]

    BACKOFF --> IDEM2
```

---

## 图七：实时通信选型

```mermaid
graph TD
    RT_NEED[实时通信需求]

    RT_NEED --> DIR{通信方向?}

    DIR -->|双向实时| WS[WebSocket\n全双工 持久连接]
    DIR -->|服务端 → 客户端| UNI[单向推送]

    UNI --> SSE_NODE[SSE\n基于 HTTP 简单]
    UNI --> LP[长轮询\n兼容性好]

    WS --> WS_SCALE[横向扩展问题\n连接是有状态的]
    WS_SCALE --> STICKY[Sticky Session]
    WS_SCALE --> REDIS_PS2[Redis Pub/Sub\n跨服务器广播]

    WS --> HB[心跳检测\n30s Ping]
    WS --> OFFLINE_MSG[离线消息\n上线后拉取]

    RT_NEED --> ORDER{消息需要有序?}
    ORDER -->|会话内有序| SEQNUM[Redis INCR\n单调序号]
    ORDER -->|全局时间有序| SNOWFLAKE2[Snowflake ID\n高位时间戳]
```

---

## 图八：ID 生成与分布式协调

```mermaid
graph LR
    ID_NEED[需要全局唯一 ID]

    ID_NEED --> UUID[UUID v4\n完全随机\n不可排序]
    ID_NEED --> UUID7[UUID v7\n时间有序\n可排序]
    ID_NEED --> AUTO[DB 自增\n单点瓶颈]
    ID_NEED --> SNOWFLAKE3[Snowflake\n64位 时间有序]
    ID_NEED --> SEGMENT[号段模式\n批量预取]

    SNOWFLAKE3 --> SNOW_STRUCT[41位时间戳\n10位机器ID\n12位序列号]
    SNOWFLAKE3 --> CLOCK_SKEW[⚠️ 时钟回拨问题\n需处理]

    SEGMENT --> DOUBLE_BUF[双 Buffer\n防止号段用尽阻塞]

    ID_NEED --> COORD[分布式协调]
    COORD --> ZK2[ZooKeeper\n强一致 CP]
    COORD --> ETCD2[etcd\nRaft 共识]

    ZK2 --> ELECTION[Leader 选举]
    ZK2 --> CONFIG[配置中心]
    ZK2 --> DL2[分布式锁\n临时顺序节点]
```

---

## 图九：地理位置系统

```mermaid
graph LR
    GEO[地理位置查询]

    GEO --> STATIC{数据是静态还是动态?}

    STATIC -->|静态 POI\n餐厅/景点| GEOHASH_DB[Geohash 预计算\n写入 DB 索引列]
    STATIC -->|动态 司机/用户| REDIS_GEO2[Redis GEO\n实时更新]

    GEOHASH_DB --> NINE[查中心 + 8 邻居格子\n共 9 个]
    GEOHASH_DB --> HAVERSINE[Haversine 精确过滤]
    GEOHASH_DB --> PRECISION[精度选择\n半径 <1km → geohash6\n半径 <5km → geohash5]

    REDIS_GEO2 --> GEORADIUS[GEORADIUS 命令\n直接返回距离排序]
    REDIS_GEO2 --> TTL_EXPIRE[TTL 30s\n离线自动清除]

    GEO --> QUADTREE[QuadTree\n内存索引\n极高 QPS]
    QUADTREE --> MEMORY[全量放内存\n~130MB / 2亿POI]
```

---

## 图十：概念总线（快速跳转）

> 这张图不关注方向，只标注"哪些概念经常一起出现"。

```mermaid
graph LR
    RAFT2 --- ETCD3[etcd]
    RAFT2 --- CONSENSUS[共识 = 基础\n分布式锁/配置中心\n/Leader选举]

    SAGA2[Saga] --- OUTBOX3[Outbox]
    SAGA2 --- IDEM3[幂等键]
    OUTBOX3 --- KAFKA2[Kafka]
    IDEM3 --- REDIS3[Redis]

    CACHE2[缓存] --- DISTLOCK3[分布式锁]
    CACHE2 --- BLOOM2[布隆过滤器]
    CACHE2 --- CB3[熔断器]

    CQRS2[CQRS] --- ES2[Elasticsearch]
    CQRS2 --- EVENTSRC2[Event Sourcing]
    EVENTSRC2 --- KAFKA2

    SNOWFLAKE4[Snowflake ID] --- CASSANDRA2[Cassandra]
    SNOWFLAKE4 --- CHAT[聊天系统\n消息有序]

    FANOUT3[Fanout] --- REDIS3
    FANOUT3 --- KAFKA2
    FANOUT3 --- NEWSFEED[News Feed]

    GEOHASH2[Geohash] --- REDIS3
    GEOHASH2 --- RIDE[打车系统]
    GEOHASH2 --- NEARBY[附近搜索]
```

---

## 阅读建议

**面试前一天**，按这个顺序扫一遍各图：

1. **图一（全局）**：建立"问题 → 方向"的直觉
2. **图二（一致性）**：CAP 是最高频的考点，必须烂熟
3. **图六（容错）**：几乎每个系统设计题都要提
4. **图四（缓存）**：穿透/击穿/雪崩三连问必考
5. **图五（消息）**：Kafka 相关的 case study 很多

**做 Mock 时**，画架构图之后，对照图三（存储选型）和图七（实时通信）检查有没有遗漏的考虑点。
