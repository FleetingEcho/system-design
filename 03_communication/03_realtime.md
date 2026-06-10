# 实时通信：WebSocket / 长轮询 / SSE

## TL;DR

- **短轮询（Short Polling）**：客户端定时请求，简单但浪费——只在没有更好方案时用
- **长轮询（Long Polling）**：服务器挂起请求直到有数据，比短轮询高效
- **SSE（Server-Sent Events）**：服务器单向推流，轻量——适合通知、数据更新
- **WebSocket**：全双工双向通信，真正实时——适合聊天、游戏、协同编辑

---

## 四种实现方式对比

### 短轮询（Short Polling）

客户端每隔固定时间主动发请求询问有没有新数据：

```
客户端                    服务器
  |---请求：有新消息吗?-->|
  |<--响应：没有---------|
  (等待 5 秒)
  |---请求：有新消息吗?-->|
  |<--响应：有，这是消息--|
  (等待 5 秒)
  |---请求：有新消息吗?-->|
  |<--响应：没有---------|
```

**缺点：**
- 大量无效请求（大多数情况下没有新消息）
- 数据不是实时的（有 5 秒延迟）
- 对服务器和网络都是浪费

**仅适合：** 对实时性要求极低（分钟级），或者作为其他方案的降级兜底。

---

### 长轮询（Long Polling）

客户端发请求，服务器**挂起请求**直到有新数据，再返回。客户端收到响应后立即发下一个请求：

```
客户端                    服务器
  |---请求：有新消息吗?-->|
  |                       | 挂起，等待消息...（最长等 30 秒）
  |                       | 有新消息了！
  |<--响应：这是消息------|
  |---请求：有新消息吗?-->| （立即发下一个请求）
  |                       | 挂起...
```

**优点：** 比短轮询高效，数据几乎实时（只有建立连接的延迟）
**缺点：** 每次响应后要重新建立连接（HTTP 握手开销）；服务器需要维护大量挂起的请求；并发量高时对服务器有压力

**适合：** 需要实时性但浏览器/环境不支持 WebSocket，或者消息频率较低。

---

### SSE（Server-Sent Events）

基于 HTTP 的单向**服务器推流**，连接一次后服务器持续推送数据，客户端不能发消息：

```
客户端                    服务器
  |---HTTP 请求--------->|
  |<--建立 SSE 连接------|
  |<--data: 消息1--------|
  |<--data: 消息2--------|  （持续推送）
  |<--data: 消息3--------|
```

**响应格式（text/event-stream）：**
```
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"type":"price_update","price":100.5}\n\n
data: {"type":"price_update","price":101.2}\n\n
event: error
data: {"message":"connection unstable"}\n\n
```

**优点：**
- 基于 HTTP，防火墙友好，不需要特殊协议
- 浏览器原生支持，自动重连
- 比 WebSocket 轻量（单向通信场景没必要用全双工）

**缺点：**
- 只能服务器推客户端，客户端不能发消息
- 受 HTTP/1.1 每个域名 6 个连接的限制（HTTP/2 无此限制）

**适合：** 新闻更新、股票行情、通知推送、实时日志——所有"服务器单向推"的场景。

---

### WebSocket

真正的**全双工双向通信**。通过 HTTP Upgrade 从 HTTP 连接升级为 WebSocket 连接，之后双方可以随时互相发送消息：

```
客户端                    服务器
  |---HTTP Upgrade 请求->|   握手
  |<--101 Switching------|
  |<=====WebSocket 连接==>|   （持久连接，双向）
  |--发消息: 你好-------->|
  |<-发消息: 你也好-------|
  |--发消息: 今天天气?---->|
  |<-发消息: 晴天---------|
```

**握手过程：**

```http
客户端发送：
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

服务器响应：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

升级成功后，连接就是一个纯 TCP 连接，帧格式是 WebSocket 协议定义的（非 HTTP）。

**WebSocket 的特点：**
- 全双工：客户端和服务器都可以随时发消息
- 持久连接：不需要重复握手
- 低开销：WebSocket 帧头部只有 2-14 字节（HTTP 头部几百字节）
- 支持文本和二进制消息

**适合：** 聊天系统、多人游戏、协同编辑、实时仪表盘、视频通话信令

---

## 四种方式对比

| 维度 | 短轮询 | 长轮询 | SSE | WebSocket |
|------|--------|--------|-----|-----------|
| 方向 | 单向（客→服） | 单向（服→客） | 单向（服→客） | **双向** |
| 连接 | 每次新建 | 每次响应后重建 | 持久 | **持久** |
| 实时性 | 低（有轮询间隔） | 高 | 高 | **最高** |
| 服务器开销 | 高（大量请求） | 中 | 低 | **低** |
| 协议 | HTTP | HTTP | HTTP | WebSocket |
| 浏览器支持 | 全部 | 全部 | 全部（现代） | 全部（现代） |
| 适用场景 | 不推荐 | 简单通知 | 服务器推流 | 聊天/游戏/协同 |

---

## WebSocket 服务的规模化

单台 WebSocket 服务器维护的连接数有上限（受内存和文件描述符限制，通常 10-100 万连接/台）。当用户量大时，需要多台服务器，带来新问题：

### 问题：连接在不同服务器上

```
用户 A 连接在 Server 1
用户 B 连接在 Server 2

用户 A 发消息给用户 B：
Server 1 收到消息，但用户 B 在 Server 2 上
怎么把消息发给 Server 2？
```

### 解决方案：Pub/Sub 总线

用 Redis Pub/Sub 或消息队列作为服务器间的消息总线：

```
用户A（连Server1）发消息给用户B
  ↓
Server1 把消息发布到 Redis Channel "user_B_messages"
  ↓
Server2 订阅了 "user_B_messages"，收到消息
  ↓
Server2 通过用户B的 WebSocket 连接发送消息
```

```typescript
// Server 1（用户A连接的服务器）
ws.on('message', async (data) => {
  const { to, content } = JSON.parse(data);
  // 不知道目标用户在哪台服务器，发布到 Redis
  await redisPublisher.publish(`user:${to}:messages`, JSON.stringify({
    from: currentUserId,
    content
  }));
});

// Server 2（用户B连接的服务器）
// 启动时订阅目标用户的频道
await redisSubscriber.subscribe(`user:${userId}:messages`);
redisSubscriber.on('message', (channel, message) => {
  // 找到对应的 WebSocket 连接，推送消息
  const ws = connectedUsers.get(userId);
  if (ws) ws.send(message);
});
```

### 连接状态管理

用户与服务器的映射关系需要存储：

```
Redis Hash:
user_connections: {
  "user:1001": "server-3"   // 用户 1001 连接在 server-3 上
  "user:1002": "server-1"
}
```

用户断开时删除记录，用户重连时更新。

---

## 心跳（Heartbeat）机制

网络设备（NAT、防火墙）可能会在连接空闲一段时间后断开 TCP 连接，需要心跳保活：

```typescript
// 客户端心跳
const heartbeatInterval = setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ type: 'ping' }));
  }
}, 30000);  // 每 30 秒发一次

// 服务端响应
ws.on('message', (data) => {
  const message = JSON.parse(data);
  if (message.type === 'ping') {
    ws.send(JSON.stringify({ type: 'pong' }));
    return;
  }
  // 处理业务消息...
});

// 服务端检测死连接
ws.on('pong', () => {
  ws.isAlive = true;
});

// 定时检查，超时没响应的连接直接关闭
const interval = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) return ws.terminate();
    ws.isAlive = false;
    ws.ping();
  });
}, 60000);
```

---

## 与 Node.js/TS 生态的类比

```typescript
// 使用 ws 库（纯 WebSocket）
import WebSocket, { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  const userId = parseToken(req.headers.authorization);

  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());
    // 处理消息...
    broadcast(wss, message);  // 广播给所有连接
  });

  ws.on('close', () => {
    // 清理连接状态
  });
});

// 使用 Socket.io（更高层抽象，内置房间、断线重连等）
import { Server } from 'socket.io';
const io = new Server(httpServer);

io.on('connection', (socket) => {
  socket.join(`room:${roomId}`);  // 加入房间

  socket.on('chat-message', (msg) => {
    io.to(`room:${roomId}`).emit('chat-message', msg);  // 发给房间内所有人
  });
});
```

---

## 常见陷阱

1. **WebSocket 连接没有做认证**：HTTP Upgrade 请求时验证 Token，否则任何人都能建立连接
2. **没有心跳机制**：长时间空闲的连接被网络设备悄悄断开，客户端以为还连着，消息发不出去
3. **服务器内存泄漏**：连接断开时没有清理对应的事件监听器和状态，内存持续增长
4. **广播风暴**：向所有在线用户广播时，服务器逐个发送，10 万用户 = 10 万次发送，CPU 打爆
5. **忽略 WebSocket 的连接数上限**：一台服务器不能无限维持连接，需要提前规划水平扩展方案

---

## 面试常见问答

### 简单

**Q：WebSocket 和 HTTP 有什么关系？**

A：WebSocket 连接建立时使用 HTTP 协议进行握手（HTTP Upgrade），服务器返回 101 Switching Protocols 后，连接从 HTTP 升级为 WebSocket 协议。之后双方的通信就不再使用 HTTP 格式，而是 WebSocket 的帧格式，运行在 TCP 之上。所以 WebSocket 是借用 HTTP 握手建立连接，实际通信不是 HTTP。好处：可以复用 80/443 端口，穿越防火墙。

---

**Q：什么时候用 SSE，什么时候用 WebSocket？**

A：核心判断：**通信方向**。如果只需要服务器推数据给客户端（通知、实时行情、日志流），SSE 更简单，基于 HTTP，不需要特殊服务器支持，浏览器自动处理重连。如果需要客户端也主动发消息（聊天、游戏、协同编辑），必须用 WebSocket 的全双工特性。如果用了 HTTP/2，SSE 也没有每域名连接数限制，大多数单向推送场景 SSE 够用。

---

### 中等

**Q：设计一个聊天系统，如何处理用户离线时发来的消息？**

A：离线消息分两部分处理：

**存储**：消息发出时，先持久化到数据库（Cassandra 按用户 ID + 时间戳分区），无论接收方是否在线都先写库。

**在线投递**：如果接收方在线，通过 WebSocket 连接实时推送，同时标记消息为"已投递"。

**离线投递**：接收方不在线时，消息只存库，不推送。接收方上线后（WebSocket 建立连接时）：
1. 查询数据库，拉取自上次在线后的所有未读消息
2. 批量推送给客户端
3. 客户端确认收到后，标记为"已读"

**推拉结合**：大量离线消息时，先推一个"你有 N 条未读消息"的通知，客户端再主动分页拉取，避免一次性推送大量数据。

---

### 难

**Q：设计一个支持 1000 万在线用户的实时消息系统，WebSocket 层如何扩展？**

A：1000 万连接不可能在一台服务器上，需要多层扩展设计：

**WebSocket 网关层（Stateful）：**
专门负责维护 WebSocket 连接，每台机器维持 50-100 万连接（内存充足时）：
- 20 台机器可以支撑 1000 万连接
- 每台机器只维护连接，不处理业务逻辑
- 通过 Load Balancer（L4，TCP 层）分配连接

**路由层（Redis）：**
记录每个用户连接在哪台 WebSocket 机器上：
```
user_router: { userId → machineId }
```

**消息处理层（无状态）：**
接收消息后，查 Redis 找到目标用户所在的机器，通过内网发送。如果目标不在线，存消息队列（Kafka）。

**消息队列（Kafka）：**
- WebSocket 机器发出的消息写入 Kafka
- 消息处理服务消费 Kafka，做持久化、离线消息队列
- 如需要，做消息的扩散（群聊：一条消息 → 扩散到所有成员）

**连接热重启（Zero-downtime）：**
WebSocket 服务器升级时，客户端自动重连到其他实例，路由表更新。客户端需要实现指数退避重连（避免重启后所有客户端同时重连）。

---

## 关联文档

- [../06_case_studies/06_chat_system.md](../06_case_studies/06_chat_system.md) — 完整聊天系统设计案例
- [02_async.md](02_async.md) — 消息队列在实时系统中的角色
- [../04_distributed/06_distributed_lock.md](../04_distributed/06_distributed_lock.md) — 实时系统中的并发控制
