# WebSocket 服务端架构（Socket.io）

> Socket.io 服务端：房间（Rooms）、命名空间、Redis Adapter 横向扩展、鉴权、事件广播。
> 与 `frontend/system_design/14_realtime_frontend.md` 配合，覆盖完整 WebSocket 架构。

---

## Socket.io vs 原生 WebSocket

```
原生 WebSocket：轻量，但需要自己实现：
  - 断线重连
  - 心跳检测
  - 房间/频道分组
  - 消息确认（Acknowledgement）
  - 多节点广播

Socket.io：在 WebSocket 上封装，开箱即得：
  - 自动降级（WebSocket → HTTP Long Polling，应对某些网络限制）
  - 命名空间（Namespace）和房间（Room）
  - 消息确认（带回调的 emit）
  - 多节点支持（Redis Adapter）
  - 自动重连

适合选 Socket.io 的场景：
  - 需要向房间广播（聊天室、游戏房间、协作文档）
  - 需要水平扩展（多个服务器实例）
  - 需要消息确认机制
```

---

## 基本服务端架构

```typescript
// src/socket/index.ts
import { Server, Socket } from 'socket.io';
import { createServer } from 'http';
import express from 'express';
import { verifyAccessToken } from '../lib/jwt';
import { UnauthorizedError } from '../lib/errors';

const app = express();
const httpServer = createServer(app);

export const io = new Server(httpServer, {
  cors: {
    origin: process.env.FRONTEND_URL,
    credentials: true,
  },
  // 连接超时和心跳
  pingInterval: 25_000,   // 每 25s 发一次 ping
  pingTimeout: 60_000,    // 60s 无响应断开
  // 每条消息最大大小（防止攻击）
  maxHttpBufferSize: 1e6,  // 1MB
});

httpServer.listen(3000);
```

---

## 鉴权中间件

```typescript
// src/socket/auth.middleware.ts
import { Socket } from 'socket.io';
import { verifyAccessToken } from '../lib/jwt';

// Socket.io 中间件：连接时鉴权
io.use((socket, next) => {
  // Token 从 auth 对象传入（前端：socket = io(url, { auth: { token } })）
  const token = socket.handshake.auth.token as string | undefined;

  if (!token) {
    return next(new Error('Authentication required'));
  }

  try {
    const user = verifyAccessToken(token);
    // 将用户信息挂载到 socket 上
    socket.data.user = user;
    next();
  } catch {
    next(new Error('Invalid or expired token'));
  }
});

// 命名空间级别的鉴权（不同命名空间不同权限）
const adminNs = io.of('/admin');
adminNs.use((socket, next) => {
  const user = socket.data.user;
  if (user?.role !== 'admin') {
    return next(new Error('Admin access required'));
  }
  next();
});
```

---

## 房间（Rooms）和事件广播

```typescript
// src/socket/chat.handler.ts
import { Server, Socket } from 'socket.io';

// 消息类型定义（确保前后端一致）
interface ChatMessage {
  id: string;
  roomId: string;
  userId: string;
  userName: string;
  content: string;
  createdAt: string;
}

interface TypingEvent {
  roomId: string;
  userId: string;
  userName: string;
}

export function registerChatHandlers(io: Server, socket: Socket) {
  const user = socket.data.user;

  // 加入房间
  socket.on('room:join', async (roomId: string) => {
    // 验证用户有权访问该房间
    const hasAccess = await checkRoomAccess(user.sub, roomId);
    if (!hasAccess) {
      socket.emit('error', { code: 'FORBIDDEN', message: 'No access to room' });
      return;
    }

    await socket.join(roomId);

    // 通知房间内其他人有人加入
    socket.to(roomId).emit('room:user_joined', {
      userId: user.sub,
      userName: user.name,
    });

    // 给加入者发最近 50 条历史消息
    const history = await getRecentMessages(roomId, 50);
    socket.emit('room:history', history);
  });

  // 发送消息
  socket.on('message:send', async (data: { roomId: string; content: string }, ack) => {
    // 验证用户在该房间中
    if (!socket.rooms.has(data.roomId)) {
      ack?.({ error: 'Not in room' });
      return;
    }

    // 持久化到数据库
    const message: ChatMessage = {
      id: randomUUID(),
      roomId: data.roomId,
      userId: user.sub,
      userName: user.name,
      content: data.content.slice(0, 2000),  // 限制长度
      createdAt: new Date().toISOString(),
    };

    await saveMessage(message);

    // 广播给房间内所有人（包括发送者）
    io.to(data.roomId).emit('message:new', message);

    // 确认发送成功（消息确认机制）
    ack?.({ success: true, messageId: message.id });
  });

  // 正在输入指示器
  socket.on('typing:start', (roomId: string) => {
    socket.to(roomId).emit('typing:user', {
      userId: user.sub,
      userName: user.name,
      roomId,
    } as TypingEvent);
  });

  socket.on('typing:stop', (roomId: string) => {
    socket.to(roomId).emit('typing:stopped', { userId: user.sub, roomId });
  });

  // 离开房间
  socket.on('room:leave', (roomId: string) => {
    socket.leave(roomId);
    socket.to(roomId).emit('room:user_left', { userId: user.sub });
  });

  // 断开连接
  socket.on('disconnect', (reason) => {
    // socket.rooms 在 disconnect 后已清空，需要在 disconnecting 中处理
    logger.info({ userId: user.sub, reason }, 'Socket disconnected');
  });

  // disconnecting：还在房间中，可以广播离开通知
  socket.on('disconnecting', () => {
    for (const roomId of socket.rooms) {
      if (roomId !== socket.id) {  // 排除自己的私有房间
        socket.to(roomId).emit('room:user_left', { userId: user.sub });
      }
    }
  });
}

// 注册 handler
io.on('connection', (socket) => {
  registerChatHandlers(io, socket);
});
```

---

## Redis Adapter（多节点扩展）

```
单节点问题：
  Node 1 上的用户 A 发消息 → io.to('room:123').emit(...)
  只有连接到 Node 1 的用户收到消息
  连接到 Node 2 的用户 B 在同一个房间，但收不到消息

Redis Adapter 解法：
  emit → 先写 Redis Pub/Sub → 所有节点订阅 → 所有节点转发给本地连接的用户
```

```typescript
// src/socket/index.ts（带 Redis Adapter）
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(createAdapter(pubClient, subClient));

// 现在 io.to('room:123').emit() 会广播到所有节点上该房间的成员
// io.of('/chat').to('room:123').emit() 同样有效
```

```typescript
// 服务端主动向特定用户发消息（用户 ID → Socket ID 映射）
// 问题：Socket ID 是连接级别的，用户可能有多个连接（多设备）

// 方案：用 Redis 存 userId → Set<socketId> 映射
class UserSocketManager {
  private redis: RedisClient;

  async addSocket(userId: string, socketId: string) {
    await this.redis.sAdd(`user:sockets:${userId}`, socketId);
    await this.redis.expire(`user:sockets:${userId}`, 86400);  // 1 天
  }

  async removeSocket(userId: string, socketId: string) {
    await this.redis.sRem(`user:sockets:${userId}`, socketId);
  }

  async getSocketIds(userId: string): Promise<string[]> {
    return this.redis.sMembers(`user:sockets:${userId}`);
  }
}

// 发送给特定用户的所有设备
async function notifyUser(userId: string, event: string, data: unknown) {
  const socketIds = await userSocketManager.getSocketIds(userId);
  for (const socketId of socketIds) {
    io.to(socketId).emit(event, data);
  }
}
```

---

## 在线状态（Presence）

```typescript
// src/socket/presence.ts
// 记录用户在线状态，断线时通知相关用户

io.on('connection', async (socket) => {
  const userId = socket.data.user.sub;

  // 上线：记录到 Redis
  await redis.hSet('online_users', userId, JSON.stringify({
    userId,
    socketId: socket.id,
    lastSeen: new Date().toISOString(),
  }));

  // 广播上线事件给好友
  const friendIds = await getFriendIds(userId);
  for (const friendId of friendIds) {
    const friendSocketIds = await userSocketManager.getSocketIds(friendId);
    for (const socketId of friendSocketIds) {
      io.to(socketId).emit('presence:online', { userId });
    }
  }

  socket.on('disconnecting', async () => {
    await redis.hDel('online_users', userId);
    await userSocketManager.removeSocket(userId, socket.id);

    // 广播下线
    for (const friendId of await getFriendIds(userId)) {
      const friendSocketIds = await userSocketManager.getSocketIds(friendId);
      for (const socketId of friendSocketIds) {
        io.to(socketId).emit('presence:offline', { userId });
      }
    }
  });
});

// 查询在线状态
async function getOnlineStatus(userIds: string[]): Promise<Record<string, boolean>> {
  const results = await redis.hmGet('online_users', userIds);
  return Object.fromEntries(
    userIds.map((id, i) => [id, results[i] !== null])
  );
}
```

---

## 错误处理与速率限制

```typescript
// 全局错误处理
io.on('connection', (socket) => {
  socket.use(([event, ...args], next) => {
    // 所有事件的中间件（类似 Express 的 app.use）
    logger.info({ event, userId: socket.data.user?.sub }, 'Socket event');
    next();
  });

  // 防止消息轰炸：限流
  const messageRateLimiter = new RateLimiter({ tokensPerInterval: 10, interval: 'second' });

  socket.on('message:send', async (data, ack) => {
    if (!await messageRateLimiter.removeTokens(1)) {
      ack?.({ error: 'Too many messages', code: 'RATE_LIMITED' });
      return;
    }
    // 正常处理...
  });
});

// 监控连接数
setInterval(() => {
  const connectionCount = io.engine.clientsCount;
  metrics.gauge('socket.connections', connectionCount);
}, 10_000);
```

---

## 面试追问

**Q: Socket.io 横向扩展时除了 Redis Adapter 还有什么方案？**
A: 粘性会话（Sticky Sessions）+ 不用 Adapter：通过 Nginx/HAProxy IP Hash 将同一用户的请求路由到同一节点，只要用户不换 IP 就不会分到不同节点。缺点：节点宕机时该节点用户全部断线，而且无法跨节点广播。Redis Adapter 是正确方案，Sticky Session 只是权宜之计。另外，Socket.io 还有专门的 `@socket.io/cluster-adapter`（单机多进程）和 `@socket.io/mongo-adapter`（用 MongoDB 替代 Redis）。

**Q: 大量用户在同一个房间，广播性能如何？**
A: `io.to(roomId).emit()` 会遍历房间内所有 Socket 并发送，房间很大时（如 10000 人直播间）会有延迟。优化：分层广播（用 Redis Pub/Sub 扇出到各节点，各节点并行发给本地连接）；限制最大房间大小，超过阈值拆分子房间；只广播"增量"而不是全量状态。

**Q: 如何处理消息不可靠（丢包）？**
A: Socket.io 的 `emit(event, data, ack)` 第三个参数是确认回调，接收方可以确认收到。如果没收到 ack（超时），发送方重发。对于关键消息（如支付通知），在数据库记录"待确认"状态，收到 ack 后标记"已确认"。这类似于消息队列的 "at-least-once delivery"。
