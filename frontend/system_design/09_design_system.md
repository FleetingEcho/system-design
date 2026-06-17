# Design System / 组件库架构

> Design System 不只是一套 UI 组件，它是**设计语言、开发规范、工程基础设施**的三位一体。
> 本文覆盖：Design Token 体系、组件库架构、Storybook、发包策略、版本管理、Monorepo 组织。

---

## 面试框架（45分钟怎么答）

**第一步（开场）**：先说清 Design System 的三层价值——统一视觉语言（Design Token）+ 复用组件（减少 Bug）+ 工程基础设施（Storybook/Changesets）
**第二步（核心）**：Design Token 三层模型（Primitive → Semantic → Component）；Radix UI 无障碍原语 + shadcn/ui 实现层；CSS Variables 主题切换
**第三步（深挖）**：Changesets 语义化版本管理（patch/minor/major）；Storybook 视觉回归测试（Chromatic）；如何让消费方平滑升级（migration guide + codemods）
**差异化得分点**：提出 Token 命名规范（不用 blue-500，用 color-primary-default）；能解释 Radix UI 如何解决键盘导航和 ARIA 问题

---

## 架构图：Design System 三层模型

```mermaid
graph TD
    subgraph Layer1["第一层：Primitive Tokens 原始值"]
        P1[color-blue-500: #3b82f6]
        P2[spacing-4: 16px]
        P3[font-size-base: 16px]
        P4[radius-md: 6px]
    end

    subgraph Layer2["第二层：Semantic Tokens 语义值"]
        S1[color-primary-default → color-blue-500]
        S2[color-text-secondary → color-gray-600]
        S3[spacing-component-padding → spacing-4]
    end

    subgraph Layer3["第三层：Component Tokens 组件值"]
        C1[button-bg-primary → color-primary-default]
        C2[button-radius → radius-md]
        C3[input-border-color → color-border-default]
    end

    subgraph Output["产物"]
        CSS[CSS Variables :root{}]
        SD[Style Dictionary JSON → CSS / iOS / Android]
        Storybook[Storybook 组件文档]
    end

    Layer1 --> Layer2
    Layer2 --> Layer3
    Layer3 --> Output
```

---

## 为什么需要 Design System

### 没有 Design System 的问题

```
10 个团队，10 种 Button：
  团队A: <button style="background: #1677ff; border-radius: 4px">
  团队B: <button style="background: #0052cc; border-radius: 6px">
  团队C: <Button variant="primary"> （自己封装的）

问题：
  - 视觉不一致，品牌认知碎片化
  - 同样的 Modal 写了 8 遍，各有各的 Bug
  - 设计师改了品牌主色，需要改 200 个文件
  - 新人入职不知道用哪个组件
```

### Design System 的价值

```
统一的 Design Token（颜色/间距/字体）
  ↓
基础组件库（Button/Input/Modal...）
  ↓
业务组件库（SearchBar/OrderCard...）
  ↓
各业务团队（直接使用，专注业务）

好处：
  品牌主色改一个 Token，全局生效
  无障碍（a11y）修复一次，所有团队受益
  设计师和开发者说同一种语言
```

---

## Design Token 体系

### Token 分层架构（三层模型）

```
Layer 1 — Global Token（原始值）
  color-blue-500: #1677ff
  color-blue-600: #0958d9
  spacing-4: 16px
  font-size-base: 14px

Layer 2 — Semantic Token（语义化，绑定用途）
  color-primary: {color-blue-500}
  color-primary-hover: {color-blue-600}
  spacing-component-padding: {spacing-4}

Layer 3 — Component Token（组件级）
  button-primary-bg: {color-primary}
  button-primary-bg-hover: {color-primary-hover}
  button-padding-x: {spacing-component-padding}
```

**为什么要三层？**
- 改品牌主色：只改 `color-primary` → 所有用到它的组件自动更新
- 换主题（深色/浅色）：只替换 Semantic Token 层 → Global Token 不动
- 组件微调：改 Component Token → 不影响其他组件

### TypeScript + CSS Variables 实现

```typescript
// tokens/global.ts — 原始值（单一来源）
export const globalTokens = {
  color: {
    blue: {
      50: '#e6f4ff',
      100: '#bae0ff',
      500: '#1677ff',
      600: '#0958d9',
      900: '#003eb3',
    },
    red: { 500: '#ff4d4f', 600: '#f5222d' },
    gray: { 100: '#f5f5f5', 500: '#8c8c8c', 900: '#141414' },
  },
  spacing: { 1: '4px', 2: '8px', 3: '12px', 4: '16px', 6: '24px', 8: '32px' },
  fontSize: { sm: '12px', base: '14px', lg: '16px', xl: '20px', '2xl': '24px' },
  radius: { sm: '4px', md: '6px', lg: '8px', full: '9999px' },
  shadow: {
    sm: '0 1px 2px rgba(0,0,0,0.05)',
    md: '0 4px 6px rgba(0,0,0,0.07)',
  },
} as const;

// tokens/semantic.ts — 语义化映射
export const lightTheme = {
  'color-primary': globalTokens.color.blue[500],
  'color-primary-hover': globalTokens.color.blue[600],
  'color-danger': globalTokens.color.red[500],
  'color-text': globalTokens.color.gray[900],
  'color-text-secondary': globalTokens.color.gray[500],
  'color-bg': '#ffffff',
  'color-bg-elevated': globalTokens.color.gray[100],
  'color-border': '#d9d9d9',
};

export const darkTheme: typeof lightTheme = {
  'color-primary': globalTokens.color.blue[500],
  'color-primary-hover': globalTokens.color.blue[100],
  'color-danger': globalTokens.color.red[500],
  'color-text': 'rgba(255,255,255,0.85)',
  'color-text-secondary': 'rgba(255,255,255,0.45)',
  'color-bg': '#141414',
  'color-bg-elevated': '#1f1f1f',
  'color-border': '#303030',
};

// utils/applyTheme.ts — 注入 CSS Variables
export function applyTheme(theme: typeof lightTheme, root = document.documentElement) {
  Object.entries(theme).forEach(([key, value]) => {
    root.style.setProperty(`--${key}`, value);
  });
}

// 切换主题
applyTheme(darkTheme);
```

```css
/* 组件 CSS 使用变量（自动跟随主题切换）*/
.button-primary {
  background: var(--color-primary);
  color: #fff;
  border-radius: var(--radius-md, 6px);
}
.button-primary:hover {
  background: var(--color-primary-hover);
}
```

### Style Dictionary（Token 多平台输出）

```javascript
// style-dictionary.config.js
// 一份 Token 定义 → 输出 CSS Variables / iOS Swift / Android XML / JS 对象
module.exports = {
  source: ['tokens/**/*.json'],
  platforms: {
    css: {
      transformGroup: 'css',
      prefix: 'ds',
      buildPath: 'dist/css/',
      files: [{ destination: 'variables.css', format: 'css/variables' }],
    },
    ios: {
      transformGroup: 'ios-swift',
      buildPath: 'dist/ios/',
      files: [{ destination: 'StyleDictionary.swift', format: 'ios-swift/class.swift' }],
    },
    js: {
      transformGroup: 'js',
      buildPath: 'dist/js/',
      files: [{ destination: 'tokens.js', format: 'javascript/es6' }],
    },
  },
};
```

---

## 组件库架构

### 组件分类

```
Primitive（原子组件）
  Button / Input / Checkbox / Radio / Select
  特点：无业务逻辑，高度可定制，a11y 完善

Composite（复合组件）
  Form / DatePicker / Table / Modal
  特点：由 Primitive 组合，有内部状态

Pattern（模式组件）
  SearchWithFilter / PaginatedList / CRUDTable
  特点：有业务模式，跨项目复用

Template（页面模板）
  AdminLayout / AuthLayout / DashboardLayout
  特点：页面级骨架
```

### 无头组件（Headless）模式

> 把逻辑和样式分离。逻辑层处理状态和 a11y，样式层完全由消费者控制。

```typescript
// 无头 Combobox（只负责逻辑，不负责样式）
// 使用 Radix UI Primitives

import * as Select from '@radix-ui/react-select';

// 消费者完全控制样式（用 Tailwind 或 CSS Modules）
function CountrySelect({ value, onChange }: Props) {
  return (
    <Select.Root value={value} onValueChange={onChange}>
      <Select.Trigger className="flex items-center gap-2 border rounded px-3 py-2">
        <Select.Value placeholder="Select country" />
        <Select.Icon>▼</Select.Icon>
      </Select.Trigger>

      <Select.Portal>
        <Select.Content className="bg-white shadow-lg rounded-lg">
          <Select.Viewport>
            {countries.map(c => (
              <Select.Item key={c.code} value={c.code}
                className="px-3 py-2 hover:bg-blue-50 cursor-pointer">
                <Select.ItemText>{c.name}</Select.ItemText>
              </Select.Item>
            ))}
          </Select.Viewport>
        </Select.Content>
      </Select.Portal>
    </Select.Root>
  );
  // Radix 内置处理：键盘导航、ARIA 属性、焦点管理、屏幕阅读器支持
}
```

### shadcn/ui 模式（Copy-paste 组件）

```bash
# shadcn/ui 不是传统 npm 包，而是把组件源码复制到项目里
npx shadcn-ui@latest add button dialog

# 生成到 components/ui/button.tsx（你拥有源码，可以自由修改）
```

**优势**：不受版本升级限制，完全可定制。
**劣势**：无法享受上游 Bug 修复，需要手动同步更新。

### 组件 API 设计原则

```typescript
// ✓ 遵循 HTML 原生属性（extends native props）
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
  icon?: React.ReactNode;  // 左侧图标
}

// ✓ 使用 Slot 模式（允许消费者替换根元素）
import { Slot } from '@radix-ui/react-slot';

interface ButtonProps {
  asChild?: boolean;  // 渲染为子组件而不是 <button>
}

function Button({ asChild, children, ...props }: ButtonProps) {
  const Comp = asChild ? Slot : 'button';
  return <Comp {...props}>{children}</Comp>;
}

// 使用：渲染为 <a> 标签但保留 Button 样式
<Button asChild>
  <a href="/dashboard">Go to Dashboard</a>
</Button>

// ✓ 复合组件模式（语义清晰）
<Dialog>
  <Dialog.Trigger>Open</Dialog.Trigger>
  <Dialog.Content>
    <Dialog.Title>Confirm</Dialog.Title>
    <Dialog.Description>Are you sure?</Dialog.Description>
    <Dialog.Footer>
      <Dialog.Close>Cancel</Dialog.Close>
    </Dialog.Footer>
  </Dialog.Content>
</Dialog>
```

---

## Storybook

### 核心用途

```
1. 组件隔离开发（不需要跑整个 App）
2. 可视化文档（设计师和开发者共同语言）
3. 视觉回归测试（Chromatic 集成）
4. 交互测试（play function）
```

### Story 编写规范

```typescript
// components/Button/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { within, userEvent, expect } from '@storybook/test';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  // 自动生成 Controls（参数面板）
  argTypes: {
    variant: { control: 'select', options: ['primary', 'secondary', 'danger'] },
    size: { control: 'radio', options: ['sm', 'md', 'lg'] },
    disabled: { control: 'boolean' },
    loading: { control: 'boolean' },
    onClick: { action: 'clicked' },  // 记录点击事件
  },
  parameters: {
    layout: 'centered',
    // 生成 a11y 报告
    a11y: { config: { rules: [{ id: 'color-contrast', enabled: true }] } },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

// 基础故事
export const Primary: Story = {
  args: { variant: 'primary', children: 'Click me' },
};

export const Loading: Story = {
  args: { variant: 'primary', loading: true, children: 'Saving...' },
};

// 交互测试（play function）— 在 Storybook 中运行 UI 测试
export const ClickInteraction: Story = {
  args: { variant: 'primary', children: 'Submit' },
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const button = canvas.getByRole('button', { name: /submit/i });

    // 验证初始状态
    await expect(button).not.toBeDisabled();

    // 模拟点击
    await userEvent.click(button);

    // 验证 loading 状态（如果组件有 loading 逻辑）
    // await expect(button).toHaveAttribute('aria-busy', 'true');
  },
};

// 所有变体的展示（用于视觉回归测试）
export const AllVariants: Story = {
  render: () => (
    <div className="flex gap-4">
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="danger">Danger</Button>
      <Button variant="ghost">Ghost</Button>
    </div>
  ),
};
```

---

## 发包策略

### Monorepo 结构

```
packages/
  tokens/                 ← Design Tokens（@myds/tokens）
    src/
      global.ts
      semantic.ts
    package.json
  components/             ← 基础组件（@myds/components）
    src/
      Button/
        Button.tsx
        Button.stories.tsx
        Button.test.tsx
        index.ts
      Input/
      ...
    package.json
  hooks/                  ← 通用 Hooks（@myds/hooks）
  icons/                  ← 图标（@myds/icons）
  utils/                  ← 工具函数（@myds/utils）

apps/
  storybook/              ← Storybook 文档站
  docs/                   ← 文档网站（Nextra/Docusaurus）
```

### 构建配置（tsup）

```typescript
// packages/components/tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['cjs', 'esm'],     // 同时输出 CommonJS 和 ESM
  dts: true,                   // 生成 .d.ts 类型文件
  splitting: true,             // 代码分割（按需引入）
  treeshake: true,
  external: ['react', 'react-dom'],  // peer deps 不打包进去
  clean: true,
});

// package.json
{
  "name": "@myds/components",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",   // ESM
      "require": "./dist/index.js",   // CJS
      "types": "./dist/index.d.ts"
    },
    "./button": {
      "import": "./dist/button.mjs",  // 支持按需引入单个组件
      "require": "./dist/button.js"
    }
  },
  "peerDependencies": {
    "react": ">=18",
    "react-dom": ">=18"
  }
}
```

### 版本管理（Changesets）

```bash
# 1. 开发者提交变更时，描述变更类型
pnpm changeset
# → 选择受影响的包
# → 选择版本号类型（patch/minor/major）
# → 写变更描述

# 2. CI 自动合并 changeset → 更新版本 + CHANGELOG
pnpm changeset version

# 3. 发布到 npm
pnpm changeset publish
```

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - name: Create Release PR or Publish
        uses: changesets/action@v1
        with:
          publish: pnpm changeset publish
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## 消费端接入

### 按需引入（Tree Shaking）

```typescript
// ✓ 按需引入（推荐，只打包用到的组件）
import { Button, Input } from '@myds/components';

// 或更精确的路径引入
import { Button } from '@myds/components/button';
import { Input } from '@myds/components/input';

// ❌ 全量引入（整个库都打包进去）
import * as DS from '@myds/components';
```

### 主题定制

```typescript
// 消费者应用 — 覆盖 Semantic Token 实现品牌定制
import { applyTheme, lightTheme } from '@myds/tokens';

// 在应用根部定制主题（只覆盖需要改的 token）
applyTheme({
  ...lightTheme,
  'color-primary': '#e85d04',         // 品牌橙色
  'color-primary-hover': '#c1440d',
  'radius-md': '2px',                 // 更方正的风格
});
```

---

## 面试常见追问

**Q: Design Token 和 CSS Variables 有什么关系？**
A: Design Token 是平台无关的设计决策（JSON/TypeScript 定义）；CSS Variables 是 Token 在 Web 平台的实现方式之一。同一套 Token 可以输出 CSS Variables（Web）、Swift Constants（iOS）、XML（Android）。

**Q: 组件库升级 breaking change 怎么办？**
A: 语义化版本（major 版本升级）+ 迁移文档 + Codemod（自动代码迁移脚本）。大型组织通常维护两个大版本并行一段时间，给消费者迁移窗口。

## 常见踩坑

**踩坑1：组件 Props 设计过于宽泛（任意传 className/style）**
❌ 错误：每个组件都接受 `className` 和 `style` prop，消费者随意改样式，设计一致性彻底失控。
✓ 正确：提供有限的 `variant`、`size`、`tone` 等语义化 prop，不暴露底层样式控制，用 Design Token 约束变化范围。
原因：Design System 的核心价值是"有约束的一致性"，过度灵活等于没有约束。

**踩坑2：Breaking Change 未做版本管理和 Migration Guide**
❌ 错误：直接修改 Button 组件的 prop 名（`variant` → `appearance`），所有消费方编译报错，每次更新都是"破坏性发布"。
✓ 正确：遵循 semver，breaking change 升 major 版本，同时提供 codemod 脚本自动迁移，保持至少一个大版本的兼容期。
原因：Design System 是内部基础设施，消费方团队无法频繁人工迁移，自动化工具是大规模采纳的关键。

**踩坑3：组件库未做 Tree Shaking，全量引入体积过大**
❌ 错误：`import { Button } from '@ds/components'` 实际引入了整个组件库 bundle（2MB），即使只用了一个组件。
✓ 正确：配置 `package.json` 的 `exports` 字段支持 ESM，开启 `sideEffects: false`，消费方 bundler 可以 tree shake 掉未使用的组件。
原因：组件库不支持 tree shaking 会严重膨胀消费方的 bundle 体积，影响首屏性能。

**踩坑4：Storybook 与生产组件不同步**
❌ 错误：组件逻辑改了但 Storybook story 没更新，文档展示的是旧行为，消费方按文档写代码却不能运行。
✓ 正确：CI 中运行 `storybook build` 并与上个版本做视觉回归测试（Chromatic），story 变化需要审批通过。
原因：文档即合约，文档与实现不一致比没有文档更危险，会建立错误的信任。

---

**Q: 无头组件（Headless）和普通封装组件怎么选？**
A: 设计系统强定制需求用无头（Radix / Headless UI），消费者自己控制样式；追求开箱即用用带样式的组件（shadcn/ui / Ant Design）。大公司通常内部用无头层 + 自己的样式层。

**Q: 如何测量 Design System 的使用率？**
A: 在组件内上报使用事件（analytics），或用 `@storybook/addon-coverage` 分析哪些组件有测试覆盖。更简单的方式：扫描各业务仓库的 `import from '@myds/components'` 计数。
