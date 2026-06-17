# Feature Flag / A/B 测试

> Feature Flag（功能开关）让代码发布和功能发布解耦：代码可以上线但功能默认关闭，
> 按用户/比例/地区灰度开放，出问题立即关闭，无需回滚代码。

---

## Feature Flag vs A/B 测试

```
Feature Flag（功能开关）：
  目的：控制功能可见性（发布控制）
  问题：新功能上线了吗？对哪些用户开放？
  例子：新结账流程只对内测用户开启

A/B 测试（实验）：
  目的：数据驱动决策（效果验证）
  问题：方案 A 和方案 B 哪个转化率更高？
  例子：按钮文案"立即购买" vs "加入购物车" 哪个点击率更高

两者共用同一套基础设施（分流逻辑 + 用户分桶 + 数据采集）
```

---

## 分流策略

### 用户分桶（Bucketing）

```typescript
// 确定性哈希分桶：同一用户每次进入同一实验组（避免体验跳变）
function assignBucket(userId: string, experimentId: string, buckets = 100): number {
  // MurmurHash 或 FNV 哈希（低碰撞，分布均匀）
  const hash = murmurhash3(`${userId}:${experimentId}`);
  return Math.abs(hash) % buckets;  // 0-99
}

// 检查是否在实验组
function isInVariant(userId: string, flag: FeatureFlag, variant: string): boolean {
  const bucket = assignBucket(userId, flag.id);
  const allocation = flag.variants.find(v => v.name === variant);
  if (!allocation) return false;

  // 变体 A 占 0-49（50%），变体 B 占 50-99（50%）
  return bucket >= allocation.start && bucket < allocation.end;
}
```

### 分流维度

```typescript
interface FeatureFlag {
  id: string;
  name: string;
  enabled: boolean;
  rules: FlagRule[];
  variants: Variant[];
}

interface FlagRule {
  type: 'user_id' | 'user_email' | 'percentage' | 'attribute' | 'environment';
  operator: 'in' | 'not_in' | 'contains' | 'less_than';
  value: string | string[] | number;
}

// 规则示例
const newCheckoutFlag: FeatureFlag = {
  id: 'new-checkout',
  name: '新结账流程',
  enabled: true,
  rules: [
    { type: 'environment', operator: 'in', value: ['production'] },
    { type: 'user_email', operator: 'contains', value: '@internal.com' },  // 内部员工
  ],
  variants: [
    { name: 'control', start: 0, end: 50 },   // 旧流程 50%
    { name: 'treatment', start: 50, end: 100 }, // 新流程 50%
  ],
};
```

---

## 客户端 vs 服务端 vs 边缘分流

### 客户端分流

```typescript
// SDK 在浏览器端运行，直接计算用户属于哪个变体
import { GrowthBook } from '@growthbook/growthbook-react';

const gb = new GrowthBook({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: 'sdk-xxx',
  attributes: {
    id: userId,
    email: userEmail,
    country: 'CN',
    plan: 'pro',
  },
  trackingCallback: (experiment, result) => {
    // 上报实验曝光（用于统计分析）
    analytics.track('Experiment Viewed', {
      experimentId: experiment.key,
      variationId: result.variationId,
    });
  },
});

await gb.loadFeatures();

// 使用 Flag
const showNewCheckout = gb.isOn('new-checkout');
const buttonText = gb.getFeatureValue('checkout-button-text', 'Buy Now');
```

**优点**：无服务端调用，延迟极低。
**缺点**：Flag 规则暴露在客户端（不能包含敏感逻辑）；存在 FOUC 闪烁问题。

### 服务端分流（推荐用于 SSR）

```typescript
// Next.js App Router — 服务端决策，客户端直接渲染正确版本
import { GrowthBook } from '@growthbook/growthbook';
import { cookies } from 'next/headers';

async function getGrowthBook(userId: string): Promise<GrowthBook> {
  const gb = new GrowthBook({
    apiHost: process.env.GROWTHBOOK_API_HOST,
    clientKey: process.env.GROWTHBOOK_CLIENT_KEY,
    attributes: { id: userId },
  });

  await gb.loadFeatures({ timeout: 1000 });  // 带超时，避免阻塞
  return gb;
}

// page.tsx（Server Component）
export default async function CheckoutPage() {
  const userId = (await cookies()).get('userId')?.value ?? '';
  const gb = await getGrowthBook(userId);

  // 服务端已决策，HTML 直接包含正确内容（无 FOUC）
  if (gb.isOn('new-checkout')) {
    return <NewCheckoutFlow />;
  }
  return <LegacyCheckoutFlow />;
}
```

### 边缘分流（性能最佳）

```typescript
// Vercel Edge Middleware / Cloudflare Worker — 在 CDN 节点分流
// 用户请求到达边缘节点 → 分流决策 → Rewrite 到不同页面
// 无冷启动，延迟 < 5ms，且用户看不到 URL 变化

// middleware.ts（Next.js Edge Middleware）
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export const config = { matcher: ['/checkout'] };

export function middleware(req: NextRequest) {
  const userId = req.cookies.get('userId')?.value;

  // 确定性分桶（无需外部调用）
  const bucket = userId ? hashBucket(userId, 'new-checkout') : Math.random() * 100;

  const isNewCheckout = bucket < 50;  // 50% 进新流程

  const response = NextResponse.rewrite(
    new URL(isNewCheckout ? '/checkout-v2' : '/checkout-v1', req.url)
  );

  // 写入 Cookie，保证用户下次进同一组
  if (!userId) {
    response.cookies.set('userId', crypto.randomUUID(), { maxAge: 365 * 24 * 60 * 60 });
  }

  // 记录分流结果（供分析）
  response.headers.set('X-Experiment-Variant', isNewCheckout ? 'new' : 'control');

  return response;
}
```

---

## 防止 FOUC（UI 闪烁）

```
FOUC 场景：
  1. 服务端返回 HTML（默认旧版本 UI）
  2. JS 加载完毕，Flag SDK 初始化
  3. 发现用户应该看到新版本
  4. 页面突然切换 → 用户看到闪烁

解决方案（优先级从高到低）：
  1. 服务端/边缘分流（根本解法）
  2. 在 <head> 中内联同步脚本（极小，先于渲染执行）
  3. 初始渲染隐藏内容，Flag 加载完后显示
```

```typescript
// 方案 2：内联脚本（极简，< 500 bytes）
// 在 HTML head 中同步执行，避免 FOUC
function getInlineScript(flagKey: string): string {
  return `
    (function() {
      try {
        var flags = JSON.parse(localStorage.getItem('feature-flags') || '{}');
        if (flags['${flagKey}']) {
          document.documentElement.setAttribute('data-flag-${flagKey}', 'true');
        }
      } catch(e) {}
    })();
  `;
}
// CSS 根据 data attribute 控制显示
// [data-flag-new-checkout] .new-checkout { display: block; }
// [data-flag-new-checkout] .legacy-checkout { display: none; }
```

---

## GrowthBook（开源）

```typescript
// 完整集成示例（React + Next.js）
import { GrowthBookProvider, useFeature, useFeatureIsOn } from '@growthbook/growthbook-react';

// _app.tsx 或 layout.tsx
export function Providers({ children }: { children: React.ReactNode }) {
  const gb = useRef(new GrowthBook({
    apiHost: process.env.NEXT_PUBLIC_GROWTHBOOK_HOST,
    clientKey: process.env.NEXT_PUBLIC_GROWTHBOOK_KEY,
    enableDevMode: process.env.NODE_ENV !== 'production',
    subscribeToChanges: true,  // 实时更新（无需刷新）
  }));

  const { user } = useSession();

  useEffect(() => {
    gb.current.setAttributes({
      id: user?.id,
      email: user?.email,
      plan: user?.plan,
      country: navigator.language.split('-')[1],
    });
    gb.current.loadFeatures();
  }, [user]);

  return (
    <GrowthBookProvider growthbook={gb.current}>
      {children}
    </GrowthBookProvider>
  );
}

// 组件中使用
function PricingPage() {
  const showAnnualDiscount = useFeatureIsOn('annual-discount-banner');
  const ctaText = useFeature('pricing-cta-text').value ?? 'Get Started';

  return (
    <div>
      {showAnnualDiscount && <AnnualDiscountBanner />}
      <Button>{ctaText}</Button>
    </div>
  );
}
```

### GrowthBook vs LaunchDarkly

| 维度 | GrowthBook | LaunchDarkly |
|------|-----------|-------------|
| 定价 | 开源免费（可自托管）| SaaS，按 MAU 收费 |
| 统计分析 | 内置贝叶斯统计 | 需要额外配置 |
| 部署 | 自托管 / 托管服务 | 纯 SaaS |
| SDK 数量 | 20+ 语言 | 30+ 语言 |
| 适合 | 中小团队、数据敏感、预算有限 | 大型企业、需要 SLA |

---

## A/B 测试统计基础

```
实验设计：
  假设：新按钮文案提升点击率
  指标：点击率（Primary）/ 转化率（Secondary）
  样本量：使用功率分析计算（避免样本量不足导致假阳性）
  运行时间：至少 2 周（捕获周末效应）

统计显著性：
  p-value < 0.05（5% 显著性水平）→ 拒绝零假设
  置信区间：95% 置信区间不含 0 → 效果显著

GrowthBook 贝叶斯方法（更直观）：
  "变体 B 优于控制组的概率是 95%"
  "预期提升：+12%（95% CI: +8% ~ +16%）"
```

---

## 实验数据上报

```typescript
// 曝光上报（用户看到了实验）
function trackExperimentView(experimentId: string, variantId: string) {
  analytics.track('$experiment_started', {
    experiment_id: experimentId,
    variant_id: variantId,
    timestamp: Date.now(),
    session_id: getSessionId(),
    // 关联业务指标需要的字段
    user_id: getCurrentUserId(),
    page: window.location.pathname,
  });
}

// 转化上报（用户完成了目标行为）
function trackConversion(goalId: string) {
  analytics.track('$goal_completed', {
    goal_id: goalId,
    timestamp: Date.now(),
    session_id: getSessionId(),
  });
}
```

---

## 面试常见追问

**Q: Feature Flag 和 Git Feature Branch 的区别？**
A: Feature Branch 是代码隔离（合并前独立），合并到 main 就面向所有用户。Feature Flag 是运行时隔离，代码合并到 main 但功能默认关闭，按规则灰度开放。Feature Flag 让"主干开发（Trunk-Based Development）"成为可能，CI/CD 更简洁。

**Q: A/B 测试中如何避免"多重比较问题"（看谁赢就停止）？**
A: 提前用统计功率分析确定样本量，达到样本量再看结果（不要中途窥视）；或使用序贯测试方法（如 mSPRT）允许中途观察。GrowthBook 的贝叶斯方法天然适合中途查看，因为它给出的是概率而不是 p-value。

**Q: 如何处理 Flag 清理（Technical Debt）？**
A: ①每个 Flag 创建时设置 TTL（预期删除日期）；②CI 中检测超期 Flag 并告警；③Flag 关闭后，旧代码路径用 Codemod 自动删除。未清理的 Flag 积累会让代码难以理解（"这个 if 还需要吗？"）。

**Q: 边缘分流有什么限制？**
A: Edge Middleware 运行在受限环境（无 Node.js API、内存限制、无数据库访问），只能做简单的哈希分流。复杂规则（如"只对 Pro 用户开启"）需要先把用户属性存入 Cookie，边缘节点读 Cookie 来决策。
