# 设计 Mini Redux

> 考察点：单向数据流、发布订阅、中间件链（洋葱模型）、函数式编程。
> 理解 Redux 后，任何状态管理库（Zustand、Jotai）都是变体。

---

## 需求分析

```
核心 API：
  createStore(reducer, preloadedState?, enhancer?)
    → { getState, dispatch, subscribe }

中间件：
  applyMiddleware(...middlewares)  → StoreEnhancer
  内置中间件：logger、thunk

工具函数：
  combineReducers(reducers)   → 合并多个 reducer
  compose(...fns)             → 函数组合（中间件链基础）

设计原则：
  - 单一数据源（Single Source of Truth）
  - State 只读（只能通过 dispatch action 修改）
  - Reducer 是纯函数（相同输入必须相同输出，无副作用）
```

---

## createStore

```typescript
type Reducer<S, A> = (state: S | undefined, action: A) => S;
type Listener = () => void;
type Unsubscribe = () => void;
type Dispatch<A> = (action: A) => A;

interface Store<S, A> {
  getState(): S;
  dispatch: Dispatch<A>;
  subscribe(listener: Listener): Unsubscribe;
}

type StoreEnhancer<S, A> = (
  createStore: (reducer: Reducer<S, A>, preloadedState?: S) => Store<S, A>
) => (reducer: Reducer<S, A>, preloadedState?: S) => Store<S, A>;

function createStore<S, A extends { type: string }>(
  reducer: Reducer<S, A>,
  preloadedState?: S,
  enhancer?: StoreEnhancer<S, A>
): Store<S, A> {
  // 如果有 enhancer（applyMiddleware），委托给它创建 store
  if (enhancer) {
    return enhancer(createStore)(reducer, preloadedState);
  }

  let state: S = preloadedState ?? reducer(undefined, { type: '@@INIT' } as A);
  const listeners = new Set<Listener>();
  let isDispatching = false;

  function getState(): S {
    if (isDispatching) throw new Error('Cannot call getState() during dispatch');
    return state;
  }

  function dispatch(action: A): A {
    if (isDispatching) throw new Error('Reducers may not dispatch actions');

    try {
      isDispatching = true;
      state = reducer(state, action);  // 纯函数调用，产生新 state
    } finally {
      isDispatching = false;
    }

    // 通知所有订阅者
    listeners.forEach(listener => listener());

    return action;
  }

  function subscribe(listener: Listener): Unsubscribe {
    listeners.add(listener);
    return () => listeners.delete(listener);
  }

  return { getState, dispatch, subscribe };
}
```

---

## combineReducers

```typescript
type ReducerMap<S> = {
  [K in keyof S]: Reducer<S[K], { type: string }>;
};

function combineReducers<S extends Record<string, unknown>>(
  reducers: ReducerMap<S>
): Reducer<S, { type: string }> {
  const keys = Object.keys(reducers) as (keyof S)[];

  return function combination(state = {} as S, action) {
    let hasChanged = false;
    const nextState = {} as S;

    keys.forEach(key => {
      const reducer = reducers[key];
      const previousStateForKey = state[key];
      const nextStateForKey = reducer(previousStateForKey, action);

      nextState[key] = nextStateForKey;
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey;
    });

    // 只有真正变化时返回新对象（引用相等性优化）
    return hasChanged ? nextState : state;
  };
}
```

---

## compose（函数组合）

```typescript
// compose(f, g, h)(x) === f(g(h(x)))
function compose<T>(...fns: ((arg: T) => T)[]): (arg: T) => T {
  if (fns.length === 0) return (arg: T) => arg;
  if (fns.length === 1) return fns[0];
  return fns.reduce((a, b) => (arg: T) => a(b(arg)));
}
```

---

## applyMiddleware（中间件系统）

```typescript
// 中间件签名：store => next => action => result
type Middleware<S, A> = (
  store: Pick<Store<S, A>, 'getState' | 'dispatch'>
) => (next: Dispatch<A>) => (action: A | unknown) => unknown;

function applyMiddleware<S, A extends { type: string }>(
  ...middlewares: Middleware<S, A>[]
): StoreEnhancer<S, A> {
  return (createStore) => (reducer, preloadedState) => {
    const store = createStore(reducer, preloadedState);

    let dispatch: Dispatch<A> = () => {
      throw new Error('Dispatching while constructing middleware chain is not allowed.');
    };

    // 给每个中间件暴露 getState 和 dispatch（注意：dispatch 是最终的增强版本）
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action: A) => dispatch(action),  // 闭包引用，指向最终 dispatch
    };

    // 每个中间件拿到 middlewareAPI，返回 (next => action => result) 函数
    const chain = middlewares.map(middleware => middleware(middlewareAPI));

    // 从右往左组合，最右边的中间件最先调用 store.dispatch（真正的 dispatch）
    dispatch = compose(...chain)(store.dispatch) as Dispatch<A>;

    return { ...store, dispatch };
  };
}
```

---

## 内置中间件

```typescript
// Logger 中间件：打印每次 dispatch 前后的 state
const loggerMiddleware: Middleware<unknown, { type: string }> =
  (store) => (next) => (action) => {
    console.group(`action: ${(action as { type: string }).type}`);
    console.log('prev state:', store.getState());
    const result = next(action as { type: string });
    console.log('next state:', store.getState());
    console.groupEnd();
    return result;
  };

// Thunk 中间件：支持 dispatch 函数（异步 action）
type ThunkAction<S, A> = (
  dispatch: Dispatch<A>,
  getState: () => S
) => unknown;

const thunkMiddleware: Middleware<unknown, { type: string }> =
  ({ dispatch, getState }) => (next) => (action) => {
    if (typeof action === 'function') {
      // action 是函数（thunk）→ 调用它，传入 dispatch 和 getState
      return (action as ThunkAction<unknown, { type: string }>)(
        dispatch as Dispatch<{ type: string }>,
        getState
      );
    }
    // action 是普通对象 → 直接传给下一个中间件
    return next(action as { type: string });
  };
```

---

## 完整使用示例

```typescript
// 定义 State 和 Actions
interface CounterState { count: number; }
interface TodoState { items: string[]; }
interface AppState { counter: CounterState; todos: TodoState; }

type CounterAction = { type: 'INCREMENT' } | { type: 'DECREMENT' } | { type: 'RESET' };
type TodoAction = { type: 'ADD_TODO'; payload: string } | { type: 'CLEAR_TODOS' };
type AppAction = CounterAction | TodoAction;

// Reducers
function counterReducer(
  state: CounterState = { count: 0 },
  action: AppAction
): CounterState {
  switch (action.type) {
    case 'INCREMENT': return { count: state.count + 1 };
    case 'DECREMENT': return { count: state.count - 1 };
    case 'RESET':     return { count: 0 };
    default:          return state;
  }
}

function todosReducer(
  state: TodoState = { items: [] },
  action: AppAction
): TodoState {
  switch (action.type) {
    case 'ADD_TODO':   return { items: [...state.items, action.payload] };
    case 'CLEAR_TODOS': return { items: [] };
    default:           return state;
  }
}

// 创建 Store
const rootReducer = combineReducers<AppState>({
  counter: counterReducer as Reducer<CounterState, { type: string }>,
  todos: todosReducer as Reducer<TodoState, { type: string }>,
});

const store = createStore(
  rootReducer,
  undefined,
  applyMiddleware(loggerMiddleware, thunkMiddleware)
);

// 同步 dispatch
store.dispatch({ type: 'INCREMENT' });
console.log(store.getState().counter);  // { count: 1 }

// 异步 dispatch（Thunk）
const fetchTodos = () => async (dispatch: Dispatch<AppAction>) => {
  const todos = await fetch('/api/todos').then(r => r.json());
  todos.forEach((todo: string) => dispatch({ type: 'ADD_TODO', payload: todo }));
};
store.dispatch(fetchTodos() as unknown as AppAction);

// 订阅
const unsubscribe = store.subscribe(() => {
  console.log('State changed:', store.getState());
});

// 取消订阅
unsubscribe();
```

---

## 时间旅行调试

```typescript
// Redux DevTools 的核心：保存每次 action 和 state 快照
class TimeTravel<S, A extends { type: string }> {
  private history: { action: A; state: S }[] = [];
  private currentIndex = -1;
  private store: Store<S, A>;

  constructor(reducer: Reducer<S, A>) {
    this.store = createStore(reducer);
  }

  dispatch(action: A) {
    // 时间旅行后 dispatch 新 action，清除未来历史
    this.history = this.history.slice(0, this.currentIndex + 1);

    const result = this.store.dispatch(action);
    this.history.push({ action, state: this.store.getState() });
    this.currentIndex++;
    return result;
  }

  jumpTo(index: number) {
    // 从初始 state 重放 action 到指定位置
    this.currentIndex = Math.max(0, Math.min(index, this.history.length - 1));
    // 实际 Redux DevTools 通过重放 reducer 实现
    console.log('Jump to state:', this.history[this.currentIndex].state);
  }
}
```

---

## 面试追问

**Q: Redux 的中间件是如何工作的？**
A: 中间件是 `store => next => action => result` 的柯里化函数。`applyMiddleware` 将所有中间件排成链，从左到右依次包裹，形成洋葱模型。每个中间件调用 `next(action)` 将控制权传给下一个中间件，最内层调用真正的 `store.dispatch`。Thunk 中间件在这里拦截函数类型的 action，直接调用而不传给下一层。

**Q: 为什么 Reducer 必须是纯函数？**
A: 纯函数（相同输入→相同输出，无副作用）的好处：①可预测（便于调试和测试）；②支持时间旅行（可以重放 action 序列）；③`combineReducers` 通过引用相等比较（`nextState !== state`）判断是否有变化，如果 reducer 直接改变原 state，这个比较就失效了。

**Q: Redux 和 Zustand 的核心区别？**
A: Redux 强制单向数据流和不可变 state，中间件机制强大但样板代码多；Zustand 直接允许 `set` 修改 state（内部通过 `immer` 保持不可变），API 极简，没有 action/reducer 的概念，适合中小型项目。两者底层都是发布订阅模式。
