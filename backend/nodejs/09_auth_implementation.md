# Node.js 鉴权实现

> JWT Refresh Token 轮换、OAuth2 PKCE、Session 管理。
> 理论见 `backend/system_design/01_fundamentals/05_auth_security.md`，本篇专注 Node.js 实现细节。

---

## JWT：Access + Refresh 双 Token 架构

```
为什么需要 Refresh Token？

Access Token 短寿命（15min）→ 减少泄露风险
Refresh Token 长寿命（7d）  → 减少重新登录频率

单 Token 困境：
  - 长寿命 Token 泄露后攻击窗口大
  - 短寿命 Token 需要频繁登录
  - 无法"强制下线"（Token 无法撤销，只能等过期）

双 Token 方案：
  Access Token（15min）→ 放在内存，每次请求带上
  Refresh Token（7d）  → 存数据库，放 HttpOnly Cookie
  每次 Access Token 过期 → 用 Refresh Token 换新 Access Token
  Refresh Token 轮换（Rotation）→ 换新时旧的立即作废
```

---

## Token 生成与验证

```typescript
// src/lib/jwt.ts
import jwt from 'jsonwebtoken';
import { z } from 'zod';

const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET!;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET!;

export const accessTokenPayloadSchema = z.object({
  sub: z.string(),    // userId
  email: z.string(),
  role: z.enum(['user', 'admin']),
  iat: z.number(),
  exp: z.number(),
});

export type AccessTokenPayload = z.infer<typeof accessTokenPayloadSchema>;

export function signAccessToken(payload: Omit<AccessTokenPayload, 'iat' | 'exp'>): string {
  return jwt.sign(payload, ACCESS_SECRET, { expiresIn: '15m' });
}

export function signRefreshToken(userId: string): string {
  return jwt.sign({ sub: userId }, REFRESH_SECRET, { expiresIn: '7d' });
}

export function verifyAccessToken(token: string): AccessTokenPayload {
  const raw = jwt.verify(token, ACCESS_SECRET);
  return accessTokenPayloadSchema.parse(raw);
}

export function verifyRefreshToken(token: string): { sub: string } {
  return jwt.verify(token, REFRESH_SECRET) as { sub: string };
}
```

---

## Refresh Token 轮换（Rotation）

```typescript
// src/services/auth.service.ts
import { prisma } from '../lib/prisma';
import { signAccessToken, signRefreshToken, verifyRefreshToken } from '../lib/jwt';
import { UnauthorizedError } from '../lib/errors';
import bcrypt from 'bcryptjs';
import { randomUUID } from 'crypto';

export class AuthService {
  async login(email: string, password: string) {
    const user = await prisma.user.findUnique({ where: { email } });
    if (!user) throw new UnauthorizedError('Invalid credentials');

    const valid = await bcrypt.compare(password, user.password);
    if (!valid) throw new UnauthorizedError('Invalid credentials');

    return this.issueTokens(user.id, user.email, user.role);
  }

  async refresh(rawRefreshToken: string) {
    // 1. 验证签名
    let payload: { sub: string };
    try {
      payload = verifyRefreshToken(rawRefreshToken);
    } catch {
      throw new UnauthorizedError('Invalid refresh token');
    }

    // 2. 查数据库（验证 Token 存在且未被使用/撤销）
    const storedToken = await prisma.refreshToken.findUnique({
      where: { token: rawRefreshToken },
      include: { user: true },
    });

    if (!storedToken || storedToken.revokedAt || new Date() > storedToken.expiresAt) {
      // Token 不存在或已撤销 → 可能是重用攻击（Replay Attack）
      // 撤销该用户所有 Token（安全保守策略）
      if (storedToken) {
        await prisma.refreshToken.updateMany({
          where: { userId: storedToken.userId },
          data: { revokedAt: new Date() },
        });
      }
      throw new UnauthorizedError('Refresh token reuse detected');
    }

    // 3. Rotation：撤销旧 Token，发新 Token
    await prisma.refreshToken.update({
      where: { id: storedToken.id },
      data: { revokedAt: new Date() },
    });

    return this.issueTokens(storedToken.user.id, storedToken.user.email, storedToken.user.role);
  }

  async logout(refreshToken: string) {
    await prisma.refreshToken.updateMany({
      where: { token: refreshToken },
      data: { revokedAt: new Date() },
    });
  }

  async logoutAll(userId: string) {
    // 强制下线：撤销该用户所有 Refresh Token
    await prisma.refreshToken.updateMany({
      where: { userId, revokedAt: null },
      data: { revokedAt: new Date() },
    });
  }

  private async issueTokens(userId: string, email: string, role: string) {
    const accessToken = signAccessToken({ sub: userId, email, role: role as any });
    const refreshTokenValue = signRefreshToken(userId);

    await prisma.refreshToken.create({
      data: {
        token: refreshTokenValue,
        userId,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      },
    });

    return { accessToken, refreshToken: refreshTokenValue };
  }
}
```

---

## Token 存储策略

```typescript
// src/routes/auth.routes.ts
import { Router, Request, Response } from 'express';
import { AuthService } from '../services/auth.service';

const COOKIE_OPTIONS = {
  httpOnly: true,    // JS 无法读取（防 XSS）
  secure: process.env.NODE_ENV === 'production',  // HTTPS only in prod
  sameSite: 'strict' as const,  // 防 CSRF
  maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 天（毫秒）
  path: '/auth/refresh',  // 只在 refresh 路径发送，减少泄露面
};

const authService = new AuthService();
const router = Router();

router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const { accessToken, refreshToken } = await authService.login(email, password);

  // Refresh Token → HttpOnly Cookie（JS 不可读）
  res.cookie('refreshToken', refreshToken, COOKIE_OPTIONS);

  // Access Token → 响应体（前端存内存，不存 localStorage）
  res.json({ accessToken });
});

router.post('/refresh', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) throw new UnauthorizedError('No refresh token');

  const { accessToken, refreshToken: newRefreshToken } = await authService.refresh(refreshToken);

  // 设置新的 Refresh Token Cookie
  res.cookie('refreshToken', newRefreshToken, COOKIE_OPTIONS);
  res.json({ accessToken });
});

router.post('/logout', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (refreshToken) await authService.logout(refreshToken);
  res.clearCookie('refreshToken', { path: '/auth/refresh' });
  res.json({ message: 'Logged out' });
});
```

---

## JWT 鉴权中间件

```typescript
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { verifyAccessToken, AccessTokenPayload } from '../lib/jwt';
import { UnauthorizedError, ForbiddenError } from '../lib/errors';

declare global {
  namespace Express {
    interface Request {
      user?: AccessTokenPayload;
    }
  }
}

export function authenticate(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    throw new UnauthorizedError('Missing authorization header');
  }

  const token = header.slice(7);
  try {
    req.user = verifyAccessToken(token);
    next();
  } catch (err: any) {
    if (err.name === 'TokenExpiredError') {
      throw new UnauthorizedError('Access token expired');
    }
    throw new UnauthorizedError('Invalid access token');
  }
}

export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) throw new UnauthorizedError('Not authenticated');
    if (!roles.includes(req.user.role)) {
      throw new ForbiddenError(`Required role: ${roles.join(' or ')}`);
    }
    next();
  };
}

// 路由使用
router.get('/admin/users', authenticate, requireRole('admin'), listUsersHandler);
router.get('/me', authenticate, getMeHandler);
```

---

## OAuth2 PKCE 流程（第三方登录）

```
PKCE（Proof Key for Code Exchange）解决什么问题？
  传统授权码流程中，Authorization Code 在重定向 URL 传输，可能被拦截。
  PKCE：客户端生成 code_verifier（随机字符串），发送其哈希 code_challenge。
  服务端收到 Authorization Code 后，验证 code_verifier 与之前的 code_challenge 匹配。
  即使 code 被拦截，攻击者没有 code_verifier 也无法换取 Access Token。

流程：
  1. 前端生成 code_verifier（随机字符串）和 code_challenge（SHA-256 哈希）
  2. 重定向到 OAuth 提供商，带 code_challenge
  3. 用户授权后，提供商重定向回来，带 authorization_code
  4. 前端将 code + code_verifier 发给自己的后端
  5. 后端用 code + code_verifier 换取 access_token（提供商验证 code_verifier）
```

```typescript
// src/lib/pkce.ts —— PKCE 工具函数
import crypto from 'crypto';

export function generateCodeVerifier(): string {
  return crypto.randomBytes(32).toString('base64url');  // 43-128 字符
}

export function generateCodeChallenge(verifier: string): string {
  return crypto.createHash('sha256').update(verifier).digest('base64url');
}

// src/routes/oauth.routes.ts
import axios from 'axios';

const GITHUB_CLIENT_ID = process.env.GITHUB_CLIENT_ID!;
const GITHUB_CLIENT_SECRET = process.env.GITHUB_CLIENT_SECRET!;
const REDIRECT_URI = `${process.env.APP_URL}/auth/callback/github`;

// Step 1: 重定向到 GitHub
router.get('/auth/github', (req, res) => {
  const state = crypto.randomBytes(16).toString('hex');
  // state 存 session（防 CSRF）
  req.session.oauthState = state;

  const params = new URLSearchParams({
    client_id: GITHUB_CLIENT_ID,
    redirect_uri: REDIRECT_URI,
    scope: 'user:email',
    state,
  });

  res.redirect(`https://github.com/login/oauth/authorize?${params}`);
});

// Step 2: 处理回调
router.get('/auth/callback/github', async (req, res) => {
  const { code, state } = req.query as { code: string; state: string };

  // 验证 state（防 CSRF）
  if (state !== req.session.oauthState) {
    throw new UnauthorizedError('Invalid OAuth state');
  }
  delete req.session.oauthState;

  // 用 code 换 access_token
  const tokenRes = await axios.post(
    'https://github.com/login/oauth/access_token',
    { client_id: GITHUB_CLIENT_ID, client_secret: GITHUB_CLIENT_SECRET, code },
    { headers: { Accept: 'application/json' } }
  );
  const githubAccessToken = tokenRes.data.access_token;

  // 获取用户信息
  const userRes = await axios.get('https://api.github.com/user', {
    headers: { Authorization: `Bearer ${githubAccessToken}` },
  });
  const emailRes = await axios.get('https://api.github.com/user/emails', {
    headers: { Authorization: `Bearer ${githubAccessToken}` },
  });

  const primaryEmail = emailRes.data.find((e: any) => e.primary)?.email;
  const githubUser = userRes.data;

  // Upsert 用户
  const user = await prisma.user.upsert({
    where: { email: primaryEmail },
    create: {
      email: primaryEmail,
      name: githubUser.name ?? githubUser.login,
      githubId: String(githubUser.id),
    },
    update: { githubId: String(githubUser.id) },
  });

  const { accessToken, refreshToken } = await authService.issueTokens(user.id, user.email, user.role);
  res.cookie('refreshToken', refreshToken, COOKIE_OPTIONS);
  res.redirect(`${process.env.FRONTEND_URL}/auth/success?token=${accessToken}`);
});
```

---

## Prisma Schema：Refresh Token 表

```prisma
model RefreshToken {
  id        String    @id @default(cuid())
  token     String    @unique
  userId    String
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  expiresAt DateTime
  revokedAt DateTime?
  createdAt DateTime  @default(now())

  @@index([userId])
}
```

---

## 面试追问

**Q: Access Token 放内存还是 localStorage？**
A: 放内存（JS 变量）。localStorage 会被 XSS 读取，一旦 XSS 就能偷走 Token 长期使用；内存里的 Token 页面刷新就消失，XSS 窗口只有当前 Session。缺点是刷新页面需要用 Refresh Token 重新换 Access Token，这是正常流程，不是问题。

**Q: Refresh Token 轮换检测到 Token 被重用怎么办？**
A: 检测到重用说明 Refresh Token 可能泄露（攻击者在用合法用户的 Token）。保守策略：撤销该用户所有 Refresh Token，强制重新登录。这会影响合法用户体验，但安全优先。也可以只撤销当前 Token 家族（Family），记录 Token 链路，只影响被泄露的设备。

**Q: JWT 和 Session Cookie 怎么选？**
A: 分布式多节点服务用 JWT（无状态，不需要共享 Session Store）；需要即时强制下线、安全要求高的场景用 Session + Redis（随时能删 Session）。实际上双 Token 方案是折中：Access Token 无状态（JWT），Refresh Token 有状态（数据库），兼顾性能和可撤销性。
