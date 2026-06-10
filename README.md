# 系统设计面试笔记

面向面试的系统设计自学文档。61 篇，覆盖从基础概念到完整系统设计的全链路。

**核心线索**：一个系统在规模变大、节点增多、网络不可靠时，会在哪里崩溃？我们怎么应对？

---

## 从这里开始

| 文件 | 说明 |
|------|------|
| [如何使用这套文档](00_how_to_use.md) | 三条阅读路线（从零入门 / 查漏补缺 / 面试冲刺），预计时间 |

---

## 01 基础概念

系统设计的语言基础。不懂这里的词，后面的讨论寸步难行。

| 文件 | 核心内容 |
|------|---------|
| [01 度量体系](01_fundamentals/01_metrics.md) | Latency / Throughput / Availability；SLA/SLO/SLI；P99 vs 平均值 |
| [02 CAP 定理](01_fundamentals/02_cap_theorem.md) | 分区时只能选 C 或 A；CP vs AP 系统举例；PACELC 扩展 |
| [03 ACID 与 BASE](01_fundamentals/03_acid_base.md) | 事务四特性；BASE 的含义；两者适用场景 |
| [04 网络基础](01_fundamentals/04_network_basics.md) | TCP/UDP/HTTP；DNS 解析链路；CDN 工作原理；**L4 vs L7 负载均衡；算法；Anycast** |
| [05 认证与安全](01_fundamentals/05_auth_security.md) | Session vs JWT；OAuth2；SQL 注入 / XSS / CSRF；**API 签名防重放；PII 加密；mTLS；Secret 管理** |
| [06 微服务与部署](01_fundamentals/06_microservices_deployment.md) | 服务发现；Kubernetes 核心概念；蓝绿/金丝雀部署；API Gateway vs Service Mesh |
| [07 核心数据结构](01_fundamentals/07_data_structures_for_sd.md) | Bloom Filter / Skip List / LSM Tree / Merkle Tree / B+ Tree — 底层原理与应用场景 |

---

## 02 存储系统

面试中"选什么数据库"是高频决策。这里建立选型框架，而不是罗列参数。

| 文件 | 核心内容 |
|------|---------|
| [00 选型框架](02_storage/00_choice_framework.md) | 决策树：什么场景用什么存储，以及为什么 |
| [01 关系型数据库](02_storage/01_rdbms.md) | B+Tree 索引；主从复制；水平分片；一致性哈希 |
| [02 NoSQL](02_storage/02_nosql.md) | KV / 文档 / 宽列 / 图 四类对比；Cassandra LSM Tree；适用场景 |
| [03 缓存](02_storage/03_cache.md) | Cache-Aside / Write-Through / Write-Back；缓存穿透 / 击穿 / 雪崩及防御 |
| [04 对象存储](02_storage/04_object_storage.md) | S3 架构；分片上传；Pre-signed URL；冷热分层 |

---

## 03 通信方式

客户端和服务之间、服务和服务之间怎么通信——选择不同，设计差异巨大。

| 文件 | 核心内容 |
|------|---------|
| [01 同步通信](03_communication/01_sync.md) | REST vs gRPC vs GraphQL；API Gateway；幂等性设计 |
| [02 异步通信](03_communication/02_async.md) | Kafka vs RabbitMQ；Exactly-Once；Outbox 模式；消息积压处理 |
| [03 实时通信](03_communication/03_realtime.md) | WebSocket / SSE / 长轮询对比；WebSocket 横向扩展；心跳与离线消息 |

---

## 04 分布式系统

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

---

## 05 方法论

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

---

## 06 实战案例

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

---

## 知识图谱

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

## 07 真实公司架构拆解

理解真实系统如何演进，比死记架构图更有价值。每篇聚焦**为什么这么做**，而不是结果是什么。

| 文件 | 公司 | 核心看点 |
|------|------|---------|
| [01 Twitter](07_real_world/01_twitter.md) | Twitter / X | 推拉混合 Fanout；Timeline 预计算；Snowflake ID；大 V 问题演进 |
| [02 Uber](07_real_world/02_uber.md) | Uber / 滴滴 | H3 六边形索引；ETA 匹配；CAS 防双派；动态定价流处理 |
| [03 Netflix](07_real_world/03_netflix.md) | Netflix | Open Connect 自建 CDN；预推送；HLS 自适应码率；混沌工程 |
| [04 Discord](07_real_world/04_discord.md) | Discord | MongoDB→Cassandra→ScyllaDB 迁移；SFU 语音；大群 Fanout 分片 |
| [05 WhatsApp](07_real_world/05_whatsapp.md) | WhatsApp | Erlang 200 万连接/台；E2EE Signal Protocol；消息不持久化设计 |
