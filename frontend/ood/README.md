# 前端 OOD 题库

> 前端工程师面试专项：用 TypeScript 实现浏览器/框架核心机制。
> 与后端 OOD（停车场、电梯、象棋）不同，前端 OOD 考察 **JS 运行时机制与框架设计**。

---

## 题目列表

| 文件 | 题目 | 考察点 |
|------|------|--------|
| [01 EventEmitter](01_event_emitter.md) | 实现 EventEmitter | 观察者模式；on/off/once/emit；内存泄漏防护；泛型类型安全 |
| [02 Promise](02_promise.md) | 实现 Promise（Promises/A+） | 状态机；then 链式调用；微任务队列；循环引用检测；all/race/allSettled |
| [03 SPA Router](03_spa_router.md) | 设计前端路由 | History API；路径→正则；嵌套路由；导航守卫；懒加载路由 |
| [04 Virtual DOM](04_virtual_dom.md) | Virtual DOM + Diff | VNode 树；createElement；同层 Diff；带 key 列表优化；O(n) 复杂度 |
| [05 Mini Redux](05_mini_redux.md) | 设计 Mini Redux | createStore；中间件链（洋葱模型）；combineReducers；thunk；时间旅行 |
| [06 工具函数](06_utils.md) | Debounce/Throttle/LazyLoad/Memoize | 闭包；定时器；IntersectionObserver；LRU 缓存 |

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
