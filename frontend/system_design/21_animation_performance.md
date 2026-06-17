# Web 动画性能

> 流畅动画的目标是 60fps（每帧 16.7ms）或 120fps（8.3ms）。
> 本文覆盖：浏览器渲染流水线、合成层优化、CSS vs JS 动画、requestAnimationFrame、Framer Motion。

---

## 浏览器渲染流水线（关键知识）

```
每一帧的工作：
  JS → Style → Layout → Paint → Composite
        ↑        ↑        ↑        ↑
      样式计算   布局计算  绘制像素  合成层叠加

触发不同阶段的代价：

高代价（触发 Layout + Paint + Composite）：
  width, height, top, left, margin, padding,
  font-size, display, position...

中代价（触发 Paint + Composite）：
  color, background, box-shadow, border-color,
  outline, visibility...

低代价（只触发 Composite）：
  transform: translate/rotate/scale
  opacity
  filter（部分）
  → 这些属性在 GPU 合成层上处理，不经过主线程
```

**结论：动画尽量只改 `transform` 和 `opacity`。**

---

## CSS 动画

### 高性能写法

```css
/* ✓ 只用 transform + opacity，GPU 加速 */
.card-enter {
  opacity: 0;
  transform: translateY(20px);
  transition: opacity 0.3s ease, transform 0.3s ease;
}
.card-enter-active {
  opacity: 1;
  transform: translateY(0);
}

/* ✗ 触发 Layout，每帧都要重排 */
.card-bad {
  transition: top 0.3s, height 0.3s;  /* top/height 触发 Layout */
}
```

### will-change（提前提升合成层）

```css
/* 告诉浏览器"这个元素将要动画"，提前创建合成层 */
.animated-element {
  will-change: transform, opacity;
}

/* ⚠️ 注意：不要滥用 will-change
   - 每个合成层占用 GPU 内存
   - 只在确实会动画的元素上使用
   - 动画结束后移除（或用 JS 动态添加/删除）
*/
.animated-element:hover {
  will-change: transform;
}
/* hover 结束后浏览器自动清理 */
```

### CSS Animation vs Transition

```css
/* Transition：从 A 状态到 B 状态 */
.button {
  background: blue;
  transition: background 0.2s ease;
}
.button:hover {
  background: darkblue;
}

/* Animation：关键帧，可循环，更复杂 */
@keyframes pulse {
  0%   { transform: scale(1); opacity: 1; }
  50%  { transform: scale(1.05); opacity: 0.8; }
  100% { transform: scale(1); opacity: 1; }
}

.loading-indicator {
  animation: pulse 1.5s ease-in-out infinite;
  /* 告诉浏览器这个动画不影响其他元素的布局 */
  animation-fill-mode: both;
}
```

---

## JS 动画：requestAnimationFrame

```typescript
// requestAnimationFrame：在浏览器下次绘制前执行，与屏幕刷新率同步
// 比 setTimeout/setInterval 精确，页面隐藏时自动暂停（节省 CPU）

function animateElement(element: HTMLElement, targetX: number, duration: number) {
  const startX = element.getBoundingClientRect().left;
  const startTime = performance.now();

  function tick(currentTime: number) {
    const elapsed = currentTime - startTime;
    const progress = Math.min(elapsed / duration, 1);  // 0 → 1

    // 缓动函数（ease-out）
    const eased = 1 - Math.pow(1 - progress, 3);
    const currentX = startX + (targetX - startX) * eased;

    // 只改 transform，不改 left（避免 Layout）
    element.style.transform = `translateX(${currentX - startX}px)`;

    if (progress < 1) {
      requestAnimationFrame(tick);
    }
  }

  requestAnimationFrame(tick);
}

// 取消动画
function useAnimation() {
  const rafRef = useRef<number>();

  const start = useCallback((callback: FrameRequestCallback) => {
    const animate = (time: number) => {
      callback(time);
      rafRef.current = requestAnimationFrame(animate);
    };
    rafRef.current = requestAnimationFrame(animate);
  }, []);

  const stop = useCallback(() => {
    if (rafRef.current) cancelAnimationFrame(rafRef.current);
  }, []);

  useEffect(() => stop, [stop]);  // 组件卸载时停止

  return { start, stop };
}
```

---

## Web Animations API（WAAPI）

```typescript
// 比 CSS 更灵活，比 JS requestAnimationFrame 更高效
// 在合成线程运行，不阻塞主线程

const element = document.querySelector('.card')!;

// 创建动画
const animation = element.animate(
  [
    { opacity: 0, transform: 'translateY(20px)' },
    { opacity: 1, transform: 'translateY(0)' },
  ],
  {
    duration: 300,
    easing: 'cubic-bezier(0.4, 0, 0.2, 1)',  // Material Design ease
    fill: 'forwards',
  }
);

// 控制动画
animation.pause();
animation.play();
animation.reverse();
animation.cancel();

// 等待完成
await animation.finished;
element.remove();
```

---

## Framer Motion

> React 最流行的动画库，声明式 API，性能优秀（底层用 WAAPI + CSS variables）。

```typescript
import { motion, AnimatePresence, useAnimation, useInView } from 'framer-motion';

// 基础动画
function Card() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -20 }}
      transition={{ duration: 0.3, ease: 'easeOut' }}
      whileHover={{ scale: 1.02 }}     // hover 时缩放
      whileTap={{ scale: 0.98 }}       // 点击时缩放
    >
      Card Content
    </motion.div>
  );
}

// AnimatePresence：挂载/卸载动画
function Modal({ isOpen, onClose }: { isOpen: boolean; onClose: () => void }) {
  return (
    <AnimatePresence>
      {isOpen && (
        <motion.div
          className="modal-overlay"
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
          onClick={onClose}
        >
          <motion.div
            className="modal-content"
            initial={{ scale: 0.9, opacity: 0 }}
            animate={{ scale: 1, opacity: 1 }}
            exit={{ scale: 0.9, opacity: 0 }}
            transition={{ type: 'spring', stiffness: 300, damping: 30 }}
            onClick={e => e.stopPropagation()}
          >
            Modal Content
          </motion.div>
        </motion.div>
      )}
    </AnimatePresence>
  );
}

// 滚动触发动画
function AnimatedSection({ children }: { children: React.ReactNode }) {
  const ref = useRef(null);
  const isInView = useInView(ref, { once: true, margin: '-100px' });

  return (
    <motion.div
      ref={ref}
      initial={{ opacity: 0, x: -50 }}
      animate={isInView ? { opacity: 1, x: 0 } : {}}
      transition={{ duration: 0.5 }}
    >
      {children}
    </motion.div>
  );
}

// 列表错开动画（stagger）
const containerVariants = {
  hidden: {},
  visible: {
    transition: {
      staggerChildren: 0.1,   // 每个子元素延迟 0.1s
      delayChildren: 0.2,
    },
  },
};

const itemVariants = {
  hidden: { opacity: 0, y: 20 },
  visible: { opacity: 1, y: 0 },
};

function ProductList({ products }: { products: Product[] }) {
  return (
    <motion.ul variants={containerVariants} initial="hidden" animate="visible">
      {products.map(p => (
        <motion.li key={p.id} variants={itemVariants}>
          <ProductCard product={p} />
        </motion.li>
      ))}
    </motion.ul>
  );
}

// layout 动画（自动处理元素位置变化）
function SortableList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => (
        <motion.li key={item.id} layout>  {/* layout prop 自动动画位置变化 */}
          {item.name}
        </motion.li>
      ))}
    </ul>
  );
}
```

---

## 性能检测与优化

### 检测 Layout Thrashing（布局抖动）

```typescript
// ✗ 读写交替触发多次 Layout（Layout Thrashing）
function badAnimation(elements: HTMLElement[]) {
  elements.forEach(el => {
    const height = el.offsetHeight;     // 读（触发 Layout）
    el.style.height = height + 10 + 'px';  // 写（使 Layout 失效）
    // 下一次循环再读，触发强制 Layout
  });
}

// ✓ 批量读，批量写
function goodAnimation(elements: HTMLElement[]) {
  // 批量读
  const heights = elements.map(el => el.offsetHeight);

  // 批量写（统一修改，只触发一次 Layout）
  elements.forEach((el, i) => {
    el.style.height = heights[i] + 10 + 'px';
  });
}

// 或用 FastDOM 库自动批处理
import fastdom from 'fastdom';

fastdom.measure(() => {
  const height = element.offsetHeight;
  fastdom.mutate(() => {
    element.style.height = height + 10 + 'px';
  });
});
```

### Chrome DevTools 性能分析

```
Performance 面板关键指标：

Long Tasks（红色块）：
  > 50ms 的任务，会阻塞主线程，导致动画卡顿

Rendering（绿色）：
  Layout（紫色）：触发重排次数
  Paint（绿色）：触发重绘面积
  Composite（小绿块）：合成（理想情况下只有这个）

Layer 面板：
  查看哪些元素提升为合成层（黄色边框）
  层数过多会占用 GPU 内存
```

### 减少动画帧丢失

```typescript
// 使用 will-change 动态管理
function useWillChange(ref: React.RefObject<HTMLElement>) {
  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const addWillChange = () => { el.style.willChange = 'transform'; };
    const removeWillChange = () => { el.style.willChange = 'auto'; };

    el.addEventListener('mouseenter', addWillChange);
    el.addEventListener('mouseleave', removeWillChange);
    el.addEventListener('animationstart', addWillChange);
    el.addEventListener('animationend', removeWillChange);

    return () => {
      el.removeEventListener('mouseenter', addWillChange);
      el.removeEventListener('mouseleave', removeWillChange);
    };
  }, [ref]);
}
```

---

## 减少动画对用户的干扰

```css
/* 尊重用户的"减少动画"系统设置（prefers-reduced-motion）
   无障碍和可访问性的重要一环 */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

```typescript
// React + Framer Motion 中检测
import { useReducedMotion } from 'framer-motion';

function AnimatedCard() {
  const shouldReduceMotion = useReducedMotion();

  return (
    <motion.div
      animate={{ opacity: 1, y: 0 }}
      initial={{ opacity: 0, y: shouldReduceMotion ? 0 : 20 }}  // 减少动画时跳过位移
      transition={{ duration: shouldReduceMotion ? 0 : 0.3 }}
    >
      Content
    </motion.div>
  );
}
```

---

## 面试常见追问

**Q: 为什么 `transform` 比 `top/left` 性能好？**
A: `top/left` 属于几何属性，修改后浏览器必须重新计算 Layout（影响文档流中的所有相邻元素），再 Paint，再 Composite。`transform` 是在合成阶段处理的，该元素被提升到独立的合成层（Compositor Layer），变换在 GPU 上完成，完全绕过 Layout 和 Paint，不影响其他元素。

**Q: `will-change: transform` 为什么不能滥用？**
A: 每个合成层都需要在 GPU 内存中存储该层的位图（Texture）。移动设备 GPU 内存有限（通常 256MB-1GB），层数过多会导致内存压力，触发层合并（Layer Squashing），反而造成掉帧。原则：只在即将动画的元素上短暂添加，动画结束后移除。

**Q: CSS 动画和 JS 动画各自的优缺点？**
A: CSS 动画：声明式、简单、浏览器可优化到合成线程（`transform/opacity`）；但难以精确控制时序、JS 无法干预中间帧、缓动函数受限。JS 动画（rAF/WAAPI）：完全可编程、可中途暂停/反向/读取状态；缺点是在主线程运行（rAF），主线程繁忙时掉帧。Framer Motion 结合了两者：声明式 API，底层用 WAAPI 在合成线程运行。

**Q: 什么是 FLIP 动画技术？**
A: FLIP（First, Last, Invert, Play）是一种让"昂贵动画"变流畅的技巧：①记录元素初始位置（First）；②执行 DOM 变化到最终状态（Last）；③计算位移差，用 `transform` 把元素"反向"拉回初始位置（Invert）；④动画到 `transform: none`（Play）。整个动画只用 `transform`，成本极低。Framer Motion 的 `layout` prop 自动实现了 FLIP。
