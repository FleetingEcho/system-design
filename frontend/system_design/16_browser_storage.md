# 浏览器存储选型

> 前端有六种存储机制，各有适用场景。面试常问："这个数据你会存在哪里，为什么？"

---

## 六种存储机制对比

| 存储 | 容量 | 生命周期 | 同步/异步 | 可访问范围 | 跨 Tab | SW 访问 |
|------|------|---------|---------|----------|--------|---------|
| **Cookie** | 4KB | 可设过期 | 同步 | 同源 + 可跨子域 | ✓ | ✗ |
| **LocalStorage** | 5-10MB | 永久 | 同步 | 同源 | ✓ | ✗ |
| **SessionStorage** | 5-10MB | Tab 关闭 | 同步 | 同源 + 同 Tab | ✗ | ✗ |
| **IndexedDB** | 50MB-无限 | 永久 | 异步 | 同源 | ✓ | ✓ |
| **Cache API** | 50MB-无限 | 永久 | 异步 | 同源 | ✓ | ✓ |
| **OPFS** | GB 级 | 永久 | 异步 | 同源 | Worker 内 | ✓ |

---

## Cookie

### 核心属性

```typescript
// 设置 Cookie（服务端，推荐）
res.cookie('session', token, {
  httpOnly: true,     // JS 无法读取（防 XSS 窃取）
  secure: true,       // 仅 HTTPS 传输
  sameSite: 'strict', // 防 CSRF
  maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 天（毫秒）
  domain: '.example.com',            // 允许子域名共享（如 api.example.com）
  path: '/',
});

// 客户端 JS 设置（只能设置非 HttpOnly 的）
document.cookie = 'theme=dark; max-age=31536000; path=/; samesite=lax';

// 读取 Cookie（JS 端）
function getCookie(name: string): string | null {
  const match = document.cookie.match(new RegExp(`(?:^|; )${name}=([^;]*)`));
  return match ? decodeURIComponent(match[1]) : null;
}
```

### 适用场景

```
✓ Session Token（HttpOnly，防 XSS）
✓ 跨子域共享状态（domain=.example.com）
✓ 服务端需要读取的偏好（语言、地区）
✓ 第三方 OAuth Token

✗ 大量数据（4KB 限制）
✗ 敏感数据明文存储（加密后存）
```

---

## LocalStorage

```typescript
// 简单 KV 存储，同步 API
localStorage.setItem('theme', 'dark');
const theme = localStorage.getItem('theme');  // 'dark' | null
localStorage.removeItem('theme');
localStorage.clear();  // 清空所有

// 只能存字符串，对象需要序列化
localStorage.setItem('user', JSON.stringify({ id: '123', name: 'Alice' }));
const user = JSON.parse(localStorage.getItem('user') ?? 'null');

// 封装带类型的 LocalStorage
function createStorage<T>(key: string, defaultValue: T) {
  return {
    get(): T {
      try {
        const item = localStorage.getItem(key);
        return item ? (JSON.parse(item) as T) : defaultValue;
      } catch {
        return defaultValue;
      }
    },
    set(value: T) {
      localStorage.setItem(key, JSON.stringify(value));
    },
    remove() {
      localStorage.removeItem(key);
    },
  };
}

const cartStorage = createStorage<CartItem[]>('cart', []);
cartStorage.set([{ id: '1', quantity: 2 }]);
```

### 跨 Tab 通信

```typescript
// LocalStorage 变化会触发其他 Tab 的 storage 事件
window.addEventListener('storage', (event) => {
  if (event.key === 'cart' && event.newValue) {
    const newCart = JSON.parse(event.newValue);
    updateCartUI(newCart);
  }
});

// 注意：storage 事件只在"其他 Tab"触发，不在当前 Tab 触发
```

### 适用场景

```
✓ 用户偏好（主题、语言、侧边栏状态）
✓ 非敏感的持久化数据（草稿、搜索历史）
✓ 简单缓存（无需索引查询）

✗ 敏感数据（明文，JS 可读，XSS 风险）
✗ 大量数据（5MB 限制）
✗ 需要索引/查询的数据
✗ Service Worker 内使用
```

---

## SessionStorage

```typescript
// 与 LocalStorage API 完全相同，但只在当前 Tab 有效
sessionStorage.setItem('checkout-step', '2');
sessionStorage.setItem('form-draft', JSON.stringify(formData));

// Tab 关闭后自动清除
// 刷新页面数据仍在（不同于内存状态）
// 多 Tab 之间不共享（区别于 LocalStorage）
```

### 适用场景

```
✓ 多步骤表单的中间状态（防刷新丢失）
✓ 页面内的临时状态（搜索过滤器）
✓ 不应跨 Tab 共享的数据（独立的购物流程）

✗ 需要 Tab 间同步
✗ Tab 关闭后需要保留
```

---

## IndexedDB

> 功能最强的客户端存储，但 API 复杂，推荐用 Dexie.js 封装。

```typescript
import Dexie, { type Table } from 'dexie';

// 定义数据库结构（含版本迁移）
class AppDB extends Dexie {
  drafts!: Table<Draft>;
  offlineActions!: Table<OfflineAction>;
  cachedUsers!: Table<User>;

  constructor() {
    super('AppDB');

    // v1 初始建表
    this.version(1).stores({
      drafts: '++id, postId, updatedAt',
      offlineActions: '++id, type, createdAt',
    });

    // v2 新增 cachedUsers 表（数据迁移）
    this.version(2).stores({
      drafts: '++id, postId, updatedAt',
      offlineActions: '++id, type, createdAt',
      cachedUsers: 'id, email, &username',  // & 表示唯一索引
    }).upgrade(tx => {
      // 数据迁移逻辑（如需要）
    });
  }
}

const db = new AppDB();

// 事务操作（原子性）
async function saveDraftWithMedia(draft: Draft, media: MediaFile[]) {
  await db.transaction('rw', db.drafts, async () => {
    const draftId = await db.drafts.put(draft);
    // 事务失败会自动回滚
  });
}

// 复杂查询（利用索引）
const recentDrafts = await db.drafts
  .where('updatedAt').above(Date.now() - 7 * 24 * 60 * 60 * 1000)  // 7天内
  .reverse()
  .limit(10)
  .toArray();

// 游标查询（处理海量数据，不全加载到内存）
await db.drafts.each(draft => {
  if (draft.content.length > 10000) {
    // 处理长文章
  }
});
```

### 适用场景

```
✓ 离线数据存储（待同步的操作队列）
✓ 大量结构化数据（联系人、消息记录）
✓ 需要索引查询的本地数据
✓ Service Worker 内需要访问的数据

✗ 简单 KV 存储（用 LocalStorage 更简单）
✗ 需要跨域访问
```

---

## Cache API

```typescript
// 专为 Service Worker 缓存设计，也可在主线程使用
// 存储 Request/Response 对（HTTP 响应缓存）

// 打开缓存（按版本命名，便于清理旧版本）
const cache = await caches.open('app-v1.2.3');

// 存入缓存
await cache.put('/api/products', new Response(JSON.stringify(products), {
  headers: { 'Content-Type': 'application/json', 'Cache-Control': 'max-age=300' },
}));

// 读取缓存
const cached = await caches.match('/api/products');
if (cached) {
  const data = await cached.json();
}

// 删除旧版本缓存（在 SW activate 事件中）
const expectedCaches = ['app-v1.2.3', 'fonts-v1'];
const allCaches = await caches.keys();
await Promise.all(
  allCaches
    .filter(name => !expectedCaches.includes(name))
    .map(name => caches.delete(name))
);
```

### 适用场景

```
✓ Service Worker 离线缓存策略
✓ 缓存 API 响应（用于离线）
✓ 缓存静态资源（JS/CSS/图片）

✗ 非 HTTP 数据存储（用 IndexedDB）
✗ 需要复杂查询
```

---

## OPFS（Origin Private File System）

> Chrome 86+ / Safari 15.2+，允许 Web App 访问一个私有沙盒文件系统，
> 性能接近原生，适合 GB 级文件操作（SQLite、视频编辑、本地 IDE）。

```typescript
// 获取 OPFS 根目录
const root = await navigator.storage.getDirectory();

// 创建文件
const fileHandle = await root.getFileHandle('database.sqlite', { create: true });

// 写入（使用 AccessHandle，性能最高，仅在 Worker 中可用）
// 在 Web Worker 中：
const accessHandle = await fileHandle.createSyncAccessHandle();
const encoder = new TextEncoder();
const data = encoder.encode('Hello, OPFS!');
accessHandle.write(data, { at: 0 });
accessHandle.flush();
accessHandle.close();

// 主线程读取（异步）
const file = await fileHandle.getFile();
const content = await file.text();
```

### 实际应用：SQLite in WASM

```typescript
// @sqlite.org/sqlite-wasm 使用 OPFS 作为持久化层
import sqlite3InitModule from '@sqlite.org/sqlite-wasm';

const sqlite3 = await sqlite3InitModule({ print: console.log });
const db = new sqlite3.oo1.OpfsDb('/myapp.sqlite');  // 数据持久化到 OPFS

db.exec('CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, content TEXT)');
db.exec({ sql: 'INSERT INTO notes (content) VALUES (?)', bind: ['Hello World'] });
const rows = db.exec({ sql: 'SELECT * FROM notes', returnValue: 'resultRows' });
```

### 适用场景

```
✓ 本地 SQLite 数据库（超大量结构化数据）
✓ 视频/音频编辑（本地文件处理）
✓ 本地代码编辑器（VS Code Web）
✓ 游戏存档（大量二进制数据）

✗ 小数据（用 LocalStorage/IndexedDB 更简单）
✗ 需要主线程同步访问（OPFS 同步 API 只在 Worker 中可用）
```

---

## 存储配额管理

```typescript
// 查询存储配额和使用量
async function checkStorageQuota() {
  const estimate = await navigator.storage.estimate();

  const usedMB = ((estimate.usage ?? 0) / 1024 / 1024).toFixed(1);
  const quotaMB = ((estimate.quota ?? 0) / 1024 / 1024).toFixed(1);
  const usagePercent = ((estimate.usage ?? 0) / (estimate.quota ?? 1) * 100).toFixed(1);

  console.log(`Storage: ${usedMB}MB / ${quotaMB}MB (${usagePercent}%)`);

  // 使用率 > 80% 时告警
  if ((estimate.usage ?? 0) / (estimate.quota ?? 1) > 0.8) {
    notifyUserToCleanup();
  }
}

// 申请持久化存储（不被浏览器自动清除）
async function requestPersistentStorage(): Promise<boolean> {
  if (await navigator.storage.persisted()) return true;  // 已持久化

  const granted = await navigator.storage.persist();
  if (granted) {
    console.log('Persistent storage granted');
  } else {
    console.warn('Persistent storage denied - data may be evicted');
  }
  return granted;
}
```

---

## 选型决策树

```
需要服务端也能读取？
  是 → Cookie（HttpOnly）

需要防 XSS / 安全 Token？
  是 → Cookie（HttpOnly + Secure）

简单用户偏好、非敏感？
  是 → LocalStorage

当前 Tab 临时状态、关闭即清？
  是 → SessionStorage

大量结构化数据 / 离线 / 需要索引查询？
  是 → IndexedDB（Dexie.js）

Service Worker 缓存 HTTP 响应？
  是 → Cache API

GB 级文件 / 本地 SQLite？
  是 → OPFS

数据是服务端的缓存副本（非持久化）？
  考虑 → TanStack Query 内存缓存（无需持久化到存储）
```

---

## 面试常见追问

**Q: JWT 应该存在 LocalStorage 还是 Cookie？**
A: 安全首选 **HttpOnly Cookie**（JS 无法读取，防 XSS 窃取）。LocalStorage 存 JWT 的风险是 XSS 攻击可以直接读取。如果必须用 LocalStorage（如跨域 API），至少要有完善的 XSS 防御（CSP）。Access Token 可存内存（最安全，但页面刷新丢失），Refresh Token 存 HttpOnly Cookie。

**Q: LocalStorage 的同步 API 为什么是性能问题？**
A: LocalStorage 读写是同步的，在主线程执行，可能阻塞渲染。大数据量的读写（如序列化 1000 条记录）会导致页面卡顿。解决方案：小数据无所谓，大数据用 IndexedDB（异步）；或用 Web Worker 做 LocalStorage 操作（但 Worker 无法访问 LocalStorage，需用 IndexedDB）。

**Q: 浏览器什么时候会清除 IndexedDB 数据？**
A: 三种情况：①存储空间不足时（Eviction），按 LRU 清除（除非申请了 Persistent Storage）；②用户手动清除浏览器数据；③隐私模式（Incognito）关闭时。`navigator.storage.persist()` 申请持久化保护后，只有用户主动清除才会删除。

**Q: 多 Tab 如何实时同步状态？**
A: 四种方案：①LocalStorage + `storage` 事件（简单，只能传字符串，当前 Tab 不触发）；②BroadcastChannel（推荐，更灵活，可传对象，所有 Tab 含当前 Tab 都收到）；③SharedWorker（多 Tab 共享一个 Worker，适合复杂状态）；④Service Worker + `clients.matchAll()`（最强大，也最复杂）。
