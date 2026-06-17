# 前端 OOD 题库

> 前端工程师面试专项：用 TypeScript 实现浏览器/框架核心机制。
> 与后端 OOD（停车场、电梯、象棋）不同，前端 OOD 考察 **JS 运行时机制与框架设计**。

---

## 题目列表（待补充）

| 题目 | 考察点 |
|------|--------|
| 实现 EventEmitter | 观察者模式、内存泄漏防护、once/off |
| 实现 Promise | 状态机、链式调用、微任务队列 |
| 设计前端路由（SPA Router） | History API、路由匹配、嵌套路由 |
| 实现 Virtual DOM + Diff 算法 | 树遍历、Reconciliation、最小化 DOM 操作 |
| 设计 Mini Redux | 单向数据流、中间件链、时间旅行调试 |
| 实现 Debounce / Throttle | 闭包、定时器、立即执行模式 |
| 实现图片懒加载组件 | Intersection Observer、加载队列 |
| 设计 React Hook（useRequest） | 异步状态机、竞态条件处理、缓存 |

---

## 与后端 OOD 的区别

```
后端 OOD：类建模 → 聚焦领域实体和关系（停车场、预订系统）
前端 OOD：机制实现 → 聚焦运行时行为和 API 设计（事件系统、调度器）

共同点：
  - 都要求清晰的接口设计
  - 都要分析边界条件和异常
  - 都要考虑扩展性
```

---

> 题目正在补充中，参考 OOD 理论指南：[backend/ood/guide](../../backend/ood/guide/00_what_is_ood.md)
