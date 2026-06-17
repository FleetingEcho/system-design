# 错误监控与边界

> 生产环境的错误不可避免，关键是：**快速发现 → 精准定位 → 优雅降级**。
> 本文覆盖：React Error Boundary、全局错误捕获、Sentry 集成、Source Map、错误告警策略。

---

## 错误分类

```
前端错误类型
├── JS 运行时错误（TypeError、ReferenceError）
├── Promise 未处理拒绝（unhandledrejection）
├── 网络请求错误（fetch 失败、超时）
├── 资源加载失败（script/image 404）
├── React 渲染错误（组件 throw，需 Error Boundary）
└── 自定义业务错误（支付失败、数据校验失败）
```

---

## React Error Boundary

> Error Boundary 是 React 的错误隔离机制：子树抛出的错误不会导致整个应用崩溃。
> **注意**：只能捕获渲染阶段、生命周期、构造函数中的错误，**不能捕获**事件处理器、异步代码。

```typescript
// components/ErrorBoundary.tsx
import { Component, type ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode | ((error: Error, reset: () => void) => ReactNode);
  onError?: (error: Error, info: { componentStack: string }) => void;
}

interface State {
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null };

  static getDerivedStateFromError(error: Error): State {
    return { error };
  }

  componentDidCatch(error: Error, info: { componentStack: string }) {
    // 上报到错误监控服务
    this.props.onError?.(error, info);
    reportError(error, { componentStack: info.componentStack });
  }

  reset = () => this.setState({ error: null });

  render() {
    const { error } = this.state;
    if (!error) return this.props.children;

    const { fallback } = this.props;
    if (typeof fallback === 'function') return fallback(error, this.reset);
    if (fallback) return fallback;

    // 默认降级 UI
    return (
      <div role="alert" className="error-boundary-fallback">
        <h2>页面出现错误</h2>
        <p>{error.message}</p>
        <button onClick={this.reset}>重试</button>
      </div>
    );
  }
}
```

### 多层级 Error Boundary

```typescript
// 粒度分三层：全局 → 页面 → 组件块

// app/layout.tsx — 最外层，兜底
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <ErrorBoundary
          fallback={<GlobalErrorPage />}
          onError={(err) => reportToSentry(err, { level: 'fatal' })}
        >
          {children}
        </ErrorBoundary>
      </body>
    </html>
  );
}

// 页面级：保护局部功能块
function ProductPage() {
  return (
    <div>
      <ProductInfo />   {/* 核心内容，不包裹 ErrorBoundary */}

      {/* 非核心推荐区，出错不影响主流程 */}
      <ErrorBoundary
        fallback={<div>推荐商品加载失败</div>}
        onError={(err) => reportToSentry(err, { level: 'warning' })}
      >
        <RecommendationSection />
      </ErrorBoundary>

      {/* 评论区，出错不影响购买 */}
      <ErrorBoundary fallback={null}>  {/* 静默失败 */}
        <ReviewSection />
      </ErrorBoundary>
    </div>
  );
}
```

### react-error-boundary（推荐社区库）

```typescript
// react-error-boundary 提供 hooks 支持，比手写 class 更方便
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary';

function DataWidget() {
  const { showBoundary } = useErrorBoundary();

  // 在事件处理器中手动触发 Error Boundary（原生不支持）
  const handleClick = async () => {
    try {
      await fetchData();
    } catch (err) {
      showBoundary(err);  // 主动触发最近的 Error Boundary
    }
  };

  return <button onClick={handleClick}>加载数据</button>;
}

// withErrorBoundary HOC
const SafeWidget = withErrorBoundary(DataWidget, {
  fallbackRender: ({ error, resetErrorBoundary }) => (
    <div>
      <p>出错了: {error.message}</p>
      <button onClick={resetErrorBoundary}>重试</button>
    </div>
  ),
  onReset: () => {
    // 重置相关状态（如清除缓存）
    queryClient.invalidateQueries();
  },
});
```

---

## 全局错误捕获

```typescript
// lib/error-handler.ts — 应用入口处注册

// 1. 同步 JS 错误
window.addEventListener('error', (event) => {
  // 过滤跨域脚本错误（message = "Script error."，无法获取详情）
  if (event.message === 'Script error.') return;

  reportToSentry({
    type: 'js_error',
    message: event.message,
    filename: event.filename,
    lineno: event.lineno,
    colno: event.colno,
    error: event.error,
  });
});

// 2. Promise unhandled rejection
window.addEventListener('unhandledrejection', (event) => {
  // 阻止浏览器控制台打印（已由我们的监控处理）
  event.preventDefault();

  const error = event.reason instanceof Error
    ? event.reason
    : new Error(String(event.reason));

  reportToSentry({
    type: 'unhandled_rejection',
    error,
  });
});

// 3. 资源加载失败（图片/脚本/样式）
window.addEventListener('error', (event) => {
  const target = event.target as HTMLElement;
  if (target instanceof HTMLScriptElement || target instanceof HTMLLinkElement || target instanceof HTMLImageElement) {
    reportToSentry({
      type: 'resource_error',
      url: (target as HTMLScriptElement).src || (target as HTMLLinkElement).href,
      tagName: target.tagName,
    });
  }
}, true);  // capture phase，才能捕获资源加载错误

// 4. Fetch/XHR 错误（拦截器）
const originalFetch = window.fetch;
window.fetch = async (...args) => {
  try {
    const response = await originalFetch(...args);
    if (!response.ok && response.status >= 500) {
      reportToSentry({
        type: 'api_error',
        url: args[0]?.toString(),
        status: response.status,
      });
    }
    return response;
  } catch (err) {
    reportToSentry({ type: 'network_error', url: args[0]?.toString(), error: err as Error });
    throw err;
  }
};
```

---

## Sentry 集成

```typescript
// app/instrumentation.ts（Next.js 15+）
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.NEXT_PUBLIC_APP_VERSION,

  // 采样率（生产环境不需要 100%，降低成本）
  sampleRate: 1.0,            // 错误 100% 上报
  tracesSampleRate: 0.1,      // 性能追踪只采 10%

  // 过滤不重要的错误
  ignoreErrors: [
    'ResizeObserver loop limit exceeded',
    'Non-Error exception captured',
    /^Network Error$/,
    /ChunkLoadError/,          // 部署新版本时旧 chunk 找不到，正常现象
  ],

  beforeSend(event, hint) {
    const error = hint.originalException as Error;

    // 过滤已知的第三方错误
    if (error?.stack?.includes('chrome-extension://')) return null;

    // 添加用户上下文
    const user = getCurrentUser();
    if (user) {
      event.user = { id: user.id, email: user.email };
    }

    return event;
  },

  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration({
      maskAllText: true,           // 隐藏敏感文本（GDPR）
      blockAllMedia: false,
      sessionSampleRate: 0.1,      // 10% 会话录制
      errorSampleRate: 1.0,        // 出错时 100% 录制
    }),
  ],
});
```

```typescript
// 手动上报（业务逻辑中）
import * as Sentry from '@sentry/nextjs';

// 上报错误 + 额外上下文
function handlePaymentError(error: Error, orderId: string) {
  Sentry.withScope(scope => {
    scope.setTag('feature', 'payment');
    scope.setTag('order_id', orderId);
    scope.setLevel('fatal');
    scope.setContext('payment', {
      orderId,
      amount: order.amount,
      currency: order.currency,
    });
    Sentry.captureException(error);
  });
}

// 上报自定义事件（非 Error）
Sentry.captureMessage('Payment method not available', 'warning');

// 面包屑（记录用户操作路径，帮助复现）
Sentry.addBreadcrumb({
  category: 'user-action',
  message: 'Clicked checkout button',
  level: 'info',
  data: { cartItemCount: cart.items.length },
});
```

---

## Source Map（生产环境定位源码位置）

```typescript
// next.config.js — 上传 source map 到 Sentry，然后从 CDN 删除
const { withSentryConfig } = require('@sentry/nextjs');

const nextConfig = {
  // 不向浏览器暴露 source map
  productionBrowserSourceMaps: false,
};

module.exports = withSentryConfig(nextConfig, {
  org: 'my-org',
  project: 'my-project',
  // 构建时自动上传 source map 到 Sentry，然后删除本地文件
  widenClientFileUpload: true,
  hideSourceMaps: true,
  // release 版本与代码关联（方便 Sentry 归因到具体 release）
  release: { name: process.env.NEXT_PUBLIC_APP_VERSION },
});
```

```
流程：
构建产物（.js + .js.map）
    ↓
Sentry CLI 上传 .js.map → Sentry 服务器
    ↓
删除 CDN 上的 .js.map（用户看不到源码）
    ↓
Sentry 收到错误（包含混淆后的行列号）
    ↓
Sentry 用 .js.map 还原 → 显示源码文件名 + 行号
```

---

## 错误告警策略

```typescript
// 告警分级（避免告警轰炸）

const ALERT_RULES = {
  // P0：立即告警（付款失败、登录失败）
  fatal: {
    threshold: 1,         // 1 次即告警
    channel: 'pagerduty', // 电话呼叫 oncall
    window: '5m',
  },

  // P1：5 分钟内超过 10 次告警
  error: {
    threshold: 10,
    channel: 'slack#alerts',
    window: '5m',
  },

  // P2：1 小时内新增超过 100 次
  warning: {
    threshold: 100,
    channel: 'slack#warnings',
    window: '1h',
  },
};

// Sentry Alert Rule 配置（在 Sentry 控制台设置，或用 API）
// 条件：错误率相比上周同期增加 20%（环比告警，避免绝对值误报）
```

---

## 错误边界 + TanStack Query 集成

```typescript
// TanStack Query 的错误与 Error Boundary 结合

// 全局错误处理（在 QueryClient 中配置）
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        // 4xx 错误不重试（客户端问题），5xx 重试最多 3 次
        if (error instanceof ApiError && error.status < 500) return false;
        return failureCount < 3;
      },
      throwOnError: (error) => {
        // 只有 5xx 错误抛给 Error Boundary，4xx 让组件自己处理
        return error instanceof ApiError && error.status >= 500;
      },
    },
  },
});

// 组件中使用
function UserProfile({ userId }: { userId: string }) {
  const { data, error, isError } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    throwOnError: false,  // 覆盖全局配置，此处不抛给 Error Boundary
  });

  if (isError) {
    if (error instanceof ApiError && error.status === 404) {
      return <NotFound />;
    }
    return <RetryButton onRetry={() => refetch()} error={error} />;
  }

  return <UserCard user={data} />;
}
```

---

## 面试常见追问

**Q: Error Boundary 为什么不能捕获事件处理器中的错误？**
A: Error Boundary 基于 React 的渲染机制（`getDerivedStateFromError` 在渲染阶段触发）。事件处理器不在 React 的渲染调用栈中，错误不会冒泡到 Error Boundary。解决方案：事件处理器中用 `try/catch` 捕获，需要显示 Error Boundary 的错误 UI 时用 `react-error-boundary` 的 `useErrorBoundary().showBoundary(err)` 手动触发。

**Q: Source Map 会不会泄露源码？**
A: 如果将 `.map` 文件部署到 CDN，任何人都能访问。正确做法：只上传到 Sentry（私有存储），生产 CDN 不部署 `.map`。`next.config.js` 中 `hideSourceMaps: true` 会在构建后自动删除。

**Q: 如何区分"偶发错误"和"系统性问题"？**
A: 看错误率曲线（绝对数量 vs 占请求百分比）：①单个用户重复报同一错误 → 可能是特定设备/浏览器问题；②错误率突然上升 → 关联最近的部署（用 Sentry 的 release tracking）；③多个地区同时出现 → 可能是 CDN 或 API 问题。Sentry 的 "Issue Grouping"（相同错误聚合）和 "Releases" 功能是关键。

**Q: 如何防止 Error Boundary 隐藏真正的 Bug？**
A: Error Boundary 不是为了"压住"错误，而是优雅降级。上报是必须的（`componentDidCatch` 中调用 Sentry）；开发环境应让错误显示完整堆栈（React DevTools 会在 Error Boundary 捕获后显示 overlay）；Error Boundary 的粒度越细越好，能让其他部分正常工作。
