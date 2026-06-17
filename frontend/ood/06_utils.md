# 实现工具函数：Debounce / Throttle / LazyLoad / Memoize

> 考察点：闭包、定时器管理、IntersectionObserver、缓存策略。
> 高频前端手写题，每个工具函数背后都有一个清晰的设计问题。

---

## Debounce（防抖）

**问题**：搜索框输入时，每次按键都触发 API 请求，浪费资源。
**方案**：延迟执行，如果在等待期内再次触发，重新计时。

```typescript
function debounce<T extends unknown[], R>(
  fn: (...args: T) => R,
  delay: number,
  options?: { leading?: boolean; trailing?: boolean }
): (...args: T) => void {
  const { leading = false, trailing = true } = options ?? {};
  let timer: ReturnType<typeof setTimeout> | null = null;
  let lastCallTime: number | null = null;

  return function (...args: T) {
    const isFirstCall = leading && lastCallTime === null;

    if (timer) clearTimeout(timer);
    lastCallTime = Date.now();

    // leading 模式：第一次立即执行
    if (isFirstCall) fn(...args);

    // trailing 模式：延迟后执行最后一次调用
    if (trailing) {
      timer = setTimeout(() => {
        if (trailing && !isFirstCall) fn(...args);
        timer = null;
        lastCallTime = null;
      }, delay);
    }
  };
}

// 取消功能（用于组件卸载时清理）
function debounceWithCancel<T extends unknown[], R>(
  fn: (...args: T) => R,
  delay: number
) {
  let timer: ReturnType<typeof setTimeout> | null = null;

  const debounced = (...args: T) => {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      fn(...args);
      timer = null;
    }, delay);
  };

  debounced.cancel = () => {
    if (timer) {
      clearTimeout(timer);
      timer = null;
    }
  };

  debounced.flush = (...args: T) => {
    debounced.cancel();
    fn(...args);
  };

  return debounced;
}

// 使用
const searchUsers = debounceWithCancel(
  (query: string) => fetch(`/api/search?q=${query}`),
  300
);

// React 组件卸载时
// useEffect(() => () => searchUsers.cancel(), []);
```

---

## Throttle（节流）

**问题**：滚动事件每秒触发 60 次，每次都执行复杂计算，卡顿。
**方案**：固定时间窗口内只执行一次。

```typescript
// 时间戳版（leading + 不精确 trailing）
function throttleTimestamp<T extends unknown[], R>(
  fn: (...args: T) => R,
  interval: number
): (...args: T) => void {
  let lastTime = 0;

  return function (...args: T) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn(...args);
    }
  };
}

// 定时器版（有 trailing，最后一次调用会在 interval 后执行）
function throttleTimer<T extends unknown[], R>(
  fn: (...args: T) => R,
  interval: number
): (...args: T) => void {
  let timer: ReturnType<typeof setTimeout> | null = null;

  return function (...args: T) {
    if (!timer) {
      timer = setTimeout(() => {
        fn(...args);
        timer = null;
      }, interval);
    }
  };
}

// 完整版：leading + trailing，精确控制
function throttle<T extends unknown[], R>(
  fn: (...args: T) => R,
  interval: number,
  options?: { leading?: boolean; trailing?: boolean }
): (...args: T) => void {
  const { leading = true, trailing = true } = options ?? {};
  let lastTime = 0;
  let timer: ReturnType<typeof setTimeout> | null = null;
  let lastArgs: T | null = null;

  const execute = (args: T) => {
    lastTime = Date.now();
    fn(...args);
  };

  return function (...args: T) {
    const now = Date.now();
    const remaining = interval - (now - lastTime);

    if (remaining <= 0 || remaining > interval) {
      if (timer) {
        clearTimeout(timer);
        timer = null;
      }
      if (leading || lastTime !== 0) execute(args);
      else lastTime = now;
    } else if (trailing) {
      lastArgs = args;
      if (!timer) {
        timer = setTimeout(() => {
          lastTime = leading ? Date.now() : 0;
          timer = null;
          if (lastArgs) execute(lastArgs);
          lastArgs = null;
        }, remaining);
      }
    }
  };
}

// Debounce vs Throttle 使用场景对比：
// debounce：搜索框输入、窗口 resize 结束后计算、表单 validation
// throttle：scroll 事件、mousemove 轨迹追踪、游戏帧率控制
```

---

## LazyLoad（图片懒加载）

**问题**：页面有 100 张图片，首次加载全部请求，带宽浪费且页面慢。
**方案**：图片进入视口时再加载。

```typescript
class LazyLoader {
  private observer: IntersectionObserver;
  private loadedImages = new WeakSet<Element>();

  constructor(options?: {
    rootMargin?: string;   // 提前加载距离
    threshold?: number;    // 可见比例触发点
    placeholder?: string;  // 占位图 URL
  }) {
    const {
      rootMargin = '200px',  // 提前 200px 开始加载
      threshold = 0,
    } = options ?? {};

    this.observer = new IntersectionObserver(
      this._handleIntersection.bind(this),
      { rootMargin, threshold }
    );
  }

  // 观察单张图片（data-src 存放真实 URL）
  observe(img: HTMLImageElement): void {
    if (this.loadedImages.has(img)) return;
    this.observer.observe(img);
  }

  // 观察容器内所有懒加载图片
  observeAll(container: Element = document.body): void {
    const images = container.querySelectorAll<HTMLImageElement>('img[data-src]');
    images.forEach(img => this.observe(img));
  }

  disconnect(): void {
    this.observer.disconnect();
  }

  private _handleIntersection(entries: IntersectionObserverEntry[]): void {
    entries.forEach(entry => {
      if (!entry.isIntersecting) return;

      const img = entry.target as HTMLImageElement;
      const src = img.dataset.src;
      if (!src) return;

      this._loadImage(img, src);
      this.observer.unobserve(img);
      this.loadedImages.add(img);
    });
  }

  private _loadImage(img: HTMLImageElement, src: string): void {
    // 先加载到内存，加载完成后再赋给 img.src（避免闪烁）
    const tempImg = new Image();
    tempImg.onload = () => {
      img.src = src;
      img.removeAttribute('data-src');
      img.classList.add('loaded');
    };
    tempImg.onerror = () => {
      img.classList.add('error');
    };
    tempImg.src = src;
  }
}

// HTML 使用：
// <img data-src="/images/photo.jpg" src="/placeholder.jpg" alt="..." />

// 初始化
const lazyLoader = new LazyLoader({ rootMargin: '300px' });
lazyLoader.observeAll();

// React Hook 版本
function useLazyImage(src: string) {
  const ref = useRef<HTMLImageElement>(null);
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    const img = ref.current;
    if (!img) return;

    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        img.src = src;
        img.onload = () => setLoaded(true);
        observer.disconnect();
      }
    }, { rootMargin: '200px' });

    observer.observe(img);
    return () => observer.disconnect();
  }, [src]);

  return { ref, loaded };
}
```

---

## Memoize（记忆化）

**问题**：纯函数被频繁以相同参数调用，重复计算浪费 CPU。
**方案**：缓存函数调用结果，相同参数直接返回缓存。

```typescript
// 基础版：单参数，用 Map 缓存
function memoize<T, R>(fn: (arg: T) => R): (arg: T) => R {
  const cache = new Map<T, R>();
  return (arg: T) => {
    if (cache.has(arg)) return cache.get(arg)!;
    const result = fn(arg);
    cache.set(arg, result);
    return result;
  };
}

// 多参数版：序列化 key
function memoizeMulti<T extends unknown[], R>(
  fn: (...args: T) => R,
  resolver?: (...args: T) => string  // 自定义 key 生成器
): (...args: T) => R {
  const cache = new Map<string, R>();

  return (...args: T) => {
    const key = resolver ? resolver(...args) : JSON.stringify(args);

    if (cache.has(key)) return cache.get(key)!;
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// LRU 缓存版（限制缓存大小，防止内存泄漏）
function memoizeLRU<T extends unknown[], R>(
  fn: (...args: T) => R,
  maxSize = 100
): (...args: T) => R {
  // Map 保持插入顺序，可模拟 LRU
  const cache = new Map<string, R>();

  return (...args: T) => {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      // 访问时移到末尾（最近使用）
      const value = cache.get(key)!;
      cache.delete(key);
      cache.set(key, value);
      return value;
    }

    const result = fn(...args);

    // 超出容量时删除最久未使用的（Map 头部）
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }

    cache.set(key, result);
    return result;
  };
}

// 使用示例
const expensiveFibonacci = memoize((n: number): number => {
  if (n <= 1) return n;
  return expensiveFibonacci(n - 1) + expensiveFibonacci(n - 2);
});

// React.memo 等价物（组件级别）：对比 props 决定是否重渲染
// useMemo：缓存计算结果
// useCallback：缓存函数引用（函数的 memoize）
const memoizedValue = useMemo(() => expensiveCalc(a, b), [a, b]);
```

---

## 综合面试追问

**Q: Debounce 和 Throttle 如何选择？**
A: 关键区别是"最后一次调用"的处理。Debounce 等待最后一次调用后才执行（适合"等用户停下来再执行"，如搜索建议、resize 后重新布局）。Throttle 保证固定频率执行（适合"持续执行但限频"，如 scroll 计算位置、mousemove 绘图）。如果需要实时反馈但限频，用 Throttle；如果结果只取最终状态，用 Debounce。

**Q: IntersectionObserver 相比 scroll 事件的优势？**
A: scroll 事件在主线程同步触发，每次滚动可能每帧 60 次，且计算 `getBoundingClientRect` 触发 Layout 回流；IntersectionObserver 在后台线程（Compositor）异步触发，浏览器批量处理，不阻塞主线程，性能更好，且 API 更简洁（不需要手动计算可见性）。

**Q: Memoize 的 WeakMap 和 Map 有什么区别？**
A: Map 持有对 key 的强引用，如果 key 是对象（如 DOM 节点、React 组件），即使原始引用被释放，缓存中的对象也不会被 GC。WeakMap 对 key 是弱引用，key 被 GC 时缓存自动清理，适合以对象作为 key 的缓存场景，防止内存泄漏。

**Q: React.memo、useMemo、useCallback 的区别？**
A: `React.memo` 包裹组件，props 不变时跳过重渲染（组件级 memoize）；`useMemo` 缓存计算结果，依赖不变时返回上次的值（值级 memoize）；`useCallback` 缓存函数引用，依赖不变时返回同一个函数（函数引用的 memoize，防止 `useEffect` 的依赖数组每次都是新函数）。三者都是空间换时间，不要过度使用。
