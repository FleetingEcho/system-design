# gRPC 与微服务间通信

> gRPC TypeScript 实现、服务发现、Saga 编排、服务网格。
> 面试场景：将单体拆分为微服务，或设计多服务间的通信协议。

---

## gRPC vs REST vs GraphQL

```
gRPC（推荐用于内部微服务通信）：
  ✓ 二进制 Protobuf（序列化比 JSON 快 5-10x，体积小 3-5x）
  ✓ HTTP/2 多路复用（同一连接并发多个请求）
  ✓ 强类型 schema（.proto 文件，跨语言代码生成）
  ✓ 原生流式 RPC（Server/Client/Bidirectional Streaming）
  ✗ 浏览器不支持（需要 gRPC-Web 或 Envoy 代理）
  ✗ 可读性差（不能直接 curl 调试）

REST（对外公共 API）：
  ✓ 浏览器直接调用，curl 友好
  ✗ JSON 性能开销

GraphQL（BFF 层，多端数据聚合）：
  ✓ 前端按需取数，减少 over-fetching
  ✗ 内部服务用太重（schema introspection 开销）

原则：对外 REST/GraphQL，内部 gRPC。
```

---

## Protobuf Schema 设计

```protobuf
// proto/user.proto
syntax = "proto3";
package user;

option java_package = "com.example.user";  // 如有 Java 服务

// 服务定义
service UserService {
  // Unary RPC：请求-响应
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc ValidateToken(ValidateTokenRequest) returns (ValidateTokenResponse);

  // Server Streaming：服务端流（如：推送用户活动流）
  rpc WatchUserEvents(WatchUserEventsRequest) returns (stream UserEvent);

  // Client Streaming：客户端流（如：批量上传）
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  string id = 1;
  string username = 2;
  string email = 3;
  int64 created_at = 4;  // Unix timestamp ms
  bool found = 5;
}

message CreateUserRequest {
  string username = 1;
  string email = 2;
  string password_hash = 3;
}

message CreateUserResponse {
  string id = 1;
  bool success = 2;
  string error = 3;  // 错误信息（success = false 时）
}

message ValidateTokenRequest {
  string token = 1;
}

message ValidateTokenResponse {
  bool valid = 1;
  string user_id = 2;
  repeated string roles = 3;
}

// proto/order.proto
syntax = "proto3";
package order;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc CancelOrder(CancelOrderRequest) returns (CancelOrderResponse);
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
  string idempotency_key = 3;  // 幂等键
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
}

message CreateOrderResponse {
  string order_id = 1;
  string status = 2;   // PENDING | CONFIRMED | FAILED
}
```

---

## TypeScript gRPC 服务端实现

```typescript
// 代码生成（package.json scripts）：
// "@grpc/grpc-js": "^1.9.0"
// "@grpc/proto-loader": "^0.7.0"
// "ts-proto": "^1.165.0"  （更好的 TypeScript 类型）
//
// 生成命令：
// protoc --plugin=./node_modules/.bin/protoc-gen-ts_proto \
//   --ts_proto_out=./src/generated \
//   --ts_proto_opt=outputServices=grpc-js,env=node,esModuleInterop=true \
//   proto/user.proto

// src/services/user-grpc-server.ts
import * as grpc from '@grpc/grpc-js';
import { UserServiceImplementation, GetUserRequest, GetUserResponse } from '../generated/user';

// 实现 UserService
const userServiceImpl: UserServiceImplementation = {
  async getUser(call: grpc.ServerUnaryCall<GetUserRequest, GetUserResponse>, callback) {
    try {
      const { userId } = call.request;
      const user = await prisma.user.findUnique({ where: { id: userId } });

      if (!user) {
        callback(null, { id: '', username: '', email: '', createdAt: 0n, found: false });
        return;
      }

      callback(null, {
        id: user.id,
        username: user.username,
        email: user.email,
        createdAt: BigInt(user.createdAt.getTime()),
        found: true,
      });
    } catch (err) {
      callback({
        code: grpc.status.INTERNAL,
        message: (err as Error).message,
      });
    }
  },

  async createUser(call, callback) {
    const { username, email, passwordHash } = call.request;
    try {
      const user = await prisma.user.create({
        data: { username, email, passwordHash },
      });
      callback(null, { id: user.id, success: true, error: '' });
    } catch (err) {
      if ((err as any).code === 'P2002') {  // Prisma 唯一约束
        callback(null, { id: '', success: false, error: '邮箱已注册' });
      } else {
        callback({ code: grpc.status.INTERNAL, message: (err as Error).message });
      }
    }
  },

  async validateToken(call, callback) {
    try {
      const payload = await verifyAccessToken(call.request.token);
      callback(null, { valid: true, userId: payload.sub, roles: payload.roles });
    } catch {
      callback(null, { valid: false, userId: '', roles: [] });
    }
  },

  // Server Streaming：推送用户事件
  watchUserEvents(call: grpc.ServerWritableStream<WatchUserEventsRequest, UserEvent>) {
    const { userId } = call.request;

    const subscription = eventBus.subscribe(`user:${userId}`, (event: UserEvent) => {
      if (call.writable) {
        call.write(event);
      } else {
        subscription.unsubscribe();  // 客户端断开后取消订阅
      }
    });

    call.on('cancelled', () => subscription.unsubscribe());
    call.on('error', () => subscription.unsubscribe());
  },
};

// 启动 gRPC 服务器
export function startUserGrpcServer(port = 50051): grpc.Server {
  const server = new grpc.Server({
    'grpc.max_receive_message_length': 10 * 1024 * 1024,   // 10MB
    'grpc.max_send_message_length': 10 * 1024 * 1024,
    'grpc.keepalive_time_ms': 10_000,
    'grpc.keepalive_timeout_ms': 5_000,
  });

  server.addService(UserServiceService, userServiceImpl);

  server.bindAsync(
    `0.0.0.0:${port}`,
    grpc.ServerCredentials.createInsecure(),  // 生产用 createSsl()
    (err, port) => {
      if (err) throw err;
      console.log(`gRPC UserService listening on port ${port}`);
    }
  );

  return server;
}
```

---

## gRPC 客户端（带连接池和重试）

```typescript
// src/clients/user-service.client.ts

import * as grpc from '@grpc/grpc-js';
import { UserServiceClient } from '../generated/user';

// 单例客户端（复用 HTTP/2 连接）
let _userClient: UserServiceClient | null = null;

export function getUserServiceClient(): UserServiceClient {
  if (_userClient) return _userClient;

  _userClient = new UserServiceClient(
    process.env.USER_SERVICE_URL ?? 'user-service:50051',
    grpc.credentials.createInsecure(),
    {
      // 重试策略（gRPC 内置）
      'grpc.service_config': JSON.stringify({
        methodConfig: [{
          name: [{ service: 'user.UserService' }],
          retryPolicy: {
            maxAttempts: 3,
            initialBackoff: '0.1s',
            maxBackoff: '1s',
            backoffMultiplier: 2,
            retryableStatusCodes: ['UNAVAILABLE', 'DEADLINE_EXCEEDED'],
          },
          timeout: '5s',  // 每次 RPC 调用超时
        }],
      }),
    }
  );

  return _userClient;
}

// Promisify wrapper（将 callback 风格转为 async/await）
export function getUser(userId: string): Promise<GetUserResponse> {
  return new Promise((resolve, reject) => {
    getUserServiceClient().getUser(
      { userId },
      new grpc.Metadata(),
      (err, response) => {
        if (err) reject(err);
        else resolve(response!);
      }
    );
  });
}

// 带 deadline 的调用
export function getUserWithDeadline(userId: string, deadlineMs = 3000): Promise<GetUserResponse> {
  return new Promise((resolve, reject) => {
    const deadline = new Date(Date.now() + deadlineMs);
    getUserServiceClient().getUser(
      { userId },
      { deadline },  // 截止时间
      (err, response) => {
        if (err) {
          if (err.code === grpc.status.DEADLINE_EXCEEDED) {
            reject(new Error(`UserService timeout after ${deadlineMs}ms`));
          } else {
            reject(err);
          }
        } else {
          resolve(response!);
        }
      }
    );
  });
}
```

---

## Saga 模式：跨服务分布式事务

```typescript
// 场景：下单流程涉及多个服务
// 1. OrderService 创建订单
// 2. InventoryService 扣减库存
// 3. PaymentService 扣款
// 4. NotificationService 发通知
//
// 任何步骤失败，需要补偿（Compensate）已完成的步骤

// src/sagas/create-order.saga.ts

interface SagaStep<T> {
  name: string;
  execute: (ctx: T) => Promise<T>;
  compensate: (ctx: T) => Promise<void>;
}

class Saga<T> {
  private steps: SagaStep<T>[] = [];
  private completedSteps: SagaStep<T>[] = [];

  addStep(step: SagaStep<T>): this {
    this.steps.push(step);
    return this;
  }

  async execute(context: T): Promise<T> {
    let ctx = context;

    for (const step of this.steps) {
      try {
        console.log(`[Saga] Executing step: ${step.name}`);
        ctx = await step.execute(ctx);
        this.completedSteps.push(step);
      } catch (err) {
        console.error(`[Saga] Step failed: ${step.name}`, err);
        // 逆序补偿已完成的步骤
        await this.compensate(ctx);
        throw new SagaFailedError(`Saga failed at step "${step.name}"`, { cause: err });
      }
    }

    return ctx;
  }

  private async compensate(ctx: T) {
    // 逆序执行补偿（LIFO）
    const toCompensate = [...this.completedSteps].reverse();
    for (const step of toCompensate) {
      try {
        console.log(`[Saga] Compensating: ${step.name}`);
        await step.compensate(ctx);
      } catch (compErr) {
        // 补偿失败：记录告警，人工介入（不能再抛出，否则掩盖原始错误）
        console.error(`[Saga] Compensation failed for step: ${step.name}`, compErr);
        await alertingService.sendCritical(`Saga compensation failed: ${step.name}`);
      }
    }
  }
}

// 创建订单 Saga
interface OrderSagaContext {
  userId: string;
  items: OrderItem[];
  idempotencyKey: string;
  orderId?: string;
  reservationId?: string;
  paymentId?: string;
}

const createOrderSaga = new Saga<OrderSagaContext>()
  .addStep({
    name: 'CreateOrder',
    execute: async (ctx) => {
      const { orderId } = await orderService.createOrder({
        userId: ctx.userId,
        items: ctx.items,
        idempotencyKey: ctx.idempotencyKey,
      });
      return { ...ctx, orderId };
    },
    compensate: async (ctx) => {
      if (ctx.orderId) await orderService.cancelOrder({ orderId: ctx.orderId });
    },
  })
  .addStep({
    name: 'ReserveInventory',
    execute: async (ctx) => {
      const { reservationId } = await inventoryService.reserve({
        orderId: ctx.orderId!,
        items: ctx.items,
      });
      return { ...ctx, reservationId };
    },
    compensate: async (ctx) => {
      if (ctx.reservationId) await inventoryService.release({ reservationId: ctx.reservationId! });
    },
  })
  .addStep({
    name: 'ProcessPayment',
    execute: async (ctx) => {
      const { paymentId } = await paymentService.charge({
        userId: ctx.userId,
        orderId: ctx.orderId!,
        idempotencyKey: ctx.idempotencyKey,
      });
      return { ...ctx, paymentId };
    },
    compensate: async (ctx) => {
      if (ctx.paymentId) await paymentService.refund({ paymentId: ctx.paymentId! });
    },
  })
  .addStep({
    name: 'SendNotification',
    execute: async (ctx) => {
      await notificationService.sendOrderConfirmation({ userId: ctx.userId, orderId: ctx.orderId! });
      return ctx;
    },
    compensate: async () => {
      // 通知无法撤回，记录日志即可
    },
  });

// 使用
export async function handleCreateOrder(req: Request, res: Response) {
  const { userId, items } = req.body;
  const idempotencyKey = req.headers['idempotency-key'] as string;

  try {
    const result = await createOrderSaga.execute({ userId, items, idempotencyKey });
    res.json({ orderId: result.orderId, status: 'confirmed' });
  } catch (err) {
    if (err instanceof SagaFailedError) {
      res.status(422).json({ error: err.message });
    } else {
      throw err;
    }
  }
}
```

---

## 服务发现

```typescript
// 生产环境：Kubernetes 内置 DNS 服务发现
// service name: user-service → DNS: user-service.default.svc.cluster.local:50051

// 动态服务发现（非 K8s 环境，用 Consul）：
import Consul from 'consul';

const consul = new Consul({ host: process.env.CONSUL_HOST });

async function discoverService(serviceName: string): Promise<string> {
  const services = await consul.health.service({ service: serviceName, passing: true });
  if (!services.length) throw new Error(`No healthy instances of ${serviceName}`);

  // 随机负载均衡
  const instance = services[Math.floor(Math.random() * services.length)];
  return `${instance.Service.Address}:${instance.Service.Port}`;
}

// 客户端侧负载均衡（gRPC 内置）
// grpc.credentials 可结合 channel 配置 DNS 解析 + 负载均衡
```

---

## 面试追问

**Q: gRPC 和 REST 性能差距多大？什么时候值得引入？**
A: 高频内部调用时差距显著：Protobuf 比 JSON 序列化快 5-10x，体积小 3-5x；HTTP/2 多路复用减少连接开销。但引入 gRPC 的成本是：proto 文件管理（需要 schema registry）、调试工具（不能 curl）、浏览器不支持（需 gRPC-Web 或 Envoy）。建议：内部服务调用 QPS > 1000 或 P99 延迟敏感时引入；否则 REST 足够，别过早优化。

**Q: Saga 补偿失败怎么处理？**
A: 补偿失败是最麻烦的情况（已经在补偿错误了，补偿又出错）。工程上的解法：① 补偿操作设计为幂等（可以安全重试）；② 失败写入"补偿待处理"表，有独立的后台任务重试；③ 告警通知人工介入；④ 永远不能在补偿过程中抛出异常（会掩盖原始错误），改为记录并继续补偿其他步骤。这就是为什么 Saga 不能完全自动化——最终一致性需要有人工兜底。

**Q: Saga 编排式 vs 协调式的区别？**
A: 编排式（Orchestration）：中央 Saga 协调者按顺序调用各服务（本篇实现方式），逻辑集中，易于追踪；耦合在协调者上。协调式（Choreography）：每个服务监听事件自行决定下一步（如：OrderCreated → InventoryService 监听后扣减库存），无中心协调，去耦；但分布式逻辑难以追踪和调试。大多数团队选编排式，因为可观测性更好（一个 Saga 实例可以记录完整状态）。
