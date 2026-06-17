# 消息队列消费端模式

> BullMQ Worker 生产级实现：幂等性、并发控制、死信队列、优先级、限流、分布式调度。
> 面试场景：设计可靠的异步任务处理系统（邮件发送、视频转码、数据导出）。

---

## 为什么需要消息队列

```
同步 HTTP 调用的问题：
  ✗ 耗时任务阻塞响应（视频转码、大报表导出）
  ✗ 下游服务不可用时请求直接失败（无重试）
  ✗ 突发流量打垮下游（无缓冲）

消息队列解决：
  ✓ 异步解耦：生产者立即返回，消费者后台处理
  ✓ 削峰填谷：队列作为缓冲，消费者按能力消费
  ✓ 可靠重试：失败自动重试，超出次数进死信队列
  ✓ 优先级：VIP 任务优先处理
  ✓ 延迟任务：定时执行（如：注册 5 分钟后发欢迎邮件）
```

---

## BullMQ 架构

```
Producer（API 层）    Queue（Redis）        Consumer（Worker 层）
    │                      │                       │
    │  ─── enqueue ────→   │  ←─── poll ─────────  │
    │                      │                       │
    │                      │  ───── job ─────────→ │
    │                      │                       │ process()
    │                      │                       │ onSuccess → remove
    │                      │                       │ onFail → retry/DLQ
    │                      │
  主队列（active/waiting/delayed/prioritized）
  死信队列（failed，超出 maxAttempts）
```

---

## 类型安全的 Job 定义

```typescript
// src/jobs/types.ts
// 集中定义所有 Job 类型（JobName → JobData 映射）

export interface JobDataMap {
  'email:send': {
    to: string;
    template: 'welcome' | 'reset-password' | 'order-confirm';
    variables: Record<string, string>;
    userId?: string;
  };
  'video:transcode': {
    videoId: string;
    sourceUrl: string;
    targetFormats: ('720p' | '1080p' | '4k')[];
    priority?: 'low' | 'normal' | 'high';
  };
  'report:generate': {
    reportId: string;
    userId: string;
    filters: Record<string, unknown>;
    outputFormat: 'pdf' | 'xlsx' | 'csv';
  };
  'notification:push': {
    userId: string;
    title: string;
    body: string;
    data?: Record<string, string>;
  };
}

export type JobName = keyof JobDataMap;
```

---

## 类型安全的队列客户端（生产者）

```typescript
// src/jobs/queue.ts

import { Queue, JobsOptions } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis(process.env.REDIS_URL!, {
  maxRetriesPerRequest: null,  // BullMQ 要求
  enableReadyCheck: false,
});

// 类型安全的队列 Map
const queues = new Map<JobName, Queue>();

function getQueue<T extends JobName>(name: T): Queue<JobDataMap[T]> {
  if (!queues.has(name)) {
    queues.set(name, new Queue(name, {
      connection,
      defaultJobOptions: {
        removeOnComplete: { count: 1000, age: 24 * 3600 },  // 保留最近 1000 条完成记录 24h
        removeOnFail: false,       // 失败的保留（用于分析）
        attempts: 3,               // 默认最多重试 3 次
        backoff: {
          type: 'exponential',
          delay: 1000,             // 首次重试间隔 1s，之后指数增长
        },
      },
    }));
  }
  return queues.get(name) as Queue<JobDataMap[T]>;
}

// 泛型 enqueue 函数
export async function enqueue<T extends JobName>(
  jobName: T,
  data: JobDataMap[T],
  options?: JobsOptions
) {
  const queue = getQueue(jobName);
  return queue.add(jobName, data, options);
}

// 语义化封装（推荐业务层使用这些函数）
export const jobs = {
  sendEmail: (data: JobDataMap['email:send'], opts?: JobsOptions) =>
    enqueue('email:send', data, opts),

  transcodeVideo: (data: JobDataMap['video:transcode'], opts?: JobsOptions) =>
    enqueue('video:transcode', data, {
      priority: data.priority === 'high' ? 1 : data.priority === 'low' ? 10 : 5,
      ...opts,
    }),

  generateReport: (data: JobDataMap['report:generate'], opts?: JobsOptions) =>
    enqueue('report:generate', data, {
      attempts: 2,    // 报表生成只重试 1 次
      timeout: 120_000,  // 2 分钟超时
      ...opts,
    }),

  // 延迟任务（注册 5 分钟后发欢迎邮件）
  sendDelayedEmail: (data: JobDataMap['email:send'], delayMs: number) =>
    enqueue('email:send', data, { delay: delayMs }),

  // 去重任务（同一 jobId 不重复入队）
  pushNotification: (data: JobDataMap['notification:push']) =>
    enqueue('notification:push', data, {
      jobId: `notification:${data.userId}:${Date.now()}`,  // 唯一 ID 防重复
    }),
};
```

---

## Worker 实现（消费者）

```typescript
// src/workers/email.worker.ts

import { Worker, Job, UnrecoverableError } from 'bullmq';

// 幂等性 key 记录（防止重复处理）
async function isProcessed(jobId: string): Promise<boolean> {
  const key = `job:processed:${jobId}`;
  const result = await redis.set(key, '1', 'EX', 86400, 'NX');  // 24h TTL，NX = 不存在才设置
  return result === null;  // null = key 已存在 = 已处理
}

export function startEmailWorker() {
  const worker = new Worker<JobDataMap['email:send']>(
    'email:send',
    async (job: Job<JobDataMap['email:send']>) => {
      // 幂等性检查（重试时防止重复发送）
      if (await isProcessed(job.id!)) {
        console.log(`Job ${job.id} already processed, skipping`);
        return;  // 直接返回成功，不重复处理
      }

      const { to, template, variables, userId } = job.data;

      // 更新进度（UI 可以展示进度条）
      await job.updateProgress(10);

      const content = await renderEmailTemplate(template, variables);

      await job.updateProgress(50);

      await emailProvider.send({ to, subject: getSubject(template), html: content });

      await job.updateProgress(100);

      // 记录到用户通知历史
      if (userId) {
        await prisma.notification.create({ data: { userId, type: 'email', template } });
      }
    },
    {
      connection,
      concurrency: 10,          // 同时处理 10 个邮件（IO 密集，可以高并发）
      limiter: {
        max: 100,               // 每分钟最多处理 100 个（保护邮件服务商速率限制）
        duration: 60_000,
      },
      stalledInterval: 30_000,  // 30s 没有心跳视为 stalled job，重新入队
    }
  );

  // 事件监听
  worker.on('completed', (job) => {
    console.log(`Email job ${job.id} completed`);
  });

  worker.on('failed', (job, err) => {
    console.error(`Email job ${job?.id} failed (attempt ${job?.attemptsMade}):`, err.message);

    // 最终失败（超出 maxAttempts）→ 告警
    if (job?.attemptsMade === job?.opts.attempts) {
      alertingService.sendAlert({
        severity: 'high',
        message: `Email delivery permanently failed`,
        metadata: { jobId: job?.id, to: job?.data.to },
      });
    }
  });

  worker.on('stalled', (jobId) => {
    console.warn(`Email job ${jobId} stalled`);
  });

  return worker;
}
```

---

## 视频转码 Worker（CPU 密集，低并发 + 进度上报）

```typescript
// src/workers/video.worker.ts

export function startVideoWorker() {
  return new Worker<JobDataMap['video:transcode']>(
    'video:transcode',
    async (job) => {
      const { videoId, sourceUrl, targetFormats } = job.data;

      // CPU 密集任务：低并发（避免服务器过载）
      // 进度：按格式逐个处理，上报进度

      const total = targetFormats.length;
      let completed = 0;

      for (const format of targetFormats) {
        // 不可恢复错误：源文件不存在，不应重试
        const exists = await checkSourceExists(sourceUrl);
        if (!exists) {
          throw new UnrecoverableError(`Source video not found: ${sourceUrl}`);
          // UnrecoverableError 直接进死信队列，不重试
        }

        await transcodeVideo({ sourceUrl, format, outputKey: `${videoId}/${format}.mp4` });

        completed++;
        await job.updateProgress(Math.round((completed / total) * 100));
      }

      // 更新 DB 状态
      await prisma.video.update({
        where: { id: videoId },
        data: { status: 'ready', formats: targetFormats },
      });

      // 触发通知
      await jobs.sendEmail({
        to: await getUserEmail(videoId),
        template: 'welcome',
        variables: { videoId },
      });
    },
    {
      connection,
      concurrency: 2,    // CPU 密集：同时只处理 2 个
      lockDuration: 300_000,  // 5 分钟锁（转码耗时长）
    }
  );
}
```

---

## 死信队列（DLQ）处理

```typescript
// src/workers/dlq-processor.ts
// 监控失败队列，按策略处理：人工审查 / 降级处理 / 告警

import { QueueEvents, Queue } from 'bullmq';

export async function setupDLQMonitoring() {
  // 监听所有队列的失败事件
  const queueNames: JobName[] = ['email:send', 'video:transcode', 'report:generate'];

  for (const queueName of queueNames) {
    const queueEvents = new QueueEvents(queueName, { connection });

    queueEvents.on('failed', async ({ jobId, failedReason }) => {
      const queue = getQueue(queueName);
      const job = await queue.getJob(jobId);
      if (!job || job.attemptsMade < (job.opts.attempts ?? 3)) return;  // 还有重试机会

      // 最终失败：进入 DLQ 处理流程
      console.error(`[DLQ] Job permanently failed`, { queueName, jobId, failedReason });

      await handleDLQ(queueName, job);
    });
  }
}

async function handleDLQ(queueName: JobName, job: Job) {
  switch (queueName) {
    case 'email:send':
      // 邮件发送失败：降级到数据库队列（下次登录时站内信）
      await prisma.inboxMessage.create({
        data: {
          userId: job.data.userId,
          content: `邮件发送失败（${job.data.template}），请检查邮箱地址`,
        },
      });
      break;

    case 'video:transcode':
      // 转码失败：通知用户，更新状态
      await prisma.video.update({
        where: { id: job.data.videoId },
        data: { status: 'failed', failureReason: job.failedReason },
      });
      await alertingService.sendHigh(`Video transcode permanently failed: ${job.data.videoId}`);
      break;

    case 'report:generate':
      // 报表失败：发 Slack 告警 + 人工处理
      await slackNotifier.send({
        channel: '#ops-alerts',
        text: `报表生成失败 reportId=${job.data.reportId} userId=${job.data.userId}`,
      });
      break;
  }
}

// 定期重试 DLQ（如果下游服务恢复）
export async function retryDLQJobs(queueName: JobName, maxJobs = 10) {
  const queue = getQueue(queueName);
  const failedJobs = await queue.getFailed(0, maxJobs - 1);

  for (const job of failedJobs) {
    console.log(`Retrying failed job ${job.id}`);
    await job.retry();
  }
}
```

---

## 监控与可观测性

```typescript
// src/workers/monitoring.ts

import { QueueEvents, Queue } from 'bullmq';
import { register, Gauge, Counter, Histogram } from 'prom-client';

const jobsCompleted = new Counter({ name: 'bullmq_jobs_completed_total', labelNames: ['queue'] });
const jobsFailed = new Counter({ name: 'bullmq_jobs_failed_total', labelNames: ['queue'] });
const jobsActive = new Gauge({ name: 'bullmq_jobs_active', labelNames: ['queue'] });
const jobsWaiting = new Gauge({ name: 'bullmq_jobs_waiting', labelNames: ['queue'] });
const jobDuration = new Histogram({
  name: 'bullmq_job_duration_seconds',
  labelNames: ['queue'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60],
});

export function setupWorkerMetrics(queueName: string, worker: Worker) {
  const startTimes = new Map<string, number>();

  worker.on('active', (job) => {
    startTimes.set(job.id!, Date.now());
    jobsActive.inc({ queue: queueName });
  });

  worker.on('completed', (job) => {
    jobsCompleted.inc({ queue: queueName });
    jobsActive.dec({ queue: queueName });
    const start = startTimes.get(job.id!);
    if (start) {
      jobDuration.observe({ queue: queueName }, (Date.now() - start) / 1000);
      startTimes.delete(job.id!);
    }
  });

  worker.on('failed', () => {
    jobsFailed.inc({ queue: queueName });
    jobsActive.dec({ queue: queueName });
  });
}

// 定期更新队列深度指标
export async function updateQueueMetrics(queueName: JobName) {
  const queue = getQueue(queueName);
  const counts = await queue.getJobCounts('waiting', 'active', 'delayed', 'failed');

  jobsWaiting.set({ queue: queueName }, counts.waiting + counts.delayed);
  // 队列积压告警：waiting > 1000 时发告警
  if (counts.waiting > 1000) {
    alertingService.sendHigh(`Queue ${queueName} backlog: ${counts.waiting} jobs`);
  }
}
```

---

## 面试追问

**Q: 如何保证 Worker 幂等性（消息至少一次 at-least-once 语义）？**
A: BullMQ 默认 at-least-once：Worker crash 或超时后，job 会被重新分配。因此必须保证 `process()` 幂等。方案：① Redis `SETNX` 记录已处理 jobId（本篇实现）；② DB 唯一约束（如 `notification.idempotencyKey = jobId`，重复插入时 catch P2002）；③ 操作本身天然幂等（如 `UPDATE ... SET status='done' WHERE id=? AND status='pending'`，多次执行结果相同）。

**Q: BullMQ 的 `concurrency` 和 `limiter` 有什么区别？**
A: `concurrency` 控制同时处理多少个 job（Worker 内部并发）；`limiter` 控制速率（每 duration 最多处理 max 个）。两者正交：`concurrency: 10, limiter: { max: 100, duration: 60000 }` 表示同时最多 10 个并发执行，但每分钟不超过 100 个（防止下游限流）。CPU 密集任务：concurrency 低（2-4）；IO 密集任务：concurrency 高（10-50）。

**Q: Worker 进程 OOM 了，正在处理的 job 怎么办？**
A: BullMQ 用 Redis 的 `BZPOPMIN`（阻塞弹出）加 `lockDuration` 心跳锁：Worker 每 `stalledInterval` 续期锁，如果 Worker crash 不续期，超时后 job 状态从 active 变回 waiting，被其他 Worker 重新拾取。所以 `lockDuration` 必须大于任务最长预期执行时间（转码任务设 300s），否则还没完成就被认为 stalled 而重新入队，导致重复执行。
