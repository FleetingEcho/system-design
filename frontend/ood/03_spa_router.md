# 设计 SPA Router

> 考察点：History API、路由匹配算法、嵌套路由、导航守卫、代码分割集成。
> React Router、Vue Router 的核心机制都基于此。

---

## 需求分析

```
核心功能：
  - 路径匹配（静态 /about、动态 /user/:id、通配符 *）
  - 两种模式：hash 模式（#/path）、history 模式（/path）
  - 编程式导航：push / replace / go / back
  - 监听路由变化，通知视图更新

扩展功能：
  - 嵌套路由（children）
  - 导航守卫（beforeEach、afterEach）
  - 路由元信息（meta）
  - 懒加载（import()）
  - 查询参数解析（?key=value）
  - 404 通配符路由
```

---

## 类图

```
Router
  - routes: RouteRecord[]
  - mode: 'hash' | 'history'
  - currentRoute: Route
  - guards: NavigationGuard[]
  + push(to): Promise<void>
  + replace(to): Promise<void>
  + back(): void
  + go(delta): void
  + beforeEach(guard): () => void
  + resolve(path): RouteMatch

RouteRecord
  - path: string
  - component: Component | (() => Promise<Component>)
  - children?: RouteRecord[]
  - meta?: Record<string, unknown>

Route（当前路由快照）
  - path: string
  - params: Record<string, string>
  - query: Record<string, string>
  - meta: Record<string, unknown>
  - matched: RouteRecord[]   （嵌套匹配链）
```

---

## 路由匹配

```typescript
// 将路由路径转换为正则表达式
interface RouteRecord {
  path: string;
  component: unknown;
  children?: RouteRecord[];
  meta?: Record<string, unknown>;
}

interface RouteMatch {
  record: RouteRecord;
  params: Record<string, string>;
  matched: RouteRecord[];  // 从父到子的完整链
}

function pathToRegex(path: string): { regex: RegExp; keys: string[] } {
  const keys: string[] = [];

  const pattern = path
    .replace(/\//g, '\\/')                     // 转义斜杠
    .replace(/:([^/]+)/g, (_, key) => {        // :param → 捕获组
      keys.push(key);
      return '([^\\/]+)';
    })
    .replace(/\*/g, '(.*)');                   // * → 通配符

  return {
    regex: new RegExp(`^${pattern}$`),
    keys,
  };
}

function matchRoute(
  pathname: string,
  routes: RouteRecord[],
  parentMatched: RouteRecord[] = []
): RouteMatch | null {
  for (const record of routes) {
    const fullPath = parentMatched.length > 0
      ? parentMatched[parentMatched.length - 1].path + record.path
      : record.path;

    const { regex, keys } = pathToRegex(fullPath);
    const match = pathname.match(regex);

    if (match) {
      // 提取动态参数
      const params: Record<string, string> = {};
      keys.forEach((key, i) => {
        params[key] = decodeURIComponent(match[i + 1] ?? '');
      });

      const matched = [...parentMatched, record];

      // 尝试匹配子路由
      if (record.children?.length) {
        const childMatch = matchRoute(pathname, record.children, matched);
        if (childMatch) return childMatch;
      }

      return { record, params, matched };
    }
  }
  return null;
}
```

---

## Router 核心实现

```typescript
interface Route {
  path: string;
  params: Record<string, string>;
  query: Record<string, string>;
  meta: Record<string, unknown>;
  matched: RouteRecord[];
}

type NavigationGuard = (
  to: Route,
  from: Route,
  next: (redirect?: string | false) => void
) => void;

class Router {
  private routes: RouteRecord[];
  private mode: 'hash' | 'history';
  private guards: NavigationGuard[] = [];
  private listeners: Set<(route: Route) => void> = new Set();

  currentRoute: Route;

  constructor(options: { routes: RouteRecord[]; mode?: 'hash' | 'history' }) {
    this.routes = options.routes;
    this.mode = options.mode ?? 'history';
    this.currentRoute = this._createRoute(this._getPath());

    // 监听浏览器前进/后退
    window.addEventListener(
      this.mode === 'hash' ? 'hashchange' : 'popstate',
      () => this._handleLocationChange()
    );
  }

  // ─── 导航 ─────────────────────────────────────────────

  async push(to: string): Promise<void> {
    await this._navigate(to, false);
  }

  async replace(to: string): Promise<void> {
    await this._navigate(to, true);
  }

  back(): void { window.history.back(); }
  forward(): void { window.history.forward(); }
  go(delta: number): void { window.history.go(delta); }

  // ─── 守卫 ─────────────────────────────────────────────

  beforeEach(guard: NavigationGuard): () => void {
    this.guards.push(guard);
    return () => {
      const idx = this.guards.indexOf(guard);
      if (idx !== -1) this.guards.splice(idx, 1);
    };
  }

  // ─── 订阅路由变化（供框架集成用）─────────────────────

  subscribe(listener: (route: Route) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  // ─── 内部方法 ─────────────────────────────────────────

  private async _navigate(path: string, replace: boolean): Promise<void> {
    const to = this._createRoute(path);
    const from = this.currentRoute;

    // 运行导航守卫链
    const passed = await this._runGuards(to, from);

    if (passed === false) return;                       // 守卫中止
    if (typeof passed === 'string') {                  // 守卫重定向
      return this._navigate(passed, replace);
    }

    // 更新浏览器 URL
    const url = this.mode === 'hash' ? `#${path}` : path;
    if (replace) {
      window.history.replaceState({ path }, '', url);
    } else {
      window.history.pushState({ path }, '', url);
    }

    this.currentRoute = to;
    this.listeners.forEach(l => l(to));
  }

  private async _runGuards(
    to: Route,
    from: Route
  ): Promise<true | false | string> {
    for (const guard of this.guards) {
      const result = await new Promise<true | false | string>((resolve) => {
        guard(to, from, (redirect) => {
          if (redirect === undefined) resolve(true);
          else if (redirect === false) resolve(false);
          else resolve(redirect);
        });
      });
      if (result !== true) return result;
    }
    return true;
  }

  private _handleLocationChange(): void {
    const path = this._getPath();
    const route = this._createRoute(path);
    this.currentRoute = route;
    this.listeners.forEach(l => l(route));
  }

  private _getPath(): string {
    if (this.mode === 'hash') {
      return window.location.hash.slice(1) || '/';
    }
    return window.location.pathname + window.location.search;
  }

  private _createRoute(rawPath: string): Route {
    const [pathname, search = ''] = rawPath.split('?');
    const match = matchRoute(pathname, this.routes);
    const query = Object.fromEntries(new URLSearchParams(search));

    return {
      path: pathname,
      params: match?.params ?? {},
      query,
      meta: match?.record.meta ?? {},
      matched: match?.matched ?? [],
    };
  }
}
```

---

## 懒加载路由

```typescript
// 路由级别的代码分割
const routes: RouteRecord[] = [
  {
    path: '/',
    component: () => import('./pages/Home'),   // 动态 import，返回 Promise
  },
  {
    path: '/user/:id',
    component: () => import('./pages/UserProfile'),
  },
  {
    path: '/admin',
    meta: { requiresAuth: true },
    component: () => import('./pages/Admin'),
    children: [
      {
        path: '/dashboard',
        component: () => import('./pages/AdminDashboard'),
      },
    ],
  },
  {
    path: '*',  // 404
    component: () => import('./pages/NotFound'),
  },
];

const router = new Router({ routes, mode: 'history' });

// 导航守卫：权限控制
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !isLoggedIn()) {
    next('/login');   // 重定向到登录页
  } else {
    next();           // 放行
  }
});
```

---

## React 集成

```typescript
// 使用 useSyncExternalStore 订阅路由状态
import { useSyncExternalStore } from 'react';

function useRouter() {
  const route = useSyncExternalStore(
    (listener) => router.subscribe(listener),
    () => router.currentRoute,
    () => router.currentRoute  // SSR snapshot
  );
  return { route, router };
}

function Link({ to, children }: { to: string; children: React.ReactNode }) {
  const handleClick = (e: React.MouseEvent) => {
    e.preventDefault();
    router.push(to);
  };
  return <a href={to} onClick={handleClick}>{children}</a>;
}
```

---

## 面试追问

**Q: hash 模式和 history 模式的区别？**
A: hash 模式（`/#/path`）利用 `hashchange` 事件，`#` 后的内容不发送到服务器，兼容性好，不需要服务端配置；history 模式（`/path`）用 `pushState/popState`，URL 干净，但刷新时浏览器会向服务器请求该路径，需服务端配置 fallback（所有路径返回 `index.html`）。

**Q: 动态路由匹配（`:id`）的实现原理？**
A: 将路由 path 编译成正则表达式，`:id` 转换为捕获组 `([^/]+)`，匹配成功后按位置提取捕获值赋给 params。面试中可以写出 `pathToRegex` 函数展示理解。

**Q: 路由守卫的执行顺序？**
A: 以 Vue Router 为例：`beforeEach`（全局）→ `beforeEnter`（路由级）→ `beforeRouteEnter`（组件级）→ 解析异步组件 → `afterEach`（全局）。每个守卫调用 `next()` 才继续，调用 `next(false)` 中止。

**Q: 如何实现路由的过渡动画？**
A: 监听路由变化，给旧路由和新路由分别加 leave/enter 的 CSS 类，利用 `TransitionGroup`（React）或 `<Transition>`（Vue）。可以在 `meta` 中指定动画类型，守卫中读取 `to.meta.transition` 决定动画方向。
