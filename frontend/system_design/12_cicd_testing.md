# 前端 CI/CD 与测试策略

> 好的测试策略让你敢于重构；好的 CI/CD 让发布变成日常操作而不是惊险时刻。
> 本文覆盖：测试金字塔、Vitest、Playwright、视觉回归、Lighthouse CI、Preview Deploy。

---

## 测试金字塔

```
        /\
       /E2E\         少量 — 慢、成本高、接近真实（Playwright）
      /------\
     /Integration\   适量 — 测试组件交互（Testing Library）
    /------------\
   /  Unit Tests  \  大量 — 快、便宜（Vitest）
  /________________\

原则：
  越底层的测试越多，越上层的测试越少
  不要为了覆盖率而写测试，要为信心而写
```

---

## 单元测试：Vitest

### 为什么选 Vitest 而不是 Jest

```
Vitest vs Jest：
  ✓ 原生 ESM 支持（Jest 需要 Babel 转换）
  ✓ 与 Vite 共享配置（无需重复配置 TypeScript / Path alias）
  ✓ 速度更快（HMR + 并行执行）
  ✓ API 兼容 Jest（迁移成本低）
  ✓ 内置 UI 模式（vitest --ui）
```

### 配置

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: 'jsdom',      // 模拟浏览器环境
    globals: true,             // describe/it/expect 无需 import
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 70,
      },
    },
  },
});

// src/test/setup.ts
import '@testing-library/jest-dom';  // expect(el).toBeInTheDocument() 等 matcher
import { vi } from 'vitest';

// 全局 Mock（所有测试都需要的）
vi.mock('next/navigation', () => ({
  useRouter: () => ({ push: vi.fn(), replace: vi.fn() }),
  usePathname: () => '/',
  useSearchParams: () => new URLSearchParams(),
}));
```

### 纯函数测试

```typescript
// utils/price.test.ts
import { describe, it, expect } from 'vitest';
import { formatPrice, calculateDiscount, applyPromoCode } from './price';

describe('formatPrice', () => {
  it('formats integer price with 2 decimal places', () => {
    expect(formatPrice(1000)).toBe('¥1,000.00');
  });

  it('handles zero correctly', () => {
    expect(formatPrice(0)).toBe('¥0.00');
  });

  it('formats negative price for refunds', () => {
    expect(formatPrice(-50.5)).toBe('-¥50.50');
  });
});

describe('applyPromoCode', () => {
  const cart = { items: [{ price: 100 }, { price: 200 }], total: 300 };

  it('applies percentage discount', () => {
    const result = applyPromoCode(cart, { type: 'percentage', value: 10 });
    expect(result.total).toBe(270);
    expect(result.discount).toBe(30);
  });

  it('rejects expired promo code', () => {
    const expired = { type: 'percentage', value: 20, expiresAt: new Date('2020-01-01') };
    expect(() => applyPromoCode(cart, expired)).toThrow('Promo code expired');
  });
});
```

### React 组件测试（Testing Library）

```typescript
// components/SearchInput/SearchInput.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { vi, describe, it, expect, beforeEach } from 'vitest';
import { SearchInput } from './SearchInput';

// Mock 防抖（避免等待真实延迟）
vi.mock('lodash-es/debounce', () => ({
  default: (fn: Function) => fn,  // 直接执行，不防抖
}));

describe('SearchInput', () => {
  const mockOnSearch = vi.fn();

  beforeEach(() => mockOnSearch.mockClear());

  it('renders placeholder correctly', () => {
    render(<SearchInput onSearch={mockOnSearch} placeholder="Search products" />);
    expect(screen.getByPlaceholderText('Search products')).toBeInTheDocument();
  });

  it('calls onSearch when user types', async () => {
    const user = userEvent.setup();
    render(<SearchInput onSearch={mockOnSearch} />);

    await user.type(screen.getByRole('searchbox'), 'iPhone');

    await waitFor(() => {
      expect(mockOnSearch).toHaveBeenCalledWith('iPhone');
    });
  });

  it('clears input when clear button clicked', async () => {
    const user = userEvent.setup();
    render(<SearchInput onSearch={mockOnSearch} />);

    const input = screen.getByRole('searchbox');
    await user.type(input, 'test');
    await user.click(screen.getByRole('button', { name: /clear/i }));

    expect(input).toHaveValue('');
    expect(mockOnSearch).toHaveBeenLastCalledWith('');
  });

  it('shows loading state when isLoading prop is true', () => {
    render(<SearchInput onSearch={mockOnSearch} isLoading />);
    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });
});
```

### Hooks 测试

```typescript
// hooks/useCart.test.ts
import { renderHook, act } from '@testing-library/react';
import { useCart } from './useCart';

describe('useCart', () => {
  it('adds item to cart', () => {
    const { result } = renderHook(() => useCart());

    act(() => {
      result.current.addItem({ id: '1', name: 'iPhone', price: 999, quantity: 1 });
    });

    expect(result.current.items).toHaveLength(1);
    expect(result.current.total).toBe(999);
  });

  it('increments quantity for existing item', () => {
    const { result } = renderHook(() => useCart());
    const item = { id: '1', name: 'iPhone', price: 999, quantity: 1 };

    act(() => { result.current.addItem(item); });
    act(() => { result.current.addItem(item); });

    expect(result.current.items[0].quantity).toBe(2);
    expect(result.current.total).toBe(1998);
  });
});
```

---

## 集成测试：Testing Library + MSW

```typescript
// MSW（Mock Service Worker）— 在测试中拦截真实 HTTP 请求
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/products', () => {
    return HttpResponse.json([
      { id: '1', name: 'iPhone 16', price: 5999 },
      { id: '2', name: 'MacBook Pro', price: 14999 },
    ]);
  }),

  http.post('/api/cart', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ success: true, cartId: 'cart-123', ...body });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// ProductList 集成测试（真实 fetch + 真实渲染）
describe('ProductList', () => {
  it('loads and displays products', async () => {
    render(<ProductList />);

    // 初始 loading 状态
    expect(screen.getByRole('progressbar')).toBeInTheDocument();

    // 等待数据加载
    await screen.findByText('iPhone 16');
    expect(screen.getByText('¥5,999.00')).toBeInTheDocument();
    expect(screen.getByText('MacBook Pro')).toBeInTheDocument();
  });

  it('shows error state on API failure', async () => {
    server.use(
      http.get('/api/products', () => new HttpResponse(null, { status: 500 }))
    );

    render(<ProductList />);
    await screen.findByText(/failed to load/i);
  });
});
```

---

## E2E 测试：Playwright

### 配置

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,  // CI 中重试 2 次（防止偶发失败）
  workers: process.env.CI ? 1 : undefined,
  fullyParallel: true,
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['github'],  // GitHub Actions 注释
  ],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',    // 失败时录制 trace
    screenshot: 'only-on-failure',
    video: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile', use: { ...devices['iPhone 14'] } },
  ],
  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Page Object Model（POM）

```typescript
// e2e/pages/CheckoutPage.ts
import { type Page, type Locator, expect } from '@playwright/test';

export class CheckoutPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly addressInput: Locator;
  readonly submitButton: Locator;
  readonly successMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.addressInput = page.getByLabel('Shipping Address');
    this.submitButton = page.getByRole('button', { name: 'Place Order' });
    this.successMessage = page.getByRole('alert', { name: 'Order Confirmed' });
  }

  async goto() {
    await this.page.goto('/checkout');
  }

  async fillShipping(data: { email: string; address: string }) {
    await this.emailInput.fill(data.email);
    await this.addressInput.fill(data.address);
  }

  async placeOrder() {
    await this.submitButton.click();
    await expect(this.successMessage).toBeVisible({ timeout: 10_000 });
  }
}

// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';
import { CheckoutPage } from './pages/CheckoutPage';

test.describe('Checkout Flow', () => {
  test('completes checkout successfully', async ({ page }) => {
    const checkout = new CheckoutPage(page);
    await checkout.goto();

    await checkout.fillShipping({
      email: 'test@example.com',
      address: '北京市朝阳区某路 1 号',
    });

    await checkout.placeOrder();

    // 验证跳转到确认页
    await expect(page).toHaveURL(/\/order-confirmation/);
  });

  test('shows validation error for invalid email', async ({ page }) => {
    const checkout = new CheckoutPage(page);
    await checkout.goto();
    await checkout.emailInput.fill('not-an-email');
    await checkout.submitButton.click();

    await expect(page.getByText('Please enter a valid email')).toBeVisible();
  });
});
```

### API Mock（Playwright Route）

```typescript
// 在 E2E 测试中 Mock 外部 API（避免依赖真实后端）
test('shows payment failure message', async ({ page }) => {
  // 拦截支付 API，返回失败
  await page.route('/api/payment', async route => {
    await route.fulfill({
      status: 402,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Insufficient funds' }),
    });
  });

  await page.goto('/checkout');
  await page.getByRole('button', { name: 'Pay Now' }).click();
  await expect(page.getByText('Payment failed: Insufficient funds')).toBeVisible();
});
```

---

## 视觉回归测试：Chromatic

```typescript
// Storybook + Chromatic — 捕获 UI 意外变化

// 每次 PR：
// 1. Chromatic 对所有 Stories 截图
// 2. 与基准线对比
// 3. 有 UI 变化 → 在 PR 中标记，需要人工审核（"接受"或"拒绝"）

// package.json
"chromatic": "npx chromatic --project-token=$CHROMATIC_PROJECT_TOKEN"

// GitHub Actions
- name: Publish to Chromatic
  uses: chromaui/action@v1
  with:
    projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
    onlyChanged: true  # 只测试变化的组件（加速）
    exitZeroOnChanges: true  # UI 变化不阻塞 CI（等人工审核）
```

### Playwright 视觉快照

```typescript
// 不依赖 Storybook 的视觉回归方案
test('product card renders correctly', async ({ page }) => {
  await page.goto('/products/1');

  const card = page.locator('[data-testid="product-card"]');

  // 首次运行生成基准截图，后续对比
  await expect(card).toHaveScreenshot('product-card.png', {
    maxDiffPixelRatio: 0.02,  // 允许 2% 像素差异（字体渲染差异）
    threshold: 0.1,
  });
});
```

---

## Lighthouse CI

```typescript
// .lighthouserc.js — 在 CI 中自动跑 Lighthouse，阈值不达标则失败
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/products',
        'http://localhost:3000/checkout',
      ],
      numberOfRuns: 3,  // 跑 3 次取中位数（减少噪音）
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.85 }],   // 性能 > 85
        'categories:accessibility': ['error', { minScore: 0.9 }],  // 无障碍 > 90
        'categories:best-practices': ['warn', { minScore: 0.9 }],
        'categories:seo': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
      },
    },
    upload: {
      target: 'lhci',
      serverBaseUrl: 'https://lhci.internal.example.com',  // 自托管 LHCI Server
    },
  },
};
```

---

## Preview Deployment（预览部署）

```yaml
# .github/workflows/preview.yml — 每个 PR 自动部署预览环境
name: Preview Deploy
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Vercel
        id: deploy
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          scope: ${{ secrets.VERCEL_ORG_ID }}

      - name: Comment PR with Preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Preview Deployment\n\n✅ Preview URL: ${{ steps.deploy.outputs.preview-url }}`
            });

      - name: Run Lighthouse on Preview
        uses: treosh/lighthouse-ci-action@v11
        with:
          urls: ${{ steps.deploy.outputs.preview-url }}
          configPath: .lighthouserc.js
```

---

## 完整 CI 流水线

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  lint-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint          # ESLint
      - run: pnpm typecheck     # tsc --noEmit

  unit-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - run: pnpm install --frozen-lockfile
      - run: pnpm test --coverage
      - uses: codecov/codecov-action@v4  # 上传覆盖率报告

  e2e:
    runs-on: ubuntu-latest
    needs: [lint-typecheck]  # 先通过 lint 再跑 E2E
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - run: pnpm install --frozen-lockfile
      - run: pnpm playwright install --with-deps chromium
      - run: pnpm build
      - run: pnpm e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  bundle-size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

---

## 面试常见追问

**Q: 测试覆盖率 80% 够吗？**
A: 覆盖率是必要非充分条件。80% 覆盖率可以有很多无意义的测试（只 cover 代码行，不测行为），也可以 60% 覆盖率但测了所有关键路径。更重要的是：核心业务逻辑（支付、认证、数据校验）接近 100%；UI 展示逻辑适量；工具函数高覆盖。

**Q: E2E 测试太慢，如何优化？**
A: ①并行执行（多 Worker）；②只对关键路径跑 E2E，其他用集成测试替代；③用 API 直接 setup 测试数据（不用 UI 操作创建用户/订单）；④Feature Branch 只跑受影响的测试（Playwright 的 `--shard` 分片）；⑤预构建好的 App，不在 CI 中实时编译。

**Q: Preview Deploy 和 Staging 环境的区别？**
A: Preview Deploy 是每个 PR 独立的临时环境（合并后自动删除），用于 CR 审查和快速验证；Staging 是长期存在的共享环境，接近生产配置，用于集成测试和 QA 验收。两者不互相替代。

**Q: 如何防止测试代码腐烂（测试越来越难维护）？**
A: ①测试行为而不是实现（不要 mock 内部方法，通过公开 API 测试）；②用 Page Object Model 封装 UI 细节；③测试命名描述用户故事（"when user clicks submit, shows success message"）；④删除重复的无意义测试，宁少勿滥。
