# OOD：任务队列（Job Queue）

> 设计一个支持优先级、重试、延迟执行的任务队列（类似 BullMQ）。
> 面试考察：队列抽象、Worker 模型、失败重试策略、并发控制。

---

## 需求分析

```
功能需求：
  1. 生产者：将任务加入队列（支持延迟执行）
  2. 消费者：Worker 从队列取任务并执行
  3. 优先级：高优先级任务先执行
  4. 重试：失败任务自动重试（指数退避）
  5. 并发控制：限制同时执行的 Worker 数量
  6. 任务状态：pending / active / completed / failed

非功能需求：
  - 任务至少执行一次（at-least-once）
  - Worker 崩溃时任务不丢失
  - 支持任务取消
```

---

## 核心接口设计

```typescript
// 任务定义
interface Job<TData = unknown> {
  id: string;
  name: string;       // 任务类型名称（用于匹配 Processor）
  data: TData;
  priority: number;   // 数字越大优先级越高
  attempts: number;   // 已尝试次数
  maxAttempts: number;
  status: JobStatus;
  createdAt: Date;
  scheduledAt: Date;  // 不早于此时间执行（支持延迟）
  startedAt?: Date;
  completedAt?: Date;
  failedAt?: Date;
  lastError?: string;
  result?: unknown;
}

type JobStatus = 'pending' | 'active' | 'completed' | 'failed' | 'cancelled';

// 任务处理器
type Processor<TData = unknown, TResult = unknown> = (
  job: Job<TData>
) => Promise<TResult>;

// 添加任务的选项
interface AddJobOptions {
  priority?: number;   // 默认 0
  delay?: number;      // 延迟执行（毫秒）
  maxAttempts?: number;  // 最大重试次数，默认 3
}

// 队列接口
interface IQueue {
  add<TData>(name: string, data: TData, options?: AddJobOptions): Promise<Job<TData>>;
  process<TData>(name: string, concurrency: number, processor: Processor<TData>): void;
  getJob(jobId: string): Promise<Job | null>;
  removeJob(jobId: string): Promise<void>;
  pause(): Promise<void>;
  resume(): Promise<void>;
  close(): Promise<void>;
}
```

---

## 内存实现（面试用，生产用 Redis）

```typescript
// src/queue/job-queue.ts
import { EventEmitter } from 'events';
import { randomUUID } from 'crypto';

class JobQueue extends EventEmitter implements IQueue {
  private jobs = new Map<string, Job>();
  private processors = new Map<string, { concurrency: number; fn: Processor }>();
  private activeWorkers = new Map<string, number>();  // name → 当前 active workers 数
  private isPaused = false;
  private pollingInterval: NodeJS.Timeout | null = null;

  constructor(private options: { pollIntervalMs?: number } = {}) {
    super();
    this.startPolling();
  }

  async add<TData>(name: string, data: TData, options: AddJobOptions = {}): Promise<Job<TData>> {
    const job: Job<TData> = {
      id: randomUUID(),
      name,
      data,
      priority: options.priority ?? 0,
      attempts: 0,
      maxAttempts: options.maxAttempts ?? 3,
      status: 'pending',
      createdAt: new Date(),
      scheduledAt: new Date(Date.now() + (options.delay ?? 0)),
    };

    this.jobs.set(job.id, job as Job);
    this.emit('job:added', job);
    return job;
  }

  process<TData>(name: string, concurrency: number, processor: Processor<TData>): void {
    this.processors.set(name, { concurrency, fn: processor as Processor });
    this.activeWorkers.set(name, 0);
  }

  private startPolling() {
    const pollMs = this.options.pollIntervalMs ?? 500;

    this.pollingInterval = setInterval(async () => {
      if (this.isPaused) return;
      await this.processNextJobs();
    }, pollMs);

    this.pollingInterval.unref();  // 不阻止进程退出
  }

  private async processNextJobs() {
    const now = new Date();

    for (const [name, { concurrency, fn }] of this.processors) {
      const activeCount = this.activeWorkers.get(name) ?? 0;
      const available = concurrency - activeCount;
      if (available <= 0) continue;

      // 取最高优先级的 pending 任务（已到执行时间）
      const pendingJobs = [...this.jobs.values()]
        .filter(j =>
          j.name === name &&
          j.status === 'pending' &&
          j.scheduledAt <= now
        )
        .sort((a, b) => b.priority - a.priority || a.createdAt.getTime() - b.createdAt.getTime());

      const toProcess = pendingJobs.slice(0, available);
      for (const job of toProcess) {
        this.executeJob(job, fn);
      }
    }
  }

  private async executeJob(job: Job, processor: Processor) {
    const name = job.name;

    // 标记为 active
    job.status = 'active';
    job.startedAt = new Date();
    job.attempts += 1;
    this.activeWorkers.set(name, (this.activeWorkers.get(name) ?? 0) + 1);
    this.emit('job:active', job);

    try {
      const result = await processor(job);

      job.status = 'completed';
      job.completedAt = new Date();
      job.result = result;
      this.emit('job:completed', job);
    } catch (err: any) {
      job.lastError = err.message;

      if (job.attempts >= job.maxAttempts) {
        // 超过重试次数 → 彻底失败
        job.status = 'failed';
        job.failedAt = new Date();
        this.emit('job:failed', job, err);
      } else {
        // 重试：指数退避
        const backoffMs = Math.pow(2, job.attempts) * 1000;  // 2s, 4s, 8s...
        job.status = 'pending';
        job.scheduledAt = new Date(Date.now() + backoffMs);
        this.emit('job:retry', job, err);
      }
    } finally {
      this.activeWorkers.set(name, (this.activeWorkers.get(name) ?? 1) - 1);
    }
  }

  async getJob(jobId: string): Promise<Job | null> {
    return this.jobs.get(jobId) ?? null;
  }

  async removeJob(jobId: string): Promise<void> {
    const job = this.jobs.get(jobId);
    if (!job) return;
    if (job.status === 'active') throw new Error('Cannot remove active job');
    this.jobs.delete(jobId);
  }

  async pause(): Promise<void> {
    this.isPaused = true;
  }

  async resume(): Promise<void> {
    this.isPaused = false;
  }

  async close(): Promise<void> {
    if (this.pollingInterval) {
      clearInterval(this.pollingInterval);
      this.pollingInterval = null;
    }
  }
}
```

---

## 使用示例

```typescript
// 创建队列
const queue = new JobQueue({ pollIntervalMs: 100 });

// 注册处理器
queue.process<{ email: string; template: string }>(
  'send-email',
  5,  // 最多 5 个并发
  async (job) => {
    console.log(`Sending email to ${job.data.email} (attempt ${job.attempts})`);
    await emailService.send(job.data.email, job.data.template);
    return { sentAt: new Date() };
  }
);

queue.process<{ reportId: string }>(
  'generate-report',
  2,  // 生成报告是 CPU 密集，限制 2 个并发
  async (job) => {
    return reportService.generate(job.data.reportId);
  }
);

// 事件监听
queue.on('job:completed', (job) => {
  logger.info({ jobId: job.id, name: job.name }, 'Job completed');
});
queue.on('job:failed', (job, err) => {
  logger.error({ jobId: job.id, error: err.message }, 'Job permanently failed');
  alerting.notify(`Job ${job.name} failed after ${job.maxAttempts} attempts`);
});

// 生产者：添加任务
await queue.add('send-email', { email: 'user@example.com', template: 'welcome' });

// 高优先级任务
await queue.add('send-email', { email: 'vip@example.com', template: 'premium' }, {
  priority: 10,
});

// 延迟任务（5 分钟后执行）
await queue.add('send-email', { email: 'user@example.com', template: 'reminder' }, {
  delay: 5 * 60 * 1000,
});
```

---

## 生产级：BullMQ（Redis 后端）

```typescript
// BullMQ 是生产级 Job Queue，后端用 Redis
// 核心概念与上面的实现一致，API 非常相似

import { Queue, Worker, QueueEvents } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis({ host: 'localhost', port: 6379 });

// 生产者
const emailQueue = new Queue('email', { connection });

await emailQueue.add('send-welcome', { email: 'user@example.com' }, {
  priority: 1,
  delay: 5000,
  attempts: 3,
  backoff: { type: 'exponential', delay: 2000 },
  removeOnComplete: { count: 1000 },   // 完成后保留最近 1000 条记录
  removeOnFail: { count: 5000 },
});

// 消费者（可以在独立进程中运行）
const emailWorker = new Worker(
  'email',
  async (job) => {
    await emailService.send(job.data.email, job.name);
    return { sentAt: new Date() };
  },
  {
    connection,
    concurrency: 5,
    limiter: {
      max: 100,    // 每秒最多处理 100 个任务（速率限制）
      duration: 1000,
    },
  }
);

// 事件
const queueEvents = new QueueEvents('email', { connection });
queueEvents.on('completed', ({ jobId, returnvalue }) => {
  logger.info({ jobId }, 'Email sent');
});
queueEvents.on('failed', ({ jobId, failedReason }) => {
  logger.error({ jobId, reason: failedReason }, 'Email failed');
});
```

---

## 面试追问

**Q: 如何保证任务不丢失（Worker 崩溃）？**
A: 关键设计：任务取出时不立即删除，而是加锁（设置 "active" 状态 + 锁过期时间）。Worker 崩溃后，锁过期，任务回到 "pending" 队列重新被消费。BullMQ 用 Redis 的原子性操作（Lua 脚本）实现这个模式，锁过期时间通常是 30 秒，Worker 正常处理时定期续期（Heartbeat）。代价是 at-least-once 语义（任务可能执行多次），Processor 应该是幂等的。

**Q: 如何实现任务优先级？**
A: 内存实现用排序数组（每次取最高优先级）。生产实现（BullMQ/Redis）用 Sorted Set：score 是优先级+时间戳，`ZPOPMIN` 取最低分（最高优先级）任务。BullMQ 的优先级是 1（最高）到 MAX_INT（最低），内部用 Redis ZADD score = priority * MAX_TIMESTAMP + timestamp 保证同优先级按时间排序。

**Q: 分布式场景下如何避免任务被多个 Worker 重复执行？**
A: Redis 的 SET NX（Set if Not Exists）实现分布式锁：Worker 取任务时执行 `SET job:{id}:lock workerId PX 30000 NX`，成功则获得锁，失败说明已被其他 Worker 锁定。Worker 每 15 秒续期（`PEXPIRE job:{id}:lock 30000`），任务完成后释放锁。这正是 BullMQ 的内部实现原理。
