# 系统设计：分布式 Key-Value 存储（设计 DynamoDB/Cassandra）

## TL;DR

设计一个类 DynamoDB 的分布式 KV 存储，核心挑战：**如何在数千台节点上均匀分布数据，同时保证高可用和可调一致性**。答案是：一致性哈希 + Quorum 读写 + 向量时钟冲突解决 + Gossip 协议故障检测。

---

## 需求澄清

```
功能需求：
  - put(key, value)：写入或更新
  - get(key)：读取，key 不存在返回 null
  - delete(key)：删除
  - 支持大 value（最大 10MB）

非功能需求：
  - 低延迟：P99 读 < 10ms，写 < 20ms
  - 高可用：99.99%（允许部分节点故障时继续工作）
  - 可扩展：水平扩展到 1000+ 节点
  - 可调一致性：可配置强一致 / 最终一致
  - 数据规模：100TB+
```

---

## 与竞品横向对比

| 特性 | Redis | DynamoDB | Cassandra | etcd | 你设计的 |
|------|-------|---------|-----------|------|---------|
| 数据模型 | KV / 多结构 | KV + 范围 | 宽列 | KV | KV |
| 一致性 | 强（单节点）| 可调 | 可调 | 强（Raft）| 可调 |
| 可用性 | AP | AP | AP | CP | AP |
| 持久化 | AOF/RDB | 是 | 是 | 是 | WAL+SSTable |
| 水平扩展 | 需 Cluster | 自动 | 自动 | 有限 | 一致性哈希 |
| 适用场景 | 缓存/会话 | 大规模业务 | 时序/日志 | 配置/选主 | 通用存储 |

---

## 整体架构

```mermaid
flowchart TD
    Client["客户端"] --> LB["负载均衡"]
    LB --> N1["节点 A\n协调者"]
    LB --> N2["节点 B"]
    LB --> N3["节点 C"]

    subgraph 一致性哈希环
        N1 --- N2 --- N3 --- N4["节点 D"] --- N5["节点 E"] --- N1
    end

    N1 -->|"复制"| N2
    N1 -->|"复制"| N3

    subgraph 每个节点内部
        WAL["WAL\n顺序写"] --> Mem["MemTable\n内存跳表"]
        Mem -->|"写满flush"| SST["SSTable L0\n不可变文件"]
        SST -->|"Compaction"| SST2["SSTable L1/L2..."]
        BF["Bloom Filter\n每个SSTable一个"] --> SST
    end
```

---

## 核心设计一：一致性哈希（数据分布）

```mermaid
flowchart LR
    subgraph 哈希环（0 ~ 2^32-1）
        direction LR
        A["节点A\n@100"] --- B["节点B\n@200"] --- C["节点C\n@300"] --- D["节点D\n@400"] --- A
        K1["key='user:1'\nhash=150\n→ 落在节点B"] 
        K2["key='order:5'\nhash=350\n→ 落在节点D"]
    end
```

**虚拟节点（解决数据倾斜）：**

```
每个物理节点 → 100~200 个虚拟节点（不同哈希位置）
  节点A → [A#1@50, A#2@180, A#3@320, ...]
  节点B → [B#1@100, B#2@250, B#3@380, ...]

效果：数据分布更均匀，节点上下线时迁移数据量 ≈ 1/N
```

**关键参数（仿 Dynamo）：**
```
N = 3（副本数）
W = 2（写 Quorum：至少2个节点确认）
R = 2（读 Quorum：至少2个节点响应）
W + R > N → 强一致；W=1,R=1 → 最终一致（高性能）

协调节点（Coordinator）：客户端请求到达的节点
  → 找到 key 在哈希环上对应的 N 个节点
  → 并行发送读/写请求，收到 Quorum 数量的响应即返回
```

---

## 核心设计二：写入流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant Coord as 协调节点
    participant N1 as 节点1（Primary）
    participant N2 as 节点2（副本）
    participant N3 as 节点3（副本）

    C->>Coord: put("user:1", {...})
    Coord->>Coord: hash("user:1") → 定位 N 个节点
    par 并行写入
        Coord->>N1: write(key, value, vector_clock)
        Coord->>N2: write(key, value, vector_clock)
        Coord->>N3: write(key, value, vector_clock)
    end
    N1-->>Coord: ACK
    N2-->>Coord: ACK
    Note over Coord: 收到 W=2 个 ACK，满足 Quorum
    Coord-->>C: 200 OK
    N3-->>Coord: ACK（迟到，仍记录）
```

**写入节点内部流程（LSM Tree）：**

```mermaid
flowchart LR
    Put["put(k,v)"] --> WAL["①写 WAL\n顺序写磁盘\n崩溃恢复用"]
    WAL --> Mem["②写 MemTable\n内存跳表\nO(log n)"]
    Mem -->|"写满 64MB"| L0["③ flush → SSTable L0\n不可变有序文件"]
    L0 -->|"Compaction"| L1["④ 合并 → SSTable L1\n去重,排序,删标记清理"]
```

---

## 核心设计三：读取流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant Coord as 协调节点
    participant N1 as 节点1
    participant N2 as 节点2
    participant N3 as 节点3（慢）

    C->>Coord: get("user:1")
    par 并行读 N 个节点
        Coord->>N1: read("user:1")
        Coord->>N2: read("user:1")
        Coord->>N3: read("user:1")
    end
    N1-->>Coord: {value:"Alice", vc:[1,2,0]}
    N2-->>Coord: {value:"Bob", vc:[1,3,0]}（更新版本）
    Note over Coord: 收到 R=2 个响应，比较向量时钟
    Note over Coord: vc[1,3,0] > vc[1,2,0] → 取最新版本
    Coord-->>C: {value:"Bob"}
    Coord->>N1: 异步修复（Read Repair）写入最新版本
```

**节点内部读取（LSM Tree 读路径）：**

```
查找 key "user:1"：
  1. MemTable（内存跳表）→ O(log n)
  2. L0 SSTable（可能多个，有重叠）
     → 用 Bloom Filter 过滤不含此 key 的文件
     → 二分查找索引块 → 读数据块
  3. L1, L2... 依次向下（每层文件不重叠，二分定位）

Bloom Filter 的作用：
  key 不存在时，大概率 Bloom Filter 就能判断 → 跳过磁盘读
  避免读放大（Read Amplification）
```

---

## 核心设计四：冲突解决（向量时钟）

```mermaid
sequenceDiagram
    participant A as 节点A
    participant B as 节点B
    participant C as 节点C

    Note over A,C: 初始状态：value="v0", vc=[0,0,0]
    A->>A: write "v1", vc=[1,0,0]
    A->>B: 复制 "v1", vc=[1,0,0]
    Note over A,B: 网络分区！
    A->>A: write "v2", vc=[2,0,0]
    C->>C: write "v3", vc=[0,0,1]
    Note over A,C: 分区恢复，两个版本冲突
    Note over A,C: vc=[2,0,0] 和 vc=[0,0,1] 无法比较（并发）
    Note over A,C: → 返回两个版本给客户端，由客户端或业务逻辑合并
```

**Last-Write-Wins（LWW）vs 向量时钟：**

```
LWW（Cassandra 默认）：
  每个写操作带时间戳，取最新时间戳的值
  优点：简单
  缺点：依赖时钟同步，可能丢失数据

向量时钟（DynamoDB 早期）：
  记录每个节点的逻辑时钟 [c_A, c_B, c_C, ...]
  可以检测到并发写（两个向量时钟无法比较大小）
  优点：不丢数据（保留所有并发版本）
  缺点：需要客户端合并逻辑，向量时钟可能无限增长
```

---

## 核心设计五：故障检测（Gossip 协议）

```mermaid
flowchart TD
    subgraph Gossip传播
        A["节点A\n心跳计数:100"] -->|"每秒随机选2个节点"| B["节点B"]
        A --> C["节点C"]
        B -->|"继续传播"| D["节点D"]
        B --> E["节点E"]
    end
    
    subgraph 故障判断
        F["节点F 心跳停更新"] -->|"超过阈值时间"| G["标记为'疑似故障'"]
        G -->|"多数节点确认"| H["标记为'已故障'"]
        H --> I["触发数据重分配"]
    end
```

**对比中心化心跳 vs Gossip：**

| | 中心化心跳 | Gossip |
|--|---------|--------|
| 单点故障 | 有（心跳服务器故障）| 无 |
| 网络带宽 | O(n)（所有节点→中心）| O(n log n) |
| 收敛时间 | 快 | O(log n) 轮次 |
| 一致性 | 强 | 最终一致 |

---

## 核心设计六：数据修复（Merkle Tree 反熵）

```mermaid
flowchart TD
    A["节点A：Merkle Root = Hash_A"] 
    B["节点B：Merkle Root = Hash_B"]
    A <-->|"对比 Root Hash"| B
    
    subgraph 不同时递归对比
        A1["A 左子树\nHash_A1"] <-->|"不同"| B1["B 左子树\nHash_B1"]
        A2["A 右子树\nHash_A2"] <-->|"相同，跳过"| B2["B 右子树\nHash_B2"]
        A1 --> Leaf["定位到具体\n哪个 key range 有差异"]
        Leaf --> Sync["只同步差异部分\n而非全量数据"]
    end
```

---

## 面试追问

**Q: W+R > N 为什么能保证强一致？**

写入确认的 W 个节点和读取的 R 个节点，至少有 1 个重叠（W+R>N → 重叠≥1）。这个重叠节点一定有最新的写入，读到的值不会是旧值。

**Q: 如果协调节点在写入 W 个确认前宕机怎么办？**

其他节点已写入的数据不丢（有 WAL），客户端收不到响应重试即可（幂等写入）。通过幂等键防止重复写入。

**Q: Hinted Handoff 是什么？**

目标节点暂时不可用时，协调者把写入数据临时存在另一个"提示"节点上，等目标节点恢复后转发。保证高可用（AP），但可能短暂一致性问题。

**Q: 为什么 Cassandra 用 LWW 而不是向量时钟？**

向量时钟需要客户端实现合并逻辑，增加使用复杂度；向量时钟大小随节点数增长可能无界。LWW 简单、性能好，大多数场景（写入覆盖）够用，代价是极端情况下可能丢写。
