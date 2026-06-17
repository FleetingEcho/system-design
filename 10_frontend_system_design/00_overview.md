# 前端系统设计专题概览

> 面向 TypeScript / 全栈工程师的系统设计补充模块。
> 后端系统设计已有完整体系（见 01-09 章），本专题聚焦**前端特有的规模化挑战**。

---

## 为什么需要专门的前端系统设计

后端系统设计关心：QPS、存储、一致性、容错。

前端系统设计额外关心：
- **首屏速度**：用户等待超过 3s 跳出率翻倍
- **交互响应**：INP < 200ms 才算"良好"
- **客户端状态**：几十个组件共享数据，如何不乱
- **缓存边界**：CDN / Service Worker / 内存缓存，各管一段
- **跨团队协作**：微前端解决的是组织问题，不只是技术问题

---

## 本专题文件索引

| 文件 | 主题 | 核心内容 |
|------|------|---------|
| [01 SSR/SSG/ISR 架构](01_ssr_ssg_isr.md) | 渲染策略 | 四种渲染模式对比；ISR 失效策略；CDN + Edge 协作；缓存雪崩防御 |
| [02 微前端架构](02_micro_frontend.md) | 应用拆分 | Module Federation；路由治理；共享状态；样式隔离；独立部署流水线 |
| [03 BFF 架构](03_bff.md) | Backend for Frontend | GraphQL BFF + DataLoader（N+1）；tRPC 类型安全 API；REST vs GraphQL 选型 |
| [04 前端性能监控](04_frontend_monitoring.md) | RUM | Core Web Vitals 采集 SDK；Beacon API；ClickHouse 时序存储；环比异常告警 |
| [05 框架横向对比](05_framework_comparison.md) | 生态选型 | Next.js / Remix / TanStack Start / Astro / Nuxt / SvelteKit 对比；选型决策树 |
| [06 状态管理](06_state_management.md) | 状态架构 | 服务端状态（TanStack Query / SWR）vs 客户端状态（Zustand / Jotai / Redux Toolkit）|
| [07 性能优化](07_performance_optimization.md) | 加载 + 运行时 | 代码分割；bundle 优化；虚拟滚动（TanStack Virtual）；Web Worker；关键渲染路径 |
| [08 前端安全](08_security.md) | 安全 | XSS / CSRF / CSP；NextAuth.js / Clerk 认证方案；HttpOnly Cookie vs JWT 存储 |

---

## 面试中如何呈现前端系统设计

前端系统设计面试与后端一样，**也用 45 分钟框架**（见 [09_interview_guide](../09_ood_guide/09_interview_guide.md)），但侧重点不同：

```
需求澄清（5 min）
  ↓ 目标用户数？DAU？首屏目标？SEO 需求？团队规模？
架构概览（10 min）
  ↓ 渲染策略选型 → 数据获取方式 → 状态管理边界
核心难点深挖（20 min）
  ↓ 缓存策略 / 代码分割 / 跨团队隔离 / 监控
追问应对（10 min）
  ↓ "如果 DAU 涨 10 倍？" / "如何保证一致性？"
```

**差异化得分点**：能主动提出 Core Web Vitals 指标、能量化说出 CDN 命中率对 TTFB 的影响、能解释 Module Federation 的 shared 配置如何防止 React 双实例。
