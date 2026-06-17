# 前端项目架构

> 当一个项目增长到 10 万行代码、50 名工程师，如何保持可维护性？
> 本文覆盖：Feature-Sliced Design、Monorepo 工具链、模块边界、依赖方向约束。

---

## 面试框架（45分钟怎么答）

**第一步（开场）**：先说"文件组织随项目规模演变"——小项目按类型（components/hooks/api）→ 中项目按功能（Feature-Sliced Design）→ 大项目 Monorepo（Turborepo/Nx）
**第二步（核心）**：FSD 六层架构（app/pages/widgets/features/entities/shared）；依赖只能从上往下，不能反向；Barrel Exports 反模式（不用 index.ts 重导出）
**第三步（深挖）**：Turborepo 缓存加速（远程缓存 Vercel Remote Cache）；ESLint 模块边界规则（no-restricted-imports 约束层级依赖）；Changesets 管理共享包版本
**差异化得分点**：能说出 Barrel Export 反模式的危害（工具链难以 Tree Shaking；IDE 引用查找变慢）

---

## 架构图：Feature-Sliced Design 六层依赖关系

```mermaid
graph TD
    App[app 应用初始化/路由/Provider] -->|依赖| Pages
    Pages[pages 页面组合层] -->|依赖| Widgets
    Widgets[widgets 独立 UI 块 Header/Sidebar] -->|依赖| Features
    Features[features 用户功能 auth/cart/search] -->|依赖| Entities
    Entities[entities 业务实体 User/Product/Order] -->|依赖| Shared
    Shared[shared 通用 UI/工具/配置]

    style App fill:#ff6b6b,color:#fff
    style Pages fill:#ffa94d,color:#fff
    style Widgets fill:#ffd43b,color:#000
    style Features fill:#69db7c,color:#000
    style Entities fill:#4dabf7,color:#000
    style Shared fill:#da77f2,color:#000

    note1[依赖方向 只能从上往下 不能反向]
```

---

## 文件组织的演变

### 阶段 1：按类型分组（小项目）

```
src/
  components/     ← 所有组件
  hooks/          ← 所有 hooks
  utils/          ← 工具函数
  pages/          ← 页面
  store/          ← 状态管理
  api/            ← 接口调用
```

**问题**：项目变大后，同一功能的文件分散在各处，修改一个功能要在 5 个文件夹之间跳转。

### 阶段 2：按功能分组（中型项目）

```
src/
  features/
    auth/
      components/
      hooks/
      api/
      store/
    products/
      components/
      hooks/
      api/
    checkout/
  shared/         ← 真正可复用的通用代码
  pages/          ← 路由页面，只引用 features
```

**问题**：没有规范，不同人写出不同结构，features 之间互相引用造成循环依赖。

### 阶段 3：Feature-Sliced Design（大型项目）

---

## Feature-Sliced Design（FSD）

### 核心原则：层（Layer）有单向依赖规则

```
app/          ← 全局配置（路由、Provider、全局样式）
  ↑ 可引用所有层
pages/        ← 页面组合（只负责布局组合）
  ↑ 可引用 widgets, features, entities, shared
widgets/      ← 独立功能块（Header、Sidebar）
  ↑ 可引用 features, entities, shared
features/     ← 用户交互功能（AddToCart、AuthByEmail）
  ↑ 可引用 entities, shared
entities/     ← 业务实体（Product、User、Order）
  ↑ 可引用 shared
shared/       ← 通用代码（UI 组件库、工具函数、API 客户端）
  ↑ 不可引用任何层
```

**关键规则**：每层只能引用它下面的层，不能引用上层（单向依赖）。

### 每层的切片（Slice）

```
features/
  add-to-cart/        ← Slice（按功能切分）
    ui/               ← 组件
      AddToCartButton.tsx
    model/            ← 状态（store slice）
      addToCartSlice.ts
    api/              ← 接口
      addToCart.ts
    lib/              ← 工具
    index.ts          ← 公开 API（只导出外部需要的）

entities/
  product/
    ui/
      ProductCard.tsx
      ProductRating.tsx
    model/
      productStore.ts
      types.ts
    api/
      fetchProduct.ts
    index.ts
```

### 切片的公开 API（index.ts）

```typescript
// features/add-to-cart/index.ts — 只暴露外部需要的接口
// 外部模块只能通过这个文件引用，不能直接引用内部文件

export { AddToCartButton } from './ui/AddToCartButton';
export { addToCartSlice, addToCart } from './model/addToCartSlice';
// 不导出内部实现细节（如 cartCalculator 等）
```

```typescript
// ✓ 正确引用方式（通过 index.ts）
import { AddToCartButton } from '@/features/add-to-cart';

// ✗ 错误引用方式（绕过公开 API）
import { AddToCartButton } from '@/features/add-to-cart/ui/AddToCartButton';
// 这样会破坏封装，内部重构时会影响到外部
```

### 实际目录示例

```
src/
  app/
    providers/
      ThemeProvider.tsx
      QueryProvider.tsx
    styles/
      global.css
    router.tsx

  pages/
    home/
      index.tsx           ← 引用 widgets/ProductGrid, widgets/Header
    product/
      [id]/index.tsx      ← 引用 widgets/ProductDetail
    checkout/
      index.tsx

  widgets/
    header/
      ui/Header.tsx       ← 引用 features/auth, entities/user
      index.ts
    product-grid/
      ui/ProductGrid.tsx  ← 引用 entities/product, features/add-to-cart
      model/
      index.ts

  features/
    add-to-cart/
    auth-by-email/
    search-products/
    apply-promo-code/

  entities/
    product/
    user/
    order/
    cart/

  shared/
    ui/                   ← 基础组件（Button, Input, Modal）
    api/                  ← Axios/Fetch 封装
    lib/                  ← 工具函数
    config/               ← 环境变量
```

---

## Monorepo 工具链

### 为什么用 Monorepo

```
多个相关仓库的痛点：
  - 跨仓库改动需要多个 PR，协调成本高
  - 版本依赖管理复杂（A 依赖 B v1.2，C 依赖 B v1.3）
  - 代码共享麻烦（复制粘贴 or 发布 npm 包）
  - CI 无法感知跨仓库的变更影响

Monorepo 优势：
  - 原子提交（一次改动跨多个包）
  - 统一版本、统一工具链
  - 变更影响分析（只 build/test 受影响的包）
```

### Turborepo 配置

```json
// turbo.json
{
  "$schema": "https://turborepo.org/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],   // 先 build 依赖包，再 build 当前包
      "outputs": [".next/**", "dist/**"],
      "cache": true
    },
    "test": {
      "dependsOn": ["^build"],   // 测试前确保依赖已构建
      "cache": true
    },
    "lint": {
      "cache": true
    },
    "dev": {
      "cache": false,
      "persistent": true          // 长进程（dev server）
    }
  },
  "globalEnv": ["NODE_ENV", "NEXT_PUBLIC_API_URL"]
}
```

```
packages/
  ui/               (@myapp/ui)
  config/           (@myapp/config — ESLint、TS 配置)
  utils/            (@myapp/utils)

apps/
  web/              (引用 @myapp/ui, @myapp/utils)
  admin/            (引用 @myapp/ui, @myapp/utils)
  docs/

pnpm-workspace.yaml:
  packages:
    - 'apps/*'
    - 'packages/*'
```

```bash
# 只构建受当前 git diff 影响的包（CI 加速）
turbo build --filter=...[HEAD^1]

# 运行所有 app 的 dev server
turbo dev

# 只测试 web app 及其依赖
turbo test --filter=web...
```

### Nx（更强大的 Monorepo 工具）

```bash
# Nx 自动分析依赖图
npx nx graph  # 可视化依赖关系

# 受影响的包才运行（基于 git diff）
npx nx affected --target=test

# 代码生成（Generator）
npx nx g @nx/react:component Button --project=ui
```

---

## 路径别名（TypeScript Path Mapping）

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@ui/*": ["./packages/ui/src/*"],
      "@/features/*": ["./src/features/*"],
      "@/entities/*": ["./src/entities/*"],
      "@/shared/*": ["./src/shared/*"]
    }
  }
}
```

```typescript
// ✓ 使用别名（可读性好，重构时只改 tsconfig）
import { Button } from '@/shared/ui';
import { ProductCard } from '@/entities/product';
import { AddToCartButton } from '@/features/add-to-cart';

// ✗ 相对路径地狱（难以维护）
import { Button } from '../../../../shared/ui/Button';
```

---

## 依赖方向约束

### ESLint 规则（import/no-restricted-paths）

```javascript
// .eslintrc.js — 自动检测违反 FSD 层级规则的 import
module.exports = {
  rules: {
    'import/no-restricted-paths': ['error', {
      zones: [
        // shared 不能引用上层
        {
          target: './src/shared',
          from: ['./src/features', './src/entities', './src/widgets', './src/pages'],
          message: 'shared layer cannot import from upper layers',
        },
        // entities 不能引用 features/widgets/pages
        {
          target: './src/entities',
          from: ['./src/features', './src/widgets', './src/pages'],
          message: 'entities layer cannot import from features/widgets/pages',
        },
        // features 不能引用 widgets/pages
        {
          target: './src/features',
          from: ['./src/widgets', './src/pages'],
          message: 'features layer cannot import from widgets/pages',
        },
        // 切片之间不能互相引用（同层隔离）
        {
          target: './src/features/add-to-cart',
          from: ['./src/features/!(add-to-cart)/**'],
          message: 'features cannot import from other features (use entities/shared instead)',
        },
      ],
    }],
  },
};
```

### dependency-cruiser（可视化依赖图）

```javascript
// .dependency-cruiser.js
module.exports = {
  forbidden: [
    {
      name: 'no-circular',
      severity: 'error',
      comment: '循环依赖',
      from: {},
      to: { circular: true },
    },
    {
      name: 'not-to-dev-dep',
      severity: 'error',
      comment: '生产代码不能引用 devDependencies',
      from: { pathNot: ['test', 'spec', 'mock'] },
      to: { dependencyTypes: ['devDependency'] },
    },
  ],
};

// 生成依赖关系图
npx depcruise --include-only "^src" --output-type dot src | dot -T svg > dependencies.svg
```

---

## Barrel Exports（桶导出）反模式

```typescript
// ❌ 反模式：所有组件都从根 index 导出
// components/index.ts
export { Button } from './Button';
export { Input } from './Input';
export { Modal } from './Modal';
export { DataTable } from './DataTable';
// ... 100 个组件全在这里

// 问题：
// 1. 引用任何一个组件都会解析整个 barrel 文件
// 2. Tree shaking 在某些情况下失效
// 3. 构建工具需要处理所有导出，即使只用了一个
// 4. 循环依赖更容易发生

// ✓ 推荐：路径直接指向文件，或保持 barrel 小（同一功能内）
import { Button } from '@/shared/ui/Button';  // 直接路径

// 或只在 feature/entity 的公开 API 中用 barrel（控制范围）
// features/add-to-cart/index.ts  ← 这里用 barrel 是合理的
export { AddToCartButton } from './ui/AddToCartButton';
```

---

## 代码规范工具链

```json
// package.json（根目录）
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "typecheck": "tsc --noEmit",
    "format": "prettier --write \"**/*.{ts,tsx,json,md}\"",
    "format:check": "prettier --check \"**/*.{ts,tsx,json,md}\""
  }
}
```

```yaml
# .github/workflows/quality.yml
- name: Lint
  run: pnpm lint
- name: Type Check
  run: pnpm typecheck
- name: Format Check
  run: pnpm format:check
- name: Dependency Constraints
  run: npx depcruise --config .dependency-cruiser.js src
```

### Husky + lint-staged（提交前自动检查）

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": "prettier --write"
  }
}
```

```bash
# 安装 husky
pnpm husky init
echo "pnpm lint-staged" > .husky/pre-commit
```

---

## 面试常见追问

**Q: FSD 和 Atomic Design 有什么区别？**
A: Atomic Design（原子、分子、有机体、模板、页面）是 UI 组件的分层设计方法，关注组件粒度；FSD 是整个应用的架构方法，关注功能模块的组织和依赖方向。两者可以结合：shared/ui 层内部用 Atomic Design 组织，整体架构用 FSD。

**Q: Monorepo 构建速度慢怎么办？**
A: ①Turborepo/Nx 的远程缓存（Remote Cache）：CI 构建结果上传到云端，其他开发者/CI 直接复用（零重复构建）；②只构建/测试受影响的包（`--affected`）；③增量类型检查（TypeScript Project References）；④Docker 层缓存（node_modules 单独一层，代码变化不重新安装依赖）。

## 常见踩坑

**踩坑1：FSD 架构中 feature 层相互引用**
❌ 错误：`features/cart` 直接导入 `features/auth` 的用户状态，形成 feature 层的循环或双向依赖，违反 FSD 单向规则。
✓ 正确：共享状态提取到 `entities/user` 层，feature 只从 entities 或 shared 获取数据，不直接引用其他 feature。
原因：FSD 的核心规则是"高层不能被低层引用"，feature 之间的互相依赖打破了模块边界，导致修改一个影响多个。

**踩坑2：Monorepo 中 workspace 包没有严格的 public API**
❌ 错误：业务代码直接从 `@company/ui/src/components/Button/index.ts` 深路径导入，绕过包的 public API，UI 库重构内部结构后所有消费方编译报错。
✓ 正确：每个 workspace 包的 `package.json` 配置 `exports` 字段明确声明 public entry，内部结构变化不影响外部接口。
原因：包的 public API 是团队间的契约，深路径导入是"实现细节依赖"，违反封装原则。

**踩坑3：组件放错层级（所有组件放 components 目录）**
❌ 错误：所有组件不分业务/通用堆在 `src/components/`，随着项目增长找不到文件，业务逻辑和通用逻辑耦合在一起。
✓ 正确：区分通用 UI 组件（`shared/ui`）、业务实体组件（`entities/product`）、特性组件（`features/checkout`），按 FSD 或类似分层放置。
原因：混放导致两个问题：难以查找（索引性差）+ 难以复用（业务逻辑和通用逻辑耦合，无法单独提取）。

**踩坑4：过早引入 Monorepo 带来巨大配置开销**
❌ 错误：团队 3 人、1 个应用时就上 Turborepo + PNPM Workspace，花大量时间配置 pipeline、包间依赖，实际没有代码需要共享。
✓ 正确：单一应用用单仓库，等到真正需要跨应用共享代码（2+ 个应用、团队 > 10 人）再迁移 Monorepo。
原因：Monorepo 工具链有实质性的配置和维护成本，小团队早期应集中精力在业务而不是工程基础设施。

---

**Q: 大型项目中如何避免循环依赖？**
A: ①FSD 单向依赖规则（根本解法）；②dependency-cruiser 在 CI 中检测；③如果两个 feature 需要互相引用，说明它们应该合并，或者提取公共部分到 entities/shared；④用事件/消息总线解耦（但会使代码更难追踪，谨慎使用）。

**Q: 什么时候用 Monorepo，什么时候用多仓库？**
A: Monorepo 适合：紧密相关的多个应用/包（共享代码多）、需要原子提交、统一技术栈。多仓库适合：完全独立的项目（不共享代码）、不同团队不同发布周期、技术栈差异大。实践中，同一产品的多个端（Web/Admin/Mobile）适合 Monorepo；完全不同的产品线适合多仓库。
