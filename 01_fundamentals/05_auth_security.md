# 认证与安全基础

## TL;DR

几乎每个系统设计题里都有"如何鉴权"这个问题。本文覆盖面试必须知道的内容：Session vs JWT、OAuth2/SSO 流程、HTTPS、API 安全，以及常见安全漏洞的防御。

---

## 认证（Authentication）vs 授权（Authorization）

面试中两个词经常混淆：

```
认证（Authentication）：你是谁？
  → 登录，验证用户名/密码，颁发凭证

授权（Authorization）：你能做什么？
  → 检查用户是否有权限访问某个资源（RBAC、ACL）

先认证，再授权。没有认证的授权没有意义。
```

---

## Session vs Token（JWT）

这是最基础的面试考点。

### Session（传统方案）

```
登录流程：
1. 用户提交用户名/密码
2. 服务器验证，创建 Session（随机字符串），存到 Redis
   session_id → { userId: 123, role: 'admin', expires: ... }
3. 服务器在 Cookie 里设置 session_id（HttpOnly, Secure）
4. 之后每次请求，浏览器自动带上 Cookie
5. 服务器查 Redis，验证 session_id 是否有效

优点：
  随时可以让 Session 失效（删 Redis Key）
  Session 内容存服务器，客户端不可见/不可篡改
  
缺点：
  有状态（Stateful）：每次请求都要查 Redis（有网络开销）
  横向扩展时，多台服务器需要共享同一个 Redis（Session 集中存储）
```

### JWT（JSON Web Token）

```
JWT 结构：Header.Payload.Signature（Base64 编码）

Header:  { "alg": "HS256", "typ": "JWT" }
Payload: { "userId": 123, "role": "admin", "exp": 1700000000 }
Signature: HMACSHA256(base64(Header) + "." + base64(Payload), secret_key)

登录流程：
1. 用户提交用户名/密码
2. 服务器验证，生成 JWT（包含 userId、role、过期时间）
3. JWT 返回给客户端（通常存 localStorage 或 Cookie）
4. 之后每次请求，带上 JWT（Authorization: Bearer <token>）
5. 服务器用 secret_key 验证签名，解码 Payload
   → 无需查数据库！
```

**JWT 的验证：**

```typescript
import jwt from 'jsonwebtoken';

const SECRET_KEY = process.env.JWT_SECRET!; // 只有服务器知道

// 生成 Token（登录时）
function generateToken(userId: string, role: string): string {
  return jwt.sign(
    { userId, role },
    SECRET_KEY,
    { expiresIn: '24h' }
  );
}

// 验证 Token（每次请求时）
function verifyToken(token: string): { userId: string; role: string } {
  try {
    return jwt.verify(token, SECRET_KEY) as any;
  } catch (e) {
    throw new Error('Invalid or expired token');
  }
}

// 中间件
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'No token' });
  
  try {
    req.user = verifyToken(token);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

### Session vs JWT 对比

| | Session | JWT |
|--|---------|-----|
| 状态 | 有状态（存 Redis）| 无状态 |
| 性能 | 每次查 Redis | 本地验证，无网络开销 |
| 吊销 | 立即删 Redis Key | **无法立即吊销**（等到过期）|
| 横向扩展 | 需要共享 Redis | 无需共享状态，天然适合微服务 |
| 存储位置 | 服务器 | 客户端（需防 XSS）|
| 适合场景 | 需要随时踢出用户（如安全敏感系统）| 微服务、无状态 API |

**JWT 的核心缺陷：无法立即吊销**

```
场景：用户被封号，但 JWT 还有 23 小时有效
→ 在 JWT 过期前，用户仍然可以用这个 Token 访问系统

解决方案（二选一）：
1. 短有效期（15 分钟）+ Refresh Token
   Access Token：15 分钟过期（损失不大）
   Refresh Token：7 天（存服务器，可以吊销）
   
2. Token 黑名单（Redis）
   被吊销的 Token 存入 Redis 黑名单（TTL = Token 剩余有效期）
   每次验证时额外查 Redis
   缺点：又变成有状态了，失去 JWT 的优势
```

---

## OAuth2 / SSO（第三方登录）

"用 Google 账号登录"的背后机制：

```
角色：
  Resource Owner：用户
  Client：你的应用（第三方 App）
  Authorization Server：Google 的授权服务器
  Resource Server：Google 的 API（如 Gmail、Drive）

OAuth2 Authorization Code Flow（最安全的流程）：

1. 用户点击"用 Google 登录"
2. 你的 App 重定向到 Google 授权页面：
   https://accounts.google.com/oauth/authorize
   ?client_id=YOUR_APP_ID
   &redirect_uri=https://yourapp.com/callback
   &scope=email profile
   &response_type=code
   &state=random_csrf_token    ← 防 CSRF

3. 用户在 Google 登录，同意授权
4. Google 重定向回你的 App，带上授权码：
   https://yourapp.com/callback?code=AUTH_CODE&state=...

5. 你的 App 后端用 AUTH_CODE 换 Access Token（服务器间调用）：
   POST https://oauth2.googleapis.com/token
   { code, client_id, client_secret, redirect_uri }
   → 返回 { access_token, id_token, refresh_token }

6. 用 Access Token 调用 Google API 获取用户信息：
   GET https://www.googleapis.com/oauth2/v3/userinfo
   Authorization: Bearer ACCESS_TOKEN
   → { email, name, picture }

7. 在自己的数据库里创建/查找用户，生成自己的 Session/JWT
```

**为什么需要 Authorization Code 而不是直接返回 Token？**

```
直接返回 Token（Implicit Flow，已废弃）：
  URL 里带 Token → 出现在浏览器历史、服务器日志、Referer 头
  → Token 可能泄露

Authorization Code Flow：
  URL 只带 code（一次性，短期有效）
  Token 通过服务器间的 HTTPS 请求换取（不经过浏览器）
  → 更安全
```

---

## HTTPS 工作原理（面试简答版）

```
TLS Handshake（握手）：

1. Client Hello：客户端告知支持的 TLS 版本、加密套件
2. Server Hello：服务器选定加密套件，发送证书（含公钥）
3. 客户端验证证书（CA 签名是否可信）
4. Key Exchange：双方协商生成对称密钥（Session Key）
   → 用非对称加密（RSA/ECDH）安全地交换密钥
5. 握手完成，后续通信用对称加密（AES）加密传输

为什么既用非对称又用对称？
  非对称加密（RSA）：安全但慢（适合握手交换密钥）
  对称加密（AES）：快（适合后续大量数据传输）
```

**HTTP vs HTTPS 的关键区别：**

```
HTTP：明文传输，中间人可以看到所有内容
HTTPS：
  加密（Confidentiality）：中间人看不到内容
  完整性（Integrity）：内容不能被篡改
  身份验证（Authentication）：确认你连的是真正的 example.com 而不是钓鱼网站
```

---

## 密码存储

**绝对不能明文存储密码**：

```typescript
import bcrypt from 'bcrypt';

// 注册时（存储 Hash）
async function hashPassword(password: string): Promise<string> {
  const saltRounds = 12; // 计算 cost（越高越慢，越安全，12 是推荐值）
  return await bcrypt.hash(password, saltRounds);
  // 结果类似：$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LeWnFgfnB.e8E.Omi
}

// 登录时（验证）
async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return await bcrypt.compare(password, hash);
}
```

**为什么用 bcrypt 而不是 SHA256：**

```
SHA256 太快（每秒 10 亿次计算）→ 暴力破解 + 彩虹表攻击可行

bcrypt / Argon2：
  有 salt（每次 Hash 结果不同，防彩虹表）
  有 cost factor（调整计算时间，现在算慢，未来升级 CPU 可以调高）
  每次验证约 100ms（对用户无感，但对攻击者：100ms × 10^9 次 = 3 年）
```

---

## 常见安全漏洞（OWASP Top 10 精选）

### SQL 注入

```typescript
// ❌ 危险：字符串拼接
const userId = req.body.userId; // 攻击者输入："1; DROP TABLE users; --"
db.query(`SELECT * FROM users WHERE id = ${userId}`);

// ✅ 安全：参数化查询
db.query('SELECT * FROM users WHERE id = ?', [userId]);

// ✅ 安全：ORM（自动参数化）
await User.findOne({ where: { id: userId } });
```

### XSS（跨站脚本攻击）

```typescript
// ❌ 危险：直接输出用户输入到 HTML
res.send(`<h1>欢迎, ${req.query.name}</h1>`);
// 攻击者输入：<script>document.cookie → 攻击者服务器</script>

// ✅ 安全：转义 HTML 特殊字符
import { escape } from 'html-escaper';
res.send(`<h1>欢迎, ${escape(req.query.name)}</h1>`);

// ✅ 安全：设置 Content-Security-Policy 头
res.setHeader('Content-Security-Policy', "default-src 'self'");

// Cookie 设置 HttpOnly（JS 无法读取，防止 XSS 窃取 Cookie）
res.cookie('session_id', sessionId, {
  httpOnly: true,  // JS 不可读
  secure: true,    // 只通过 HTTPS 传输
  sameSite: 'strict' // 防 CSRF
});
```

### CSRF（跨站请求伪造）

```
攻击原理：
  用户登录了 bank.com
  访问恶意网站 evil.com
  evil.com 的页面里有：
    <img src="https://bank.com/transfer?to=attacker&amount=1000">
  浏览器自动带上 bank.com 的 Cookie → 转账成功！

防御：
1. SameSite Cookie：
   Cookie 设置 sameSite: 'strict'
   → 跨站请求不带 Cookie

2. CSRF Token：
   每个表单/请求带一个服务器生成的随机 Token（存在 Session 里）
   服务器验证 Token 是否匹配
   → evil.com 无法获取这个 Token

3. 检查 Origin/Referer 头（辅助方案）
```

---

## API 安全设计

### API Key vs OAuth

```
API Key（机器间调用）：
  适合：服务间通信、第三方开发者调用你的 API
  实现：随机字符串（UUID v4），存 DB，请求时在 Header 带上
  Header：Authorization: ApiKey sk_live_abc123xyz
  权限：按 API Key 做限流、权限管理

OAuth（代表用户的授权）：
  适合："用户授权第三方 App 访问他的数据"
  实现：见上面的 OAuth2 流程
```

### 权限控制（RBAC）

```typescript
// Role-Based Access Control
enum Role { Admin, Editor, Viewer }

const permissions = {
  [Role.Admin]:  ['read', 'write', 'delete', 'admin'],
  [Role.Editor]: ['read', 'write'],
  [Role.Viewer]: ['read'],
};

function checkPermission(userRole: Role, action: string): boolean {
  return permissions[userRole].includes(action);
}

// 中间件
function requirePermission(action: string) {
  return (req, res, next) => {
    if (!checkPermission(req.user.role, action)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

// 使用
app.delete('/posts/:id',
  authMiddleware,              // 先认证
  requirePermission('delete'), // 再授权
  deletePost
);
```

---

## 面试中如何讨论安全

面试官问"这个系统的安全怎么设计？"时的标准答案框架：

```
1. 传输安全：全站 HTTPS，API 只接受 HTTPS 请求

2. 认证：
   - 用户：Session Cookie 或 JWT（根据是否需要即时吊销选择）
   - 服务间：API Key 或 mTLS（双向 TLS）
   - 第三方登录：OAuth2 Authorization Code Flow

3. 授权：
   - RBAC（基于角色），资源级别的权限检查
   - 最小权限原则（只给必要的权限）

4. 数据安全：
   - 密码：bcrypt 存 Hash，永不明文
   - 敏感数据（手机号、身份证）：AES 加密存储
   - PII 数据（个人可识别信息）：单独存储、访问审计

5. 常见攻击防御：
   - SQL 注入：参数化查询 / ORM
   - XSS：输出转义、CSP Header、HttpOnly Cookie
   - CSRF：SameSite Cookie + CSRF Token
   - 暴力破解：登录限速 + 账号锁定

6. 基础设施安全：
   - API Gateway 统一处理限流、认证、日志
   - 内网服务不暴露公网
   - 定期轮换密钥和证书
```

---

## Node.js 类比

```typescript
// 完整的 Express 认证/授权中间件栈
import express from 'express';
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';
import rateLimit from 'express-rate-limit';
import helmet from 'helmet'; // 自动设置安全 Header

const app = express();

// 1. 安全 Header（防 XSS、Clickjacking 等）
app.use(helmet());

// 2. 登录限流（防暴力破解）
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 10,                   // 最多 10 次尝试
  message: 'Too many login attempts'
});

app.post('/auth/login', loginLimiter, async (req, res) => {
  const { email, password } = req.body;
  const user = await db.users.findOne({ email });

  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    return res.status(401).json({ error: 'Invalid credentials' });
    // 注意：不要说"用户不存在"或"密码错误"（信息泄露）
    // 统一返回"凭证无效"
  }

  const token = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  );

  const refreshToken = jwt.sign(
    { userId: user.id },
    process.env.REFRESH_SECRET!,
    { expiresIn: '7d' }
  );

  res.cookie('refresh_token', refreshToken, {
    httpOnly: true, secure: true, sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000
  });

  res.json({ token }); // Access Token 返回给前端（存内存，不存 localStorage）
});
```

---

## 常见陷阱

1. **JWT Secret 硬编码**：`const secret = "mysecret"` 写死在代码里，提交 GitHub 后泄露。必须通过环境变量（`process.env.JWT_SECRET`）注入，且 Secret 要足够长（256 位随机数）

2. **不区分"用户不存在"和"密码错误"**：登录失败时，返回不同信息会告诉攻击者账号是否存在，方便枚举用户。统一返回"用户名或密码错误"

3. **存 JWT 到 localStorage**：localStorage 可被 XSS 攻击读取。Access Token 存内存（React State），Refresh Token 存 HttpOnly Cookie（JS 不可读）

4. **Refresh Token 不做轮换**：每次用 Refresh Token 换 Access Token 时，应该同时颁发新的 Refresh Token（旧的作废），并记录使用情况。如果同一个 Refresh Token 被使用两次，说明被盗用，立即吊销所有 Token

---

## 面试 Q&A

### 简单

**Q: Session 和 JWT 有什么核心区别？**

A: Session 是有状态的——服务器把会话数据存在 Redis 里，每次请求都查 Redis 验证，支持随时吊销（删 Redis Key）。JWT 是无状态的——Token 本身携带用户信息，服务器只需验证签名不用查数据库，适合微服务；但代价是无法立即吊销，只能等 Token 过期，通常用短有效期（15 分钟）+ Refresh Token 来弥补。

**Q: 为什么密码要用 bcrypt 而不是 MD5 或 SHA256？**

A: MD5/SHA256 设计目标是快（CPU 密集型任务用来做 Hash），正因为太快，攻击者可以每秒计算几十亿次来暴力破解。bcrypt 有意设计得慢（通过 cost factor 控制，每次 Hash 约需 100ms），还内置了 salt（每次 Hash 结果不同，防止彩虹表）。攻击者暴力破解的成本被提高了数百万倍。

---

### 中等

**Q: OAuth2 的 Authorization Code Flow 和 Implicit Flow 有什么区别，为什么前者更安全？**

A: Implicit Flow（已废弃）直接在 URL Fragment 里返回 Access Token（`https://app.com/callback#access_token=xxx`），Token 会出现在浏览器历史记录、服务器日志、Referer 头，容易泄露。Authorization Code Flow 只在 URL 里传一次性的 code（有效期很短），真正的 Token 由服务器后端通过 HTTPS 请求换取，完全不经过浏览器，大幅降低泄露风险。

**Q: 如何防止 JWT 被盗用（Token 泄露后如何止损）？**

A: 几个措施组合：1）短有效期（15 分钟）——即使泄露，攻击窗口有限；2）Access Token 存内存（不存 localStorage），减少 XSS 能读取的机会；3）HTTPS 传输，防止中间人截取；4）绑定指纹——Token 里包含 IP 或 User-Agent 的 Hash，验证时对比（粗粒度，因为 IP 会变）；5）Refresh Token 轮换并检测重用——同一 Refresh Token 被使用两次，立即吊销所有会话。

---

### 困难

**Q: 为一个多租户 SaaS 系统（每个租户有自己的用户、角色、资源）设计认证授权架构。**

A: 分层设计：

**认证层（确认身份）：** JWT 方案，Token Payload 里包含 `{ userId, tenantId, email }`。登录时验证用户属于该租户（`SELECT * FROM users WHERE id=? AND tenant_id=?`），防止跨租户越权。

**授权层（确认权限）：** 三级 RBAC：
- 平台级（SaaS 管理员）：所有租户的超级管理员
- 租户级（每个租户的 Admin）：管理本租户内的用户和权限
- 资源级（细粒度）：某个用户对某个具体资源的权限（如只读某份文档）

权限表：`permissions(tenant_id, user_id, resource_type, resource_id, action)`，查询时过滤 `tenant_id` 保证租户隔离。

**数据隔离：** 每个 SQL 查询都必须带 `tenant_id = ?` 条件（可以用 ORM 的全局 Scope 自动注入，防止漏写）。Row Level Security（PostgreSQL 原生支持）可以在数据库层强制隔离，即使应用层漏掉了 tenant_id 过滤，数据库也拒绝返回其他租户的数据。

**审计日志：** 每个写操作记录到 `audit_logs(tenant_id, user_id, action, resource, timestamp)`，供合规审查。

---

## 关联文档

- [../04_distributed/04_fault_tolerance.md](../04_distributed/04_fault_tolerance.md) — 限流（登录限速）
- [../03_communication/01_sync.md](../03_communication/01_sync.md) — API Gateway（统一鉴权入口）
- [../02_rate_limiter.md](../06_case_studies/02_rate_limiter.md) — 接口限流防暴力破解
