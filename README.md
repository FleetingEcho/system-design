# 系统设计 + OOD 面试笔记

全程 TypeScript，分前端 / 后端两条线，覆盖系统设计与面向对象设计的完整面试准备。

```
├── frontend/
│   ├── system_design/   前端系统设计（27 篇）
│   └── ood/             前端 OOD 题（6 题）
├── backend/
│   ├── nodejs/          Node.js 运行时与服务架构（7 篇）★ 新增
│   ├── system_design/   后端系统设计（50+ 篇）
│   └── ood/
│       ├── problems/    OOD 实战题（23 道）
│       └── guide/       OOD 理论精讲（15 篇）
├── fullstack/           全栈架构（1 篇）★ 新增
└── 00_how_to_use.md
```

---

## 前端

### 前端系统设计

面向 TypeScript / 全栈工程师，覆盖后端视角看不到的前端规模化挑战。

| 文件 | 主题 | 核心内容 |
|------|------|---------|
| [00 专题概览](frontend/system_design/00_overview.md) | 总览 | 前端 vs 后端系统设计差异；45 分钟面试框架；差异化得分点 |
| [01 SSR/SSG/ISR](frontend/system_design/01_ssr_ssg_isr.md) | 渲染策略 | 四种模式对比；流式 SSR；ISR 失效策略；CDN 分层缓存；缓存雪崩防御 |
| [02 微前端架构](frontend/system_design/02_micro_frontend.md) | 应用拆分 | Module Federation；路由治理；共享状态；样式隔离；独立部署流水线 |
| [03 BFF 架构](frontend/system_design/03_bff.md) | Backend for Frontend | GraphQL BFF + DataLoader（N+1）；tRPC 类型安全 API；REST vs GraphQL 选型 |
| [04 前端性能监控](frontend/system_design/04_frontend_monitoring.md) | RUM | Core Web Vitals 采集 SDK；Beacon API；ClickHouse 时序存储；环比异常告警 |
| [05 框架横向对比](frontend/system_design/05_framework_comparison.md) | 生态选型 | Next.js / Remix / TanStack Start / Astro / Nuxt / SvelteKit；选型决策树 |
| [06 状态管理](frontend/system_design/06_state_management.md) | 状态架构 | TanStack Query / SWR（服务端状态）；Zustand / Jotai / Redux Toolkit（客户端）；乐观更新 |
| [07 性能优化](frontend/system_design/07_performance_optimization.md) | 加载+运行时 | 代码分割；bundle 优化；虚拟滚动（TanStack Virtual）；Web Worker；关键渲染路径 |
| [08 前端安全](frontend/system_design/08_security.md) | 安全 | XSS/CSRF/CSP；DOMPurify；NextAuth.js / Clerk；HttpOnly Cookie vs JWT |
| [09 Design System](frontend/system_design/09_design_system.md) | 设计系统 | Design Token 3 层模型；CSS Variables；Style Dictionary；Radix UI；shadcn/ui；Storybook；Changesets |
| [10 PWA & 离线](frontend/system_design/10_pwa_offline.md) | 离线能力 | Service Worker 生命周期；5 种缓存策略；Workbox；Dexie.js IndexedDB；Background Sync；Push Notifications |
| [11 Feature Flag & A/B](frontend/system_design/11_feature_flag_ab.md) | 功能开关 | 确定性 Hash 分桶；Client/Server/Edge 分流；FOUC 防护；GrowthBook vs LaunchDarkly；A/B 统计显著性 |
| [12 CI/CD & 测试](frontend/system_design/12_cicd_testing.md) | 工程质量 | Vitest；Testing Library；MSW v2；Playwright POM；Chromatic 视觉回归；Lighthouse CI；GitHub Actions |
| [13 国际化（i18n）](frontend/system_design/13_i18n.md) | 多语言 | next-intl ICU 格式；RTL CSS 逻辑属性；Intl API；Lokalise 翻译工作流；时区处理 |
| [14 实时通信](frontend/system_design/14_realtime_frontend.md) | 实时 | WebSocket 重连（指数退避）；客户端消息队列；乐观 UI；Presence 在线状态；SSE |
| [15 文件上传](frontend/system_design/15_file_upload.md) | 大文件 | 分块上传断点续传；Pre-signed URL 直传 S3；tus 协议；客户端图片压缩；上传安全 |
| [16 浏览器存储](frontend/system_design/16_browser_storage.md) | 存储选型 | Cookie/LocalStorage/SessionStorage/IndexedDB/Cache API/OPFS 对比；选型决策树；配额管理 |
| [17 SEO 架构](frontend/system_design/17_seo_architecture.md) | SEO | Meta 标签；Open Graph；JSON-LD 结构化数据；动态 OG 图片；Sitemap；Canonical URL |
| [18 项目架构](frontend/system_design/18_project_architecture.md) | 可维护性 | Feature-Sliced Design；Turborepo/Nx Monorepo；Barrel Exports 反模式；ESLint 层级约束 |
| [19 图片与媒体优化](frontend/system_design/19_image_optimization.md) | 资源优化 | WebP/AVIF；响应式 srcset；Next/Image；CDN 图片变换（Cloudinary/Imgix）；懒加载；视频优化 |
| [20 错误监控与边界](frontend/system_design/20_error_monitoring.md) | 稳定性 | React Error Boundary；全局错误捕获；Sentry + Source Map；错误告警分级 |
| [21 动画性能](frontend/system_design/21_animation_performance.md) | 渲染性能 | 浏览器渲染流水线；transform/opacity GPU 加速；will-change；rAF；WAAPI；Framer Motion；FLIP |
| [22 可访问性（a11y）](frontend/system_design/22_accessibility.md) | 无障碍 | WCAG 2.1 POUR；语义化 HTML；ARIA；键盘导航与焦点陷阱；颜色对比度；axe 自动检测 |
| [23 Feed 流设计](frontend/system_design/23_feed_design.md) | 信息流 | 虚拟列表；无限滚动；SSE 实时新帖；乐观更新（点赞回滚）；cursor 分页 |
| [24 协作编辑器](frontend/system_design/24_collaborative_editor_frontend.md) | 多人协作 | CRDT（Yjs）；光标同步；WebSocket 离线队列；冲突合并 |
| [25 视频播放器](frontend/system_design/25_video_player.md) | 流媒体 | HLS/DASH；自适应码率 ABR；hls.js；缓冲策略；Quality Selector；续播；PiP |
| [26 Next.js App Router](frontend/system_design/26_nextjs_app_router.md) | App Router | RSC vs Client Component 边界；Server Actions；四层缓存；Streaming + Suspense；PPR |

### 前端 OOD

| 文件 | 题目 | 考察点 |
|------|------|--------|
| [01 EventEmitter](frontend/ood/01_event_emitter.md) | 实现 EventEmitter | 观察者模式；on/off/once；内存泄漏防护；泛型类型安全 |
| [02 Promise](frontend/ood/02_promise.md) | 实现 Promise（Promises/A+） | 状态机；then 链；微任务队列；循环引用；all/race/allSettled |
| [03 SPA Router](frontend/ood/03_spa_router.md) | 设计前端路由 | History API；路径→正则；嵌套路由；导航守卫；懒加载 |
| [04 Virtual DOM](frontend/ood/04_virtual_dom.md) | Virtual DOM + Diff | VNode；createElement；同层 Diff；key 优化；O(n) |
| [05 Mini Redux](frontend/ood/05_mini_redux.md) | 设计 Mini Redux | createStore；中间件链；combineReducers；thunk；时间旅行 |
| [06 工具函数](frontend/ood/06_utils.md) | Debounce/Throttle/LazyLoad/Memoize | 闭包；定时器；IntersectionObserver；LRU 缓存 |

---

## 全栈

### Monorepo 与端到端架构

| 文件 | 主题 | 核心内容 |
|------|------|---------|
| [01 Monorepo 架构](fullstack/01_monorepo_architecture.md) | 全栈整合 | Turborepo；Zod 单一真相来源；tRPC 端到端类型安全；Prisma 类型层；共享 UI 包 |

---

## 后端

### Node.js 运行时与服务架构

TypeScript + Node.js 服务的完整知识体系，从运行时原理到生产级实践。

| 文件 | 主题 | 核心内容 |
|------|------|---------|
| [01 Event Loop](backend/nodejs/01_event_loop.md) | 运行时原理 | 六个阶段；process.nextTick vs Promise；setImmediate vs setTimeout；libuv 线程池；Event Loop 阻塞 |
| [02 Streams](backend/nodejs/02_streams.md) | 流与背压 | Readable/Writable/Transform；背压机制；pipeline vs pipe；大文件处理；CSV 解析 |
| [03 服务分层架构](backend/nodejs/03_service_architecture.md) | 项目结构 | Routes/Controllers/Services/Repositories 分层；TypeScript DI；接口隔离；单测策略 |
| [04 错误处理](backend/nodejs/04_error_handling.md) | 错误体系 | AppError 层次；全局 Express error handler；Zod 错误转换；async 路由捕获；进程级兜底 |
| [05 NestJS 架构](backend/nodejs/05_nestjs_architecture.md) | NestJS | Module/DI 生命周期；Guard/Interceptor/Pipe/Filter 执行顺序；REQUEST scope；自定义装饰器 |
| [06 生产级模式](backend/nodejs/06_production_patterns.md) | 生产实践 | Graceful Shutdown（SIGTERM）；Pino 结构化日志；AsyncLocalStorage 请求追踪；Health Check；Zod Config |
| [07 性能与扩展](backend/nodejs/07_performance_scaling.md) | 性能 | Worker Threads；Cluster；内存泄漏排查；Event Loop 监控；Profiling；Worker vs Cluster 决策 |

### OOD 理论精讲

从零系统学习面向对象设计，全程 TypeScript。

**学习路径**：

```
基础四大特性（01-04）→ SOLID 原则（05）→ 三大类设计模式（06-08）
→ 面试方法论（09）→ TS 高级类型（10）→ 反模式（11）→ GRASP（12）
→ UML 速查（13）→ 函数式 vs OOP（14）→ DDD 基础（15）
```

| 文件 | 主题 | 核心内容 |
|------|------|---------|
| [00 什么是 OOD](backend/ood/guide/00_what_is_ood.md) | 概述 | 类 vs 对象；三种类关系；interface vs abstract；四大支柱导览 |
| [01 封装](backend/ood/guide/01_encapsulation.md) | Encapsulation | 访问修饰符；不变量保护；Getter-Setter 正确用法；不泄露内部集合 |
| [02 抽象](backend/ood/guide/02_abstraction.md) | Abstraction | interface vs abstract class 决策树；抽象层次一致性；依赖注入边界 |
| [03 继承与组合](backend/ood/guide/03_inheritance.md) | Inheritance | 脆弱基类问题；Composition over Inheritance；Square-Rectangle 陷阱；Mixin |
| [04 多态](backend/ood/guide/04_polymorphism.md) | Polymorphism | 子类型多态；泛型参数多态；函数重载；消灭 if-else；鸭子类型 |
| [05 SOLID](backend/ood/guide/05_solid.md) | SOLID 原则 | SRP / OCP / LSP / ISP / DIP — 每条原则含反例→正例对比 |
| [06 创建型模式](backend/ood/guide/06_creational_patterns.md) | Creational | Factory Method / Abstract Factory / Builder（链式 API）/ Singleton / Prototype |
| [07 结构型模式](backend/ood/guide/07_structural_patterns.md) | Structural | Composite / Decorator / Adapter / Facade / Proxy（懒加载/缓存/权限）|
| [08 行为型模式](backend/ood/guide/08_behavioral_patterns.md) | Behavioral | Strategy / Observer / State / Command（Undo/Redo）/ CoR / Template Method |
| [09 面试方法论](backend/ood/guide/09_interview_guide.md) | 面试指南 | 45 分钟时间分配；需求澄清话术；类图速绘；扩展性追问应对；题目难度分级 |
| [10 TS 高级类型](backend/ood/guide/10_typescript_advanced.md) | TypeScript | 判别联合；映射类型；条件类型 infer；模板字面量类型；keyof typeof；泛型约束 |
| [11 反模式](backend/ood/guide/11_anti_patterns.md) | Anti-Patterns | 上帝类；贫血模型；原始类型偏执；特性依恋；霰弹式修改；过度设计 |
| [12 GRASP 原则](backend/ood/guide/12_grasp.md) | GRASP | 信息专家；创建者；控制器；低耦合；高内聚；多态；纯虚构；间接；防止变异 |
| [13 UML 速查表](backend/ood/guide/13_uml_cheatsheet.md) | UML | 六大关系箭头；多重性符号；白板画图技巧；代码→UML 对照表 |
| [14 函数式 vs OOP](backend/ood/guide/14_functional_vs_oop.md) | 范式混用 | class vs function 决策树；4 种场景对比；混用最佳实践 |
| [15 DDD 基础](backend/ood/guide/15_ddd_basics.md) | 领域驱动设计 | 值对象；实体；聚合根；领域服务；仓储 — OOD 理论落地实际项目的桥梁 |

### OOD 实战题

> 做题前先读 [09 面试方法论](backend/ood/guide/09_interview_guide.md)，做题时参照 [13 UML 速查](backend/ood/guide/13_uml_cheatsheet.md)。

**推荐刷题顺序（前端 / 全栈）**：
停车场 → LRU Cache → 自动贩卖机 → ATM 机 → 购物车 → 聊天室 → 日志聚合器

| 文件 | 题目 | 核心考点 |
|------|------|---------|
| [00 方法论](backend/ood/problems/00_ood_framework.md) | OOD 面试框架 | SOLID；常用设计模式；类图速记 |
| [01 停车场](backend/ood/problems/01_parking_lot.md) | Parking Lot | 多车型多态；Strategy 计费；车位分配 |
| [02 LRU Cache](backend/ood/problems/02_lru_cache.md) | LRU Cache | HashMap + 双向链表；O(1) get & put |
| [03 电梯系统](backend/ood/problems/03_elevator.md) | Elevator System | 状态机；SCAN 调度；Strategy 模式 |
| [04 扑克牌游戏](backend/ood/problems/04_card_game.md) | Card Game (Blackjack) | 抽象层次；Template Method；Ace 点数逻辑 |
| [05 自动贩卖机](backend/ood/problems/05_vending_machine.md) | Vending Machine | State 模式；状态机；预建实例切换 |
| [06 图书馆系统](backend/ood/problems/06_library_system.md) | Library System | BookCatalog vs BookItem；预约队列；逾期罚款 |
| [07 国际象棋](backend/ood/problems/07_chess_game.md) | Chess Game | 棋子多态；合法走法验证；将军检测；Board 克隆模拟 |
| [08 HashMap](backend/ood/problems/08_hashmap.md) | HashMap | 链地址法；动态 rehash；负载因子；2 的幂次原理 |
| [09 文件系统](backend/ood/problems/09_file_system.md) | File System | Composite 模式（File+Directory 同接口）；递归 size；路径解析 |
| [10 网约车](backend/ood/problems/10_ride_sharing_ood.md) | Ride Sharing | 行程状态机；车型多态；司机匹配 Strategy；计价 |
| [11 酒店预订](backend/ood/problems/11_hotel_booking.md) | Hotel Booking | 房间类型多态；防超卖乐观锁；预订状态机；计价 Strategy |
| [12 外卖系统](backend/ood/problems/12_food_delivery.md) | Food Delivery | 订单状态机；菜单 Composite；Observer 推送通知；骑手分配 |
| [13 购物车](backend/ood/problems/13_shopping_cart.md) | Shopping Cart | 实物 vs 数字商品多态；折扣 Strategy；结账防超卖 |
| [14 阻塞队列](backend/ood/problems/14_blocking_queue.md) | Blocking Queue | Producer-Consumer；条件变量；ReentrantLock 实现 |
| [15 Twitter OOD](backend/ood/problems/15_twitter_ood.md) | Twitter（OOD层）| Tweet 多态（原推/转推/回复）；Timeline 推拉模式；推荐关注 |
| [16 聊天室](backend/ood/problems/16_chat_room.md) | Chat Room | Mediator 模式；Observer 广播；消息多态；权限管理 |
| [17 日志聚合器](backend/ood/problems/17_logger.md) | Logger | Singleton；责任链（Handler 链）；格式化 Strategy；采样 |
| [18 ATM 机](backend/ood/problems/18_atm_machine.md) | ATM Machine | State 模式（Idle/CardInserted/PINVerified/Dispensing）；金额精度；事务日志 |
| [19 贪吃蛇](backend/ood/problems/19_snake_game.md) | Snake Game | Queue 模拟蛇身；O(1) 碰撞检测（HashSet）；方向约束；胜利条件 |
| [20 井字棋](backend/ood/problems/20_tic_tac_toe.md) | Tic-Tac-Toe | N×N 棋盘；O(1) 胜负检测（行列对角线计数器）；玩家多态；Minimax AI |
| [21 电影院订票](backend/ood/problems/21_cinema_booking.md) | Cinema Booking | 2D 座位图；乐观锁防超卖（CAS）；预订超时释放；区域差异定价 |
| [22 LFU Cache](backend/ood/problems/22_lfu_cache.md) | LFU Cache | 双 HashMap + LinkedHashSet；minFreq 追踪；O(1) get&put；同频 LRU |

---

### 后端系统设计

#### 01 基础概念

| 文件 | 核心内容 |
|------|---------|
| [如何使用](00_how_to_use.md) | 三条阅读路线（从零入门/查漏补缺/面试冲刺）|
| [01 度量体系](backend/system_design/01_fundamentals/01_metrics.md) | Latency / Throughput / Availability；SLA/SLO/SLI；P99 vs 平均值 |
| [02 CAP 定理](backend/system_design/01_fundamentals/02_cap_theorem.md) | 分区时只能选 C 或 A；CP vs AP；PACELC 扩展 |
| [03 ACID 与 BASE](backend/system_design/01_fundamentals/03_acid_base.md) | 事务四特性；BASE 的含义；两者适用场景 |
| [04 网络基础](backend/system_design/01_fundamentals/04_network_basics.md) | TCP/UDP/HTTP；DNS；CDN；L4 vs L7 负载均衡；Anycast |
| [05 认证与安全](backend/system_design/01_fundamentals/05_auth_security.md) | Session vs JWT；OAuth2；XSS/CSRF；API 签名；mTLS；Secret 管理 |
| [06 微服务与部署](backend/system_design/01_fundamentals/06_microservices_deployment.md) | 服务发现；Kubernetes；蓝绿/金丝雀部署；API Gateway vs Service Mesh |
| [07 核心数据结构](backend/system_design/01_fundamentals/07_data_structures_for_sd.md) | Bloom Filter / Skip List / LSM Tree / Merkle Tree / B+ Tree / HyperLogLog |
| [08 输入 URL 后发生了什么](backend/system_design/01_fundamentals/08_url_to_page.md) | DNS 递归解析；TCP 三次握手；TLS 握手；HTTP/2；浏览器渲染流水线 |
| [09 一致性哈希](backend/system_design/01_fundamentals/09_consistent_hashing.md) | 哈希环原理；虚拟节点；扩缩容迁移；TypeScript 实现；vs Range Sharding |

#### 02 存储系统

| 文件 | 核心内容 |
|------|---------|
| [00 选型框架](backend/system_design/02_storage/00_choice_framework.md) | 决策树：什么场景用什么存储，以及为什么 |
| [01 关系型数据库](backend/system_design/02_storage/01_rdbms.md) | B+Tree 索引；主从复制；水平分片；一致性哈希 |
| [02 NoSQL](backend/system_design/02_storage/02_nosql.md) | KV / 文档 / 宽列 / 图 四类对比；Cassandra LSM Tree；适用场景 |
| [03 缓存](backend/system_design/02_storage/03_cache.md) | Cache-Aside / Write-Through / Write-Back；缓存穿透/击穿/雪崩防御 |
| [04 对象存储](backend/system_design/02_storage/04_object_storage.md) | S3 架构；分片上传；Pre-signed URL；冷热分层 |
| [05 Schema 设计](backend/system_design/02_storage/05_schema_design.md) | 规范化 vs 反范式；电商/社交/RBAC/审计日志/树形结构；索引设计；分片键选择 |

#### 03 通信方式

| 文件 | 核心内容 |
|------|---------|
| [01 同步通信](backend/system_design/03_communication/01_sync.md) | REST vs gRPC vs GraphQL；API Gateway；幂等性设计 |
| [02 异步通信](backend/system_design/03_communication/02_async.md) | Kafka vs RabbitMQ；Exactly-Once；Outbox 模式；消息积压处理 |
| [03 实时通信](backend/system_design/03_communication/03_realtime.md) | WebSocket / SSE / 长轮询对比；横向扩展；心跳与离线消息 |
| [04 API 设计](backend/system_design/03_communication/04_api_design.md) | 分页策略（Offset/Cursor/Keyset）；版本管理；幂等键；请求签名；tRPC |

#### 04 分布式系统

| 文件 | 核心内容 |
|------|---------|
| [00 权衡图谱](backend/system_design/04_distributed/00_tradeoffs.md) | 所有核心 tradeoff 的汇总索引 |
| [01 一致性模型](backend/system_design/04_distributed/01_consistency_models.md) | 线性一致性 → 因果一致性 → 最终一致性；Read-Your-Writes；Quorum |
| [02 分布式事务](backend/system_design/04_distributed/02_distributed_tx.md) | 2PC 的问题；Saga（编排式/协调式）；TCC；Outbox 模式 |
| [03 共识算法](backend/system_design/04_distributed/03_consensus.md) | Raft：Leader 选举 + 日志复制；与 Paxos 对比；etcd/ZooKeeper |
| [04 容错与弹性](backend/system_design/04_distributed/04_fault_tolerance.md) | 熔断器三状态；令牌桶/漏桶/滑动窗口限流；指数退避；幂等键 |
| [05 ID 生成](backend/system_design/04_distributed/05_id_generation.md) | UUID / DB 自增 / Snowflake / 号段模式；时钟回拨问题 |
| [06 分布式锁](backend/system_design/04_distributed/06_distributed_lock.md) | Redis SETNX + Lua；ZooKeeper 临时顺序节点；Fencing Token 防脑裂 |
| [07 多地域架构](backend/system_design/04_distributed/07_multi_region.md) | Active-Active vs Passive；写冲突解决；GeoDNS；故障切换 Playbook |
| [08 Event Sourcing](backend/system_design/04_distributed/08_event_sourcing.md) | 事件溯源原理；聚合根+命令+事件；Projection；Snapshot；CQRS |

#### 05 方法论

| 文件 | 核心内容 |
|------|---------|
| [01 面试框架](backend/system_design/05_methodology/design_process/01_framework.md) | 45 分钟时间分配；各阶段该说什么；如何主动引导讨论 |
| [02 容量估算](backend/system_design/05_methodology/design_process/02_capacity_estimation.md) | QPS / 存储 / 带宽的估算公式；常用数量级记忆 |
| [03 权衡分析](backend/system_design/05_methodology/design_process/03_tradeoff_analysis.md) | 如何结构化表达"选 A 而不是 B，因为..." |
| [04 反模式与追问应对](backend/system_design/05_methodology/design_process/04_antipatterns_and_followup.md) | 7 个高频反模式；4 类追问的标准应对框架 |
| [01 数字速查](backend/system_design/05_methodology/reference/01_numbers_cheatsheet.md) | 各类系统的典型数量级（必背） |
| [02 术语表](backend/system_design/05_methodology/reference/02_glossary.md) | 100+ 术语的精确定义 |
| [03 设计模式](backend/system_design/05_methodology/reference/03_patterns.md) | 读写分离 / CQRS / Fanout / Event Sourcing 的结构与适用场景 |
| [04 可观测性](backend/system_design/05_methodology/reference/04_observability.md) | Metrics / Logs / Traces；SLO 告警；Error Budget；Burn Rate |
| [05 概念关系图](backend/system_design/05_methodology/reference/05_concept_map.md) | 10 张 Mermaid 图：所有概念之间的依赖和关联网络 |
| [06 面试官评分标准](backend/system_design/05_methodology/reference/06_interviewer_rubric.md) | 面试官真正在评的 4 个维度；L4/L5/L6 的差距 |
| [07 技术横向对比](backend/system_design/05_methodology/reference/07_tech_comparison.md) | 7 张大表：消息队列/数据库/缓存/通信协议/LB 算法/存储/一致性模型 |
| [08 流处理与数据管道](backend/system_design/05_methodology/reference/08_data_pipeline.md) | 批处理 vs 流处理；Lambda vs Kappa；Spark；Flink 时间语义/Watermark/窗口 |
| [09 Elasticsearch 深度解析](backend/system_design/05_methodology/reference/09_elasticsearch_deep.md) | Lucene Segment；Posting List；NRT；BM25；深度分页；写入/查询优化 |

#### 06 实战案例

> 先用 [空白练习模板](backend/system_design/06_case_studies/00_practice_mode.md) 限时 45 分钟独立设计，再打开对应文档对比。

| 文件 | 系统 | 核心考点 |
|------|------|---------|
| [00 面试套路](backend/system_design/06_case_studies/00_interview_patterns.md) | — | 各类题型的通用解题模板 |
| [01 URL 短链](backend/system_design/06_case_studies/01_url_shortener.md) | bit.ly | 短码生成；301 vs 302；DB 分片 |
| [02 限流器](backend/system_design/06_case_studies/02_rate_limiter.md) | API Gateway | 令牌桶/滑动窗口；Redis Lua；多数据中心 |
| [03 搜索自动补全](backend/system_design/06_case_studies/03_search_autocomplete.md) | Google 搜索框 | Trie 结构；top-K 缓存；实时 vs 批量更新 |
| [04 通知系统](backend/system_design/06_case_studies/04_notification_system.md) | Push/Email/SMS | 多渠道多队列；幂等去重；第三方故障处理 |
| [05 News Feed](backend/system_design/06_case_studies/05_news_feed.md) | 朋友圈 / Twitter | 推模式 vs 拉模式 vs 混合；大 V 问题；Timeline 缓存 |
| [06 聊天系统](backend/system_design/06_case_studies/06_chat_system.md) | 微信 / WhatsApp | WebSocket 扩展；消息有序；离线消息；已读回执 |
| [07 分布式缓存](backend/system_design/06_case_studies/07_distributed_cache.md) | Redis Cluster | 一致性哈希+虚拟节点；热点 Key；驱逐策略 |
| [08 分布式存储](backend/system_design/06_case_studies/08_distributed_storage.md) | S3 / HDFS | 元数据与数据分离；分片上传；跨 AZ 多副本 |
| [09 视频流媒体](backend/system_design/06_case_studies/09_video_streaming.md) | YouTube / Netflix | 异步转码；自适应码率 HLS/DASH；CDN 策略 |
| [10 打车系统](backend/system_design/06_case_studies/10_ride_sharing.md) | Uber / 滴滴 | 位置高频写入；Geohash 附近查询；司机匹配 |
| [11 广告点击聚合](backend/system_design/06_case_studies/11_ad_click_aggregator.md) | 广告平台 | Lambda vs Kappa；时间窗口聚合；数据倾斜；对账 |
| [12 网络爬虫](backend/system_design/06_case_studies/12_web_crawler.md) | Googlebot | URL Frontier 优先级队列；布隆过滤器去重；spider trap |
| [13 全文搜索引擎](backend/system_design/06_case_studies/13_search_engine.md) | Elasticsearch | 倒排索引；BM25；NRT；深度分页陷阱 |
| [14 支付系统](backend/system_design/06_case_studies/14_payment_system.md) | Stripe / 支付宝 | 幂等键；复式记账 Ledger；Webhook 处理；对账 |
| [15 附近搜索](backend/system_design/06_case_studies/15_proximity_service.md) | Yelp / 大众点评 | Geohash 9 格查询；静态 POI vs 动态位置；地理分片 |
| [16 酒店预订](backend/system_design/06_case_studies/16_hotel_reservation.md) | Booking.com | 库存并发防超卖；乐观锁 vs 悲观锁；Saga 取消补偿 |
| [17 协同文档编辑](backend/system_design/06_case_studies/17_collaborative_editor.md) | Google Docs | OT vs CRDT；WebSocket 实时同步；光标 ID；离线合并；版本历史 |
| [18 股票交易所](backend/system_design/06_case_studies/18_stock_exchange.md) | 低延迟交易所 | 撮合引擎；订单簿（红黑树）；LMAX Disruptor；μs 级延迟；风控；WAL |
| [19 分布式 KV 存储](backend/system_design/06_case_studies/19_key_value_store.md) | Dynamo / DynamoDB | 一致性哈希虚拟节点；N/W/R Quorum；向量时钟；Gossip；Merkle 反熵 |
| [20 Google Drive](backend/system_design/06_case_studies/20_google_drive.md) | Google Drive / Dropbox | 4MB 分块；SHA-256 内容寻址；增量同步；三路合并冲突；WebSocket 通知 |
| [21 附近的人](backend/system_design/06_case_studies/21_nearby_friends.md) | 微信附近的人 | Redis GEO；Pub/Sub 实时推送；WebSocket Fanout；隐私黑名单 |
| [22 地图导航](backend/system_design/06_case_studies/22_google_maps.md) | Google Maps | 地图瓦片 Z/X/Y；路网邻接表；A* vs Contraction Hierarchies；Flink 实时路况 |
| [23 分布式消息队列](backend/system_design/06_case_studies/23_distributed_mq.md) | 设计 Kafka | 分区路由；Commit Log 顺序写；零拷贝 sendfile；ISR 副本协议；日志压缩 |
| [24 监控告警系统](backend/system_design/06_case_studies/24_metrics_monitoring.md) | Prometheus + Alertmanager | Pull vs Push；时序数据模型；Gorilla 压缩；降采样；告警状态机 |
| [25 分布式邮件服务](backend/system_design/06_case_studies/25_distributed_email.md) | SendGrid / SES | SMTP 发送流程；SPF/DKIM/DMARC；软退信 vs 硬退信；退订抑制列表 |
| [26 游戏排行榜](backend/system_design/06_case_studies/26_gaming_leaderboard.md) | 实时排行榜 | Redis ZSet（ZADD GT/ZREVRANK）；虚拟节点分片；Top 100 缓存刷新 |
| [27 数字钱包](backend/system_design/06_case_studies/27_digital_wallet.md) | 支付宝 / PayPal | 复式记账；幂等转账；乐观锁防超扣；本地消息表跨分片；日终对账 |
| [28 推荐系统](backend/system_design/06_case_studies/28_recommendation_system.md) | 电商/视频推荐 | 两阶段召回+排序；协同过滤；向量召回（FAISS）；Wide&Deep 模型；特征工程；A/B 测试 |

#### 07 真实公司架构拆解

| 文件 | 公司 | 核心看点 |
|------|------|---------|
| [01 Twitter](backend/system_design/07_real_world/01_twitter.md) | Twitter / X | 推拉混合 Fanout；Timeline 预计算；Snowflake ID；大 V 问题演进 |
| [02 Uber](backend/system_design/07_real_world/02_uber.md) | Uber / 滴滴 | H3 六边形索引；ETA 匹配；CAS 防双派；动态定价流处理 |
| [03 Netflix](backend/system_design/07_real_world/03_netflix.md) | Netflix | Open Connect 自建 CDN；预推送；HLS 自适应码率；混沌工程 |
| [04 Discord](backend/system_design/07_real_world/04_discord.md) | Discord | MongoDB→Cassandra→ScyllaDB 迁移；SFU 语音；大群 Fanout 分片 |
| [05 WhatsApp](backend/system_design/07_real_world/05_whatsapp.md) | WhatsApp | Erlang 200 万连接/台；E2EE Signal Protocol；消息不持久化设计 |
