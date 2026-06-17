# GraphQL 服务端实现

> Schema 设计、Resolver 实现、DataLoader（N+1）、订阅（Subscription）、鉴权、性能优化。
> 面试场景：在 Node.js 项目中用 GraphQL 替换 REST，或 BFF 层聚合多个下游服务。

---

## GraphQL vs REST 选型

```
选 GraphQL 的场景：
  ✓ 前端有复杂的数据聚合需求（一次请求多个资源）
  ✓ 多端（Web/Mobile）需要不同数据形状
  ✓ BFF 层聚合多个微服务
  ✓ 需要实时订阅（Subscription）

坚守 REST 的场景：
  ✓ 简单 CRUD，关系不复杂
  ✓ 公共 API（GraphQL 的 introspection 泄露 schema）
  ✓ 文件上传（GraphQL 原生不支持）
  ✓ 缓存（HTTP GET 缓存 vs GraphQL POST 无法 CDN 缓存）
```

---

## 项目结构

```
src/
├── graphql/
│   ├── schema/
│   │   ├── user.graphql
│   │   ├── post.graphql
│   │   └── index.ts          # 合并所有 schema
│   ├── resolvers/
│   │   ├── user.resolver.ts
│   │   ├── post.resolver.ts
│   │   └── index.ts
│   ├── dataloaders/
│   │   ├── user.loader.ts
│   │   └── index.ts
│   ├── context.ts            # 请求上下文（user + loaders）
│   └── server.ts             # Apollo Server 配置
```

---

## Schema 设计

```graphql
# src/graphql/schema/user.graphql

type User {
  id: ID!
  username: String!
  email: String!
  createdAt: String!
  posts: [Post!]!     # 关联字段：触发 N+1 问题
  postCount: Int!
}

type Query {
  me: User
  user(id: ID!): User
  users(limit: Int = 20, cursor: String): UserConnection!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
}
type UserEdge {
  node: User!
  cursor: String!
}
type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

type Mutation {
  updateProfile(input: UpdateProfileInput!): User!
  changePassword(currentPassword: String!, newPassword: String!): Boolean!
}

input UpdateProfileInput {
  username: String
  bio: String
}

# src/graphql/schema/post.graphql

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!        # 关联字段：触发 N+1 问题
  tags: [String!]!
  likeCount: Int!
  viewCount: Int!
  createdAt: String!
}

type Query {
  post(id: ID!): Post
  posts(userId: ID, limit: Int = 20, cursor: String): PostConnection!
  feed(limit: Int = 20, cursor: String): PostConnection!
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
  deletePost(id: ID!): Boolean!
}

input CreatePostInput {
  title: String!
  content: String!
  tags: [String!]
}

type Subscription {
  postLiked(postId: ID!): Post!
  newComment(postId: ID!): Comment!
}
```

---

## DataLoader：解决 N+1

```typescript
// src/graphql/dataloaders/user.loader.ts
// N+1 问题：查询 100 篇 Post，每篇 Post 再查 author → 101 次 DB 查询
// DataLoader 批量合并：1 次查询所有用户

import DataLoader from 'dataloader';
import { PrismaClient } from '@prisma/client';

// DataLoader<Key, Value>
// batchFn：接收一批 key，返回对应 value 数组（顺序必须与 keys 一致）

export function createUserLoader(prisma: PrismaClient) {
  return new DataLoader<string, User | null>(
    async (userIds) => {
      // 一次 DB 查询批量获取所有用户
      const users = await prisma.user.findMany({
        where: { id: { in: [...userIds] } },
      });

      // 构建 id → user 映射（批量结果可能乱序）
      const userMap = new Map(users.map(u => [u.id, u]));

      // 按 keys 顺序返回（DataLoader 要求顺序对应）
      return userIds.map(id => userMap.get(id) ?? null);
    },
    {
      // 同一请求内同一 key 只查一次（默认开启）
      cache: true,
      // 最大批量大小（防止 IN 查询过大）
      maxBatchSize: 100,
    }
  );
}

// 同样的模式用于 posts by userId
export function createPostsByUserLoader(prisma: PrismaClient) {
  return new DataLoader<string, Post[]>(async (userIds) => {
    const posts = await prisma.post.findMany({
      where: { authorId: { in: [...userIds] } },
      orderBy: { createdAt: 'desc' },
    });

    const postsByUser = new Map<string, Post[]>();
    for (const post of posts) {
      const arr = postsByUser.get(post.authorId) ?? [];
      arr.push(post);
      postsByUser.set(post.authorId, arr);
    }

    return userIds.map(id => postsByUser.get(id) ?? []);
  });
}
```

---

## Context（请求上下文）

```typescript
// src/graphql/context.ts
// 每个请求创建新的 DataLoader 实例（DataLoader 缓存是请求级别的）

import { ApolloServer } from '@apollo/server';

export interface GraphQLContext {
  user: AuthUser | null;   // 当前登录用户（null = 未登录）
  prisma: PrismaClient;
  loaders: {
    user: ReturnType<typeof createUserLoader>;
    postsByUser: ReturnType<typeof createPostsByUserLoader>;
  };
}

export async function createContext({ req }: { req: Request }): Promise<GraphQLContext> {
  // 解析鉴权 token
  let user: AuthUser | null = null;
  const authHeader = req.headers.get('authorization');
  if (authHeader?.startsWith('Bearer ')) {
    const token = authHeader.slice(7);
    try {
      user = await verifyAccessToken(token);
    } catch {
      // token 无效：user = null，后续 resolver 自行决定是否报错
    }
  }

  return {
    user,
    prisma,
    loaders: {
      user: createUserLoader(prisma),
      postsByUser: createPostsByUserLoader(prisma),
    },
  };
}
```

---

## Resolver 实现

```typescript
// src/graphql/resolvers/user.resolver.ts

export const userResolvers = {
  Query: {
    me: (_: unknown, __: unknown, ctx: GraphQLContext) => {
      if (!ctx.user) throw new GraphQLError('请先登录', { extensions: { code: 'UNAUTHENTICATED' } });
      return ctx.prisma.user.findUnique({ where: { id: ctx.user.id } });
    },

    user: (_: unknown, { id }: { id: string }, ctx: GraphQLContext) => {
      return ctx.loaders.user.load(id);  // DataLoader 自动批量
    },

    users: async (_: unknown, { limit, cursor }: { limit: number; cursor?: string }, ctx: GraphQLContext) => {
      const take = Math.min(limit + 1, 100);  // 防止过大查询
      const users = await ctx.prisma.user.findMany({
        take,
        ...(cursor ? { skip: 1, cursor: { id: cursor } } : {}),
        orderBy: { createdAt: 'desc' },
      });

      const hasNextPage = users.length > limit;
      const edges = users.slice(0, limit).map(user => ({
        node: user,
        cursor: user.id,
      }));

      return {
        edges,
        pageInfo: {
          hasNextPage,
          endCursor: edges[edges.length - 1]?.cursor ?? null,
        },
      };
    },
  },

  Mutation: {
    updateProfile: async (
      _: unknown,
      { input }: { input: { username?: string; bio?: string } },
      ctx: GraphQLContext
    ) => {
      if (!ctx.user) throw new GraphQLError('请先登录', { extensions: { code: 'UNAUTHENTICATED' } });

      return ctx.prisma.user.update({
        where: { id: ctx.user.id },
        data: input,
      });
    },
  },

  // 关联字段 Resolver（User.posts 触发时调用）
  User: {
    posts: (parent: User, _: unknown, ctx: GraphQLContext) => {
      // DataLoader：同一请求内多个 User 的 posts 会被批量查询
      return ctx.loaders.postsByUser.load(parent.id);
    },
    postCount: async (parent: User, _: unknown, ctx: GraphQLContext) => {
      // 简单聚合用 DataLoader 或直接查
      return ctx.prisma.post.count({ where: { authorId: parent.id } });
    },
  },
};

// src/graphql/resolvers/post.resolver.ts

export const postResolvers = {
  Query: {
    post: (_: unknown, { id }: { id: string }, ctx: GraphQLContext) => {
      return ctx.prisma.post.findUnique({ where: { id } });
    },
  },

  Mutation: {
    createPost: async (
      _: unknown,
      { input }: { input: { title: string; content: string; tags?: string[] } },
      ctx: GraphQLContext
    ) => {
      if (!ctx.user) throw new GraphQLError('请先登录', { extensions: { code: 'UNAUTHENTICATED' } });

      const post = await ctx.prisma.post.create({
        data: { ...input, authorId: ctx.user.id },
      });

      // 发布 Subscription 事件
      pubsub.publish('POST_CREATED', { postCreated: post });
      return post;
    },

    deletePost: async (_: unknown, { id }: { id: string }, ctx: GraphQLContext) => {
      if (!ctx.user) throw new GraphQLError('请先登录', { extensions: { code: 'UNAUTHENTICATED' } });

      const post = await ctx.prisma.post.findUnique({ where: { id } });
      if (!post) throw new GraphQLError('帖子不存在', { extensions: { code: 'NOT_FOUND' } });
      if (post.authorId !== ctx.user.id) throw new GraphQLError('无权限', { extensions: { code: 'FORBIDDEN' } });

      await ctx.prisma.post.delete({ where: { id } });
      return true;
    },
  },

  // 关联字段 Resolver（Post.author 触发时调用）
  Post: {
    author: (parent: Post, _: unknown, ctx: GraphQLContext) => {
      return ctx.loaders.user.load(parent.authorId);  // DataLoader 批量
    },
  },
};
```

---

## Subscription（实时订阅）

```typescript
// src/graphql/subscriptions.ts
// 生产环境用 graphql-subscriptions + Redis PubSub（多节点）

import { PubSub } from 'graphql-subscriptions';
import { RedisPubSub } from 'graphql-redis-subscriptions';

// 单节点开发用内存 PubSub
export const pubsub = new PubSub();

// 多节点生产用 Redis PubSub
// export const pubsub = new RedisPubSub({
//   publisher: new Redis(process.env.REDIS_URL),
//   subscriber: new Redis(process.env.REDIS_URL),
// });

export const subscriptionResolvers = {
  Subscription: {
    postLiked: {
      subscribe: (_: unknown, { postId }: { postId: string }) => {
        return pubsub.asyncIterator(`POST_LIKED_${postId}`);
      },
      resolve: (payload: { postLiked: Post }) => payload.postLiked,
    },

    newComment: {
      subscribe: (_: unknown, { postId }: { postId: string }, ctx: GraphQLContext) => {
        // 需要鉴权的订阅
        if (!ctx.user) throw new GraphQLError('请先登录');
        return pubsub.asyncIterator(`NEW_COMMENT_${postId}`);
      },
      resolve: (payload: { newComment: Comment }) => payload.newComment,
    },
  },
};

// 在 Mutation 中触发订阅：
// pubsub.publish(`POST_LIKED_${postId}`, { postLiked: updatedPost });
```

---

## Apollo Server 配置

```typescript
// src/graphql/server.ts

import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';
import depthLimit from 'graphql-depth-limit';

const schema = makeExecutableSchema({
  typeDefs: [userTypeDefs, postTypeDefs],
  resolvers: [userResolvers, postResolvers, subscriptionResolvers],
});

const server = new ApolloServer<GraphQLContext>({
  schema,
  // 安全：限制查询深度（防止递归嵌套攻击）
  validationRules: [depthLimit(7)],
  // 生产环境关闭 introspection（防止 schema 泄露）
  introspection: process.env.NODE_ENV !== 'production',
  // 格式化错误（不暴露内部细节）
  formatError: (formattedError) => {
    if (process.env.NODE_ENV === 'production') {
      // 只暴露已知的业务错误，其余统一返回 Internal Error
      const code = formattedError.extensions?.code;
      if (['UNAUTHENTICATED', 'FORBIDDEN', 'NOT_FOUND', 'BAD_USER_INPUT'].includes(code as string)) {
        return formattedError;
      }
      return { message: 'Internal server error', extensions: { code: 'INTERNAL_SERVER_ERROR' } };
    }
    return formattedError;
  },
});

// HTTP + WebSocket 双端点
export async function startGraphQLServer(app: Express, httpServer: HTTPServer) {
  // WebSocket（用于 Subscription）
  const wsServer = new WebSocketServer({ server: httpServer, path: '/graphql' });
  const wsDispose = useServer(
    {
      schema,
      context: async (ctx) => createContext({ req: ctx.extra.request as Request }),
    },
    wsServer
  );

  await server.start();

  app.use('/graphql', cors(), json(), expressMiddleware(server, { context: createContext }));

  return { wsDispose };
}
```

---

## 鉴权模式

```typescript
// 方式一：resolver 内部检查（细粒度，适合字段级权限）
const resolver = async (_: unknown, args: unknown, ctx: GraphQLContext) => {
  if (!ctx.user) throw new GraphQLError('UNAUTHENTICATED');
  if (!ctx.user.roles.includes('admin')) throw new GraphQLError('FORBIDDEN');
  // ...
};

// 方式二：指令（Directive）— 声明式，在 schema 层控制
// @auth 指令
const authDirective = (directiveName: string) => ({
  typeDefs: `directive @${directiveName}(requires: Role = USER) on FIELD_DEFINITION`,
  transformer: (schema: GraphQLSchema) =>
    mapSchema(schema, {
      [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
        const requires = getDirective(schema, fieldConfig, directiveName)?.[0]?.['requires'];
        if (requires) {
          const { resolve = defaultFieldResolver } = fieldConfig;
          fieldConfig.resolve = async (source, args, ctx: GraphQLContext, info) => {
            if (!ctx.user) throw new GraphQLError('UNAUTHENTICATED');
            if (requires === 'ADMIN' && !ctx.user.roles.includes('admin')) {
              throw new GraphQLError('FORBIDDEN');
            }
            return resolve(source, args, ctx, info);
          };
        }
        return fieldConfig;
      },
    }),
});

// 在 schema 中使用：
// deletePost(id: ID!): Boolean! @auth(requires: ADMIN)
```

---

## 面试追问

**Q: DataLoader 的批量窗口是怎么工作的？**
A: DataLoader 利用事件循环的 microtask 机制。同一个事件循环 tick 内所有调用 `loader.load(key)` 的请求，会被收集到一个批次，等当前 tick 结束（进入下一个 tick）时一次性执行 `batchFn`。这就是为什么 DataLoader 只需要在请求级别创建一次（不是全局缓存），因为跨请求的批量没有意义——只有同一请求的 Resolver 并发执行时才会触发批量。

**Q: GraphQL N+1 是什么？DataLoader 为什么能解决？**
A: 查询 10 篇 Post，`Post.author` 字段 Resolver 每次调用 `prisma.user.findUnique`，触发 10 次 DB 查询（1 次查 posts + 10 次查 user = N+1）。DataLoader 拦截所有 `load(userId)` 调用，收集到 `[id1, id2, ..., id10]` 后一次批量查询 `WHERE id IN (id1...id10)`，10 次 → 1 次。DataLoader 还自带请求级缓存：同一请求内同一 userId 只查一次。

**Q: Subscription 在多节点部署时为什么需要 Redis PubSub？**
A: 内存 PubSub 只在单进程内通知。A 节点上的 WebSocket 连接，B 节点上发布的事件无法通知到 A。Redis PubSub（Pub/Sub 频道）是跨进程的消息总线：任何节点 publish 到 Redis，所有订阅的节点都能收到，再通过自己的 WebSocket 推给客户端。生产替代方案：Kafka（更可靠，支持消息重放）或 NATS（低延迟）。
