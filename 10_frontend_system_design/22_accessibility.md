# 可访问性（Accessibility / a11y）

> 可访问性不只是"给残障人士用的"——键盘导航、屏幕阅读器、颜色对比度
> 影响全球约 15% 的人口，同时也对 SEO 和用户体验有正向影响。

---

## 为什么 a11y 在面试中重要

```
- WCAG 2.1 是国际标准（AA 级是大多数公司的基线要求）
- 美国 ADA、欧盟 EAA 等法规要求 Web 可访问性（违规有法律风险）
- 大厂（Google、Microsoft、Apple）把 a11y 作为工程文化的一部分
- Lighthouse、axe 等工具会自动检测，CI 中失分影响团队 KPI
```

---

## WCAG 2.1 四大原则（POUR）

```
Perceivable（可感知）：信息能被感知，不只依赖单一感官
  → 图片有 alt，视频有字幕，颜色不是唯一信息载体

Operable（可操作）：功能可通过键盘操作，不依赖鼠标
  → 所有交互可键盘访问，足够的点击目标尺寸

Understandable（可理解）：内容和操作可预测
  → 表单有明确标签，错误提示清晰，语言标记正确

Robust（健壮）：兼容不同辅助技术
  → 语义化 HTML，正确使用 ARIA
```

---

## 语义化 HTML（最重要的基础）

```html
<!-- ✗ 非语义化（屏幕阅读器不知道这是按钮） -->
<div class="btn" onclick="submit()">提交</div>
<div class="nav">
  <div onclick="goto('/home')">首页</div>
</div>

<!-- ✓ 语义化（浏览器和辅助技术理解结构） -->
<button type="submit">提交</button>
<nav aria-label="主导航">
  <a href="/home">首页</a>
</nav>

<!-- 语义化 HTML 的好处：
  - 自带键盘可访问性（button 可 Tab 聚焦，Enter/Space 触发）
  - 屏幕阅读器自动播报角色（"按钮：提交"）
  - 减少 ARIA 使用量（语义化 HTML > ARIA）
-->

<!-- 文档结构 -->
<header>
  <nav aria-label="主导航">...</nav>
</header>
<main id="main-content">
  <h1>页面标题</h1>  <!-- 每页只有一个 h1 -->
  <article>
    <h2>文章标题</h2>
    <section>
      <h3>章节标题</h3>
    </section>
  </article>
</main>
<aside aria-label="相关内容">...</aside>
<footer>...</footer>
```

---

## ARIA（Accessible Rich Internet Applications）

> ARIA 是语义化 HTML 不够用时的补充。**第一原则：不必要时不用 ARIA，
> 原生 HTML 语义已经够了就直接用。**

```html
<!-- ARIA 核心三要素：role、state、property -->

<!-- role：告诉辅助技术"这是什么" -->
<div role="button" tabindex="0">自定义按钮</div>
<!-- 但优先用 <button>，上面这种写法需要自己处理键盘事件 -->

<!-- aria-label：为没有文本的元素提供描述 -->
<button aria-label="关闭对话框">✕</button>
<button aria-label="搜索">
  <SearchIcon />  <!-- 图标按钮没有文字 -->
</button>

<!-- aria-labelledby：引用页面中的文本作为标签 -->
<h2 id="dialog-title">确认删除</h2>
<div role="dialog" aria-labelledby="dialog-title" aria-modal="true">
  <p>确定要删除这条记录吗？</p>
</div>

<!-- aria-describedby：提供额外说明 -->
<input
  type="password"
  aria-describedby="password-hint"
/>
<p id="password-hint">密码至少 8 位，包含大小写字母和数字</p>

<!-- aria-expanded：展开/折叠状态 -->
<button aria-expanded="false" aria-controls="menu">菜单</button>
<ul id="menu" hidden>...</ul>

<!-- aria-live：动态内容更新通知 -->
<div aria-live="polite" aria-atomic="true">
  {/* 内容变化时屏幕阅读器会播报 */}
  {statusMessage}
</div>
<!-- polite：等用户操作结束再播报 -->
<!-- assertive：立即中断播报（用于错误） -->

<!-- aria-hidden：对辅助技术隐藏（纯装饰内容） -->
<span aria-hidden="true">★★★★☆</span>
<span className="sr-only">4 星（满分 5 星）</span>
```

---

## 键盘导航

```typescript
// 焦点管理：对话框打开时将焦点移入，关闭时还原

function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // 记录打开前的焦点
      previousFocusRef.current = document.activeElement as HTMLElement;
      // 将焦点移到 modal 的第一个可聚焦元素
      const focusable = modalRef.current?.querySelector<HTMLElement>(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );
      focusable?.focus();
    } else {
      // modal 关闭后，焦点还原到触发按钮
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  // 焦点陷阱（Tab 在 modal 内循环，不能跑到外面）
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Escape') {
      onClose();
      return;
    }

    if (e.key !== 'Tab') return;

    const focusables = Array.from(
      modalRef.current?.querySelectorAll<HTMLElement>(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      ) ?? []
    );

    const first = focusables[0];
    const last = focusables[focusables.length - 1];

    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last.focus();   // Shift+Tab 在第一个元素时跳到最后
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first.focus();  // Tab 在最后元素时跳到第一个
    }
  };

  if (!isOpen) return null;

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      onKeyDown={handleKeyDown}
    >
      <h2 id="modal-title">对话框标题</h2>
      {children}
      <button onClick={onClose}>关闭</button>
    </div>
  );
}
```

```typescript
// 自定义键盘交互（方向键导航，常用于 Listbox、Combobox、Menu）
function ListBox({ options, value, onChange }: ListBoxProps) {
  const [activeIndex, setActiveIndex] = useState(0);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setActiveIndex(prev => Math.min(prev + 1, options.length - 1));
        break;
      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex(prev => Math.max(prev - 1, 0));
        break;
      case 'Enter':
      case ' ':
        e.preventDefault();
        onChange(options[activeIndex].value);
        break;
      case 'Home':
        setActiveIndex(0);
        break;
      case 'End':
        setActiveIndex(options.length - 1);
        break;
    }
  };

  return (
    <ul
      role="listbox"
      aria-activedescendant={`option-${activeIndex}`}
      tabIndex={0}
      onKeyDown={handleKeyDown}
    >
      {options.map((opt, index) => (
        <li
          key={opt.value}
          id={`option-${index}`}
          role="option"
          aria-selected={opt.value === value}
          onClick={() => onChange(opt.value)}
        >
          {opt.label}
        </li>
      ))}
    </ul>
  );
}
```

---

## 颜色与视觉

```typescript
// WCAG 颜色对比度要求：
// AA 级：正常文本 4.5:1，大文本(18px+/粗体14px+) 3:1
// AAA 级：正常文本 7:1，大文本 4.5:1

// 检测颜色对比度
function getContrastRatio(color1: string, color2: string): number {
  const lum1 = getRelativeLuminance(color1);
  const lum2 = getRelativeLuminance(color2);
  const lighter = Math.max(lum1, lum2);
  const darker = Math.min(lum1, lum2);
  return (lighter + 0.05) / (darker + 0.05);
}

// 工具推荐：
// - Chrome DevTools（点颜色会显示对比度）
// - WebAIM Contrast Checker
// - Figma 插件：Contrast

// ✗ 不能只用颜色传递信息
<span style={{ color: 'red' }}>错误</span>  {/* 色盲用户看不出来 */}

// ✓ 颜色 + 图标/文字 双重传递
<span className="error">
  <ErrorIcon aria-hidden="true" />
  <span>邮箱格式不正确</span>
</span>
```

---

## 表单可访问性

```typescript
// 表单是 a11y 问题最集中的地方
function AccessibleForm() {
  return (
    <form noValidate onSubmit={handleSubmit}>
      {/* label 与 input 关联（for/htmlFor 或嵌套） */}
      <div>
        <label htmlFor="email">
          邮箱
          <span aria-hidden="true" className="required-mark"> *</span>
        </label>
        <input
          id="email"
          type="email"
          required
          aria-required="true"
          aria-invalid={errors.email ? 'true' : 'false'}
          aria-describedby={errors.email ? 'email-error' : 'email-hint'}
          autoComplete="email"
        />
        <span id="email-hint" className="hint">
          我们不会分享你的邮箱
        </span>
        {errors.email && (
          <span
            id="email-error"
            role="alert"          // 立即播报错误
            className="error-msg"
          >
            {errors.email.message}
          </span>
        )}
      </div>

      {/* 单选组需要 fieldset + legend */}
      <fieldset>
        <legend>性别</legend>
        <label>
          <input type="radio" name="gender" value="male" />
          男
        </label>
        <label>
          <input type="radio" name="gender" value="female" />
          女
        </label>
        <label>
          <input type="radio" name="gender" value="other" />
          其他
        </label>
      </fieldset>

      {/* 提交按钮状态 */}
      <button
        type="submit"
        aria-busy={isSubmitting}
        disabled={isSubmitting}
      >
        {isSubmitting ? '提交中...' : '提交'}
      </button>
    </form>
  );
}
```

---

## Skip Link（跳过导航）

```typescript
// 键盘用户的第一个焦点应该是"跳到主内容"链接
// 让用户跳过重复的导航，直接到主内容

function SkipLink() {
  return (
    <a
      href="#main-content"
      className="skip-link"
    >
      跳到主要内容
    </a>
  );
}

// CSS：平时隐藏，聚焦时显示
/*
.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  padding: 8px 16px;
  background: #000;
  color: #fff;
  z-index: 9999;
}
.skip-link:focus {
  top: 0;
}
*/

// 页面结构
function Layout({ children }) {
  return (
    <>
      <SkipLink />
      <Header />
      <main id="main-content" tabIndex={-1}>  {/* tabIndex=-1 允许被程序聚焦 */}
        {children}
      </main>
      <Footer />
    </>
  );
}
```

---

## 屏幕外文字（sr-only）

```css
/* 对视觉用户隐藏，但屏幕阅读器可读 */
/* 注意：不能用 display:none 或 visibility:hidden，那样屏幕阅读器也看不到 */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

```typescript
// Tailwind 中直接用 sr-only 类
<button>
  <HeartIcon aria-hidden="true" />
  <span className="sr-only">收藏商品</span>
</button>

// 图标按钮：图标对屏幕阅读器隐藏，sr-only 文字提供语义
```

---

## 自动化测试

```typescript
// @axe-core/react — 开发环境自动检测 a11y 问题
// 在浏览器控制台输出违规项
if (process.env.NODE_ENV !== 'production') {
  const axe = await import('@axe-core/react');
  axe.default(React, ReactDOM, 1000);
}

// Vitest + jest-axe — 单元测试中检测
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

test('Form has no accessibility violations', async () => {
  const { container } = render(<LoginForm />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

// Playwright — E2E 中检测
import AxeBuilder from '@axe-core/playwright';

test('Homepage should not have any a11y violations', async ({ page }) => {
  await page.goto('/');
  const accessibilityScanResults = await new AxeBuilder({ page }).analyze();
  expect(accessibilityScanResults.violations).toEqual([]);
});

// Lighthouse CI — CI/CD 中自动检测（见 12_cicd_testing.md）
// lighthouserc.js 中 accessibility 分数阈值
assertions: {
  'categories:accessibility': ['error', { minScore: 0.9 }],
}
```

---

## 常用社区工具

| 工具 | 用途 |
|------|------|
| **axe DevTools**（Chrome 扩展）| 手动检测当前页面 a11y 问题 |
| **NVDA**（Windows）/ **VoiceOver**（Mac）| 真实屏幕阅读器测试 |
| **Radix UI** | Headless 组件，内置完整 a11y（键盘/ARIA）|
| **@headlessui/react** | Tailwind 官方 Headless，完整 ARIA |
| **react-aria**（Adobe）| 最完整的 a11y hooks 库 |
| **jest-axe** | 单元测试中检测 |
| **@axe-core/playwright** | E2E 测试中检测 |

---

## 面试常见追问

**Q: ARIA role 和语义化 HTML 怎么选？**
A: 语义化 HTML 优先。`<button>` 自带 `role="button"` + 键盘支持 + 默认样式重置，不需要手动添加 ARIA。ARIA 只用于原生 HTML 无法表达的复杂组件（Combobox、Tree、DataGrid）。错误使用 ARIA 比不用更糟（如在 `<button>` 上加 `role="button"` 是冗余的；在非交互元素上加 `role="button"` 但不处理键盘事件，则会误导辅助技术）。

**Q: 如何测试键盘可访问性？**
A: 断开鼠标，用键盘操作整个页面：`Tab` 移动焦点、`Shift+Tab` 反向、`Enter/Space` 激活、方向键在组件内导航、`Escape` 关闭弹窗。检查：①所有交互都可到达；②焦点指示器可见（不要 `outline: none`）；③焦点顺序合理（与视觉顺序一致）；④弹窗有焦点陷阱。

**Q: `tabindex` 的各个值有什么区别？**
A: `tabindex="0"`：加入 Tab 顺序（自然顺序）；`tabindex="-1"`：可被程序聚焦（`el.focus()`），但不在 Tab 顺序中（用于焦点管理，如 modal 打开时 `mainRef.focus()`）；`tabindex="1"` 以上：改变 Tab 顺序（**不推荐**，破坏自然顺序，难维护）。原则：让 DOM 顺序 = 视觉顺序 = Tab 顺序，不需要手动设置正整数 tabindex。
