# 全栈鉴权（NextAuth.js v5 / Auth.js）

> Next.js App Router 中的完整鉴权：Auth.js v5（NextAuth）配置、Session 获取、Protected Routes、RBAC。
> 与 `backend/nodejs/09_auth_implementation.md` 互补：那篇讲纯 Node.js JWT 实现，本篇讲 Next.js 生态集成。

---

## 方案对比

```
自建 JWT（backend/nodejs/09）：
  + 完全控制（Token 内容、过期策略、存储方式）
  + 适合：API 服务 + 前端分离（不用 Next.js server-side session）
  - 需要手写 Refresh Token 轮换、OAuth 集成

Auth.js v5（NextAuth）：
  + 开箱即用：20+ OAuth providers（GitHub/Google/Discord...）
  + 与 Next.js App Router 深度集成（Server Component 直接 auth()）
  + 内置 CSRF 保护
  - 定制性稍差（被 Auth.js 抽象层限制）
  - 适合：Next.js 全栈应用，快速接入多个 OAuth providers

Clerk：
  + 最简单，UI 组件开箱即用
  - 付费且厂商锁定
```

---

## Auth.js v5 配置

```typescript
// auth.ts（根目录）
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';
import Google from 'next-auth/providers/google';
import Credentials from 'next-auth/providers/credentials';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/prisma';
import bcrypt from 'bcryptjs';
import { z } from 'zod';

const credentialsSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),  // 将 Session 存数据库

  providers: [
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    }),
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    Credentials({
      credentials: {
        email: { type: 'email' },
        password: { type: 'password' },
      },
      async authorize(credentials) {
        const parsed = credentialsSchema.safeParse(credentials);
        if (!parsed.success) return null;

        const user = await prisma.user.findUnique({
          where: { email: parsed.data.email },
        });
        if (!user?.password) return null;

        const valid = await bcrypt.compare(parsed.data.password, user.password);
        if (!valid) return null;

        return { id: user.id, email: user.email, name: user.name, role: user.role };
      },
    }),
  ],

  callbacks: {
    // JWT callback：向 Token 添加自定义字段
    async jwt({ token, user }) {
      if (user) {
        // 首次登录时（user 非空），从 DB 加载完整信息
        const dbUser = await prisma.user.findUnique({ where: { id: user.id as string } });
        token.id = user.id;
        token.role = dbUser?.role ?? 'user';
      }
      return token;
    },

    // Session callback：将 Token 字段暴露给前端 session
    async session({ session, token }) {
      session.user.id = token.id as string;
      session.user.role = token.role as string;
      return session;
    },
  },

  session: {
    strategy: 'jwt',    // 'jwt'（无状态）or 'database'（有状态，依赖 Adapter）
    maxAge: 30 * 24 * 60 * 60,  // 30 天
  },

  pages: {
    signIn: '/login',    // 自定义登录页
    error: '/auth/error',
  },
});
```

---

## TypeScript 类型扩展

```typescript
// types/next-auth.d.ts
import 'next-auth';
import 'next-auth/jwt';

declare module 'next-auth' {
  interface Session {
    user: {
      id: string;
      email: string;
      name: string;
      role: 'user' | 'admin';
      image?: string;
    };
  }

  interface User {
    role?: 'user' | 'admin';
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    id: string;
    role: 'user' | 'admin';
  }
}
```

---

## Route Handler（API 路由）

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth';

export const { GET, POST } = handlers;
```

---

## 获取 Session：四种方式

```typescript
// 1. Server Component（最常用）—— 直接 await，不需要 hook
import { auth } from '@/auth';

export default async function ProfilePage() {
  const session = await auth();  // 在服务端直接获取

  if (!session) redirect('/login');

  return <div>Hello, {session.user.name}</div>;
}

// 2. Server Action
'use server';
import { auth } from '@/auth';

export async function updateProfile(formData: FormData) {
  const session = await auth();
  if (!session) throw new Error('Unauthorized');

  await prisma.user.update({
    where: { id: session.user.id },
    data: { name: formData.get('name') as string },
  });
  revalidatePath('/profile');
}

// 3. Route Handler（API 路由）
import { auth } from '@/auth';
import { NextRequest } from 'next/server';

export async function GET(req: NextRequest) {
  const session = await auth();
  if (!session) return Response.json({ error: 'Unauthorized' }, { status: 401 });
  return Response.json({ user: session.user });
}

// 4. Client Component（useSession hook）
'use client';
import { useSession } from 'next-auth/react';

export function UserAvatar() {
  const { data: session, status } = useSession();

  if (status === 'loading') return <Skeleton />;
  if (!session) return <SignInButton />;
  return <img src={session.user.image ?? '/default-avatar.png'} alt={session.user.name} />;
}

// 5. 在根 layout 中包裹 SessionProvider（Client Component 才需要）
// app/layout.tsx
import { SessionProvider } from 'next-auth/react';
import { auth } from '@/auth';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const session = await auth();
  return (
    <html>
      <body>
        <SessionProvider session={session}>
          {children}
        </SessionProvider>
      </body>
    </html>
  );
}
```

---

## Protected Routes（middleware.ts）

```typescript
// middleware.ts（根目录，在 Edge Runtime 运行）
import { auth } from '@/auth';
import { NextResponse } from 'next/server';

// 定义需要保护的路由
const protectedRoutes = ['/dashboard', '/profile', '/settings', '/api/user'];
const adminRoutes = ['/admin'];
const authRoutes = ['/login', '/register'];

export default auth((req) => {
  const { nextUrl, auth: session } = req;
  const isLoggedIn = !!session;

  const isProtected = protectedRoutes.some(r => nextUrl.pathname.startsWith(r));
  const isAdmin = adminRoutes.some(r => nextUrl.pathname.startsWith(r));
  const isAuthRoute = authRoutes.some(r => nextUrl.pathname.startsWith(r));

  // 已登录用户访问登录页 → 重定向到 dashboard
  if (isAuthRoute && isLoggedIn) {
    return NextResponse.redirect(new URL('/dashboard', req.url));
  }

  // 未登录访问保护路由 → 重定向到登录页
  if (isProtected && !isLoggedIn) {
    const loginUrl = new URL('/login', req.url);
    loginUrl.searchParams.set('callbackUrl', nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }

  // Admin 路由：检查角色
  if (isAdmin) {
    if (!isLoggedIn) {
      return NextResponse.redirect(new URL('/login', req.url));
    }
    if (session.user.role !== 'admin') {
      return NextResponse.redirect(new URL('/403', req.url));
    }
  }

  return NextResponse.next();
});

export const config = {
  // 匹配所有路由（除了静态资源）
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.png$).*)'],
};
```

---

## RBAC（基于角色的权限控制）

```typescript
// src/lib/auth-utils.ts —— 权限工具函数

// 权限定义
const PERMISSIONS = {
  user: ['post:read', 'post:create', 'comment:create', 'profile:update'],
  admin: ['post:read', 'post:create', 'post:delete', 'user:manage', 'comment:create', 'comment:delete'],
} as const;

type Role = keyof typeof PERMISSIONS;
type Permission = typeof PERMISSIONS[Role][number];

export function hasPermission(role: Role, permission: Permission): boolean {
  return (PERMISSIONS[role] as readonly string[]).includes(permission);
}

// Server Component 中使用
export async function requirePermission(permission: Permission) {
  const session = await auth();
  if (!session) redirect('/login');

  if (!hasPermission(session.user.role as Role, permission)) {
    redirect('/403');
  }

  return session;
}

// 使用：
export default async function AdminPage() {
  const session = await requirePermission('user:manage');
  // 到这里确保有 user:manage 权限
  return <AdminDashboard />;
}

// Server Action 中权限检查
export async function deletePost(postId: string) {
  'use server';
  const session = await auth();
  if (!session) throw new UnauthorizedError();

  // 只有 admin 或帖子作者才能删除
  const post = await prisma.post.findUniqueOrThrow({ where: { id: postId } });
  const isOwner = post.authorId === session.user.id;
  const isAdmin = session.user.role === 'admin';

  if (!isOwner && !isAdmin) throw new ForbiddenError();

  await prisma.post.delete({ where: { id: postId } });
  revalidatePath('/posts');
}
```

---

## 登录表单（Server Action 方式）

```typescript
// app/login/page.tsx
import { signIn } from '@/auth';
import { AuthError } from 'next-auth';

export default function LoginPage({
  searchParams,
}: {
  searchParams: { callbackUrl?: string; error?: string };
}) {
  return (
    <div>
      {/* OAuth 登录 */}
      <form action={async () => {
        'use server';
        await signIn('github', { redirectTo: searchParams.callbackUrl ?? '/dashboard' });
      }}>
        <button type="submit">Login with GitHub</button>
      </form>

      {/* 邮箱密码登录 */}
      <form action={async (formData: FormData) => {
        'use server';
        try {
          await signIn('credentials', {
            email: formData.get('email'),
            password: formData.get('password'),
            redirectTo: searchParams.callbackUrl ?? '/dashboard',
          });
        } catch (err) {
          if (err instanceof AuthError) {
            // 处理登录错误（无效凭证等）
            redirect(`/login?error=${err.type}`);
          }
          throw err;  // 其他错误（如 redirect）必须重新抛出
        }
      }}>
        <input name="email" type="email" />
        <input name="password" type="password" />
        {searchParams.error === 'CredentialsSignin' && (
          <p>Invalid email or password</p>
        )}
        <button type="submit">Sign in</button>
      </form>
    </div>
  );
}
```

---

## Prisma Schema（Auth.js 需要的表）

```prisma
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  password      String?   // Credentials 登录用，OAuth 登录为 null
  role          String    @default("user")
  accounts      Account[]
  sessions      Session[]
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}
```

---

## 面试追问

**Q: Auth.js 的 JWT strategy 和 database strategy 怎么选？**
A: JWT strategy：Session 存 Cookie 的 JWT 中，无需查数据库（快），但无法实时撤销（Token 过期前一直有效）；database strategy：Session 存数据库，每次请求验证（慢），但可以随时 `signOut` 撤销。安全要求高（强制下线、账号封禁立即生效）→ database；普通应用 → JWT。Auth.js 默认推荐 JWT，PrismaAdapter 主要用于存 OAuth account 关联。

**Q: Middleware.ts 在 Edge Runtime 运行有什么限制？**
A: Edge Runtime 不支持 Node.js API（如 `fs`、`crypto`、`net`），不支持 Prisma（需要 Node.js 模块），不支持大多数 npm 包。Auth.js v5 的 `auth()` 在 middleware 中会使用 Edge 兼容模式（只验证 JWT，不查 DB）。如果需要查 DB（如检查用户是否被封禁），需要在 API Route 或 Server Component 中做，不能在 middleware。

**Q: 如何防止 CSRF 攻击？**
A: Auth.js 内置 CSRF 保护：Server Actions 由 Next.js 自动添加 CSRF Token；Auth.js 的 POST 端点（如 `/api/auth/signin`）验证请求来源（Origin 头）和 CSRF Token。自定义 API 路由：用 `SameSite=Strict` Cookie + 验证 `Origin` 头。如果用 JWT 存 Access Token（内存变量，不是 Cookie），XSS 风险代替了 CSRF 风险，需要用 DOMPurify 防 XSS。
