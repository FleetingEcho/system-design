# OOD：useRequest Hook

> 设计通用的数据请求 hook：loading/error/data 状态机、竞态条件处理、重试、轮询、手动触发。
> 原型来自 ahooks 的 `useRequest`，但面试要求从零实现核心机制。

---

## 需求分析

```
功能需求：
  1. 自动执行：组件挂载时自动发请求
  2. 手动触发：支持 manual 模式，手动调用 run()
  3. Loading / Error / Data 状态管理
  4. 竞态条件处理：快速连续请求时，只取最后一次响应
  5. 重试：失败时自动重试（指数退避）
  6. 轮询：每 N 毫秒自动刷新
  7. 请求取消：组件卸载时取消飞行中的请求（AbortController）
  8. 防抖 / 节流：减少频繁触发的请求次数
  
核心难点：
  竞态条件（Race Condition）：
    1. 搜索框快速输入 "a" → "ab" → "abc"
    2. 三个请求并发发出
    3. 网络不稳定，"ab" 的响应比 "abc" 晚回来
    4. 如果不处理：UI 最终显示 "ab" 的结果（错误！）
    
  解法一：AbortController（推荐）— 取消上一次请求
  解法二：版本号（stale closure）— 比较响应版本，丢弃旧的
```

---

## 核心实现

```typescript
// src/hooks/useRequest.ts
import { useState, useEffect, useRef, useCallback } from 'react';

// 状态机
type RequestStatus = 'idle' | 'loading' | 'success' | 'error';

interface UseRequestState<T> {
  data: T | undefined;
  error: Error | undefined;
  status: RequestStatus;
  loading: boolean;
}

interface UseRequestOptions<T, P extends unknown[]> {
  manual?: boolean;          // true = 不自动执行，等待手动 run()
  defaultParams?: P;         // 自动执行时的默认参数
  onSuccess?: (data: T, params: P) => void;
  onError?: (error: Error, params: P) => void;
  onFinally?: (params: P) => void;
  // 重试
  retryCount?: number;       // 最大重试次数（默认 0，不重试）
  retryInterval?: number;    // 重试间隔（ms，默认 1000）
  // 轮询
  pollingInterval?: number;  // 轮询间隔（ms），0 = 不轮询
  pollingWhenHidden?: boolean;  // 页面不可见时是否继续轮询（默认 false）
  // 防抖
  debounceWait?: number;
  // 节流
  throttleWait?: number;
}

interface UseRequestReturn<T, P extends unknown[]> extends UseRequestState<T> {
  run: (...params: P) => void;       // 手动触发（异步，不返回 Promise）
  runAsync: (...params: P) => Promise<T>;  // 手动触发（返回 Promise）
  cancel: () => void;                // 取消当前请求
  refresh: () => void;               // 用上次参数重新请求
  mutate: (data: T | ((old: T | undefined) => T)) => void;  // 手动修改 data
}

function useRequest<T, P extends unknown[] = []>(
  service: (...params: P) => Promise<T>,
  options: UseRequestOptions<T, P> = {}
): UseRequestReturn<T, P> {
  const {
    manual = false,
    defaultParams = [] as unknown as P,
    onSuccess,
    onError,
    onFinally,
    retryCount = 0,
    retryInterval = 1000,
    pollingInterval = 0,
    pollingWhenHidden = false,
    debounceWait,
    throttleWait,
  } = options;

  const [state, setState] = useState<UseRequestState<T>>({
    data: undefined,
    error: undefined,
    status: 'idle',
    loading: !manual,  // 自动执行时初始为 loading
  });

  // 用 ref 保存不触发重渲染的状态
  const abortControllerRef = useRef<AbortController | null>(null);
  const retryCountRef = useRef(0);
  const pollingTimerRef = useRef<NodeJS.Timeout | null>(null);
  const latestParamsRef = useRef<P>(defaultParams);
  const serviceRef = useRef(service);
  serviceRef.current = service;  // 始终指向最新的 service

  // 核心执行函数
  const runAsync = useCallback(async (...params: P): Promise<T> => {
    // 取消上一次未完成的请求（解决竞态条件）
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }
    const controller = new AbortController();
    abortControllerRef.current = controller;

    latestParamsRef.current = params;
    setState(prev => ({ ...prev, status: 'loading', loading: true }));

    try {
      // 将 AbortSignal 传给 service（service 内部需要支持 signal）
      // 这里通过 AbortController 传递，实际可以通过 context 或参数
      const data = await serviceRef.current(...params);

      // 如果请求已被取消（组件卸载 / 新请求发出），忽略此次响应
      if (controller.signal.aborted) {
        return undefined as unknown as T;
      }

      setState({ data, error: undefined, status: 'success', loading: false });
      retryCountRef.current = 0;  // 重置重试计数
      onSuccess?.(data, params);
      return data;
    } catch (err) {
      if (controller.signal.aborted) {
        return undefined as unknown as T;
      }

      const error = err instanceof Error ? err : new Error(String(err));

      // 重试逻辑
      if (retryCountRef.current < retryCount) {
        retryCountRef.current++;
        const delay = retryInterval * Math.pow(2, retryCountRef.current - 1);  // 指数退避
        await new Promise(r => setTimeout(r, delay));
        return runAsync(...params);
      }

      setState({ data: undefined, error, status: 'error', loading: false });
      onError?.(error, params);
      throw error;
    } finally {
      if (!controller.signal.aborted) {
        onFinally?.(params);
      }
    }
  }, [onSuccess, onError, onFinally, retryCount, retryInterval]);

  // 包装 run（不抛出错误，适合事件处理器）
  const run = useCallback((...params: P) => {
    runAsync(...params).catch(() => {});  // 错误通过 onError 处理
  }, [runAsync]);

  // 防抖 / 节流包装
  const debouncedRun = useMemo(() => {
    if (debounceWait) return debounce(run, debounceWait);
    if (throttleWait) return throttle(run, throttleWait);
    return run;
  }, [run, debounceWait, throttleWait]);

  // 取消当前请求
  const cancel = useCallback(() => {
    abortControllerRef.current?.abort();
    if (pollingTimerRef.current) {
      clearTimeout(pollingTimerRef.current);
    }
    setState(prev => ({ ...prev, status: 'idle', loading: false }));
  }, []);

  // 用上次参数刷新
  const refresh = useCallback(() => {
    run(...latestParamsRef.current);
  }, [run]);

  // 手动修改数据（乐观更新用）
  const mutate = useCallback((updater: T | ((old: T | undefined) => T)) => {
    setState(prev => ({
      ...prev,
      data: typeof updater === 'function'
        ? (updater as (old: T | undefined) => T)(prev.data)
        : updater,
    }));
  }, []);

  // 自动执行（非 manual 模式）
  useEffect(() => {
    if (!manual) {
      run(...defaultParams);
    }
  }, []);  // 只在挂载时执行

  // 轮询
  useEffect(() => {
    if (!pollingInterval) return;

    const poll = () => {
      if (!pollingWhenHidden && document.visibilityState === 'hidden') {
        pollingTimerRef.current = setTimeout(poll, pollingInterval);
        return;
      }
      run(...latestParamsRef.current);
      pollingTimerRef.current = setTimeout(poll, pollingInterval);
    };

    pollingTimerRef.current = setTimeout(poll, pollingInterval);

    return () => {
      if (pollingTimerRef.current) clearTimeout(pollingTimerRef.current);
    };
  }, [pollingInterval, pollingWhenHidden]);

  // 组件卸载时取消请求（防内存泄漏）
  useEffect(() => {
    return () => {
      abortControllerRef.current?.abort();
      if (pollingTimerRef.current) clearTimeout(pollingTimerRef.current);
    };
  }, []);

  return {
    ...state,
    run: debouncedRun as typeof run,
    runAsync,
    cancel,
    refresh,
    mutate,
  };
}
```

---

## 解决竞态条件：AbortController vs 版本号

```typescript
// 方案一：AbortController（推荐）
// 真正取消 HTTP 请求，节省带宽

async function searchUsers(query: string, signal?: AbortSignal) {
  const res = await fetch(`/api/users?q=${encodeURIComponent(query)}`, { signal });
  if (!res.ok) throw new Error('Search failed');
  return res.json();
}

// 在 useRequest 中使用时需要传 signal
// 实际 ahooks 通过包装 service 传入 signal：
function useRequest_with_abort<T, P extends unknown[]>(
  service: (signal: AbortSignal, ...params: P) => Promise<T>,
  options: UseRequestOptions<T, P> = {}
) {
  // ... 在 runAsync 中 service(controller.signal, ...params)
}

// 方案二：版本号（stale closure）
// 不取消网络请求，但丢弃旧响应
function useRequestWithVersion<T>(fn: () => Promise<T>) {
  const versionRef = useRef(0);
  const [state, setState] = useState<{ data?: T; loading: boolean }>({ loading: false });

  const run = useCallback(async () => {
    const version = ++versionRef.current;  // 每次请求自增版本
    setState({ loading: true });

    const data = await fn();

    // 只接受最新版本的响应（旧响应被丢弃）
    if (version === versionRef.current) {
      setState({ data, loading: false });
    }
  }, [fn]);

  return { ...state, run };
}
```

---

## 使用示例

```typescript
// 1. 自动请求（组件挂载时执行）
function UserProfile({ userId }: { userId: string }) {
  const { data, loading, error, refresh } = useRequest(
    () => fetchUser(userId),
    {
      onSuccess: (user) => console.log('Loaded:', user.name),
      retryCount: 2,
    }
  );

  if (loading) return <Skeleton />;
  if (error) return <Button onClick={refresh}>重试</Button>;
  return <div>{data!.name}</div>;
}

// 2. 搜索框（防抖 + 竞态保护）
function SearchBox() {
  const [query, setQuery] = useState('');
  const { data, loading, run } = useRequest(
    (q: string) => searchUsers(q),
    {
      manual: true,
      debounceWait: 300,  // 防抖 300ms
    }
  );

  return (
    <div>
      <input
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);
          run(e.target.value);  // 防抖处理，快速输入只发最后一次
        }}
      />
      {loading && <Spinner />}
      {data?.map(u => <UserItem key={u.id} user={u} />)}
    </div>
  );
}

// 3. 手动触发（表单提交）
function CreateUserForm() {
  const { loading, run, error } = useRequest(
    (formData: CreateUserInput) => createUser(formData),
    {
      manual: true,
      onSuccess: () => router.push('/users'),
      onError: (err) => toast.error(err.message),
    }
  );

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      run(getFormData(e.currentTarget));
    }}>
      <button type="submit" disabled={loading}>
        {loading ? '提交中...' : '创建'}
      </button>
      {error && <p>{error.message}</p>}
    </form>
  );
}

// 4. 轮询（实时订单状态）
function OrderTracker({ orderId }: { orderId: string }) {
  const { data } = useRequest(
    () => fetchOrder(orderId),
    {
      pollingInterval: 3000,
      pollingWhenHidden: false,
    }
  );

  return <div>Status: {data?.status}</div>;
}

// 5. 乐观更新（点赞）
function LikeButton({ postId, initialLiked }: { postId: string; initialLiked: boolean }) {
  const { data, mutate, run } = useRequest(
    () => fetchPost(postId),
    { manual: true }
  );

  const isLiked = data?.isLiked ?? initialLiked;

  const handleLike = () => {
    // 乐观更新：立即翻转状态
    mutate(old => ({ ...old!, isLiked: !isLiked, likeCount: (old?.likeCount ?? 0) + (isLiked ? -1 : 1) }));
    // 发请求（失败时需要回滚，此处简化）
    fetch(`/api/posts/${postId}/like`, { method: 'POST' });
  };

  return <button onClick={handleLike}>{isLiked ? '已赞' : '点赞'}</button>;
}
```

---

## 面试追问

**Q: AbortController 取消请求后，Promise 会变成什么状态？**
A: AbortController 取消后，`fetch` 会 reject 一个 `DOMException`，其 `name` 为 `"AbortError"`。需要在 catch 中判断：`if (err.name === 'AbortError') return;`，不把取消当错误处理。axios 类似，取消后 reject 一个 `CanceledError`，可以用 `axios.isCancel(err)` 判断。ioredis 等非 HTTP 请求不支持 AbortController，需要通过版本号方案处理竞态。

**Q: 为什么用 ref 保存 abortController 而不是 state？**
A: `useRef` 的修改不触发重渲染，且在 `useEffect` cleanup 和事件处理器中读取的是最新值（不受 stale closure 影响）。`AbortController` 是副作用管理的基础设施，不需要也不应该出现在渲染逻辑中，放 `ref` 正确。如果放 `state`，每次 cancel 都会触发额外渲染。

**Q: useRequest 和 TanStack Query 什么时候用哪个？**
A: TanStack Query 适合数据获取（缓存、后台刷新、跨组件共享同一份数据）；useRequest 适合操作类请求（表单提交、文件上传）或不需要缓存的场景（搜索接口、带参数的 lazy 请求）。实际项目里两者配合：GET 数据用 TQ，POST/PUT/DELETE 用 TQ 的 `useMutation`，复杂的 UI 操作逻辑（防抖、轮询、竞态）可以封装自定义 hook。
