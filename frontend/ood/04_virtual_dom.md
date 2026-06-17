# 实现 Virtual DOM + Diff 算法

> 考察点：树结构建模、Diff 算法复杂度优化、最小化 DOM 操作、key 的作用。
> React、Vue 的核心引擎，理解后面试问"React 为什么快"能答出真正原因。

---

## 需求分析

```
核心功能：
  1. createElement(type, props, ...children) → VNode
  2. render(vnode, container)    首次渲染 VNode → 真实 DOM
  3. diff(oldVNode, newVNode)    对比两棵 VNode 树
  4. patch(container, patches)  将差异应用到真实 DOM

关键约束：
  - 同层对比（不跨层移动，O(n) 而非 O(n³)）
  - key 优化列表 Diff（识别移动，减少 DOM 操作）
  - 相同 type 节点复用（不销毁重建）
  - 不同 type 节点直接替换
```

---

## VNode 数据结构

```typescript
type VNodeType = string | null;  // string: div/span, null: 文本节点

interface VNode {
  type: VNodeType;
  props: Record<string, unknown>;
  children: (VNode | string)[];
  key: string | null;
  _el?: Node;  // 对应的真实 DOM 节点（挂载后填充）
}

function createElement(
  type: string,
  props: Record<string, unknown> | null,
  ...children: (VNode | string | null | undefined)[]
): VNode {
  return {
    type,
    props: props ?? {},
    children: children.flat().filter(c => c != null) as (VNode | string)[],
    key: (props?.key as string) ?? null,
  };
}

// 简写（JSX 转换目标）
const h = createElement;

// 示例
const vdom = h('div', { class: 'container' },
  h('h1', null, 'Hello World'),
  h('ul', null,
    h('li', { key: '1' }, 'Item 1'),
    h('li', { key: '2' }, 'Item 2'),
  )
);
```

---

## 首次渲染（render）

```typescript
function render(vnode: VNode | string, container: Element): void {
  const el = createDOMNode(vnode);
  container.appendChild(el);
}

function createDOMNode(vnode: VNode | string): Node {
  // 文本节点
  if (typeof vnode === 'string') {
    return document.createTextNode(vnode);
  }

  const el = document.createElement(vnode.type as string);
  vnode._el = el;

  // 设置属性
  setProps(el, vnode.props);

  // 递归创建子节点
  vnode.children.forEach(child => {
    el.appendChild(createDOMNode(child));
  });

  return el;
}

function setProps(el: Element, props: Record<string, unknown>): void {
  Object.entries(props).forEach(([key, value]) => {
    setProp(el, key, value);
  });
}

function setProp(el: Element, key: string, value: unknown): void {
  if (key === 'key') return;  // key 不作为 DOM 属性

  if (key.startsWith('on') && typeof value === 'function') {
    // 事件监听：onClick → click
    el.addEventListener(key.slice(2).toLowerCase(), value as EventListener);
  } else if (key === 'style' && typeof value === 'object') {
    Object.assign((el as HTMLElement).style, value);
  } else if (key === 'className') {
    el.setAttribute('class', value as string);
  } else if (typeof value === 'boolean') {
    if (value) el.setAttribute(key, '');
    else el.removeAttribute(key);
  } else {
    el.setAttribute(key, String(value));
  }
}
```

---

## Diff 算法

```typescript
// 同层对比核心：old VNode 和 new VNode 对比，产生 patch 操作
type PatchType = 'REPLACE' | 'UPDATE_PROPS' | 'UPDATE_TEXT' | 'REORDER';

function diff(oldVNode: VNode | string, newVNode: VNode | string): void {
  const el = (typeof oldVNode === 'string'
    ? oldVNode
    : oldVNode._el) as Node;

  if (!el) return;

  patch(el, oldVNode, newVNode);
}

function patch(
  parent: Node,
  oldVNode: VNode | string | null,
  newVNode: VNode | string | null,
  refNode?: Node  // 插入参考点
): void {
  // Case 1: 旧节点不存在 → 新增
  if (oldVNode == null) {
    parent.insertBefore(createDOMNode(newVNode!), refNode ?? null);
    return;
  }

  // Case 2: 新节点不存在 → 删除
  if (newVNode == null) {
    parent.removeChild(typeof oldVNode === 'string'
      ? findTextNode(parent, oldVNode)!
      : oldVNode._el!
    );
    return;
  }

  // Case 3: 文本节点
  if (typeof oldVNode === 'string' || typeof newVNode === 'string') {
    const oldEl = typeof oldVNode === 'string'
      ? findTextNode(parent, oldVNode)
      : oldVNode._el;
    if (oldVNode !== newVNode) {
      parent.replaceChild(createDOMNode(newVNode), oldEl!);
    }
    return;
  }

  // Case 4: 类型不同 → 直接替换（不复用）
  if (oldVNode.type !== newVNode.type) {
    parent.replaceChild(createDOMNode(newVNode), oldVNode._el!);
    return;
  }

  // Case 5: 相同类型 → 复用 DOM 节点，更新属性和子节点
  const el = oldVNode._el as Element;
  newVNode._el = el;  // 复用

  // 更新属性
  updateProps(el, oldVNode.props, newVNode.props);

  // 更新子节点（关键：key 优化）
  diffChildren(el, oldVNode.children, newVNode.children);
}

function updateProps(
  el: Element,
  oldProps: Record<string, unknown>,
  newProps: Record<string, unknown>
): void {
  // 新增/更新属性
  Object.entries(newProps).forEach(([key, value]) => {
    if (oldProps[key] !== value) setProp(el, key, value);
  });
  // 删除旧属性
  Object.keys(oldProps).forEach(key => {
    if (!(key in newProps)) el.removeAttribute(key);
  });
}
```

---

## 带 key 的列表 Diff（核心难点）

```typescript
function diffChildren(
  parent: Element,
  oldChildren: (VNode | string)[],
  newChildren: (VNode | string)[]
): void {
  // 建立 key → oldVNode 的映射（快速查找可复用节点）
  const keyedOldMap = new Map<string, { vnode: VNode; index: number }>();
  oldChildren.forEach((child, i) => {
    if (typeof child !== 'string' && child.key != null) {
      keyedOldMap.set(child.key, { vnode: child, index: i });
    }
  });

  // 记录已处理的旧节点索引
  const patched = new Set<number>();
  // 需要移动的节点（key 存在但位置变化）
  let lastStableIndex = 0;

  newChildren.forEach((newChild, newIndex) => {
    if (typeof newChild !== 'string' && newChild.key != null) {
      const oldEntry = keyedOldMap.get(newChild.key);

      if (oldEntry) {
        patched.add(oldEntry.index);
        // 复用节点，递归 patch
        patch(parent, oldEntry.vnode, newChild);

        // 判断是否需要移动（LIS 优化的简化版）
        if (oldEntry.index < lastStableIndex) {
          // 需要移动：将节点插到正确位置
          const nextSibling = parent.childNodes[newIndex + 1] ?? null;
          parent.insertBefore(newChild._el!, nextSibling);
        } else {
          lastStableIndex = oldEntry.index;
        }
      } else {
        // 新节点（key 不存在于旧树）→ 插入
        const refNode = parent.childNodes[newIndex] ?? null;
        parent.insertBefore(createDOMNode(newChild), refNode);
      }
    } else {
      // 无 key：按位置对比
      patch(parent, oldChildren[newIndex] ?? null, newChild);
    }
  });

  // 删除旧树中未复用的节点
  oldChildren.forEach((oldChild, i) => {
    if (!patched.has(i) && typeof oldChild !== 'string' && oldChild.key != null) {
      parent.removeChild(oldChild._el!);
    }
  });
}
```

---

## 完整示例

```typescript
// 初始渲染
const container = document.getElementById('app')!;
let vdom = h('ul', null,
  h('li', { key: 'a' }, 'Apple'),
  h('li', { key: 'b' }, 'Banana'),
  h('li', { key: 'c' }, 'Cherry'),
);
render(vdom, container);

// 更新：删除 b，在头部插入 Date，a 和 c 保留并移动
const newVdom = h('ul', null,
  h('li', { key: 'd' }, 'Date'),    // 新增
  h('li', { key: 'c' }, 'Cherry'),  // 移动（从位置2到位置1）
  h('li', { key: 'a' }, 'Apple'),   // 移动（从位置0到位置2）
  // b 被删除
);

// Diff + Patch（只有 3 次 DOM 操作：1次插入、1次移动、1次删除）
diff(vdom, newVdom);
```

---

## 面试追问

**Q: 为什么 React 的 Diff 是 O(n) 而不是 O(n³)？**
A: 完整的树 Diff 是 O(n³)（枚举所有节点对 + 排列）。React 做了三个假设：①跨层移动节点的情况极少，只做同层对比；②不同 type 的节点直接替换，不尝试复用子树；③key 帮助识别同层的可复用节点。这三个假设将复杂度降到 O(n)。

**Q: key 的作用是什么？为什么不能用 index 作为 key？**
A: key 让 Diff 算法识别"这个新节点对应哪个旧节点"，从而复用 DOM 而不是重建。用 index 作为 key 时，列表重排（如在头部插入一项）会导致所有节点的 key 都变化，无法复用，等于没有 key。key 应该是稳定的唯一 ID（数据库 ID 或稳定哈希）。

**Q: React Fiber 相比简单 VNode 树有什么改进？**
A: 简单 VNode Diff 是同步递归，无法中断。Fiber 将 VNode 树转换为链表（每个节点有 child/sibling/return 指针），可以在每帧的空闲时间分片执行 Diff（`requestIdleCallback`），高优先级更新（用户交互）可以中断低优先级更新（数据加载），实现并发渲染。

**Q: Vue 和 React 的 Diff 有什么区别？**
A: Vue 3 使用"最长递增子序列（LIS）"算法优化带 key 的列表 Diff，找到不需要移动的最大子序列，只移动其他节点，最小化 DOM 移动次数。React 的简化策略是从左往右扫描，找到第一个需要移动的节点后，之后遇到需要移动的节点都移动（不找 LIS）。Vue 3 的 DOM 移动次数更少，但计算更复杂。
