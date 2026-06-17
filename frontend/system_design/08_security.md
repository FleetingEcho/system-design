# 前端安全

> 前端安全面试考点：XSS、CSRF、CSP、Auth 方案。
> 后端安全见 [01_fundamentals/05_auth_security.md](../../backend/system_design/01_fundamentals/05_auth_security.md)，本文聚焦前端特有场景。

---

## XSS（跨站脚本攻击）

### 攻击原理

```
用户提交内容：<script>document.cookie → 攻击者服务器</script>
页面直接渲染：<div><script>...</script></div>
浏览器执行脚本，窃取 Cookie / LocalStorage / 用户操作
```

### 三类 XSS

| 类型 | 数据流 | 典型场景 |
|------|--------|---------|
| **存储型**（最危险）| 用户输入 → 数据库 → 页面渲染 | 评论区、用户昵称 |
| **反射型** | 用户输入 → URL → 页面渲染 | 搜索框、错误页 |
| **DOM 型** | JS 读取 URL/存储 → 写入 DOM | 单页应用客户端路由 |

### 防御：输出编码

```typescript
// ❌ 危险：直接插入 HTML
element.innerHTML = userInput;
document.write(userInput);

// ✓ 安全：文本内容用 textContent（自动编码）
element.textContent = userInput;

// ✓ React 默认安全（JSX 自动 HTML 编码）
function Comment({ content }: { content: string }) {
  return <div>{content}</div>;  // ✓ 安全，React 自动转义
}

// ❌ React 的逃生舱（慎用，必须确保 content 是受信任的 HTML）
function RichContent({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
  // 必须在服务端用 DOMPurify sanitize 后才能使用
}
```

### DOMPurify：HTML 净化

```typescript
import DOMPurify from 'dompurify';

// 服务端净化（Node.js）
import { JSDOM } from 'jsdom';
const { window } = new JSDOM('');
const DOMPurifyServer = DOMPurify(window);

function sanitizeHTML(dirty: string): string {
  return DOMPurifyServer.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'ul', 'li'],
    ALLOWED_ATTR: ['href', 'target'],
    // 强制外链在新 tab 打开，并阻止钓鱼（noopener）
    ADD_ATTR: ['rel'],
    FORCE_BODY: true,
  });
}

// 客户端富文本渲染
function RichText({ content }: { content: string }) {
  const cleanHTML = useMemo(() => DOMPurify.sanitize(content), [content]);
  return <div dangerouslySetInnerHTML={{ __html: cleanHTML }} />;
}
```

---

## CSP（内容安全策略）

### 原理

HTTP 响应头告诉浏览器：**只允许加载来自指定来源的资源**，即使攻击者注入了脚本，浏览器也拒绝执行。

### 配置示例

```typescript
// Next.js 中配置 CSP（next.config.ts）
const ContentSecurityPolicy = `
  default-src 'self';
  script-src 'self' 'nonce-${nonce}' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https://images.example.com;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com wss://realtime.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
`;

// Middleware 中设置 CSP + 生成 nonce
import { NextResponse } from 'next/server';
import crypto from 'crypto';

export function middleware(request: NextRequest) {
  const nonce = crypto.randomBytes(16).toString('base64');

  const response = NextResponse.next({
    request: { headers: new Headers(request.headers) },
  });

  response.headers.set(
    'Content-Security-Policy',
    ContentSecurityPolicy.replace('${nonce}', nonce).replace(/\s{2,}/g, ' ').trim()
  );
  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('X-Frame-Options', 'DENY');         // 防点击劫持
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');

  return response;
}
```

### Nonce 模式（严格 CSP）

```html
<!-- 服务端为每次请求生成随机 nonce，内联脚本必须带 nonce 才能执行 -->
<!-- 攻击者注入的脚本没有正确 nonce → 被 CSP 拦截 -->
<script nonce="a3f2c9d1...">
  // 这段内联脚本可以执行（nonce 匹配）
</script>

<script>alert('XSS')</script>  <!-- 被 CSP 拦截（无 nonce）-->
```

---

## CSRF（跨站请求伪造）

### 攻击原理

```
1. 用户登录了 bank.com（Cookie 自动携带）
2. 用户访问了 evil.com
3. evil.com 包含：
   <img src="https://bank.com/transfer?to=attacker&amount=10000" />
4. 浏览器发送请求时自动带上 bank.com 的 Cookie
5. 银行以为是合法请求，执行转账
```

### 防御方案

**方案 1：SameSite Cookie（现代浏览器首选）**

```typescript
// 服务端设置 Cookie
res.cookie('session', token, {
  httpOnly: true,    // JS 无法读取（防 XSS 窃取 Cookie）
  secure: true,      // 仅 HTTPS 传输
  sameSite: 'strict', // 严格：第三方站点请求不携带 Cookie（完全防 CSRF）
  // sameSite: 'lax' — 宽松：GET 跨站请求允许，POST 不允许（兼顾 SSO 跳转）
  maxAge: 7 * 24 * 60 * 60 * 1000,
});
```

**方案 2：CSRF Token（老浏览器兼容）**

```typescript
// 服务端在响应中下发 CSRF Token
// 客户端在修改操作（POST/PUT/DELETE）中带上 Token
// 攻击者无法获取 Token（跨域脚本无法读取 DOM）

// 服务端生成
const csrfToken = crypto.randomBytes(32).toString('hex');
// 存入 Session，返回给客户端（Cookie 或 响应体）

// 客户端请求时携带
fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'X-CSRF-Token': getCsrfToken(),  // 从 Cookie 或 <meta> 标签读取
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ to: 'someone', amount: 100 }),
});

// 服务端验证
if (req.headers['x-csrf-token'] !== session.csrfToken) {
  return res.status(403).json({ error: 'CSRF validation failed' });
}
```

---

## 身份认证方案

### Cookie-Session vs JWT

| 维度 | Cookie-Session | JWT |
|------|---------------|-----|
| 存储位置 | 服务端（Session Store）| 客户端（Cookie / Header）|
| 注销 | 简单（删除 Session）| 难（Token 有效期内无法强制失效）|
| 水平扩展 | 需要 Session Store（Redis）| 无状态，天然水平扩展 |
| 跨域 | 麻烦（Cookie 跨域限制）| 灵活（Header 无跨域限制）|
| 安全性 | Token 服务端可控 | Token 泄露前无法吊销 |

**实践结论**：
- Web 应用首选 **HttpOnly Cookie + Session**（安全性更高，防 XSS 窃取）
- 纯 API / 移动端首选 **JWT**（无状态，便于分发）
- JWT 的注销问题用短过期时间（15min）+ Refresh Token 缓解

### NextAuth.js / Auth.js

```typescript
// 最流行的 Next.js 认证库，支持 40+ OAuth Provider
// app/api/auth/[...nextauth]/route.ts

import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';
import Google from 'next-auth/providers/google';
import Credentials from 'next-auth/providers/credentials';
import { DrizzleAdapter } from '@auth/drizzle-adapter';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: DrizzleAdapter(db),  // 持久化 Session 到数据库
  providers: [
    GitHub({ clientId: process.env.GITHUB_ID!, clientSecret: process.env.GITHUB_SECRET! }),
    Google({ clientId: process.env.GOOGLE_ID!, clientSecret: process.env.GOOGLE_SECRET! }),
    Credentials({
      credentials: { email: {}, password: {} },
      authorize: async (credentials) => {
        const user = await db.user.findUnique({ where: { email: credentials.email } });
        if (!user) return null;
        const valid = await bcrypt.compare(credentials.password, user.hashedPassword);
        return valid ? user : null;
      },
    }),
  ],
  callbacks: {
    session: ({ session, user }) => ({
      ...session,
      user: { ...session.user, id: user.id, role: user.role },  // 扩展 Session 字段
    }),
  },
});

// 在 Server Component 中获取 Session
const session = await auth();
if (!session) redirect('/login');
```

### Clerk（SaaS 认证服务）

```typescript
// Clerk：托管认证服务，内置 UI 组件，开箱即用
// 适合快速启动项目、不想自己管理认证基础设施

// middleware.ts — 路由级保护
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isProtected = createRouteMatcher(['/dashboard(.*)', '/admin(.*)']);

export default clerkMiddleware((auth, req) => {
  if (isProtected(req)) auth().protect();  // 未登录自动跳转 /sign-in
});

// Server Component 中获取用户
import { auth, currentUser } from '@clerk/nextjs/server';

export default async function DashboardPage() {
  const { userId } = auth();
  const user = await currentUser();
  return <div>Welcome, {user?.firstName}</div>;
}

// 客户端 hooks
import { useUser, useAuth } from '@clerk/nextjs';

function Header() {
  const { user, isLoaded } = useUser();
  const { signOut } = useAuth();
  return <button onClick={() => signOut()}>Sign out, {user?.firstName}</button>;
}
```

**Clerk vs Auth.js 选型**：
- Auth.js（NextAuth）：开源、免费、自托管，需要配置数据库
- Clerk：SaaS（有免费额度），托管、UI 组件精美、多因素认证开箱即用，规模大后收费

---

## 敏感数据保护

### 环境变量管理

```typescript
// .env.local — 服务端 + 客户端
DATABASE_URL=...           # 不带 NEXT_PUBLIC_ 前缀 → 只在服务端可见
SECRET_KEY=...

// .env.local — 只有客户端需要的公开配置
NEXT_PUBLIC_API_URL=...    # 带 NEXT_PUBLIC_ 前缀 → 打包进客户端 bundle

// ❌ 危险：私密 key 不小心暴露给客户端
NEXT_PUBLIC_DATABASE_URL=...   // ← 这会出现在浏览器 JS 里
NEXT_PUBLIC_STRIPE_SECRET=...  // ← 永远不能这样做

// 原则：只有公开配置（API 端点、Feature Flag、Analytics ID）才加 NEXT_PUBLIC_
```

### LocalStorage vs Cookie 存 Token

```
❌ LocalStorage 存 JWT：
  - JS 可以直接读取 → XSS 攻击可窃取
  - 没有 SameSite 保护

✓ HttpOnly Cookie 存 Token：
  - JS 无法读取（防 XSS）
  - SameSite=strict 防 CSRF
  - Secure 属性确保 HTTPS 传输

折中方案（需要跨域 API）：
  - Access Token 存内存（不持久化，页面刷新丢失）
  - Refresh Token 存 HttpOnly Cookie
  - Access Token 过期时，用 Refresh Token 静默续签
```

---

## 安全 Headers 速查

```typescript
// 推荐在所有 Web 应用设置的安全响应头
const securityHeaders = {
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',
  // ↑ 强制 HTTPS（HSTS），一年内浏览器拒绝 HTTP 连接

  'X-Content-Type-Options': 'nosniff',
  // ↑ 禁止 MIME 类型嗅探（防止把 JS 文件当 HTML 执行）

  'X-Frame-Options': 'DENY',
  // ↑ 禁止 iframe 嵌入（防点击劫持），CSP frame-ancestors 是现代替代

  'Referrer-Policy': 'strict-origin-when-cross-origin',
  // ↑ 跨域请求只发送 Origin（不含路径），防止 URL 中的敏感参数泄露

  'Permissions-Policy': 'camera=(), microphone=(), geolocation=(self)',
  // ↑ 限制 Web API 权限，防止恶意脚本调用摄像头/麦克风

  'Content-Security-Policy': '...', // 见上文
};
```

---

## 面试常见追问

**Q: SPA 比传统多页应用更容易受 XSS 攻击吗？**
A: 是的，风险更高。SPA 大量操作 DOM（innerHTML、eval 等），而且客户端路由让反射型 XSS 更容易触发。但现代框架（React/Vue）默认自动编码，只要避免 dangerouslySetInnerHTML 就能防御大多数 XSS。

**Q: JWT 存 localStorage 有什么问题，怎么解决？**
A: XSS 可以读取 localStorage，窃取 Token 后冒充用户。解决：用 HttpOnly Cookie 存 Token（JS 无法读取）；或存内存（安全但页面刷新丢失，需要 Refresh Token 机制）。

**Q: OAuth2 授权码流程前端需要关心什么？**
A: 三点：①state 参数（随机字符串，防 CSRF，授权回调时校验）；②PKCE（code_verifier / code_challenge，防授权码拦截，SPA 必用）；③回调后的 code 只用一次，立即换 Token 并丢弃 code。

**Q: 如何防止 npm 供应链攻击？**
A: ①锁定版本（package-lock.json / pnpm-lock.yaml 提交到 git）；②CI 中用 `npm audit` 扫描漏洞；③用 `npm install --ignore-scripts` 防止 postinstall 脚本执行；④考虑 Socket.dev 或 Snyk 监控依赖变化。
