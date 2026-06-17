# TanStack Query 进阶

> useInfiniteQuery、RSC + 服务端预取、select 防重渲染、依赖查询、Suspense 模式。
> 基础用法见 `frontend/system_design/06_state_management.md`，本篇覆盖面试差异化考点。

---

## useInfiniteQuery：无限滚动

```typescript
// 与 useQuery 的区别：数据是"页"的数组，而不是单次结果

interface PostsPage {
  posts: Post[];
  nextCursor: string | null;
  hasNextPage: boolean;
}

// 关键参数：
// getNextPageParam: 从上一页结果中提取下一页的 cursor
// initialPageParam: 第一次请求时的初始参数

function useInfinitePosts(filters: PostFilters) {
  return useInfiniteQuery({
    queryKey: ['posts', 'infinite', filters],
    queryFn: async ({ pageParam }) => {
      const res = await fetch(
        `/api/posts?cursor=${pageParam ?? ''}&limit=20&${new URLSearchParams(filters as any)}`
      );
      return res.json() as Promise<PostsPage>;
    },
    initialPageParam: '',  // 第一页 cursor 为空
    getNextPageParam: (lastPage) => lastPage.nextCursor,  // null 时停止
    staleTime: 30_000,
  });
}

// 组件使用
function PostFeed({ filters }: { filters: PostFilters }) {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
  } = useInfinitePosts(filters);

  // data.pages: PostsPage[]
  // data.pageParams: cursor[]
  const allPosts = data?.pages.flatMap(page => page.posts) ?? [];

  // IntersectionObserver 触发加载更多
  const loadMoreRef = useRef<HTMLDivElement>(null);
  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 0.1 }
    );
    if (loadMoreRef.current) observer.observe(loadMoreRef.current);
    return () => observer.disconnect();
  }, [fetchNextPage, hasNextPage, isFetchingNextPage]);

  if (isLoading) return <FeedSkeleton />;
  if (isError) return <ErrorMessage />;

  return (
    <div>
      {allPosts.map(post => <PostCard key={post.id} post={post} />)}
      <div ref={loadMoreRef} />
      {isFetchingNextPage && <Spinner />}
      {!hasNextPage && <p>全部加载完毕</p>}
    </div>
  );
}
```

---

## RSC + TanStack Query：服务端预取 + 客户端 Hydration

```
为什么需要这个模式？

纯 RSC（Server Component）：数据在服务端，无客户端交互（分页、刷新）。
纯 TanStack Query：数据在客户端，有加载状态，但首屏无数据（白屏）。

最佳组合：RSC 预取数据注入 TanStack Query 缓存 → 首屏无加载状态，
           之后客户端 TQ 接管（后台刷新、乐观更新、分页等）。
```

```typescript
// app/posts/page.tsx — Server Component 预取
import { HydrationBoundary, QueryClient, dehydrate } from '@tanstack/react-query';
import { PostFeedClient } from './PostFeedClient';

export default async function PostsPage({
  searchParams,
}: {
  searchParams: { category?: string };
}) {
  const queryClient = new QueryClient();

  // 服务端预取：注入 TanStack Query 缓存
  await queryClient.prefetchQuery({
    queryKey: ['posts', { category: searchParams.category }],
    queryFn: () => fetchPostsFromDB({ category: searchParams.category }),
  });

  // 无限滚动预取第一页
  await queryClient.prefetchInfiniteQuery({
    queryKey: ['posts', 'infinite', { category: searchParams.category }],
    queryFn: ({ pageParam }) => fetchPostsFromDB({ cursor: pageParam, limit: 20 }),
    initialPageParam: '',
    pages: 1,  // 只预取第一页
  });

  return (
    // dehydrate 将缓存序列化后传给客户端
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostFeedClient category={searchParams.category} />
    </HydrationBoundary>
  );
}

// components/PostFeedClient.tsx — Client Component 接管
'use client';
import { useInfiniteQuery } from '@tanstack/react-query';

export function PostFeedClient({ category }: { category?: string }) {
  const query = useInfiniteQuery({
    queryKey: ['posts', 'infinite', { category }],
    queryFn: ({ pageParam }) =>
      fetch(`/api/posts?cursor=${pageParam}&category=${category ?? ''}`).then(r => r.json()),
    initialPageParam: '',
    getNextPageParam: (page) => page.nextCursor,
    // 服务端预取的数据在 Hydration 时直接填充，组件挂载时不会触发额外请求
    staleTime: 30_000,
  });
  // ...
}
```

---

## select：数据转换 + 防止不必要重渲染

```typescript
// 问题：后端返回 { users: User[], total: number }
// 组件 A 只关心 users，组件 B 只关心 total
// 如果 total 变了，组件 A 应该不重渲染

const userListQuery = useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
  // select 在缓存数据变化时调用，返回值会参与比较
  // 如果返回值（users 数组引用）没变，组件不重渲染
  select: (data) => data.users,
});

const totalQuery = useQuery({
  queryKey: ['users'],       // 同一个缓存 key
  queryFn: fetchUsers,       // 只请求一次
  select: (data) => data.total,  // 不同的 select
});

// 更实用：数据转换（排序、过滤、映射）
const sortedQuery = useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  select: (data) =>
    [...data.products]
      .sort((a, b) => b.price - a.price)
      .filter(p => p.inStock),
});

// select 是 memoized 的（stale data 且没变化时不重算）
// 传稳定引用避免无限重渲染
const selectTopUsers = useCallback(
  (data: UsersResponse) => data.users.slice(0, 5),
  []  // 空依赖：引用稳定
);
const topUsersQuery = useQuery({ queryKey: ['users'], queryFn: fetchUsers, select: selectTopUsers });
```

---

## enabled：依赖查询（Dependent Queries）

```typescript
// 第二个查询依赖第一个查询的结果（如：先查用户，再查该用户的订单）

function UserOrders({ userId }: { userId?: string }) {
  // 第一个查询
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId!),
    enabled: !!userId,  // userId 不存在时不发请求
  });

  // 第二个查询依赖第一个
  const ordersQuery = useQuery({
    queryKey: ['orders', userQuery.data?.id],
    queryFn: () => fetchOrders(userQuery.data!.id),
    enabled: !!userQuery.data?.id,  // 等第一个查询成功后才发请求
  });

  if (userQuery.isLoading) return <Skeleton />;
  if (ordersQuery.isLoading) return <div>Loading orders...</div>;

  return <OrderList orders={ordersQuery.data} />;
}

// 更复杂：并行查询 + 聚合（useQueries）
function UserDashboard({ userIds }: { userIds: string[] }) {
  const queries = useQueries({
    queries: userIds.map(id => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
      staleTime: 60_000,
    })),
  });

  const users = queries
    .filter(q => q.status === 'success')
    .map(q => q.data!);

  const isLoading = queries.some(q => q.isLoading);
  return isLoading ? <Skeleton /> : <UserGrid users={users} />;
}
```

---

## refetchInterval：轮询

```typescript
// 实时数据（不用 WebSocket 的轻量方案）

function OrderStatus({ orderId }: { orderId: string }) {
  const { data } = useQuery({
    queryKey: ['order', orderId],
    queryFn: () => fetchOrder(orderId),
    // 每 3 秒轮询，直到订单完成
    refetchInterval: (query) => {
      const status = query.state.data?.status;
      // 终态时停止轮询（返回 false）
      if (status === 'delivered' || status === 'cancelled') return false;
      return 3000;
    },
    refetchIntervalInBackground: false,  // 页面不可见时停止轮询（省电）
  });

  return <div>Status: {data?.status}</div>;
}

// 结合 WebSocket 实现"WebSocket 优先，轮询兜底"
function useRealtimeOrder(orderId: string) {
  const queryClient = useQueryClient();

  // WebSocket 更新缓存
  useEffect(() => {
    const ws = new WebSocket(`/ws/orders/${orderId}`);
    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      queryClient.setQueryData(['order', orderId], update);
    };
    return () => ws.close();
  }, [orderId, queryClient]);

  // 轮询作为兜底（WebSocket 断线时仍然工作）
  return useQuery({
    queryKey: ['order', orderId],
    queryFn: () => fetchOrder(orderId),
    refetchInterval: 30_000,  // WebSocket 存在时几乎不触发（数据是 fresh 的）
  });
}
```

---

## Suspense 模式

```typescript
// Suspense 模式：loading 状态用 React Suspense 处理，不需要手动判断 isLoading

// 配置
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      suspense: true,  // 全局开启（或在每个 useQuery 单独设置）
    },
  },
});

// 或者用 useSuspenseQuery（TanStack Query v5 推荐方式）
import { useSuspenseQuery, useSuspenseInfiniteQuery } from '@tanstack/react-query';

function ProductDetail({ id }: { id: string }) {
  // 不需要处理 isLoading，Suspense 自动处理
  // data 一定有值（不是 undefined）
  const { data: product } = useSuspenseQuery({
    queryKey: ['product', id],
    queryFn: () => fetchProduct(id),
  });

  return <div>{product.name}</div>;  // TypeScript 知道 product 不是 undefined
}

// 父组件用 Suspense + ErrorBoundary 包裹
function ProductPage({ id }: { id: string }) {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<ProductSkeleton />}>
        <ProductDetail id={id} />
      </Suspense>
    </ErrorBoundary>
  );
}

// 好处：多个 useSuspenseQuery 可以并行（React 18 并发模式）
function Dashboard() {
  return (
    <Suspense fallback={<DashboardSkeleton />}>
      <DashboardContent />  {/* 内部多个 useSuspenseQuery 并行请求 */}
    </Suspense>
  );
}

function DashboardContent() {
  const { data: stats } = useSuspenseQuery({ queryKey: ['stats'], queryFn: fetchStats });
  const { data: recent } = useSuspenseQuery({ queryKey: ['recent'], queryFn: fetchRecent });
  // 两个请求并行，都完成后才渲染（整体 Suspense）
  return <div>...</div>;
}
```

---

## 缓存失效策略

```typescript
const queryClient = useQueryClient();

// 失效整个查询键
queryClient.invalidateQueries({ queryKey: ['posts'] });
// 失效所有以 ['posts'] 开头的 key
queryClient.invalidateQueries({ queryKey: ['posts'], exact: false });

// 直接更新缓存（用于乐观更新后的确认，或 WebSocket 推送）
queryClient.setQueryData(['post', postId], (old: Post | undefined) => ({
  ...old!,
  likeCount: old!.likeCount + 1,
}));

// 删除缓存（强制下次重新请求）
queryClient.removeQueries({ queryKey: ['user', userId] });

// 完整的 Mutation 生命周期：乐观更新 + 失败回滚
const updatePostMutation = useMutation({
  mutationFn: (data: { id: string; title: string }) =>
    fetch(`/api/posts/${data.id}`, { method: 'PATCH', body: JSON.stringify(data) }).then(r => r.json()),

  onMutate: async (variables) => {
    await queryClient.cancelQueries({ queryKey: ['post', variables.id] });
    const snapshot = queryClient.getQueryData(['post', variables.id]);
    queryClient.setQueryData(['post', variables.id], (old: Post) => ({
      ...old,
      title: variables.title,
    }));
    return { snapshot };  // 传给 onError
  },

  onError: (_err, variables, context) => {
    // 回滚乐观更新
    queryClient.setQueryData(['post', variables.id], context?.snapshot);
  },

  onSettled: (_data, _err, variables) => {
    // 无论成功失败，最终都重新获取服务端真实数据
    queryClient.invalidateQueries({ queryKey: ['post', variables.id] });
  },
});
```

---

## 面试追问

**Q: TanStack Query 的 staleTime 和 gcTime（cacheTime）有什么区别？**
A: `staleTime`：数据被视为"新鲜"的时长，新鲜期内不会发后台请求（即使 refetchOnWindowFocus 也不触发）。`gcTime`（原 cacheTime）：没有组件订阅后，数据在缓存中保留多久才被删除。典型配置：`staleTime: 30s, gcTime: 5min` — 30s 内数据新鲜不请求，30s 后后台刷新，5min 无人使用后从内存删除。`staleTime: Infinity` 等于说"永远不自动刷新，只有手动 invalidate"。

**Q: RSC 直接 fetch 和 TanStack Query prefetchQuery 有什么区别？**
A: RSC 直接 fetch 适合纯展示、不需要客户端互动的数据（如博文内容）；TanStack Query prefetchQuery 适合客户端需要继续操作的数据（乐观更新、分页、手动刷新）。区别：prefetchQuery 的数据会被序列化传给客户端，客户端 TQ 能感知并接管，无需重新请求；RSC 直接 fetch 的数据就是服务端 HTML，客户端 TQ 对此无感知（如果客户端再次 useQuery 同一数据，会独立发请求）。

**Q: useInfiniteQuery 与 FlatList/虚拟列表集成时注意什么？**
A: `data.pages.flatMap(p => p.items)` 每次渲染都会生成新数组引用，传给虚拟列表会触发重渲染。解法：用 `useMemo` memoize 扁平数组；或使用 TanStack Virtual + IntersectionObserver 分开管理：TQ 负责数据分页，Virtual 负责 DOM 虚拟化，两者通过 `allItems` 数组对接。
