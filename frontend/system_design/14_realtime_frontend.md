# 实时通信前端模式

> 服务端的 WebSocket/SSE 架构见 [03_communication/03_realtime.md](../../backend/system_design/03_communication/03_realtime.md)。
> 本文聚焦**前端视角**：连接管理、断线重连、客户端消息队列、乐观 UI、presence 在线状态。

---

## 连接管理

### WebSocket 客户端封装

```typescript
// lib/websocket.ts — 生产级 WebSocket 客户端

type WSEventMap = {
  message: MessageEvent;
  open: Event;
  close: CloseEvent;
  error: Event;
};

type MessageHandler = (data: unknown) => void;

class WebSocketClient {
  private ws: WebSocket | null = null;
  private url: string;
  private handlers = new Map<string, Set<MessageHandler>>();

  // 重连配置
  private reconnectAttempts = 0;
  private readonly maxReconnectAttempts = 10;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;

  // 消息队列（离线时缓存，重连后发送）
  private messageQueue: string[] = [];

  // 心跳
  private heartbeatTimer: ReturnType<typeof setInterval> | null = null;
  private lastPongAt = Date.now();
  private readonly heartbeatInterval = 25_000;
  private readonly pongTimeout = 10_000;

  constructor(url: string) {
    this.url = url;
  }

  connect() {
    if (this.ws?.readyState === WebSocket.OPEN) return;

    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('WS connected');
      this.reconnectAttempts = 0;

      // 重连成功后，发送队列中的消息
      this.flushMessageQueue();

      // 启动心跳
      this.startHeartbeat();
    };

    this.ws.onmessage = (event) => {
      try {
        const msg = JSON.parse(event.data);

        // 心跳 pong
        if (msg.type === 'pong') {
          this.lastPongAt = Date.now();
          return;
        }

        // 分发到对应 handler
        this.handlers.get(msg.type)?.forEach(handler => handler(msg.data));
        this.handlers.get('*')?.forEach(handler => handler(msg));
      } catch (err) {
        console.error('Failed to parse WS message', err);
      }
    };

    this.ws.onclose = (event) => {
      this.stopHeartbeat();

      // 非主动关闭（code 1000 = 正常关闭），自动重连
      if (event.code !== 1000) {
        this.scheduleReconnect();
      }
    };

    this.ws.onerror = () => {
      // onerror 后必然触发 onclose，在 onclose 中处理重连
    };
  }

  // 指数退避重连
  private scheduleReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnect attempts reached');
      return;
    }

    // 指数退避 + 随机抖动：1s, 2s, 4s, 8s, ... 最大 30s
    const baseDelay = Math.min(1000 * 2 ** this.reconnectAttempts, 30_000);
    const jitter = Math.random() * 1000;
    const delay = baseDelay + jitter;

    this.reconnectAttempts++;
    console.log(`Reconnecting in ${(delay / 1000).toFixed(1)}s (attempt ${this.reconnectAttempts})`);

    this.reconnectTimer = setTimeout(() => this.connect(), delay);
  }

  send(type: string, data: unknown) {
    const message = JSON.stringify({ type, data });

    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(message);
    } else {
      // 离线时入队，重连后发送
      this.messageQueue.push(message);
    }
  }

  private flushMessageQueue() {
    while (this.messageQueue.length > 0 && this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(this.messageQueue.shift()!);
    }
  }

  private startHeartbeat() {
    this.heartbeatTimer = setInterval(() => {
      if (this.ws?.readyState !== WebSocket.OPEN) return;

      // 检查上次 pong 是否超时
      if (Date.now() - this.lastPongAt > this.heartbeatInterval + this.pongTimeout) {
        console.warn('Heartbeat timeout, reconnecting...');
        this.ws.close(4000, 'Heartbeat timeout');
        return;
      }

      this.ws.send(JSON.stringify({ type: 'ping' }));
    }, this.heartbeatInterval);
  }

  private stopHeartbeat() {
    if (this.heartbeatTimer) clearInterval(this.heartbeatTimer);
  }

  on(eventType: string, handler: MessageHandler) {
    if (!this.handlers.has(eventType)) {
      this.handlers.set(eventType, new Set());
    }
    this.handlers.get(eventType)!.add(handler);
    return () => this.handlers.get(eventType)?.delete(handler);  // 返回取消订阅函数
  }

  disconnect() {
    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
    this.stopHeartbeat();
    this.ws?.close(1000, 'Client disconnect');
  }
}
```

### React Hook 封装

```typescript
// hooks/useWebSocket.ts
import { useEffect, useRef, useState, useCallback } from 'react';

type ConnectionStatus = 'connecting' | 'connected' | 'disconnected' | 'error';

export function useWebSocket(url: string) {
  const clientRef = useRef<WebSocketClient | null>(null);
  const [status, setStatus] = useState<ConnectionStatus>('disconnected');

  useEffect(() => {
    const client = new WebSocketClient(url);
    clientRef.current = client;

    const unsubOpen = client.on('open', () => setStatus('connected'));
    const unsubClose = client.on('close', () => setStatus('disconnected'));
    const unsubError = client.on('error', () => setStatus('error'));

    client.connect();
    setStatus('connecting');

    return () => {
      unsubOpen();
      unsubClose();
      unsubError();
      client.disconnect();
    };
  }, [url]);

  const send = useCallback((type: string, data: unknown) => {
    clientRef.current?.send(type, data);
  }, []);

  const subscribe = useCallback((eventType: string, handler: MessageHandler) => {
    return clientRef.current?.on(eventType, handler) ?? (() => {});
  }, []);

  return { status, send, subscribe };
}

// 使用
function ChatRoom({ roomId }: { roomId: string }) {
  const { status, send, subscribe } = useWebSocket(`wss://api.example.com/rooms/${roomId}`);
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    return subscribe('chat:message', (data) => {
      setMessages(prev => [...prev, data as Message]);
    });
  }, [subscribe]);

  return (
    <div>
      <ConnectionStatus status={status} />
      <MessageList messages={messages} />
      <MessageInput onSend={(text) => send('chat:send', { text })} />
    </div>
  );
}
```

---

## 乐观 UI（Optimistic Update）

```typescript
// 场景：发送消息 — 先显示，失败再撤回

interface Message {
  id: string;
  text: string;
  senderId: string;
  status: 'sending' | 'sent' | 'failed';
  optimisticId?: string;  // 临时 ID
}

function useChatMessages(roomId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const { send, subscribe } = useWebSocket(`wss://.../${roomId}`);

  // 接收服务端确认
  useEffect(() => {
    return subscribe('chat:ack', ({ optimisticId, serverMessage }) => {
      setMessages(prev =>
        prev.map(m =>
          m.optimisticId === optimisticId
            ? { ...serverMessage, status: 'sent' }  // 替换为服务端确认的消息
            : m
        )
      );
    });
  }, [subscribe]);

  // 接收失败通知
  useEffect(() => {
    return subscribe('chat:error', ({ optimisticId }) => {
      setMessages(prev =>
        prev.map(m =>
          m.optimisticId === optimisticId ? { ...m, status: 'failed' } : m
        )
      );
    });
  }, [subscribe]);

  const sendMessage = useCallback((text: string) => {
    const optimisticId = crypto.randomUUID();

    // 立即显示（乐观）
    setMessages(prev => [
      ...prev,
      {
        id: optimisticId,
        text,
        senderId: 'me',
        status: 'sending',
        optimisticId,
      },
    ]);

    // 发送到服务器
    send('chat:send', { text, optimisticId });
  }, [send]);

  const retryMessage = useCallback((optimisticId: string) => {
    const msg = messages.find(m => m.optimisticId === optimisticId);
    if (!msg) return;

    setMessages(prev => prev.map(m =>
      m.optimisticId === optimisticId ? { ...m, status: 'sending' } : m
    ));
    send('chat:send', { text: msg.text, optimisticId });
  }, [messages, send]);

  return { messages, sendMessage, retryMessage };
}
```

---

## Presence（在线状态）

### 架构

```
客户端连接 → 服务端注册 presence
用户断开 → 服务端广播 user_left
定时心跳 → 服务端更新 last_seen
```

```typescript
// hooks/usePresence.ts
interface PresenceUser {
  id: string;
  name: string;
  avatar: string;
  status: 'online' | 'idle' | 'offline';
  lastSeen: number;
}

export function usePresence(roomId: string) {
  const [onlineUsers, setOnlineUsers] = useState<Map<string, PresenceUser>>(new Map());
  const { subscribe, send } = useWebSocket(`wss://.../${roomId}`);

  useEffect(() => {
    // 其他用户上线
    const unsubJoin = subscribe('presence:join', (user: PresenceUser) => {
      setOnlineUsers(prev => new Map(prev).set(user.id, user));
    });

    // 其他用户下线
    const unsubLeave = subscribe('presence:leave', ({ userId }: { userId: string }) => {
      setOnlineUsers(prev => {
        const next = new Map(prev);
        next.delete(userId);
        return next;
      });
    });

    // 初始化：获取当前在线列表
    const unsubSync = subscribe('presence:sync', (users: PresenceUser[]) => {
      setOnlineUsers(new Map(users.map(u => [u.id, u])));
    });

    return () => { unsubJoin(); unsubLeave(); unsubSync(); };
  }, [subscribe]);

  // 用户活跃状态检测（鼠标移动/键盘）
  useEffect(() => {
    let idleTimer: ReturnType<typeof setTimeout>;

    const markActive = () => {
      clearTimeout(idleTimer);
      send('presence:active', {});
      idleTimer = setTimeout(() => send('presence:idle', {}), 5 * 60 * 1000);
    };

    window.addEventListener('mousemove', markActive);
    window.addEventListener('keydown', markActive);

    return () => {
      window.removeEventListener('mousemove', markActive);
      window.removeEventListener('keydown', markActive);
      clearTimeout(idleTimer);
    };
  }, [send]);

  return { onlineUsers: Array.from(onlineUsers.values()) };
}
```

---

## 实时输入提示（Typing Indicator）

```typescript
// "对方正在输入..." 功能
function useTypingIndicator(roomId: string, currentUserId: string) {
  const [typingUsers, setTypingUsers] = useState<string[]>([]);
  const { send, subscribe } = useWebSocket(`wss://.../${roomId}`);
  const typingTimerRef = useRef<ReturnType<typeof setTimeout>>();

  // 接收他人输入状态
  useEffect(() => {
    const timers = new Map<string, ReturnType<typeof setTimeout>>();

    const unsub = subscribe('typing:update', ({ userId, isTyping }: { userId: string; isTyping: boolean }) => {
      if (userId === currentUserId) return;

      if (isTyping) {
        setTypingUsers(prev => prev.includes(userId) ? prev : [...prev, userId]);
        // 3 秒没有更新则自动清除（防止对方崩溃留下永久 typing 状态）
        clearTimeout(timers.get(userId));
        timers.set(userId, setTimeout(() => {
          setTypingUsers(prev => prev.filter(id => id !== userId));
        }, 3000));
      } else {
        clearTimeout(timers.get(userId));
        setTypingUsers(prev => prev.filter(id => id !== userId));
      }
    });

    return () => { unsub(); timers.forEach(clearTimeout); };
  }, [subscribe, currentUserId]);

  // 发送自己的输入状态（防抖，避免每次键入都发送）
  const handleTyping = useCallback(() => {
    send('typing:update', { isTyping: true });

    clearTimeout(typingTimerRef.current);
    typingTimerRef.current = setTimeout(() => {
      send('typing:update', { isTyping: false });
    }, 2000);
  }, [send]);

  return { typingUsers, handleTyping };
}
```

---

## SSE（Server-Sent Events）客户端

```typescript
// 适合：单向实时推送（通知、进度、新闻 Feed）
// 相比 WebSocket：更简单、自动重连、基于 HTTP（CDN 友好）

class SSEClient {
  private eventSource: EventSource | null = null;
  private url: string;
  private handlers = new Map<string, Set<(data: unknown) => void>>();

  constructor(url: string, private token: string) {
    this.url = url;
  }

  connect() {
    // EventSource 不支持自定义 Header，通过 URL 参数传 Token
    this.eventSource = new EventSource(`${this.url}?token=${this.token}`);

    this.eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handlers.get('message')?.forEach(h => h(data));
    };

    // 自定义事件类型
    ['notification', 'order_update', 'price_change'].forEach(type => {
      this.eventSource!.addEventListener(type, (event: MessageEvent) => {
        const data = JSON.parse(event.data);
        this.handlers.get(type)?.forEach(h => h(data));
      });
    });

    // EventSource 内置自动重连（断线后浏览器自动重试）
    this.eventSource.onerror = () => {
      // 浏览器会自动处理重连，这里只需要记录日志
      console.warn('SSE connection error, browser will retry...');
    };
  }

  on(type: string, handler: (data: unknown) => void) {
    if (!this.handlers.has(type)) this.handlers.set(type, new Set());
    this.handlers.get(type)!.add(handler);
    return () => this.handlers.get(type)?.delete(handler);
  }

  disconnect() {
    this.eventSource?.close();
  }
}
```

---

## 网络状态感知

```typescript
// 监听网络状态，连接恢复时重新同步数据
function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true);
      // 触发数据重新同步
      queryClient.invalidateQueries();  // TanStack Query 重新拉取
    };
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  return isOnline;
}

// 显示网络状态提示
function NetworkStatusBanner() {
  const isOnline = useNetworkStatus();
  const [showBanner, setShowBanner] = useState(false);

  useEffect(() => {
    if (!isOnline) {
      setShowBanner(true);
    } else {
      // 网络恢复后，短暂显示"已重新连接"再消失
      const timer = setTimeout(() => setShowBanner(false), 3000);
      return () => clearTimeout(timer);
    }
  }, [isOnline]);

  if (!showBanner) return null;

  return (
    <div className={`network-banner ${isOnline ? 'reconnected' : 'offline'}`}>
      {isOnline ? '✓ 已重新连接' : '⚠ 网络连接已断开，操作将在恢复后同步'}
    </div>
  );
}
```

---

## 社区工具

| 场景 | 工具 | 说明 |
|------|------|------|
| 实时协作完整方案 | **Liveblocks** | 托管 presence/房间/CRDT，开箱即用 |
| 实时状态同步 | **Partykit** | 边缘节点 WebSocket，Cloudflare Workers 支持 |
| 实时后端 | **Supabase Realtime** | Postgres 变更订阅，基于 WebSocket |
| WebSocket 测试 | **wscat** | 命令行 WebSocket 客户端 |
| SSE 服务端 | **EventSource polyfill** | 老浏览器兼容 |

---

## 面试常见追问

**Q: WebSocket 和 HTTP 长轮询怎么选？**
A: WebSocket 适合高频双向通信（聊天、协作编辑、游戏）；长轮询适合低频单向通知（后台任务进度）且不值得维护 WebSocket 基础设施时。SSE 介于两者之间：单向、低频/中频、基于 HTTP（防火墙友好）。

**Q: 消息队列满了（用户长时间离线）怎么处理？**
A: 设置队列上限（如 100 条），超出后丢弃最旧的消息，重连后从服务端拉取完整状态（而不是依赖所有离线消息都能重放）。关键操作（支付、订单）不应依赖 WebSocket，需要独立的持久化机制。

**Q: 多 Tab 页面如何共享一个 WebSocket 连接？**
A: 用 `SharedWorker`（所有 Tab 共享一个 Worker 线程）或 `BroadcastChannel`（Tab 间消息广播）。SharedWorker 维护一个连接，所有 Tab 通过它收发消息，避免多 Tab 建立多个连接。

**Q: 如何测试实时功能？**
A: 单元测试 Mock WebSocket（`jest-websocket-mock`）；集成测试用 MSW 的 WebSocket handler；E2E 测试（Playwright）可以同时打开两个页面，验证消息从一个传到另一个。
