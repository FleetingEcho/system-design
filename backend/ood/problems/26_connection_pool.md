# OOD：数据库连接池（Connection Pool）

> 设计管理数据库连接的连接池：acquire/release、等待队列、连接健康检查、超时控制。
> 面试考察：资源管理模式、并发控制、生产者-消费者、泄漏防御。

---

## 需求分析

```
为什么需要连接池？

不用连接池：
  每次请求 → 建立新 TCP 连接（10-100ms）→ 查询 → 关闭连接
  1000 QPS = 1000 个并发 TCP 连接，数据库撑不住

用连接池：
  启动时建立 N 个复用连接
  请求来了 → acquire（从池里借一个）→ 查询 → release（归还）
  1000 QPS 但同时只用 20 个连接

功能需求：
  1. acquire：从池中获取空闲连接，无空闲则等待
  2. release：归还连接到池
  3. 最大连接数限制（超过则排队等待）
  4. 最小空闲连接（保持热连接，减少 acquire 延迟）
  5. 等待超时（等太久的请求报错）
  6. 连接健康检查（定期 ping，剔除断掉的连接）
  7. 连接泄漏检测（acquire 后很久没 release，发出警告）
```

---

## 核心接口

```typescript
// 被池管理的连接的抽象
interface PooledConnection {
  id: string;
  // 执行查询（实际的数据库连接方法）
  query(sql: string, params?: unknown[]): Promise<unknown>;
  // 检查连接是否存活
  ping(): Promise<boolean>;
  // 物理关闭连接
  close(): Promise<void>;
}

// 连接工厂（由使用者提供，池只管生命周期）
interface ConnectionFactory<T extends PooledConnection> {
  create(): Promise<T>;
  validate(connection: T): Promise<boolean>;  // 验证连接是否可用
  destroy(connection: T): Promise<void>;
}

interface PoolOptions {
  min: number;              // 最小保持连接数（默认 2）
  max: number;              // 最大连接数（默认 10）
  acquireTimeout: number;   // 等待空闲连接的超时（ms，默认 5000）
  idleTimeout: number;      // 空闲连接关闭前的等待时间（ms，默认 30000）
  leakDetectionThreshold: number;  // 连接持有超过此时间报警（ms，默认 60000）
}
```

---

## 连接池实现

```typescript
// src/pool/connection-pool.ts
import { EventEmitter } from 'events';
import { randomUUID } from 'crypto';

interface PoolEntry<T extends PooledConnection> {
  connection: T;
  id: string;
  state: 'idle' | 'active';
  createdAt: Date;
  lastUsedAt: Date;
  acquiredAt?: Date;  // 用于泄漏检测
}

class ConnectionPool<T extends PooledConnection> extends EventEmitter {
  private pool: PoolEntry<T>[] = [];
  private waitQueue: Array<{
    resolve: (conn: T) => void;
    reject: (err: Error) => void;
    addedAt: number;
  }> = [];
  private totalCreated = 0;
  private isClosing = false;

  constructor(
    private factory: ConnectionFactory<T>,
    private options: PoolOptions
  ) {
    super();
    this.initialize();
    this.startMaintenance();
  }

  private async initialize() {
    // 预热：创建最小连接数
    const promises = Array.from({ length: this.options.min }, () => this.createConnection());
    await Promise.all(promises);
  }

  // 获取连接
  async acquire(): Promise<T> {
    if (this.isClosing) {
      throw new Error('Pool is closing, cannot acquire connection');
    }

    // 先尝试获取空闲连接
    const idleEntry = this.pool.find(e => e.state === 'idle');
    if (idleEntry) {
      return this.activateEntry(idleEntry);
    }

    // 没有空闲且未达到最大数量 → 创建新连接
    if (this.pool.length < this.options.max) {
      const entry = await this.createConnection();
      return this.activateEntry(entry);
    }

    // 已达最大连接数 → 加入等待队列
    return new Promise<T>((resolve, reject) => {
      const timeout = setTimeout(() => {
        // 从队列中移除
        const idx = this.waitQueue.findIndex(w => w.resolve === resolve);
        if (idx !== -1) this.waitQueue.splice(idx, 1);

        reject(new Error(
          `Connection acquire timeout after ${this.options.acquireTimeout}ms ` +
          `(pool: ${this.pool.length}/${this.options.max}, waiting: ${this.waitQueue.length})`
        ));
      }, this.options.acquireTimeout);

      this.waitQueue.push({
        resolve: (conn) => {
          clearTimeout(timeout);
          resolve(conn);
        },
        reject: (err) => {
          clearTimeout(timeout);
          reject(err);
        },
        addedAt: Date.now(),
      });
    });
  }

  // 归还连接
  release(connection: T): void {
    const entry = this.pool.find(e => e.connection === connection);
    if (!entry) {
      this.emit('warning', 'Releasing connection not belonging to this pool');
      return;
    }
    if (entry.state !== 'active') {
      this.emit('warning', `Releasing connection that is already ${entry.state}`);
      return;
    }

    entry.state = 'idle';
    entry.lastUsedAt = new Date();
    delete entry.acquiredAt;

    // 通知等待队列
    if (this.waitQueue.length > 0) {
      const waiter = this.waitQueue.shift()!;
      waiter.resolve(this.activateEntry(entry));
    }
  }

  // 带自动释放的安全 acquire（类似 using）
  async withConnection<R>(fn: (conn: T) => Promise<R>): Promise<R> {
    const conn = await this.acquire();
    try {
      return await fn(conn);
    } finally {
      this.release(conn);
    }
  }

  private async createConnection(): Promise<PoolEntry<T>> {
    const connection = await this.factory.create();
    this.totalCreated++;

    const entry: PoolEntry<T> = {
      connection,
      id: randomUUID(),
      state: 'idle',
      createdAt: new Date(),
      lastUsedAt: new Date(),
    };

    this.pool.push(entry);
    this.emit('connection:created', { id: entry.id, total: this.pool.length });
    return entry;
  }

  private activateEntry(entry: PoolEntry<T>): T {
    entry.state = 'active';
    entry.acquiredAt = new Date();
    return entry.connection;
  }

  private async destroyEntry(entry: PoolEntry<T>): Promise<void> {
    const idx = this.pool.indexOf(entry);
    if (idx !== -1) this.pool.splice(idx, 1);

    try {
      await this.factory.destroy(entry.connection);
    } catch (err) {
      this.emit('warning', `Error destroying connection: ${(err as Error).message}`);
    }

    this.emit('connection:destroyed', { id: entry.id, total: this.pool.length });
  }

  // 维护任务：健康检查 + 泄漏检测 + 空闲连接回收
  private startMaintenance() {
    setInterval(async () => {
      const now = Date.now();

      for (const entry of [...this.pool]) {
        // 泄漏检测：连接 active 太久
        if (
          entry.state === 'active' &&
          entry.acquiredAt &&
          now - entry.acquiredAt.getTime() > this.options.leakDetectionThreshold
        ) {
          this.emit('leak:detected', {
            id: entry.id,
            heldForMs: now - entry.acquiredAt.getTime(),
          });
        }

        // 健康检查：验证 idle 连接是否存活
        if (entry.state === 'idle') {
          const idleMs = now - entry.lastUsedAt.getTime();

          // 超过空闲超时 + 超过最小连接数 → 关闭
          if (idleMs > this.options.idleTimeout && this.pool.length > this.options.min) {
            await this.destroyEntry(entry);
            continue;
          }

          // 定期 ping 检查连接是否存活
          try {
            const healthy = await this.factory.validate(entry.connection);
            if (!healthy) {
              await this.destroyEntry(entry);
              // 如果需要维持最小连接数，创建新连接
              if (this.pool.length < this.options.min) {
                await this.createConnection();
              }
            }
          } catch {
            await this.destroyEntry(entry);
          }
        }
      }
    }, 10_000);  // 每 10 秒检查一次
  }

  // 统计信息
  getStats() {
    return {
      total: this.pool.length,
      idle: this.pool.filter(e => e.state === 'idle').length,
      active: this.pool.filter(e => e.state === 'active').length,
      waiting: this.waitQueue.length,
      max: this.options.max,
      totalCreated: this.totalCreated,
    };
  }

  // 优雅关闭
  async close(): Promise<void> {
    this.isClosing = true;

    // 拒绝所有等待中的请求
    for (const waiter of this.waitQueue) {
      waiter.reject(new Error('Pool closed'));
    }
    this.waitQueue.length = 0;

    // 等待所有 active 连接释放（最多等 30s）
    const deadline = Date.now() + 30_000;
    while (this.pool.some(e => e.state === 'active') && Date.now() < deadline) {
      await new Promise(r => setTimeout(r, 100));
    }

    // 关闭所有连接
    await Promise.all(this.pool.map(e => this.destroyEntry(e)));
  }
}
```

---

## PostgreSQL 连接池使用示例

```typescript
// 实现 ConnectionFactory（适配 pg 模块）
import { Client } from 'pg';

class PostgresConnection implements PooledConnection {
  id = randomUUID();
  private client: Client;

  constructor(connectionString: string) {
    this.client = new Client({ connectionString });
  }

  async connect() {
    await this.client.connect();
  }

  async query(sql: string, params?: unknown[]) {
    return this.client.query(sql, params);
  }

  async ping() {
    try {
      await this.client.query('SELECT 1');
      return true;
    } catch {
      return false;
    }
  }

  async close() {
    await this.client.end();
  }
}

const postgresFactory: ConnectionFactory<PostgresConnection> = {
  async create() {
    const conn = new PostgresConnection(process.env.DATABASE_URL!);
    await conn.connect();
    return conn;
  },
  async validate(conn) {
    return conn.ping();
  },
  async destroy(conn) {
    await conn.close();
  },
};

const pool = new ConnectionPool(postgresFactory, {
  min: 2,
  max: 20,
  acquireTimeout: 5000,
  idleTimeout: 30_000,
  leakDetectionThreshold: 60_000,
});

// 监控
pool.on('leak:detected', ({ id, heldForMs }) => {
  logger.warn({ connectionId: id, heldForMs }, 'Possible connection leak detected');
});

// 使用（推荐：withConnection 自动释放）
const result = await pool.withConnection(async (conn) => {
  return conn.query('SELECT * FROM users WHERE id = $1', [userId]);
});

// 手动管理（需要确保 release）
const conn = await pool.acquire();
try {
  const result = await conn.query('SELECT 1');
  return result;
} finally {
  pool.release(conn);  // 必须在 finally 中释放
}
```

---

## 与 Prisma 连接池的关系

```
Prisma 内置连接池（基于 Rust 的 connection_manager）：
  - 生产时直接用 Prisma 即可，不需要自己实现
  - DATABASE_URL 中可配置：?connection_limit=20&pool_timeout=10

自己实现连接池的场景：
  - 使用原生 pg/mysql2 模块（不通过 ORM）
  - 需要连接池级别的精细控制
  - 学习/面试目的：理解池的工作原理

PgBouncer（外部连接池代理）：
  - Serverless 场景：每个函数实例都有 Prisma 连接池
    → 100 个函数实例 × 20 连接 = 2000 个 PostgreSQL 连接（超过 DB 上限）
  - PgBouncer 坐在应用和 DB 之间，对应用侧看起来有无限连接
    但对 DB 侧只维持少量真实连接（Transaction/Session pooling 模式）
  - Neon、Supabase 等托管 PostgreSQL 内置 PgBouncer
```

---

## 面试追问

**Q: 连接池为什么要设最小连接数？**
A: 避免冷启动：没有最小连接时，低流量期间连接全被回收，流量突增时需要重建连接（耗时 10-100ms），第一批请求延迟高。最小连接数保持"热连接"，随时可用。代价是空闲时也会占用数据库连接资源，最小值通常等于 CPU 核数或 2-5 个。

**Q: acquire 超时和连接超时有什么区别？**
A: acquire 超时：等待从池中获取可用连接的时间（队列等待），说明连接池满、所有连接都在使用；连接超时：实际建立 TCP 连接到数据库的时间，说明数据库无法访问或网络问题。前者是资源竞争问题（扩大连接池或优化查询），后者是基础设施问题。监控两者的告警应该不同。

**Q: 连接泄漏如何防御？**
A: ①使用 `withConnection`（类似 Python 的 `with` / Java 的 try-with-resources）确保自动释放；②泄漏检测：超过阈值（如 60s）还未释放的 active 连接，打警告日志并记录调用堆栈（acquire 时记录）；③强制超时释放：生产中谨慎（可能导致查询被中断），测试环境可以强制释放以提前发现问题。
