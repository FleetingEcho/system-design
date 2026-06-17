# Node.js 性能与扩展

> Worker Threads、Cluster、内存泄漏排查、Event Loop 监控。
> 理解 Node.js 的并发模型，才能做出正确的扩展决策。

---

## 并发模型决策树

```
收到性能问题 → 先分析瓶颈类型

I/O 密集型（数据库、网络、文件）
  → Node.js 默认就能处理（Event Loop 异步 I/O）
  → 如果并发连接太多：横向扩展（Cluster / 多 Pod）

CPU 密集型（加密、图像处理、JSON 解析大数据、机器学习推理）
  → 单线程 Node.js 会阻塞 Event Loop，导致所有请求超时
  → Worker Threads（多线程共享内存）或单独微服务

内存不足
  → 先排查内存泄漏
  → 再考虑增加内存或分片

高并发请求（QPS）
  → Cluster（多进程，利用多核）
  → 或 K8s 水平扩展（多 Pod）
```

---

## Worker Threads（CPU 密集型任务）

```typescript
// worker.ts —— 在独立线程中运行的代码
import { workerData, parentPort } from 'worker_threads';

function heavyComputation(data: number[]): number {
  // 模拟 CPU 密集操作（如图像处理、加密）
  return data.reduce((acc, n) => acc + Math.sqrt(n), 0);
}

const result = heavyComputation(workerData.numbers);
parentPort?.postMessage(result);

// main.ts —— 主线程
import { Worker } from 'worker_threads';
import path from 'path';

function runWorker(numbers: number[]): Promise<number> {
  return new Promise((resolve, reject) => {
    const worker = new Worker(path.join(__dirname, 'worker.js'), {
      workerData: { numbers },
    });
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

// Worker Pool（避免频繁创建/销毁 Worker）
import { StaticPool } from 'node-worker-threads-pool';

const pool = new StaticPool({
  size: 4,              // Worker 数量（通常 = CPU 核数 - 1）
  task: './worker.js',
});

app.post('/compute', async (req, res) => {
  // 不阻塞 Event Loop，任务在 Worker 线程中执行
  const result = await pool.exec(req.body.data);
  res.json({ result });
});

// 共享内存（SharedArrayBuffer）——零拷贝数据传递
const sharedBuffer = new SharedArrayBuffer(1024 * 1024);  // 1MB 共享内存
const sharedArray = new Float64Array(sharedBuffer);

// Worker 和主线程共享同一块内存，不需要序列化/反序列化
const worker = new Worker('./worker.js', {
  workerData: { sharedBuffer },
});
```

---

## Cluster（多核利用）

```typescript
// cluster.ts —— 主进程管理 Worker 进程
import cluster from 'cluster';
import os from 'os';
import { logger } from './lib/logger';

const NUM_WORKERS = os.cpus().length;  // 等于 CPU 核数

if (cluster.isPrimary) {
  logger.info({ workers: NUM_WORKERS }, 'Starting cluster');

  // 创建 Worker 进程（每个进程独立的 Node.js 实例）
  for (let i = 0; i < NUM_WORKERS; i++) {
    cluster.fork();
  }

  // Worker 异常退出时自动重启
  cluster.on('exit', (worker, code, signal) => {
    logger.warn({ pid: worker.process.pid, code, signal }, 'Worker died, restarting');
    cluster.fork();
  });
} else {
  // Worker 进程：各自独立运行 Express 服务
  // 多个 Worker 共享同一个 TCP 端口（OS 负责负载均衡）
  require('./server');
  logger.info({ pid: process.pid }, 'Worker started');
}

// 注意：Cluster 进程间不共享内存
// 共享状态（会话、缓存）必须放在外部（Redis）
// PM2 cluster mode 本质就是这个机制的封装
```

---

## 内存泄漏排查

```typescript
// 常见内存泄漏模式

// ❌ 1. 全局变量意外累积
const cache = {};  // 全局 Map 不断增长，从不清理
app.get('/data/:id', (req, res) => {
  cache[req.params.id] = fetchData(req.params.id);  // 永远不删除
});

// ✓ 修复：使用有容量限制的 LRU Cache
import { LRUCache } from 'lru-cache';
const cache = new LRUCache({ max: 1000, ttl: 1000 * 60 * 5 });

// ❌ 2. 事件监听器泄漏（经典）
function setupFeature(emitter: EventEmitter) {
  // 每次调用都添加新 listener，但从不移除
  emitter.on('data', handleData);
}

// ✓ 修复：返回清理函数
function setupFeature(emitter: EventEmitter) {
  const handler = handleData.bind(null);
  emitter.on('data', handler);
  return () => emitter.off('data', handler);
}

// ❌ 3. 闭包引用大对象
function processLargeData(data: Buffer) {
  const result = computeSomething(data);
  return () => {
    console.log(result);
    // 闭包持有 data 引用，data 无法被 GC
  };
}

// ✓ 修复：只保留需要的部分
function processLargeData(data: Buffer) {
  const result = computeSomething(data);
  // data 在函数返回后可以被 GC（result 不引用它）
  return () => console.log(result);
}

// ❌ 4. Promise 泄漏（未处理的 rejection 挂起）
async function badPattern() {
  const promise = longRunningOperation();  // 从不 await，但 promise 一直在内存中
  return 'done';
}
```

```typescript
// 内存泄漏检测工具

// 1. 监控内存使用趋势（生产）
setInterval(() => {
  const mem = process.memoryUsage();
  logger.info({
    heapUsed: Math.round(mem.heapUsed / 1024 / 1024) + 'MB',
    heapTotal: Math.round(mem.heapTotal / 1024 / 1024) + 'MB',
    rss: Math.round(mem.rss / 1024 / 1024) + 'MB',
    external: Math.round(mem.external / 1024 / 1024) + 'MB',
  }, 'Memory usage');
}, 30_000);

// 2. 堆快照对比（开发）
// node --inspect server.js
// Chrome DevTools → Memory → Heap Snapshot
// 取两个快照，比较新增对象

// 3. clinic.js（开源诊断工具）
// clinic doctor -- node server.js   → 诊断 Event Loop、CPU、内存
// clinic flame -- node server.js    → 火焰图
// clinic bubbleprof -- node server.js → async 操作分析
```

---

## Event Loop 监控

```typescript
// 检测 Event Loop 延迟（卡顿）
import { monitorEventLoopDelay } from 'perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 10 });  // 每 10ms 采样
histogram.enable();

setInterval(() => {
  const lag = histogram.mean / 1e6;  // 纳秒 → 毫秒
  if (lag > 100) {
    logger.warn({ lagMs: lag.toFixed(2) }, 'Event Loop lag detected');
  }
  // Prometheus 指标
  eventLoopLagGauge.set(lag);
}, 5000);

// 简单版：setInterval 漂移检测
let lastCheck = Date.now();
setInterval(() => {
  const now = Date.now();
  const lag = now - lastCheck - 1000;  // 期望 1000ms，实际差值
  if (lag > 50) logger.warn({ lagMs: lag }, 'Possible event loop blocking');
  lastCheck = now;
}, 1000);
```

---

## 性能分析（Profiling）

```bash
# 1. 内置 CPU Profiler
node --prof server.js             # 运行并生成 isolate-*.log
node --prof-process isolate-*.log # 分析，输出热点函数

# 2. V8 Inspector（推荐）
node --inspect server.js
# Chrome → chrome://inspect → 打开 DevTools
# Profiler tab → Start Profiling → 施压 → Stop
# 查看火焰图，找 CPU 热点

# 3. 0x（更好的火焰图工具）
npm install -g 0x
0x -- node server.js
# 自动生成 HTML 火焰图
```

```typescript
// 生产环境性能追踪（不影响性能的低开销方式）
import { performance, PerformanceObserver } from 'perf_hooks';

// 测量关键操作耗时
export async function measureAsync<T>(
  name: string,
  fn: () => Promise<T>
): Promise<T> {
  const start = performance.now();
  try {
    return await fn();
  } finally {
    const duration = performance.now() - start;
    if (duration > 100) {  // 只记录慢操作
      logger.warn({ operation: name, durationMs: duration.toFixed(2) }, 'Slow operation');
    }
  }
}

// 使用
const user = await measureAsync('db.findUser', () =>
  prisma.user.findUnique({ where: { id } })
);
```

---

## Worker Threads vs Cluster vs 微服务

| | Worker Threads | Cluster | 独立微服务 |
|--|--|--|--|
| **隔离性** | 共享内存 | 独立进程 | 独立部署 |
| **通信** | `postMessage`（零拷贝 SharedArrayBuffer） | IPC / 外部存储 | HTTP/gRPC/消息队列 |
| **适合** | CPU 密集型（同进程内卸载） | 利用多核、单机横向扩展 | 不同语言、独立扩展、故障隔离 |
| **开销** | 低（线程启动快） | 中（进程启动慢） | 高（网络开销、运维复杂） |
| **共享状态** | 可以（SharedArrayBuffer） | 不可以（需 Redis） | 不可以（需外部存储） |

**决策原则**：
- 同进程内 CPU 密集 → Worker Threads
- 利用服务器多核 → Cluster / PM2 / K8s replicas
- 不同团队、不同语言、独立扩展需求 → 微服务

---

## 面试追问

**Q: Node.js 的 Worker Threads 和传统多线程有什么区别？**
A: Node.js Worker Threads 默认不共享内存（每个 Worker 有独立的 V8 堆），通过 `postMessage` 传递数据（需要序列化/拷贝）。可以通过 `SharedArrayBuffer` 共享内存，但需要手动同步（`Atomics`）。传统多线程（Java/C++）默认共享内存，需要锁来保护。Node.js 的设计避免了竞态条件，但也限制了线程间通信效率。

**Q: 如何判断性能瓶颈是 CPU 还是 I/O？**
A: 看 Event Loop 延迟。I/O 瓶颈时 Event Loop 本身是空闲的（等待 I/O 完成），延迟低；CPU 瓶颈时 Event Loop 被 JS 执行占满，延迟高（其他请求在队列中等待）。工具：`clinic doctor` 会自动区分，或者监控 `eventLoopDelay` 指标（`perf_hooks.monitorEventLoopDelay`）。

**Q: PM2 cluster mode 和 Docker 多副本有什么区别？**
A: PM2 cluster mode 在单机上用 `cluster` 模块启动多进程，共享同一个 IP/端口，OS 负载均衡；Docker 多副本是多台机器（或同机不同容器），由 Kubernetes/nginx 负载均衡。PM2 适合传统部署（VPS/裸机），Docker 多副本是云原生标准方案。现代 K8s 部署通常直接用 K8s replicas，不需要 PM2（PM2 的进程管理功能被容器运行时接管）。
