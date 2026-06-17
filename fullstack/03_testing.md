# 全栈测试策略

> Vitest + React Testing Library + MSW v2 + Playwright。
> 覆盖 Next.js App Router 的测试难点：Server Components、Server Actions、tRPC。

---

## 全栈测试分层

```
                    /\
                   /E2E\
                  / Playwright /   完整用户流程（登录、下单、结账）
                /──────────────\
               / Integration   \  API 路由 + DB（TestContainers）
              /──────────────────\
             /  Component Tests  \ React 组件（RTL + MSW）
            /────────────────────\
           /    Unit Tests        \ 纯函数、hooks、utils
          /──────────────────────────\

前端特有挑战：
  - Server Components 无法用 React Testing Library 测（没有 DOM）
  - Server Actions 需要单独测
  - tRPC 测试需要模拟 context
```

---

## Vitest 全栈配置（Monorepo）

```typescript
// apps/web/vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',  // 浏览器环境（DOM API）
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      reporter: ['text', 'lcov'],
      exclude: ['**/*.stories.tsx', '**/*.test.tsx', 'src/test/**'],
    },
  },
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
    },
  },
});

// apps/api/vitest.config.ts（Node.js 环境）
export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./src/test/setup.ts'],
  },
});
```

```typescript
// apps/web/src/test/setup.ts
import '@testing-library/jest-dom';  // .toBeInTheDocument() 等 matcher
import { setupServer } from 'msw/node';
import { handlers } from './msw-handlers';

export const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// 模拟 next/navigation（测试中无法真实路由）
vi.mock('next/navigation', () => ({
  useRouter: () => ({ push: vi.fn(), back: vi.fn(), refresh: vi.fn() }),
  useSearchParams: () => new URLSearchParams(),
  usePathname: () => '/',
  redirect: vi.fn(),
}));

// 模拟 next/cache
vi.mock('next/cache', () => ({
  revalidatePath: vi.fn(),
  revalidateTag: vi.fn(),
}));
```

---

## Client Component 测试（RTL + MSW）

```typescript
// src/components/__tests__/UserProfile.test.tsx
import { render, screen, waitFor, userEvent } from '../test-utils';  // 自定义封装
import { UserProfile } from '../UserProfile';
import { http, HttpResponse } from 'msw';
import { server } from '../../test/setup';

// 自定义 render 封装（包含必要的 Providers）
// src/test/test-utils.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { SessionProvider } from 'next-auth/react';
import { type Session } from 'next-auth';

const mockSession: Session = {
  user: { id: '1', email: 'test@example.com', name: 'Test User', role: 'user' },
  expires: new Date(Date.now() + 86400000).toISOString(),
};

export function render(ui: React.ReactElement, options?: { session?: Session | null }) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },  // 测试中不重试
  });

  return rtlRender(
    <SessionProvider session={options?.session ?? mockSession}>
      <QueryClientProvider client={queryClient}>
        {ui}
      </QueryClientProvider>
    </SessionProvider>
  );
}

// --- 测试 ---
describe('UserProfile', () => {
  it('displays user info after loading', async () => {
    server.use(
      http.get('/api/users/1', () =>
        HttpResponse.json({ id: '1', name: 'Alice', email: 'alice@example.com', bio: 'Hello' })
      )
    );

    render(<UserProfile userId="1" />);

    // 加载状态
    expect(screen.getByTestId('profile-skeleton')).toBeInTheDocument();

    // 等待数据加载
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });
    expect(screen.getByText('alice@example.com')).toBeInTheDocument();
  });

  it('shows error state on network failure', async () => {
    server.use(
      http.get('/api/users/1', () => HttpResponse.json({ error: 'Not found' }, { status: 404 }))
    );

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText(/user not found/i)).toBeInTheDocument();
    });
  });

  it('updates bio on form submit', async () => {
    const user = userEvent.setup();
    const updateHandler = vi.fn();

    server.use(
      http.get('/api/users/1', () =>
        HttpResponse.json({ id: '1', name: 'Alice', bio: 'Old bio' })
      ),
      http.patch('/api/users/1', async ({ request }) => {
        const body = await request.json();
        updateHandler(body);
        return HttpResponse.json({ id: '1', name: 'Alice', bio: body.bio });
      })
    );

    render(<UserProfile userId="1" />);
    await screen.findByText('Old bio');

    await user.click(screen.getByRole('button', { name: /edit/i }));
    await user.clear(screen.getByLabelText(/bio/i));
    await user.type(screen.getByLabelText(/bio/i), 'New bio');
    await user.click(screen.getByRole('button', { name: /save/i }));

    await waitFor(() => {
      expect(updateHandler).toHaveBeenCalledWith({ bio: 'New bio' });
    });
    expect(screen.getByText('New bio')).toBeInTheDocument();
  });
});
```

---

## Server Actions 测试

```typescript
// src/actions/__tests__/user.actions.test.ts
// Server Actions 是普通 async 函数，可以直接调用测试
import { vi, describe, it, expect, beforeEach } from 'vitest';
import { updateProfile } from '../user.actions';

// Mock Next.js server-only utilities
vi.mock('next/cache', () => ({ revalidatePath: vi.fn() }));
vi.mock('next/navigation', () => ({ redirect: vi.fn() }));
vi.mock('@/auth', () => ({
  auth: vi.fn().mockResolvedValue({
    user: { id: 'user-1', email: 'test@example.com', role: 'user' },
  }),
}));
vi.mock('@/lib/prisma', () => ({
  prisma: {
    user: {
      update: vi.fn().mockResolvedValue({ id: 'user-1', name: 'New Name' }),
    },
  },
}));

import { revalidatePath } from 'next/cache';
import { prisma } from '@/lib/prisma';

describe('updateProfile', () => {
  it('updates user name and revalidates', async () => {
    const formData = new FormData();
    formData.append('name', 'New Name');

    await updateProfile(formData);

    expect(prisma.user.update).toHaveBeenCalledWith({
      where: { id: 'user-1' },
      data: { name: 'New Name' },
    });
    expect(revalidatePath).toHaveBeenCalledWith('/profile');
  });

  it('returns error for unauthenticated user', async () => {
    vi.mocked(auth).mockResolvedValueOnce(null);

    const formData = new FormData();
    formData.append('name', 'New Name');

    const result = await updateProfile(formData);
    expect(result?.error).toBe('Unauthorized');
  });
});
```

---

## tRPC 测试

```typescript
// src/server/routers/__tests__/user.router.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { createCallerFactory } from '@trpc/server';
import { appRouter } from '../index';
import { createTRPCContext } from '../context';

const createCaller = createCallerFactory(appRouter);

describe('userRouter', () => {
  describe('user.list', () => {
    it('returns paginated users', async () => {
      // 创建有鉴权的 context
      const ctx = await createTRPCContext({
        session: { user: { id: 'admin-1', role: 'admin' } } as any,
      });
      const caller = createCaller(ctx);

      const result = await caller.user.list({ page: 1, limit: 10 });

      expect(result.users).toBeInstanceOf(Array);
      expect(result.total).toBeGreaterThanOrEqual(0);
    });
  });

  describe('user.create', () => {
    it('throws UNAUTHORIZED for unauthenticated users', async () => {
      const ctx = await createTRPCContext({ session: null });
      const caller = createCaller(ctx);

      await expect(
        caller.user.create({ email: 'test@example.com', name: 'Test', password: 'pass123' })
      ).rejects.toThrowError(/UNAUTHORIZED/);
    });
  });
});
```

---

## Playwright E2E 测试

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [['html'], ['github']],  // CI 中用 github reporter
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'Mobile Chrome', use: { ...devices['Pixel 5'] } },
  ],
  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

```typescript
// e2e/auth.spec.ts —— Page Object Model 模式
import { test, expect, type Page } from '@playwright/test';

class LoginPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Sign in' }).click();
  }

  async expectError(message: string) {
    await expect(this.page.getByRole('alert')).toContainText(message);
  }
}

class DashboardPage {
  constructor(private page: Page) {}

  async expectLoaded() {
    await expect(this.page).toHaveURL('/dashboard');
    await expect(this.page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
  }
}

test.describe('Authentication', () => {
  let loginPage: LoginPage;
  let dashboardPage: DashboardPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    dashboardPage = new DashboardPage(page);
    await loginPage.goto();
  });

  test('redirects to dashboard on successful login', async ({ page }) => {
    await loginPage.login('test@example.com', 'password123');
    await dashboardPage.expectLoaded();
  });

  test('shows error on invalid credentials', async () => {
    await loginPage.login('test@example.com', 'wrongpassword');
    await loginPage.expectError('Invalid email or password');
  });

  test('redirects unauthenticated user to login', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page).toHaveURL(/\/login/);
  });
});

// e2e/checkout.spec.ts —— 复杂用户流程
test.describe('Checkout flow', () => {
  test.beforeEach(async ({ page }) => {
    // 使用 storage state 快速登录（不走 UI 登录流程）
    await page.goto('/api/test/login?userId=test-user');  // 测试专用端点
  });

  test('completes purchase end-to-end', async ({ page }) => {
    await page.goto('/products');
    await page.getByTestId('product-card').first().click();
    await page.getByRole('button', { name: 'Add to cart' }).click();
    await page.goto('/cart');
    await page.getByRole('button', { name: 'Checkout' }).click();

    // 填写收货地址
    await page.getByLabel('Address').fill('123 Main St');
    await page.getByLabel('City').fill('New York');

    // 模拟 Stripe（测试环境用测试卡号）
    await page.getByLabel('Card number').fill('4242424242424242');
    await page.getByLabel('Expiry').fill('12/26');
    await page.getByLabel('CVC').fill('123');

    await page.getByRole('button', { name: 'Place order' }).click();

    await expect(page.getByRole('heading', { name: 'Order confirmed' })).toBeVisible({
      timeout: 10_000,
    });
    await expect(page).toHaveURL(/\/orders\/\w+/);
  });
});
```

---

## CI 配置（GitHub Actions）

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]

jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo test:unit
      - uses: codecov/codecov-action@v4

  integration-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    env:
      DATABASE_URL: postgresql://test:test@localhost:5432/testdb
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm prisma migrate deploy
      - run: pnpm turbo test:integration

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm build
      - run: pnpm exec playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

---

## 面试追问

**Q: Server Components 如何测试？**
A: Server Components 是普通 async 函数，可以直接调用并断言返回值，不需要 DOM 渲染。但 Next.js 的特殊 API（`headers()`、`cookies()`、`auth()`）需要 mock。目前 React Testing Library 不原生支持 Server Components（需要 Next.js 的测试运行器 `@testing-library/nextjs`），通常对 Server Components 做集成测试（Playwright 测渲染结果）比单元测试更实用。

**Q: MSW v2 和 v1 有什么区别？**
A: v2 最大变化：API 从 `rest.get/post` 改为 `http.get/post`，响应从 `res(ctx.json({}))` 改为 `HttpResponse.json({})`；WebSocket 支持（`ws`）；ESM 原生支持；响应更接近真实 Fetch API（`Request`/`Response` 对象）。MSW 在网络层拦截（Service Worker 或 Node.js 拦截），不 mock 模块，测试更真实。

**Q: Playwright 和 Cypress 怎么选？**
A: Playwright：多浏览器（Chrome/Firefox/Safari）并行测试，速度更快，支持 Component Testing，Next.js 官方推荐；Cypress：生态成熟，调试体验好（时间旅行调试），在国内文档资源更多。新项目建议 Playwright，迁移成本高时保留 Cypress。
