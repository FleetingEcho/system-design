# 全栈部署（Docker + Vercel + CI/CD）

> Turborepo Monorepo 的 Docker 多阶段构建、Vercel 部署 Next.js、GitHub Actions CI/CD、数据库 Migration 策略。

---

## Next.js 部署选项对比

```
Vercel（官方平台）：
  + 零配置，push 即部署
  + Preview Deployments（每个 PR 有独立预览 URL）
  + Edge Functions、Image Optimization、Analytics 开箱即用
  + ISR / On-Demand Revalidation 完美支持
  - 费用随流量增长，大规模可能昂贵
  - Serverless 函数有冷启动

Docker + K8s（自托管）：
  + 完全控制，适合私有云/合规要求
  + 长连接支持好（WebSocket、Server-Sent Events）
  + 无冷启动（常驻进程）
  - 需要自己处理部署、扩容、监控
  - Next.js standalone 模式需要额外配置

Railway / Render（轻量平台）：
  + 比 Vercel 便宜，支持长连接
  + 可以部署 Docker 容器
  + 适合全栈（前端 + 后端 + DB 同一平台）
```

---

## Vercel 部署（Next.js）

```
monorepo 结构：
apps/web   → Vercel Project A
apps/admin → Vercel Project B
apps/api   → Railway/Render（如果是独立 API）
```

```json
// apps/web/vercel.json
{
  "buildCommand": "cd ../.. && pnpm turbo build --filter=web...",
  "outputDirectory": "apps/web/.next",
  "installCommand": "pnpm install --frozen-lockfile",
  "framework": "nextjs"
}
```

```bash
# Vercel CLI 配置（在 Vercel Dashboard 设置更方便）
vercel link                              # 关联 Vercel 项目
vercel env add DATABASE_URL production   # 添加环境变量
vercel --prod                            # 手动部署到生产
```

```
环境变量分级：
  本地开发：.env.local（git ignore）
  Vercel Preview：.env.preview（同一变量名，不同值）
  Vercel Production：.env.production

Next.js 环境变量规则：
  NEXT_PUBLIC_*  → 暴露给浏览器（打包进 JS）
  无前缀          → 只在服务端可用（Server Component、API Route、Server Action）
  不要把密钥放 NEXT_PUBLIC_*（会泄露到客户端）
```

---

## Docker 多阶段构建（Node.js API）

```dockerfile
# apps/api/Dockerfile
# 阶段一：依赖安装
FROM node:20-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# 只复制 package 文件（利用 Docker layer 缓存）
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY apps/api/package.json ./apps/api/
COPY packages/database/package.json ./packages/database/
COPY packages/validators/package.json ./packages/validators/

RUN corepack enable && pnpm install --frozen-lockfile

# 阶段二：构建
FROM node:20-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/apps/api/node_modules ./apps/api/node_modules
COPY . .

RUN pnpm turbo build --filter=api

# Prisma 生成（需要在 build 阶段）
RUN pnpm --filter database exec prisma generate

# 阶段三：生产镜像（只包含运行时需要的文件）
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nodeuser

# 只复制必要文件（不包含 devDependencies、源码）
COPY --from=builder --chown=nodeuser:nodejs /app/apps/api/dist ./dist
COPY --from=builder --chown=nodeuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodeuser:nodejs /app/apps/api/node_modules ./apps/api/node_modules
COPY --from=builder --chown=nodeuser:nodejs /app/packages/database/prisma ./prisma

USER nodeuser
EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3001/health || exit 1

CMD ["node", "dist/server.js"]
```

```dockerfile
# apps/web/Dockerfile（Next.js standalone 模式）
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY apps/web/package.json ./apps/web/
RUN corepack enable && pnpm install --frozen-lockfile

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NEXT_TELEMETRY_DISABLED=1
# standalone 模式：Next.js 只输出运行所需的最小文件集
RUN pnpm turbo build --filter=web

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/apps/web/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/static ./apps/web/.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000

CMD ["node", "apps/web/server.js"]
```

```javascript
// apps/web/next.config.ts
const nextConfig = {
  output: 'standalone',  // 启用 standalone 模式（Docker 必需）
  // standalone 会把所有依赖打包到 .next/standalone，不需要 node_modules
};
```

---

## Turborepo + Docker（Monorepo 精简）

```bash
# Turborepo 的 prune 命令：只保留指定 app 需要的文件
# 避免整个 Monorepo 都进 Docker context（慢且大）

npx turbo prune api --docker
# 输出：
# out/
#   json/   → 只包含 api 和其依赖的 package.json 文件
#   full/   → 完整源码（只包含 api 依赖的 packages）
#   pnpm-lock.yaml → 精简后的 lockfile
```

```dockerfile
# 使用 turbo prune 的 Dockerfile（推荐）
FROM node:20-alpine AS pruner
WORKDIR /app
COPY . .
RUN npx turbo prune api --docker

FROM node:20-alpine AS deps
WORKDIR /app
COPY --from=pruner /app/out/json/ .
COPY --from=pruner /app/out/pnpm-lock.yaml ./pnpm-lock.yaml
RUN corepack enable && pnpm install --frozen-lockfile

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo build --filter=api

# runner 阶段同上...
```

---

## GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint type-check test:unit

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push API image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: apps/api/Dockerfile
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/api:latest
            ghcr.io/${{ github.repository }}/api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            docker pull ghcr.io/${{ github.repository }}/api:latest
            docker compose -f /app/docker-compose.prod.yml up -d --no-deps api
```

---

## 数据库 Migration 策略

```
Migration 的核心挑战：
  1. 生产数据库不能停机 migration（零停机部署）
  2. Migration 和代码部署必须协调（先 migration 还是先部署？）
  3. 出错如何回滚？

零停机 Migration 原则（Expand-Contract 模式）：

Phase 1 - Expand（扩展）：
  添加新列/表，设置可为 NULL 或有默认值
  旧代码兼容（不报错）
  部署新代码（读/写新列，同时写旧列）

Phase 2 - Migrate（迁移）：
  运行数据迁移（把旧列数据复制到新列）

Phase 3 - Contract（收缩）：
  确认新代码已运行一段时间（无旧代码）
  删除旧列

例：把 full_name 拆分为 first_name + last_name
  Phase 1: 添加 first_name、last_name 列（NULL）
  Phase 2: UPDATE SET first_name = SPLIT_PART(full_name, ' ', 1)...
  Phase 3: 删除 full_name 列
```

```typescript
// package.json scripts：CI/CD 中的 migration 流程
{
  "scripts": {
    "db:migrate:deploy": "prisma migrate deploy",  // 生产：只运行未执行的 migration
    "db:migrate:dev": "prisma migrate dev",          // 开发：创建新 migration
    "db:migrate:status": "prisma migrate status",    // 检查 migration 状态
  }
}
```

```yaml
# GitHub Actions：在部署代码前运行 migration
deploy:
  steps:
    # 先 migration
    - name: Run database migrations
      run: |
        npx prisma migrate deploy
      env:
        DATABASE_URL: ${{ secrets.DATABASE_URL }}

    # 后部署（新代码兼容新 schema）
    - name: Deploy application
      run: docker compose up -d
```

```
Migration 失败回滚策略：

方案一：Prisma Migration 本身不支持自动回滚
  → 为每个 migration 手动编写 down.sql
  → 出错时手动运行 down.sql

方案二：蓝绿部署（Blue-Green Deployment）
  → 同时运行两个版本，migration 向后兼容
  → 出问题时切回旧版本（不需要回滚 DB）

方案三：功能标志（Feature Flags）
  → 新列已创建，但代码用 Flag 控制是否使用
  → 出问题关 Flag，不需要回滚 DB 也不需要重新部署
```

---

## Docker Compose（本地开发 + 生产）

```yaml
# docker-compose.yml（本地开发）
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ['5432:5432']
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports: ['6379:6379']
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:

# docker-compose.prod.yml（生产，只包含基础设施）
version: '3.8'
services:
  api:
    image: ghcr.io/myorg/myapp/api:latest
    restart: unless-stopped
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
    ports: ['3001:3001']
    healthcheck:
      test: ['CMD', 'wget', '-qO-', 'http://localhost:3001/health']
      interval: 30s
      timeout: 3s
      retries: 3

  nginx:
    image: nginx:alpine
    ports: ['80:80', '443:443']
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certbot/conf:/etc/letsencrypt:ro
    depends_on: [api]
```

---

## 面试追问

**Q: Next.js Standalone 模式有什么好处？**
A: `output: 'standalone'` 让 Next.js 把所有依赖打包到 `.next/standalone` 目录，Docker 镜像不需要整个 `node_modules`（几百 MB），最终镜像可以压缩到 200MB 以下（vs 普通方式的 1GB+）。配合多阶段构建，只把 `standalone` 目录复制到生产镜像，大幅减少攻击面和部署时间。

**Q: Vercel 和自托管什么时候切换？**
A: Vercel 适合早期（速度快、零运维）和静态/SSR 场景；以下情况考虑迁移：① 费用超过自托管成本（流量大时）；② 需要长连接（WebSocket、gRPC）；③ 合规要求数据不出境（Vercel 服务器在美国/欧洲）；④ 需要更细粒度的网络控制（VPC、内网服务调用）。迁移时，Next.js standalone 模式是关键，让 Next.js 在标准 Node.js 容器中运行。

**Q: 如何做零停机部署？**
A: 滚动更新（K8s 默认策略）：逐个替换 Pod，新 Pod Ready 后再删旧 Pod；蓝绿部署：同时运行两套，切换负载均衡器指向；金丝雀发布：5% 流量先走新版本，观察无问题后全量。三者的前提：新代码必须兼容旧 schema（DB migration 先于代码部署），API 变更要向后兼容（不能直接删字段，要先废弃）。
