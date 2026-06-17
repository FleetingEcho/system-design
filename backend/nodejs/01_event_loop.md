# Node.js Event Loop 深度

> 面试高频：执行顺序、process.nextTick vs Promise、I/O 回调时机。
> 理解这些是解释"Node.js 为什么适合 I/O 密集型"的底层依据。

---

## 架构全景

```
┌─────────────────────────────────────────────────────────┐
│                   Node.js 进程                           │
│                                                         │
│  V8 引擎                  libuv                         │
│  ├── JavaScript 执行       ├── Event Loop               │
│  ├── 调用栈（Call Stack）  ├── 线程池（4 个默认线程）    │
│  └── 垃圾回收              │   fs / crypto / dns         │
│                            └── OS 异步 I/O（epoll/kqueue）│
│                                                         │
│  微任务队列（V8 管）                                     │
│  ├── process.nextTick 队列   ← 优先于 Promise            │
│  └── Promise microtask 队列                              │
└─────────────────────────────────────────────────────────┘
```

---

## Event Loop 六个阶段

```
    ┌──────────────────────────────────────────┐
    │                                          │
┌───▼───┐   ┌──────────┐   ┌─────────────┐   │
│timers │──▶│ pending  │──▶│idle/prepare │   │
│       │   │callbacks │   │             │   │
└───────┘   └──────────┘   └──────┬──────┘   │
                                  │           │
┌───────┐   ┌──────────┐   ┌──────▼──────┐   │
│ close │◀──│  check   │◀──│    poll     │   │
│callbacks  │(setImmed)│   │(等待 I/O)   │   │
└───────┘   └──────────┘   └─────────────┘   │
    │                                         │
    └─────────────────────────────────────────┘

每个阶段切换时，先清空：
  1. process.nextTick 队列（全部）
  2. Promise microtask 队列（全部）
然后才进入下一个阶段
```

### 各阶段说明

```
timers         执行 setTimeout / setInterval 到期的回调
               注意：时间只是"最早"执行时间，实际取决于 poll 阶段耗时

pending        上一轮循环中推迟的 I/O 错误回调（TCP ECONNREFUSED 等）

idle/prepare   libuv 内部使用，忽略

poll           核心阶段：
               ① 执行 I/O 回调（fs.readFile 完成、网络请求等）
               ② 如果队列空 + 没有 setImmediate + timer 未到期：
                  在此阻塞等待新的 I/O 事件
               ③ 如果有 setImmediate：立即进入 check 阶段

check          执行 setImmediate 回调

close          执行 close 事件回调（socket.destroy()、server.close() 等）
```

---

## 执行顺序：微任务 vs 宏任务

```typescript
// 结论先行：
// process.nextTick > Promise.then > setImmediate > setTimeout(0)

console.log('1: sync');

setTimeout(() => console.log('5: setTimeout'), 0);

setImmediate(() => console.log('6: setImmediate'));

Promise.resolve().then(() => console.log('3: Promise'));

process.nextTick(() => console.log('2: nextTick'));

queueMicrotask(() => console.log('4: queueMicrotask'));

// 输出：
// 1: sync
// 2: nextTick        ← 所有 nextTick 队列先清空
// 3: Promise         ← 然后所有 Promise 微任务
// 4: queueMicrotask  ← queueMicrotask 和 Promise.then 同队列
// 5: setTimeout      ← timers 阶段
// 6: setImmediate    ← check 阶段（此处顺序不确定，看 I/O 情况）
```

### 嵌套 nextTick（重要陷阱）

```typescript
// nextTick 中再调用 nextTick，会在同一个"清空轮次"中执行
// 可能导致 I/O 饥饿（I/O 永远等不到执行机会）

process.nextTick(() => {
  console.log('nextTick 1');
  process.nextTick(() => {
    console.log('nextTick 2');  // 在 Event Loop 进入下一阶段前执行
  });
});

// 对比：Promise 链不会有这个问题，因为每次 then 进新的微任务
Promise.resolve()
  .then(() => {
    console.log('Promise 1');
    return Promise.resolve();
  })
  .then(() => console.log('Promise 2'));
```

### I/O 回调中的 setImmediate vs setTimeout

```typescript
const fs = require('fs');

fs.readFile('/tmp/test.txt', () => {
  // 在 I/O 回调（poll 阶段）中：
  // setImmediate 一定先于 setTimeout 执行
  // 因为 poll → check(setImmediate) → timers(setTimeout)

  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// 输出：immediate → timeout（确定）

// 但在顶层（非 I/O 回调）中，顺序不确定（取决于启动耗时）：
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// 输出不确定！
```

---

## async/await 与 Event Loop

```typescript
async function fetchData() {
  console.log('A');
  const result = await Promise.resolve('data');  // await 让出控制权
  console.log('C', result);  // 在微任务队列中恢复
}

console.log('start');
fetchData();
console.log('B');  // fetchData() 中的 await 让出后，继续执行同步代码

// 输出：start → A → B → C data
```

```typescript
// await 在 Event Loop 中的实质：
// await expr  ≈  Promise.resolve(expr).then(continuation)
// continuation（await 后的代码）进入 Promise 微任务队列

async function example() {
  await null;  // 即使 await null，也会延迟一个微任务
  // 这里相当于 Promise.resolve(null).then(() => { ...后续代码... })
}
```

---

## libuv 线程池

```
默认 4 个线程（可通过 UV_THREADPOOL_SIZE 调整，最大 1024）

使用线程池的操作：
  ✓ fs（文件系统，除 watch）
  ✓ dns.lookup（不是 dns.resolve）
  ✓ crypto（加密解密）
  ✓ zlib（压缩）
  ✓ http.request 的 DNS 解析部分

不使用线程池（OS 异步）：
  ✓ TCP/UDP 网络 I/O（epoll/kqueue/IOCP）
  ✓ Pipe / Unix socket
  ✓ Child process

常见问题：同时发起大量 fs 操作时，超出线程池会排队
解决：增大 UV_THREADPOOL_SIZE 或改用流式处理
```

```typescript
// 演示线程池饱和的影响
import { createHash } from 'crypto';

// 同时发起 8 个 CPU 密集的 crypto 操作（线程池默认 4）
const start = Date.now();
const promises = Array.from({ length: 8 }, () =>
  new Promise<void>(resolve => {
    // pbkdf2 使用线程池
    require('crypto').pbkdf2('password', 'salt', 100000, 64, 'sha512', () => {
      resolve();
    });
  })
);

Promise.all(promises).then(() => {
  // 前 4 个约 200ms，后 4 个得等前 4 个完成（线程池排队）
  // 总计约 400ms 而不是 200ms
  console.log(`Total: ${Date.now() - start}ms`);
});
```

---

## 实际影响：不要阻塞 Event Loop

```typescript
// 错误：同步 CPU 密集操作阻塞 Event Loop
app.get('/compute', (req, res) => {
  // JSON.parse 大文件、同步加密、复杂排序——阻塞期间所有请求都等待
  const result = heavyComputation();
  res.json(result);
});

// 正确：Worker Threads 处理 CPU 密集任务
import { Worker } from 'worker_threads';

app.get('/compute', (req, res) => {
  const worker = new Worker('./worker.js', { workerData: req.body });
  worker.once('message', result => res.json(result));
  worker.once('error', err => res.status(500).json({ error: err.message }));
});

// 正确：大数组分批处理，用 setImmediate 让出 Event Loop
async function processBatch(items: unknown[]) {
  for (let i = 0; i < items.length; i += 1000) {
    const batch = items.slice(i, i + 1000);
    await processBatchSync(batch);
    // 每批处理后让出，让 I/O 回调有机会执行
    await new Promise(resolve => setImmediate(resolve));
  }
}
```

---

## 面试追问

**Q: process.nextTick 和 Promise.then 都是微任务，区别是什么？**
A: `process.nextTick` 有自己独立的队列，优先级高于 Promise 微任务队列，在每个阶段切换前**先**清空 nextTick 队列，**再**清空 Promise 队列。过度使用 `process.nextTick` 可能导致 I/O 饥饿（I/O 回调迟迟不能执行）。Node.js 官方建议优先用 `queueMicrotask`（与 Promise 同队列）而不是 `process.nextTick`，除非真的需要在 Promise 之前执行。

**Q: 为什么 Node.js 单线程却能处理高并发？**
A: Node.js 的 JS 执行是单线程，但 I/O 操作委托给 libuv（网络 I/O 用 OS 异步接口如 epoll，文件 I/O 用线程池）。等待 I/O 期间 Event Loop 可以处理其他回调，不需要为每个连接开一个线程。适合 I/O 密集型（等待数据库、网络请求），不适合 CPU 密集型（计算会阻塞 Event Loop）。

**Q: setImmediate 和 setTimeout(0) 有什么区别？**
A: `setImmediate` 在 check 阶段执行（紧跟 poll 之后），`setTimeout(0)` 在 timers 阶段执行（Event Loop 开头）。在 I/O 回调中 `setImmediate` 一定先执行；在顶层代码中顺序不确定（取决于进程启动时间是否超过 1ms）。需要"在当前 I/O 事件后立即执行"用 `setImmediate`，不要依赖 `setTimeout(0)` 的顺序。
