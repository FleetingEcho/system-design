# Twitter/Instagram Feed 前端设计

> 高频前端系统设计题。考察：虚拟列表、无限滚动、实时推送、乐观更新。

---

## 需求理解（先问）

```
功能需求：
  - 显示关注用户的帖子流（文字 + 图片 + 视频）
  - 无限滚动加载（向下滚动自动加载更多）
  - 实时新帖通知（"查看 N 条新帖"提示）
  - 点赞/转发/评论（乐观更新 + 失败回滚）
  - 下拉刷新

非功能需求：
  - 100ms 内响应交互（点赞等）
  - 流畅滚动（60fps，无白屏）
  - 移动端优先
  - 离线支持（已加载内容可查看）
```

---

## 架构概览

```
┌────────────────────────────────────────────────────────────────┐
│                        Feed 组件树                              │
│                                                                │
│  FeedContainer                                                 │
│  ├── NewPostsBar ("查看 5 条新帖")  ← WebSocket/SSE 推送       │
│  ├── VirtualList                    ← 性能核心                  │
│  │   ├── PostCard[0]  (可见)                                   │
│  │   ├── PostCard[1]  (可见)                                   │
│  │   └── ...          (不可见的只渲染占位 div)                  │
│  └── LoadMoreSentinel  ← IntersectionObserver 触发加载         │
│                                                                │
│  数据流：                                                       │
│  FeedStore (Zustand)                                           │
│  ├── posts: Post[]         （已加载帖子）                       │
│  ├── newPosts: Post[]      （实时推送未显示的帖子）             │
│  ├── cursor: string        （分页游标）                         │
│  └── optimisticLikes: Map  （乐观更新中的点赞状态）             │
└────────────────────────────────────────────────────────────────┘
```

---

## 无限滚动

```typescript
// 使用 IntersectionObserver 监听"哨兵元素"进入视口
function useInfiniteScroll(onLoadMore: () => Promise<void>) {
  const sentinelRef = useRef<HTMLDivElement>(null);
  const isLoading = useRef(false);

  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    const observer = new IntersectionObserver(
      async ([entry]) => {
        if (entry.isIntersecting && !isLoading.current) {
          isLoading.current = true;
          try {
            await onLoadMore();
          } finally {
            isLoading.current = false;
          }
        }
      },
      {
        rootMargin: '400px',  // 距离底部 400px 时预加载
        threshold: 0,
      }
    );

    observer.observe(sentinel);
    return () => observer.disconnect();
  }, [onLoadMore]);

  return sentinelRef;
}

// 分页 cursor（比 offset 更稳定，避免新帖插入导致重复/跳过）
interface FeedCursor {
  afterId: string;    // 从该帖子 ID 之后加载
  limit: number;
}

async function loadMorePosts(cursor: FeedCursor): Promise<Post[]> {
  const res = await fetch(`/api/feed?after=${cursor.afterId}&limit=${cursor.limit}`);
  return res.json();
}
```

---

## 虚拟列表（Virtual List）

核心思路：只渲染可视区域内的元素，其他元素用占位 `div` 代替。

```typescript
interface VirtualListProps {
  items: Post[];
  estimatedItemHeight: number;  // 预估高度（实际高度动态测量）
  overscan?: number;             // 视口上下额外渲染的项数
  onItemHeightChange?: (id: string, height: number) => void;
}

function VirtualList({ items, estimatedItemHeight, overscan = 3 }: VirtualListProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [scrollTop, setScrollTop] = useState(0);
  const [containerHeight, setContainerHeight] = useState(0);

  // 缓存每个 item 的实际高度
  const heightCache = useRef(new Map<string, number>());
  const getHeight = (id: string) => heightCache.current.get(id) ?? estimatedItemHeight;

  // 计算每个 item 的 offsetTop（累加）
  const offsets = useMemo(() => {
    const result: number[] = [];
    let total = 0;
    items.forEach((item, i) => {
      result[i] = total;
      total += getHeight(item.id);
    });
    return result;
  }, [items, heightCache.current.size]);

  const totalHeight = offsets[items.length - 1] + getHeight(items[items.length - 1]?.id ?? '');

  // 计算可见区域内的 item 范围
  const { startIndex, endIndex } = useMemo(() => {
    // 二分查找第一个可见 item
    let start = 0;
    let lo = 0, hi = items.length - 1;
    while (lo <= hi) {
      const mid = (lo + hi) >> 1;
      if (offsets[mid] < scrollTop - estimatedItemHeight * overscan) {
        lo = mid + 1;
        start = mid + 1;
      } else {
        hi = mid - 1;
      }
    }

    let end = start;
    while (end < items.length && offsets[end] < scrollTop + containerHeight + estimatedItemHeight * overscan) {
      end++;
    }

    return {
      startIndex: Math.max(0, start),
      endIndex: Math.min(items.length - 1, end),
    };
  }, [scrollTop, containerHeight, offsets, items.length, estimatedItemHeight, overscan]);

  const visibleItems = items.slice(startIndex, endIndex + 1);

  return (
    <div
      ref={containerRef}
      style={{ height: '100%', overflowY: 'scroll', position: 'relative' }}
      onScroll={(e) => setScrollTop((e.target as HTMLDivElement).scrollTop)}
    >
      {/* 撑开总高度 */}
      <div style={{ height: totalHeight, position: 'relative' }}>
        {visibleItems.map((item, i) => {
          const index = startIndex + i;
          return (
            <div
              key={item.id}
              style={{
                position: 'absolute',
                top: offsets[index],
                width: '100%',
              }}
            >
              <PostCard
                post={item}
                onHeightChange={(height) => {
                  if (heightCache.current.get(item.id) !== height) {
                    heightCache.current.set(item.id, height);
                    // 触发重新计算
                  }
                }}
              />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

---

## 实时新帖推送

```typescript
// SSE（服务器推送事件）监听新帖
function useNewPosts(onNewPost: (post: Post) => void) {
  useEffect(() => {
    const es = new EventSource('/api/feed/stream');

    es.addEventListener('new_post', (e) => {
      const post = JSON.parse(e.data) as Post;
      onNewPost(post);
    });

    es.addEventListener('error', () => {
      // SSE 断线时自动重连（浏览器内置）
      console.warn('SSE connection error, browser will retry');
    });

    return () => es.close();
  }, [onNewPost]);
}

// 新帖提示条（点击后插入到列表顶部并滚动到顶部）
function NewPostsBar({ count, onShow }: { count: number; onShow: () => void }) {
  if (count === 0) return null;
  return (
    <button
      className="new-posts-bar"
      onClick={onShow}
      aria-live="polite"
      aria-label={`查看 ${count} 条新帖`}
    >
      ↑ 查看 {count} 条新帖
    </button>
  );
}
```

---

## 乐观更新（点赞）

```typescript
interface FeedStore {
  posts: Post[];
  optimisticLikes: Map<string, { original: number; optimistic: number }>;
  likePost: (postId: string) => Promise<void>;
}

const useFeedStore = create<FeedStore>((set, get) => ({
  posts: [],
  optimisticLikes: new Map(),

  likePost: async (postId: string) => {
    const post = get().posts.find(p => p.id === postId);
    if (!post) return;

    const originalCount = post.likeCount;

    // 1. 立即更新 UI（乐观）
    set(state => ({
      posts: state.posts.map(p =>
        p.id === postId
          ? { ...p, likeCount: p.likeCount + 1, isLiked: true }
          : p
      ),
      optimisticLikes: new Map(state.optimisticLikes).set(postId, {
        original: originalCount,
        optimistic: originalCount + 1,
      }),
    }));

    try {
      // 2. 发送 API 请求
      await fetch(`/api/posts/${postId}/like`, { method: 'POST' });

      // 3. 成功：清除乐观标记
      set(state => {
        const newMap = new Map(state.optimisticLikes);
        newMap.delete(postId);
        return { optimisticLikes: newMap };
      });
    } catch (error) {
      // 4. 失败：回滚到原始状态
      set(state => ({
        posts: state.posts.map(p =>
          p.id === postId
            ? { ...p, likeCount: originalCount, isLiked: false }
            : p
        ),
        optimisticLikes: (() => {
          const newMap = new Map(state.optimisticLikes);
          newMap.delete(postId);
          return newMap;
        })(),
      }));

      // 5. 提示用户
      toast.error('点赞失败，请重试');
    }
  },
}));
```

---

## 图片加载策略

```typescript
function PostMedia({ media }: { media: Post['media'] }) {
  return (
    <div className="post-media">
      {media.map((item) => (
        item.type === 'image' ? (
          // 原生懒加载 + srcset 响应式
          <img
            key={item.id}
            src={item.thumbnailUrl}       // 先显示缩略图
            data-src={item.fullUrl}
            loading="lazy"
            decoding="async"
            srcSet={`${item.url_400} 400w, ${item.url_800} 800w`}
            sizes="(max-width: 600px) 400px, 800px"
            alt={item.altText}
            style={{ aspectRatio: `${item.width}/${item.height}` }}  // 防止 CLS
          />
        ) : (
          // 视频：不自动加载，用户交互才加载
          <video
            key={item.id}
            preload="none"
            poster={item.posterUrl}
            controls
          >
            <source src={item.url} type="video/mp4" />
          </video>
        )
      ))}
    </div>
  );
}
```

---

## 性能优化总结

| 问题 | 解决方案 |
|------|---------|
| 长列表 DOM 节点过多 | Virtual List，只渲染可见项 |
| 滚动时频繁加载 | IntersectionObserver + 节流 + 防止并发 |
| 点赞响应慢 | 乐观更新，失败回滚 |
| 图片 CLS（布局偏移） | 设置 `aspect-ratio` 预占位 |
| 图片加载慢 | lazy loading + srcset 响应式 + blur placeholder |
| 新帖刷新 | SSE 推送，不轮询 |
| 组件重渲染 | `React.memo` + `useCallback` 稳定函数引用 |

---

## 面试追问

**Q: 为什么用 cursor 分页而不是 offset 分页？**
A: offset 分页在新帖插入时会导致数据重复（翻页期间新帖把原来第 N 页的内容推到第 N+1 页）或跳过。cursor 分页基于最后一条帖子的 ID，新数据插入不影响已有位置，且性能更好（`WHERE id < cursor LIMIT n` 比 `LIMIT n OFFSET m` 在大偏移时快得多）。

**Q: Virtual List 如何处理动态高度？**
A: 初始用预估高度，渲染后用 `ResizeObserver` 测量实际高度并更新 heightCache，重新计算 offsets（所有后续元素的 top 都要重算）。生产中可以用 `react-virtual`（TanStack Virtual）库，它已经处理了这些细节。

**Q: 乐观更新失败回滚时，如果用户已经进行了多个操作怎么办？**
A: 需要保存"原始服务器状态"而不是"前一次状态"。在 `optimisticLikes` 中存储服务器确认的原始值，失败时回滚到该值，不受中间乐观操作影响。同时需要排队处理（如用户快速双击点赞），用请求序列号或幂等键防止竞态。
