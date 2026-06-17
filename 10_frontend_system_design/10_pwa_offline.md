# PWA / 离线支持

> PWA（Progressive Web App）的核心是让 Web App 在不稳定或无网络环境下仍然可用。
> 三大支柱：Service Worker（拦截网络）、Web App Manifest（安装到桌面）、IndexedDB（本地持久化）。

---

## PWA 能力全景

```
PWA
├── 离线访问（Service Worker + Cache API）
├── 安装到主屏幕（Web App Manifest）
├── 推送通知（Push API + Notifications API）
├── 后台同步（Background Sync API）
├── 本地持久化（IndexedDB）
└── 设备能力（Camera / Geolocation / Bluetooth — Web APIs）
```

---

## Service Worker 生命周期

```
1. 注册（Register）
   main.js → navigator.serviceWorker.register('/sw.js')

2. 安装（Install）
   sw.js → install 事件 → 预缓存关键资源

3. 激活（Activate）
   sw.js → activate 事件 → 清理旧缓存

4. 运行（Fetch）
   所有网络请求经过 sw.js → fetch 事件 → 返回缓存/网络/混合响应

5. 更新（Update）
   检测到新 sw.js → 等待旧 SW 释放控制权 → 新 SW 激活
```

```typescript
// main.ts — 注册 Service Worker
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const reg = await navigator.serviceWorker.register('/sw.js', {
        scope: '/',
        updateViaCache: 'none',  // 每次都检查 SW 更新，不用 HTTP 缓存
      });

      // 检测到新版本，提示用户刷新
      reg.addEventListener('updatefound', () => {
        const newWorker = reg.installing!;
        newWorker.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            showUpdateBanner(); // "发现新版本，点击刷新"
          }
        });
      });
    } catch (err) {
      console.error('SW registration failed:', err);
    }
  });
}
```

---

## 五种缓存策略

### 策略 1：Cache First（缓存优先）

```typescript
// 适合：不常变化的静态资源（字体、图标、框架 JS）
self.addEventListener('fetch', (event: FetchEvent) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      if (cached) return cached;  // 命中缓存，直接返回

      // 未命中，从网络获取并缓存
      return fetch(event.request).then(response => {
        const cache = await caches.open('static-v1');
        cache.put(event.request, response.clone());
        return response;
      });
    })
  );
});
```

### 策略 2：Network First（网络优先）

```typescript
// 适合：需要最新数据但离线时可接受旧数据（新闻、Feed）
async function networkFirst(request: Request): Promise<Response> {
  try {
    const response = await fetch(request);
    const cache = await caches.open('dynamic-v1');
    cache.put(request, response.clone());
    return response;
  } catch {
    // 网络失败，降级到缓存
    const cached = await caches.match(request);
    if (cached) return cached;
    // 都没有，返回离线页
    return caches.match('/offline.html') as Promise<Response>;
  }
}
```

### 策略 3：Stale-While-Revalidate（先旧后新）

```typescript
// 适合：用户体验优先（先返回旧缓存，后台更新）
// 场景：个人资料、配置信息
async function staleWhileRevalidate(request: Request): Promise<Response> {
  const cache = await caches.open('dynamic-v1');
  const cached = await cache.match(request);

  // 后台发起更新（不等待结果）
  const fetchPromise = fetch(request).then(response => {
    cache.put(request, response.clone());
    return response;
  });

  // 立即返回缓存（如果有）；否则等网络
  return cached ?? fetchPromise;
}
```

### 策略 4：Cache Only（仅缓存）

```typescript
// 适合：完全离线资源（预缓存的 App Shell）
async function cacheOnly(request: Request): Promise<Response> {
  const cached = await caches.match(request);
  if (!cached) throw new Error('Not in cache');
  return cached;
}
```

### 策略 5：Network Only（仅网络）

```typescript
// 适合：不能缓存的请求（支付、认证、实时数据）
async function networkOnly(request: Request): Promise<Response> {
  return fetch(request);
}
```

### 策略路由（综合应用）

```typescript
// sw.ts — 按请求类型选择策略
self.addEventListener('fetch', (event: FetchEvent) => {
  const url = new URL(event.request.url);

  // 静态资源 → Cache First
  if (url.pathname.match(/\.(js|css|woff2|png|svg)$/)) {
    event.respondWith(cacheFirst(event.request));
    return;
  }

  // API 请求 → Network First
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(event.request));
    return;
  }

  // 导航请求（HTML 页面）→ App Shell 策略
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match('/app-shell.html')!)
    );
    return;
  }

  // 其他 → Stale While Revalidate
  event.respondWith(staleWhileRevalidate(event.request));
});
```

---

## Workbox（生产级 SW 框架）

手写 Service Worker 容易出错，Workbox 是 Google 出品的 SW 工具库。

```typescript
// sw.ts（使用 Workbox）
import { precacheAndRoute, cleanupOutdatedCaches } from 'workbox-precaching';
import { registerRoute, NavigationRoute } from 'workbox-routing';
import {
  CacheFirst, NetworkFirst, StaleWhileRevalidate,
  NetworkOnly
} from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';
import { BackgroundSyncPlugin } from 'workbox-background-sync';

// 预缓存（由构建工具注入资源清单）
precacheAndRoute(self.__WB_MANIFEST);
cleanupOutdatedCaches();

// 字体 — Cache First，缓存 1 年
registerRoute(
  ({ url }) => url.origin === 'https://fonts.gstatic.com',
  new CacheFirst({
    cacheName: 'google-fonts',
    plugins: [new ExpirationPlugin({ maxAgeSeconds: 365 * 24 * 60 * 60 })],
  })
);

// API — Network First，离线返回缓存
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    networkTimeoutSeconds: 3,  // 3s 超时降级到缓存
    plugins: [new ExpirationPlugin({ maxEntries: 100, maxAgeSeconds: 24 * 60 * 60 })],
  })
);

// 图片 — Stale While Revalidate
registerRoute(
  ({ request }) => request.destination === 'image',
  new StaleWhileRevalidate({
    cacheName: 'images',
    plugins: [new ExpirationPlugin({ maxEntries: 200 })],
  })
);

// 支付接口 — Network Only（绝不缓存）
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/payment'),
  new NetworkOnly()
);

// SPA 导航 — 返回 App Shell
registerRoute(
  new NavigationRoute(new NetworkFirst({ cacheName: 'pages' }))
);
```

```typescript
// vite.config.ts — Workbox 插件集成
import { VitePWA } from 'vite-plugin-pwa';

export default {
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [...],
      },
      manifest: {
        name: 'My App',
        short_name: 'App',
        theme_color: '#1677ff',
        icons: [
          { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
        ],
      },
    }),
  ],
};
```

---

## IndexedDB：客户端持久化

### 为什么不用 LocalStorage

```
LocalStorage:
  ✗ 同步 API（阻塞主线程）
  ✗ 只能存字符串
  ✗ 5MB 限制
  ✗ Service Worker 无法访问

IndexedDB:
  ✓ 异步 API（不阻塞）
  ✓ 存任意 JS 对象（结构化克隆）
  ✓ 50MB+ 配额（可申请更多）
  ✓ Service Worker 可访问
  ✓ 支持索引、事务、游标
```

### Dexie.js（推荐封装库）

```typescript
import Dexie, { type Table } from 'dexie';

interface Post {
  id: string;
  title: string;
  content: string;
  authorId: string;
  createdAt: number;
  synced: boolean;  // 是否已同步到服务器
}

interface PendingOperation {
  id?: number;
  type: 'create' | 'update' | 'delete';
  resource: string;
  data: unknown;
  createdAt: number;
}

class AppDatabase extends Dexie {
  posts!: Table<Post>;
  pendingOps!: Table<PendingOperation>;

  constructor() {
    super('AppDB');

    this.version(1).stores({
      posts: 'id, authorId, createdAt, synced',  // 主键 + 索引
      pendingOps: '++id, type, resource, createdAt',  // ++id 自增主键
    });
  }
}

export const db = new AppDatabase();

// 使用示例
async function savePostOffline(post: Post) {
  await db.transaction('rw', db.posts, db.pendingOps, async () => {
    await db.posts.put({ ...post, synced: false });
    await db.pendingOps.add({
      type: 'create',
      resource: 'posts',
      data: post,
      createdAt: Date.now(),
    });
  });
}

// 查询（利用索引）
const myPosts = await db.posts
  .where('authorId').equals(userId)
  .and(post => !post.synced)  // 未同步的帖子
  .sortBy('createdAt');
```

---

## 后台同步（Background Sync）

```typescript
// 用户离线时保存操作，网络恢复后自动重试

// sw.ts — 监听 sync 事件
self.addEventListener('sync', (event: SyncEvent) => {
  if (event.tag === 'sync-posts') {
    event.waitUntil(syncPendingPosts());
  }
});

async function syncPendingPosts() {
  const db = new AppDatabase();
  const pending = await db.pendingOps.where('resource').equals('posts').toArray();

  for (const op of pending) {
    try {
      await fetch('/api/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(op.data),
      });

      // 同步成功，标记并清理
      await db.posts.update((op.data as Post).id, { synced: true });
      await db.pendingOps.delete(op.id!);
    } catch {
      // 单条失败，继续尝试其他的（网络可能不稳定）
      console.warn(`Failed to sync op ${op.id}`);
    }
  }
}

// 主线程 — 注册同步任务
async function saveAndQueueSync(post: Post) {
  await savePostOffline(post);

  if ('serviceWorker' in navigator && 'sync' in ServiceWorkerRegistration.prototype) {
    const reg = await navigator.serviceWorker.ready;
    await reg.sync.register('sync-posts');
    // 网络恢复时自动触发，即使页面已关闭
  } else {
    // 降级：直接尝试同步
    await syncPendingPosts();
  }
}
```

---

## 推送通知

```typescript
// 1. 请求权限
async function requestNotificationPermission(): Promise<boolean> {
  const permission = await Notification.requestPermission();
  return permission === 'granted';
}

// 2. 订阅推送
async function subscribePush(serverPublicKey: string) {
  const reg = await navigator.serviceWorker.ready;

  const subscription = await reg.pushManager.subscribe({
    userVisibleOnly: true,  // 必须显示通知（不能静默）
    applicationServerKey: urlBase64ToUint8Array(serverPublicKey),
  });

  // 3. 将订阅信息发送给服务器
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  });
}

// sw.ts — 接收推送
self.addEventListener('push', (event: PushEvent) => {
  const data = event.data?.json() ?? {};

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192.png',
      badge: '/badge.png',
      data: { url: data.url },
      actions: [
        { action: 'view', title: 'View' },
        { action: 'dismiss', title: 'Dismiss' },
      ],
    })
  );
});

// 点击通知
self.addEventListener('notificationclick', (event: NotificationEvent) => {
  event.notification.close();

  if (event.action === 'view') {
    event.waitUntil(
      clients.openWindow(event.notification.data.url)
    );
  }
});
```

---

## Web App Manifest

```json
// public/manifest.json
{
  "name": "My Awesome App",
  "short_name": "MyApp",
  "description": "The best app ever",
  "start_url": "/?source=pwa",
  "display": "standalone",
  "orientation": "portrait",
  "theme_color": "#1677ff",
  "background_color": "#ffffff",
  "icons": [
    { "src": "/icons/icon-72.png", "sizes": "72x72", "type": "image/png" },
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "maskable" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ],
  "shortcuts": [
    {
      "name": "New Post",
      "short_name": "New",
      "url": "/compose",
      "icons": [{ "src": "/icons/compose.png", "sizes": "96x96" }]
    }
  ],
  "screenshots": [
    { "src": "/screenshots/home.png", "sizes": "1280x720", "type": "image/png" }
  ]
}
```

---

## 更新提示 UX

```typescript
// 检测到新 SW 后，提示用户刷新（避免强制刷新破坏用户操作）
function useServiceWorkerUpdate() {
  const [updateAvailable, setUpdateAvailable] = useState(false);

  useEffect(() => {
    if (!('serviceWorker' in navigator)) return;

    navigator.serviceWorker.ready.then(reg => {
      reg.addEventListener('updatefound', () => {
        const newWorker = reg.installing!;
        newWorker.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            setUpdateAvailable(true);
          }
        });
      });
    });
  }, []);

  const applyUpdate = () => {
    navigator.serviceWorker.controller?.postMessage({ type: 'SKIP_WAITING' });
    window.location.reload();
  };

  return { updateAvailable, applyUpdate };
}

// sw.ts — 响应 SKIP_WAITING
self.addEventListener('message', (event: MessageEvent) => {
  if (event.data?.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});
```

---

## 面试常见追问

**Q: Service Worker 和 Web Worker 的区别？**
A: Web Worker 是通用后台线程，用于 CPU 密集型计算，与页面生命周期绑定。Service Worker 是网络代理，拦截 fetch 请求，生命周期独立于页面（页面关闭后仍可运行），主要用于缓存和离线。

**Q: Cache API 和 HTTP 缓存（Cache-Control）如何配合？**
A: 两者独立工作。HTTP 缓存由浏览器控制（根据响应头）；Cache API 由 Service Worker 代码控制。Service Worker 在 HTTP 缓存层之前介入，可以返回 Cache API 中的数据，完全绕过 HTTP 缓存。最佳实践：SW 管理应用级缓存逻辑，HTTP 缓存作为次级缓存。

**Q: IndexedDB 数据会被浏览器清除吗？**
A: 可能。浏览器在存储空间不足时会驱逐数据（Eviction）。可以用 `navigator.storage.persist()` 申请持久化存储（需要用户授权），申请后不会被自动清除。用 `navigator.storage.estimate()` 查询剩余配额。

**Q: PWA 在 iOS Safari 的限制有哪些？**
A: 主要限制：①Cache API 配额 50MB（远低于 Android Chrome）；②Background Sync 不支持；③Push Notification 支持有限（iOS 16.4+ 才支持，且 App 必须添加到主屏幕）；④Service Worker 在 Safari 后台会被更激进地终止。
