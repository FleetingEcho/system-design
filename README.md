# OOD + 系统设计面试笔记

102 篇，覆盖面向对象设计与分布式系统设计的全链路面试准备，全程 TypeScript。

---

## OOD 入门精讲

从零系统学习面向对象设计，全程 TypeScript。适合 OOD 基础薄弱或想系统梳理的读者。

**学习路径**：

```
基础四大特性（01-04）
    ↓
SOLID 原则（05）          ← 面试最高频，必须能举例
    ↓
三大类设计模式（06-08）   ← 结合 08_ood 实战题一起看
    ↓
面试方法论（09）          ← 上场前必读，有完整话术模板
    ↓
TS 高级类型（10）         ← 前端/全栈加分项
反模式识别（11）          ← 能说出反模式让面试官眼前一亮
GRASP 原则（12）          ← 选读，深度加分
UML 速查（13）            ← 白板面试时随时查
函数式 vs OOP（14）       ← TS 特有，前端/全栈必读
DDD 基础（15）            ← OOD 理论落地实际项目的关键
```

| 文件 | 主题 | 核心内容 |
|------|------|---------|
| [00 什么是 OOD](09_ood_guide/00_what_is_ood.md) | 概述 | 类 vs 对象 / 三种类关系 / interface vs abstract / 四大支柱导览 |
| [01 封装](09_ood_guide/01_encapsulation.md) | Encapsulation | 访问修饰符 / 不变量保护 / Getter-Setter 正确用法 / 不泄露内部集合 |
| [02 抽象](09_ood_guide/02_abstraction.md) | Abstraction | interface vs abstract class 决策树 / 抽象层次一致性 / 依赖注入边界 |
| [03 继承与组合](09_ood_guide/03_inheritance.md) | Inheritance | 脆弱基类问题 / Composition over Inheritance / Square-Rectangle 陷阱 / Mixin |
| [04 多态](09_ood_guide/04_polymorphism.md) | Polymorphism | 子类型多态 / 泛型参数多态 / 函数重载 / 消灭 if-else / 鸭子类型 |
| [05 SOLID](09_ood_guide/05_solid.md) | SOLID 原则 | SRP / OCP / LSP / ISP / DIP — 每条原则含反例→正例对比 |
| [06 创建型模式](09_ood_guide/06_creational_patterns.md) | Creational | Factory Method / Abstract Factory / Builder（链式 API）/ Singleton / Prototype |
| [07 结构型模式](09_ood_guide/07_structural_patterns.md) | Structural | Composite / Decorator（动态叠加）/ Adapter / Facade / Proxy（懒加载/缓存/权限）|
| [08 行为型模式](09_ood_guide/08_behavioral_patterns.md) | Behavioral | Strategy / Observer / State / Command（Undo/Redo）/ CoR / Template Method |
| [09 面试方法论](09_ood_guide/09_interview_guide.md) | 面试指南 | 45分钟时间分配 / 需求澄清话术 / 类图速绘 / 扩展性追问应对 / 题目难度分级 |
| [10 TS 高级类型](09_ood_guide/10_typescript_advanced.md) | TypeScript | 判别联合 / 映射类型 / 条件类型 infer / 模板字面量类型 / keyof typeof / 泛型约束 |
| [11 反模式](09_ood_guide/11_anti_patterns.md) | Anti-Patterns | 上帝类 / 贫血模型 / 原始类型偏执 / 特性依恋 / 霰弹式修改 / 过度设计 |
| [12 GRASP 原则](09_ood_guide/12_grasp.md) | GRASP | 信息专家 / 创建者 / 控制器 / 低耦合 / 高内聚 / 多态 / 纯虚构 / 间接 / 防止变异 |
| [13 UML 速查表](09_ood_guide/13_uml_cheatsheet.md) | UML | 六大关系箭头 / 多重性符号 / 白板画图技巧 / 代码→UML 对照表 |
| [14 函数式 vs OOP](09_ood_guide/14_functional_vs_oop.md) | 范式混用 | class vs function 判断决策树 / 4 种场景对比 / 混用最佳实践 / 什么时候不要用 class |
| [15 DDD 基础](09_ood_guide/15_ddd_basics.md) | 领域驱动设计 | 值对象 / 实体 / 聚合根 / 领域服务 / 仓储 — OOD 理论到实际项目的桥梁 |



## OOD 面向对象设计

面试中常与系统设计并列考察，尤其是国内大厂和外企。用 TypeScript 实现，每题含类图、完整代码、追问解答。

### 推荐刷题顺序（前端 / 全栈）

| 优先级 | 题目 | 文件 | 核心模式 |
|--------|------|------|---------|
| ⭐ 必须会 | 停车场 | [01](08_ood/01_parking_lot.md) | 多态 + Strategy |
| ⭐ 必须会 | LRU Cache | [02](08_ood/02_lru_cache.md) | HashMap + 双向链表 |
| ⭐ 必须会 | 自动贩卖机 | [05](08_ood/05_vending_machine.md) | State 模式 |
| 重点掌握 | ATM 机 | [18](08_ood/18_atm_machine.md) | State 模式进阶 |
| 重点掌握 | 购物车 | [13](08_ood/13_shopping_cart.md) | 多态 + Strategy |
| 重点掌握 | 聊天室 | [16](08_ood/16_chat_room.md) | Mediator + Observer |
| 重点掌握 | 日志聚合器 | [17](08_ood/17_logger.md) | Singleton + CoR + Strategy |
| 有余力再看 | 电影院订票 | [21](08_ood/21_cinema_booking.md) | 并发防超卖 |
| 有余力再看 | LFU Cache | [22](08_ood/22_lfu_cache.md) | 双 HashMap O(1) |
| 有余力再看 | Twitter OOD | [15](08_ood/15_twitter_ood.md) | 多态 + 推拉模式 |

> **配合使用**：做题前先读 [09_ood_guide/09_interview_guide.md](09_ood_guide/09_interview_guide.md)（45分钟框架 + 话术），做题时对照 [09_ood_guide/13_uml_cheatsheet.md](09_ood_guide/13_uml_cheatsheet.md) 练习白板画图。

| 文件 | 题目 | 核心考点 |
|------|------|---------|
| [00 方法论](08_ood/00_ood_framework.md) | OOD 面试框架 | SOLID 原则 / 常用设计模式 / 类图速记 |
| [01 停车场](08_ood/01_parking_lot.md) | Parking Lot | 多车型多态 / Strategy 计费 / 车位分配 |
| [02 LRU Cache](08_ood/02_lru_cache.md) | LRU Cache | HashMap + 双向链表 / O(1) get & put |
| [03 电梯系统](08_ood/03_elevator.md) | Elevator System | 状态机 / SCAN 调度 / Strategy 模式 |
| [04 扑克牌游戏](08_ood/04_card_game.md) | Card Game (Blackjack) | 抽象层次 / Template Method / Ace 点数逻辑 |
| [05 自动贩卖机](08_ood/05_vending_machine.md) | Vending Machine | State 模式 / 状态机 / 预建实例切换 |
| [06 图书馆系统](08_ood/06_library_system.md) | Library System | BookCatalog vs BookItem / 预约队列 / 逾期罚款 |
| [07 国际象棋](08_ood/07_chess_game.md) | Chess Game | 棋子多态 / 合法走法验证 / 将军检测 / Board克隆模拟 |
| [08 HashMap](08_ood/08_hashmap.md) | HashMap | 链地址法 / 动态 rehash / 负载因子 / 为何用 2 的幂次 |
| [09 文件系统](08_ood/09_file_system.md) | File System | Composite 模式（File+Directory 同接口）/ 递归 size / 路径解析 |
| [10 网约车](08_ood/10_ride_sharing_ood.md) | Ride Sharing | 行程状态机 / 车型多态 / 司机匹配 Strategy / 计价 |
| [11 酒店预订](08_ood/11_hotel_booking.md) | Hotel Booking | 房间类型多态 / 防超卖乐观锁 / 预订状态机 / 计价 Strategy |
| [12 外卖系统](08_ood/12_food_delivery.md) | Food Delivery | 订单状态机 / 菜单 Composite / Observer 推送通知 / 骑手分配 |
| [13 购物车](08_ood/13_shopping_cart.md) | Shopping Cart | 实物 vs 数字商品多态 / 折扣 Strategy / 结账防超卖 |
| [14 阻塞队列](08_ood/14_blocking_queue.md) | Blocking Queue | Producer-Consumer / 条件变量 / Java ReentrantLock 实现 |
| [15 Twitter OOD](08_ood/15_twitter_ood.md) | Twitter（OOD层）| Tweet 多态（原推/转推/回复）/ Timeline 推拉模式 / 推荐关注 |
| [16 聊天室](08_ood/16_chat_room.md) | Chat Room | Mediator 模式 / Observer 广播 / 消息多态 / 权限管理 |
| [17 日志聚合器](08_ood/17_logger.md) | Logger | Singleton / 责任链（Handler链）/ 格式化 Strategy / 采样 |
| [18 ATM 机](08_ood/18_atm_machine.md) | ATM Machine | State 模式（Idle/CardInserted/PINVerified/Dispensing）/ 金额精度 / 事务日志 |
| [19 贪吃蛇](08_ood/19_snake_game.md) | Snake Game | Queue 模拟蛇身 / O(1) 碰撞检测（HashSet）/ 方向约束 / 胜利条件 |
| [20 井字棋](08_ood/20_tic_tac_toe.md) | Tic-Tac-Toe | N×N 棋盘 / O(1) 胜负检测（行列对角线计数器）/ 玩家多态 / Minimax AI |
| [21 电影院订票](08_ood/21_cinema_booking.md) | Cinema Booking | 2D 座位图 / 乐观锁防超卖（CAS）/ 预订超时释放 / 区域差异定价 |
| [22 LFU Cache](08_ood/22_lfu_cache.md) | LFU Cache | 双 HashMap + LinkedHashSet / minFreq 追踪 / O(1) get&put / 同频 LRU |

---

## 系统设计

**核心线索**：一个系统在规模变大、节点增多、网络不可靠时，会在哪里崩溃？我们怎么应对？

### 从这里开始

| 文件 | 说明 |
|------|------|
| [如何使用这套文档](00_how_to_use.md) | 三条阅读路线（从零入门 / 查漏补缺 / 面试冲刺），预计时间 |

---


### 01 基础概念

系统设计的语言基础。不懂这里的词，后面的讨论寸步难行。

| 文件 | 核心内容 |
|------|---------|
| [01 度量体系](01_fundamentals/01_metrics.md) | Latency / Throughput / Availability；SLA/SLO/SLI；P99 vs 平均值 |
| [08 输入URL后发生了什么](01_fundamentals/08_url_to_page.md) | DNS递归解析；TCP三次握手；TLS握手；HTTP/2多路复用；浏览器渲染流水线；CRP优化 |
| [02 CAP 定理](01_fundamentals/02_cap_theorem.md) | 分区时只能选 C 或 A；CP vs AP 系统举例；PACELC 扩展 |
| [03 ACID 与 BASE](01_fundamentals/03_acid_base.md) | 事务四特性；BASE 的含义；两者适用场景 |
| [04 网络基础](01_fundamentals/04_network_basics.md) | TCP/UDP/HTTP；DNS 解析链路；CDN 工作原理；**L4 vs L7 负载均衡；算法；Anycast** |
| [05 认证与安全](01_fundamentals/05_auth_security.md) | Session vs JWT；OAuth2；SQL 注入 / XSS / CSRF；**API 签名防重放；PII 加密；mTLS；Secret 管理** |
| [06 微服务与部署](01_fundamentals/06_microservices_deployment.md) | 服务发现；Kubernetes 核心概念；蓝绿/金丝雀部署；API Gateway vs Service Mesh |
| [07 核心数据结构](01_fundamentals/07_data_structures_for_sd.md) | Bloom Filter / Skip List / LSM Tree / Merkle Tree / B+ Tree / **HyperLogLog / Count-Min Sketch** — 底层原理、面试场景决策树 |
| [09 一致性哈希深度解析](01_fundamentals/09_consistent_hashing.md) | 哈希环原理；虚拟节点（VNode）数学分析；扩缩容迁移流程；TypeScript 实现；vs Range Sharding 对比 |

---

### 02 存储系统

面试中"选什么数据库"是高频决策。这里建立选型框架，而不是罗列参数。

| 文件 | 核心内容 |
|------|---------|
| [00 选型框架](02_storage/00_choice_framework.md) | 决策树：什么场景用什么存储，以及为什么 |
| [01 关系型数据库](02_storage/01_rdbms.md) | B+Tree 索引；主从复制；水平分片；一致性哈希 |
| [02 NoSQL](02_storage/02_nosql.md) | KV / 文档 / 宽列 / 图 四类对比；Cassandra LSM Tree；适用场景 |
| [03 缓存](02_storage/03_cache.md) | Cache-Aside / Write-Through / Write-Back；缓存穿透 / 击穿 / 雪崩及防御 |
| [04 对象存储](02_storage/04_object_storage.md) | S3 架构；分片上传；Pre-signed URL；冷热分层 |
| [05 Schema 设计](02_storage/05_schema_design.md) | 规范化 vs 反范式化；高频 Schema 案例（电商/社交/RBAC/审计日志/树形结构）；索引设计；分片键选择 |

---

### 03 通信方式

客户端和服务之间、服务和服务之间怎么通信——选择不同，设计差异巨大。

| 文件 | 核心内容 |
|------|---------|
| [01 同步通信](03_communication/01_sync.md) | REST vs gRPC vs GraphQL；API Gateway；幂等性设计 |
| [02 异步通信](03_communication/02_async.md) | Kafka vs RabbitMQ；Exactly-Once；Outbox 模式；消息积压处理 |
| [03 实时通信](03_communication/03_realtime.md) | WebSocket / SSE / 长轮询对比；WebSocket 横向扩展；心跳与离线消息 |
| [04 API 设计](03_communication/04_api_design.md) | 分页策略（Offset/Cursor/Keyset）；版本管理；幂等键；RESTful 规范；请求签名；tRPC 类型安全 API |

---

### 04 分布式系统

面试的深水区。每一篇都是面试官喜欢追问的方向。

| 文件 | 核心内容 |
|------|---------|
| [00 权衡图谱](04_distributed/00_tradeoffs.md) | 所有核心 tradeoff 的汇总索引，快速建立决策框架 |
| [01 一致性模型](04_distributed/01_consistency_models.md) | 线性一致性 → 因果一致性 → 最终一致性；Read-Your-Writes；Quorum |
| [02 分布式事务](04_distributed/02_distributed_tx.md) | 2PC 的问题；Saga（编排式 vs 协调式）；TCC；Outbox 模式 |
| [03 共识算法](04_distributed/03_consensus.md) | Raft：Leader 选举 + 日志复制；与 Paxos 对比；etcd/ZooKeeper 应用 |
| [04 容错与弹性](04_distributed/04_fault_tolerance.md) | 熔断器三状态；令牌桶 / 漏桶 / 滑动窗口限流；指数退避；幂等键 |
| [05 ID 生成](04_distributed/05_id_generation.md) | UUID / DB 自增 / Snowflake / 号段模式；时钟回拨问题 |
| [06 分布式锁](04_distributed/06_distributed_lock.md) | Redis SETNX + Lua；ZooKeeper 临时顺序节点；Fencing Token 防脑裂 |
| [07 多地域架构 + RTO/RPO](04_distributed/07_multi_region.md) | Active-Active vs Passive；写冲突解决；GeoDNS；故障切换 Playbook |
| [08 Event Sourcing](04_distributed/08_event_sourcing.md) | 事件溯源原理；聚合根+命令+事件；Projection；Snapshot；CQRS 组合；Schema Evolution |

---

### 05 方法论

怎么在 45 分钟内有条理地完成一次设计，以及面试官实际在打什么分。

### 设计流程

| 文件 | 核心内容 |
|------|---------|
| [01 面试框架](05_methodology/design_process/01_framework.md) | 45 分钟时间分配；各阶段该说什么；如何主动引导讨论 |
| [02 容量估算](05_methodology/design_process/02_capacity_estimation.md) | QPS / 存储 / 带宽的估算公式；常用数量级记忆 |
| [03 权衡分析](05_methodology/design_process/03_tradeoff_analysis.md) | 如何结构化地表达"我选 A 而不是 B，因为..." |
| [04 反模式与被追问应对](05_methodology/design_process/04_antipatterns_and_followup.md) | 7 个高频反模式；4 类追问的标准应对框架；面试前速查清单 |

### 参考速查

| 文件 | 核心内容 |
|------|---------|
| [01 数字速查](05_methodology/reference/01_numbers_cheatsheet.md) | 各类系统的典型数量级（必背）：DB QPS、缓存延迟、磁盘速度等 |
| [02 术语表](05_methodology/reference/02_glossary.md) | 100+ 术语的精确定义，面试前扫一遍 |
| [03 设计模式](05_methodology/reference/03_patterns.md) | 读写分离 / CQRS / Fanout / Event Sourcing 的结构与适用场景 |
| [04 可观测性](05_methodology/reference/04_observability.md) | Metrics / Logs / Traces 三支柱；SLO 告警；Error Budget；Burn Rate |
| [05 概念关系图](05_methodology/reference/05_concept_map.md) | **10 张 Mermaid 图**：所有概念之间的依赖和关联网络 |
| [06 面试官评分标准](05_methodology/reference/06_interviewer_rubric.md) | 面试官真正在评的 4 个维度；L4/L5/L6 的差距在哪里；常见失分模式 |
| [07 技术横向对比速查表](05_methodology/reference/07_tech_comparison.md) | **7 张大表**：消息队列 / 数据库 / 缓存策略 / 通信协议 / LB 算法 / 存储类型 / 一致性模型 |
| [08 流处理与数据管道](05_methodology/reference/08_data_pipeline.md) | 批处理 vs 流处理；Lambda vs Kappa 架构；Spark 基础；Flink 时间语义 / Watermark / 窗口 |
| [09 Elasticsearch 深度解析](05_methodology/reference/09_elasticsearch_deep.md) | Lucene Segment；Posting List；NRT refresh vs flush；BM25；深度分页；写入/查询优化 |

---

### 06 实战案例

每篇 case study 有完整设计、核心难点、面试问答。

**练习方式**：先用[空白模板](06_case_studies/00_practice_mode.md)限时 45 分钟独立设计，再打开对应文档对比。

| 文件 | 系统 | 核心考点 |
|------|------|---------|
| [00 面试套路](06_case_studies/00_interview_patterns.md) | — | 各类题型的通用解题模板 |
| [00 空白练习模板](06_case_studies/00_practice_mode.md) | 全部 17 题 | **主动回忆用**，只有问题没有答案 |
| [01 URL 短链](06_case_studies/01_url_shortener.md) | bit.ly | 短码生成（哈希 vs 计数器）；301 vs 302；DB 分片 |
| [02 限流器](06_case_studies/02_rate_limiter.md) | API Gateway | 令牌桶 / 滑动窗口；Redis Lua；多数据中心 |
| [03 搜索自动补全](06_case_studies/03_search_autocomplete.md) | Google 搜索框 | Trie 结构；top-K 缓存；实时 vs 批量更新 |
| [04 通知系统](06_case_studies/04_notification_system.md) | Push/Email/SMS | 多渠道多队列；幂等去重；第三方服务故障处理 |
| [05 News Feed](06_case_studies/05_news_feed.md) | 朋友圈 / Twitter | 推模式 vs 拉模式 vs 混合；大 V 问题；Timeline 缓存 |
| [06 聊天系统](06_case_studies/06_chat_system.md) | 微信 / WhatsApp | WebSocket 扩展；消息有序；离线消息；已读回执 |
| [07 分布式缓存](06_case_studies/07_distributed_cache.md) | Redis Cluster | 一致性哈希 + 虚拟节点；热点 Key；驱逐策略 |
| [08 分布式存储](06_case_studies/08_distributed_storage.md) | S3 / HDFS | 元数据与数据分离；分片上传；跨 AZ 多副本 |
| [09 视频流媒体](06_case_studies/09_video_streaming.md) | YouTube / Netflix | 异步转码；自适应码率 HLS/DASH；CDN 策略 |
| [10 打车系统](06_case_studies/10_ride_sharing.md) | Uber / 滴滴 | 位置高频写入；Geohash 附近查询；司机匹配 |
| [11 广告点击聚合](06_case_studies/11_ad_click_aggregator.md) | 广告平台 | Lambda vs Kappa；时间窗口聚合；数据倾斜；对账 |
| [12 网络爬虫](06_case_studies/12_web_crawler.md) | Googlebot | URL Frontier 优先级队列；布隆过滤器去重；spider trap |
| [13 全文搜索引擎](06_case_studies/13_search_engine.md) | Elasticsearch | 倒排索引；BM25；NRT 机制；深度分页陷阱 |
| [14 支付系统](06_case_studies/14_payment_system.md) | Stripe / 支付宝 | 幂等键；复式记账 Ledger；Webhook 处理；对账 |
| [15 附近搜索](06_case_studies/15_proximity_service.md) | Yelp / 大众点评 | Geohash 9 格查询；静态 POI vs 动态位置；地理分片 |
| [16 酒店预订](06_case_studies/16_hotel_reservation.md) | Booking.com | 库存并发防超卖；乐观锁 vs 悲观锁；Saga 取消补偿 |
| [17 协同文档编辑](06_case_studies/17_collaborative_editor.md) | Google Docs | OT vs CRDT；WebSocket 实时同步；光标 ID；离线合并；版本历史 |
| [18 股票交易所](06_case_studies/18_stock_exchange.md) | 低延迟交易所 | 撮合引擎；订单簿（红黑树）；LMAX Disruptor；μs级延迟；风控；WAL故障恢复 |
| [19 分布式 KV 存储](06_case_studies/19_key_value_store.md) | Dynamo / DynamoDB | 一致性哈希虚拟节点；N/W/R Quorum；向量时钟冲突；Gossip 故障检测；Merkle 反熵 |
| [20 Google Drive](06_case_studies/20_google_drive.md) | Google Drive / Dropbox | 4MB 分块上传；SHA-256 内容寻址；增量同步；三路合并冲突；WebSocket/SSE 通知 |
| [21 附近的人](06_case_studies/21_nearby_friends.md) | 微信附近的人 | Redis GEO；Pub/Sub 实时推送；WebSocket Fanout；隐私黑名单；vs 附近搜索对比 |
| [22 地图导航](06_case_studies/22_google_maps.md) | Google Maps | 地图瓦片 Z/X/Y；路网邻接表；A* vs Contraction Hierarchies；GPS探针→Kafka→Flink 实时路况 |
| [23 分布式消息队列](06_case_studies/23_distributed_mq.md) | 设计 Kafka | 分区路由；Commit Log 顺序写；零拷贝 sendfile；ISR 副本协议；消费者组再平衡；日志压缩 |
| [24 监控告警系统](06_case_studies/24_metrics_monitoring.md) | Prometheus + Alertmanager | Pull vs Push；时序数据模型（TSID+标签倒排索引）；Gorilla 压缩；降采样；告警状态机 |
| [25 分布式邮件服务](06_case_studies/25_distributed_email.md) | SendGrid / SES | SMTP 发送流程；SPF/DKIM/DMARC；软退信 vs 硬退信；退订抑制列表；开信追踪像素 |
| [26 游戏排行榜](06_case_studies/26_gaming_leaderboard.md) | 实时排行榜 | Redis ZSet（ZADD GT/ZREVRANK）；虚拟节点分片；Top 100 缓存刷新；好友榜推拉结合 |
| [27 数字钱包](06_case_studies/27_digital_wallet.md) | 支付宝 / PayPal | 复式记账；幂等转账（idempotency_key）；乐观锁防超扣；本地消息表跨分片；日终对账 |

---

### 知识图谱

不知道从哪里看起？先看这张图：

```
需求 → 规模 → 读多？→ 缓存 / 读写分离
                   写多？→ Kafka / NoSQL / 分片
                   数据大？→ 分片 + 对象存储

需求 → 可靠性 → 节点故障 → 复制 + 共识（Raft）
                 级联故障 → 熔断 + 限流
                 重复请求 → 幂等键

需求 → 一致性 → 强一致 → 分布式锁 / 2PC
                 跨服务事务 → Saga + Outbox
                 最终一致 → 事件驱动 + 补偿
```

完整的概念关系图（Mermaid 可视化）见 → [05_concept_map.md](05_methodology/reference/05_concept_map.md)

---

### 07 真实公司架构拆解

理解真实系统如何演进，比死记架构图更有价值。每篇聚焦**为什么这么做**，而不是结果是什么。

| 文件 | 公司 | 核心看点 |
|------|------|---------|
| [01 Twitter](07_real_world/01_twitter.md) | Twitter / X | 推拉混合 Fanout；Timeline 预计算；Snowflake ID；大 V 问题演进 |
| [02 Uber](07_real_world/02_uber.md) | Uber / 滴滴 | H3 六边形索引；ETA 匹配；CAS 防双派；动态定价流处理 |
| [03 Netflix](07_real_world/03_netflix.md) | Netflix | Open Connect 自建 CDN；预推送；HLS 自适应码率；混沌工程 |
| [04 Discord](07_real_world/04_discord.md) | Discord | MongoDB→Cassandra→ScyllaDB 迁移；SFU 语音；大群 Fanout 分片 |
| [05 WhatsApp](07_real_world/05_whatsapp.md) | WhatsApp | Erlang 200 万连接/台；E2EE Signal Protocol；消息不持久化设计 |

---

## 10 前端系统设计专题

面向 TypeScript / 全栈工程师的补充模块，覆盖后端视角看不到的前端规模化挑战。

| 文件 | 主题 | 核心内容 |
|------|------|---------|
| [00 专题概览](10_frontend_system_design/00_overview.md) | 总览 | 前端系统设计 vs 后端对比；45分钟框架；差异化得分点 |
| [01 SSR/SSG/ISR](10_frontend_system_design/01_ssr_ssg_isr.md) | 渲染策略 | 四种模式对比；流式 SSR；ISR 失效策略；CDN 分层缓存；缓存雪崩防御 |
| [02 微前端架构](10_frontend_system_design/02_micro_frontend.md) | 应用拆分 | Module Federation 配置；路由治理；共享状态方案；样式隔离；独立部署流水线 |
| [03 BFF 架构](10_frontend_system_design/03_bff.md) | Backend for Frontend | GraphQL BFF + DataLoader（N+1解决）；tRPC 类型安全 API；REST vs GraphQL 选型 |
| [04 前端性能监控](10_frontend_system_design/04_frontend_monitoring.md) | RUM | Core Web Vitals 采集 SDK；Beacon API；ClickHouse 时序存储；环比异常告警 |
| [05 框架横向对比](10_frontend_system_design/05_framework_comparison.md) | 生态选型 | Next.js / Remix+RR v7 / TanStack Start / Astro / Nuxt / SvelteKit；选型决策树 |
| [06 状态管理](10_frontend_system_design/06_state_management.md) | 状态架构 | 服务端状态（TanStack Query/SWR）vs 客户端状态（Zustand/Jotai/Redux Toolkit）；乐观更新 |
| [07 性能优化](10_frontend_system_design/07_performance_optimization.md) | 加载+运行时 | 代码分割；bundle 优化；虚拟滚动（TanStack Virtual）；Web Worker；关键渲染路径 |
| [08 前端安全](10_frontend_system_design/08_security.md) | 安全 | XSS/CSRF/CSP；DOMPurify；NextAuth.js/Clerk 认证方案；HttpOnly Cookie vs JWT 存储 |
| [09 Design System](10_frontend_system_design/09_design_system.md) | 设计系统 | Design Token 3 层模型；CSS Variables 主题；Style Dictionary；Radix UI；shadcn/ui；Storybook；tsup 构建；Changesets |
| [10 PWA & 离线](10_frontend_system_design/10_pwa_offline.md) | 离线能力 | Service Worker 生命周期；5 种缓存策略；Workbox VitePWA；Dexie.js IndexedDB；Background Sync；Push Notifications |
| [11 Feature Flag & A/B](10_frontend_system_design/11_feature_flag_ab.md) | 功能开关 | 确定性 Hash 分桶；Client/Server/Edge 分流；FOUC 防护；GrowthBook vs LaunchDarkly；A/B 统计显著性 |
| [12 CI/CD & 测试](10_frontend_system_design/12_cicd_testing.md) | 工程质量 | Vitest；Testing Library；MSW v2；Playwright POM；Chromatic 视觉回归；Lighthouse CI；GitHub Actions 流水线 |
| [13 国际化（i18n）](10_frontend_system_design/13_i18n.md) | 多语言 | next-intl ICU 格式；RTL CSS 逻辑属性；Intl API（日期/数字/相对时间）；Lokalise 翻译工作流 |
| [14 实时通信](10_frontend_system_design/14_realtime_frontend.md) | 实时 | WebSocket 重连（指数退避）；客户端消息队列；乐观 UI；Presence 在线状态；SSE；网络感知 |
| [15 文件上传](10_frontend_system_design/15_file_upload.md) | 大文件 | 分块上传断点续传；Pre-signed URL 直传 S3；tus 协议；客户端图片压缩；并发控制；上传安全 |
| [16 浏览器存储](10_frontend_system_design/16_browser_storage.md) | 存储选型 | Cookie/LocalStorage/SessionStorage/IndexedDB/Cache API/OPFS 对比；选型决策树；配额管理 |
| [17 SEO 架构](10_frontend_system_design/17_seo_architecture.md) | SEO | Meta 标签；Open Graph；JSON-LD 结构化数据；动态 OG 图片（@vercel/og）；Sitemap；Canonical URL |
| [18 项目架构](10_frontend_system_design/18_project_architecture.md) | 可维护性 | Feature-Sliced Design；Turborepo/Nx Monorepo；Barrel Exports 反模式；ESLint 层级约束；dependency-cruiser |
| [19 图片与媒体优化](10_frontend_system_design/19_image_optimization.md) | 资源优化 | WebP/AVIF 格式选型；响应式 srcset；Next/Image；CDN 图片变换（Cloudinary/Imgix）；懒加载；视频优化 |
| [20 错误监控与边界](10_frontend_system_design/20_error_monitoring.md) | 稳定性 | React Error Boundary；全局错误捕获；Sentry 集成与 Source Map；错误告警分级；react-error-boundary |
| [21 动画性能](10_frontend_system_design/21_animation_performance.md) | 渲染性能 | 浏览器渲染流水线；transform/opacity GPU 加速；will-change；rAF；WAAPI；Framer Motion；FLIP 技术 |
| [22 可访问性（a11y）](10_frontend_system_design/22_accessibility.md) | 无障碍 | WCAG 2.1 POUR；语义化 HTML；ARIA；键盘导航与焦点陷阱；颜色对比度；表单 a11y；axe 自动化检测 |

---
