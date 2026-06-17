# 图片与媒体优化

> 图片通常占页面总字节数的 50-70%，是 LCP 和带宽优化的第一战场。
> 本文覆盖：格式选型、响应式图片、懒加载、CDN 图片变换、Next/Image、视频优化。

---

## 图片格式选型

| 格式 | 压缩 | 透明 | 动图 | 浏览器支持 | 适用场景 |
|------|------|------|------|-----------|---------|
| **JPEG** | 有损 | ✗ | ✗ | 全部 | 照片、复杂场景 |
| **PNG** | 无损 | ✓ | ✗ | 全部 | Logo、需要透明的图标 |
| **WebP** | 有损/无损 | ✓ | ✓ | 95%+ | 替代 JPEG/PNG 首选 |
| **AVIF** | 有损/无损 | ✓ | ✓ | 90%+ | 最高压缩率（比 WebP 再小 20-50%）|
| **SVG** | 矢量 | ✓ | ✓ | 全部 | 图标、插图、可缩放图形 |
| **GIF** | 有损 | ✓ | ✓ | 全部 | 已过时，用 WebP/video 替代 |

### 格式压缩对比（同一张照片）

```
原始 PNG:  1200KB
JPEG 80%:   180KB  (↓85%)
WebP 80%:   120KB  (↓90%)
AVIF 80%:    65KB  (↓95%)
```

---

## 响应式图片（srcset + sizes）

```html
<!-- 基础 srcset：浏览器根据设备像素密度选择 -->
<img
  src="hero-800.jpg"
  srcset="hero-400.jpg 400w, hero-800.jpg 800w, hero-1600.jpg 1600w"
  sizes="(max-width: 768px) 100vw,
         (max-width: 1200px) 50vw,
         800px"
  alt="Hero image"
/>
<!--
  sizes 含义：
  - 视口 ≤ 768px → 图片占 100vw
  - 视口 ≤ 1200px → 图片占 50vw
  - 其他 → 固定 800px
  浏览器据此 + 设备 DPR 选择最合适的源
-->

<!-- picture 元素：不同格式回退 + 艺术指导 -->
<picture>
  <!-- AVIF（最优先，最小） -->
  <source
    type="image/avif"
    srcset="hero-400.avif 400w, hero-800.avif 800w"
    sizes="(max-width: 768px) 100vw, 50vw"
  />
  <!-- WebP（回退） -->
  <source
    type="image/webp"
    srcset="hero-400.webp 400w, hero-800.webp 800w"
    sizes="(max-width: 768px) 100vw, 50vw"
  />
  <!-- JPEG（最终回退，所有浏览器支持） -->
  <img src="hero-800.jpg" alt="Hero image" width="800" height="600" />
</picture>
```

---

## Next/Image 组件

> Next/Image 自动处理：格式转换（WebP/AVIF）、响应式 srcset、懒加载、尺寸优化、blur placeholder。

```typescript
import Image from 'next/image';

// 1. 静态导入（自动获取宽高，构建时优化）
import heroImg from '@/public/hero.jpg';

function HeroSection() {
  return (
    <Image
      src={heroImg}
      alt="Hero image"
      priority          // LCP 图片加上 priority，取消懒加载，添加 <link rel="preload">
      placeholder="blur" // 模糊占位（静态导入自动生成，无需额外配置）
      quality={85}       // 压缩质量（默认 75）
      sizes="(max-width: 768px) 100vw, 50vw"
    />
  );
}

// 2. 动态 URL（需要指定 width/height 或 fill）
function ProductImage({ url, name }: { url: string; name: string }) {
  return (
    <div style={{ position: 'relative', aspectRatio: '4/3' }}>
      <Image
        src={url}
        alt={name}
        fill                    // 填充父容器（代替 layout="fill"）
        style={{ objectFit: 'cover' }}
        sizes="(max-width: 768px) 100vw, 33vw"
        placeholder="blur"
        blurDataURL="data:image/jpeg;base64,/9j/4AAQ..."  // 手动提供 blur placeholder
      />
    </div>
  );
}

// 3. 外部域名图片（需要在 next.config.js 中配置）
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'images.example.com' },
      { protocol: 'https', hostname: '*.cloudfront.net' },
    ],
    formats: ['image/avif', 'image/webp'],  // 优先 AVIF，回退 WebP
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    minimumCacheTTL: 60 * 60 * 24 * 30,    // 30 天缓存
  },
};
```

### Next/Image 工作原理

```
请求 /_next/image?url=...&w=800&q=75
          ↓
  Next.js Image Optimization API
          ↓
  1. 从源拉取原始图片
  2. 转换为 WebP/AVIF
  3. 调整到请求尺寸
  4. 返回 + Cache-Control: public, max-age=2592000
          ↓
  CDN 缓存（后续请求直接从 CDN 返回）
```

---

## 懒加载

```typescript
// 原生懒加载（现代浏览器，推荐）
<img src="photo.jpg" loading="lazy" alt="..." />

// Intersection Observer（更多控制）
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const imgRef = useRef<HTMLImageElement>(null);
  const [loaded, setLoaded] = useState(false);
  const [inView, setInView] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setInView(true);
          observer.disconnect();
        }
      },
      { rootMargin: '200px' }  // 提前 200px 开始加载（避免滚动时看到空白）
    );

    if (imgRef.current) observer.observe(imgRef.current);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={imgRef} className="img-container">
      {inView && (
        <img
          src={src}
          alt={alt}
          onLoad={() => setLoaded(true)}
          style={{ opacity: loaded ? 1 : 0, transition: 'opacity 0.3s' }}
        />
      )}
      {!loaded && <div className="skeleton" />}
    </div>
  );
}
```

---

## CDN 图片变换

> 主流 CDN（Cloudinary、Imgix、Cloudflare Images）支持通过 URL 参数动态变换图片，
> 无需预先生成所有尺寸。

```typescript
// Cloudinary URL 变换
function cloudinaryUrl(
  publicId: string,
  options: {
    width?: number;
    height?: number;
    quality?: number;
    format?: 'auto' | 'webp' | 'avif';
    fit?: 'fill' | 'crop' | 'scale';
  }
): string {
  const transforms = [
    options.width && `w_${options.width}`,
    options.height && `h_${options.height}`,
    options.quality && `q_${options.quality ?? 'auto'}`,
    options.format && `f_${options.format ?? 'auto'}`,
    options.fit && `c_${options.fit}`,
    'dpr_auto',  // 自动适配设备像素密度
  ].filter(Boolean).join(',');

  return `https://res.cloudinary.com/myapp/image/upload/${transforms}/${publicId}`;
}

// 使用
const src = cloudinaryUrl('products/iphone-16', {
  width: 800,
  quality: 85,
  format: 'auto',  // 自动选择最佳格式
  fit: 'fill',
});
// → https://res.cloudinary.com/myapp/image/upload/w_800,q_85,f_auto,c_fill,dpr_auto/products/iphone-16

// Imgix（同理）
function imgixUrl(baseUrl: string, params: Record<string, string | number>) {
  const url = new URL(baseUrl);
  Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, String(v)));
  return url.toString();
}
```

---

## Blur Placeholder 生成

```typescript
// 服务端：将图片缩成极小的 base64（用于模糊占位）
import sharp from 'sharp';

async function generateBlurDataURL(imagePath: string): Promise<string> {
  const buffer = await sharp(imagePath)
    .resize(10, 10, { fit: 'inside' })  // 缩到 10x10
    .toFormat('jpeg', { quality: 50 })
    .toBuffer();

  return `data:image/jpeg;base64,${buffer.toString('base64')}`;
  // 结果约 200-500 字节，可内联到 HTML
}

// 使用（在 getStaticProps 中预处理）
export async function getStaticProps() {
  const blurDataURL = await generateBlurDataURL('./public/hero.jpg');
  return { props: { blurDataURL } };
}
```

---

## 视频优化

```html
<!-- 代替 GIF 动图：视频文件小 10 倍以上 -->
<video autoplay loop muted playsinline>
  <source src="animation.av1.mp4" type="video/mp4; codecs=av01">  <!-- 最小 -->
  <source src="animation.webm" type="video/webm">
  <source src="animation.mp4" type="video/mp4">  <!-- 最终回退 -->
</video>
```

```typescript
// 视频懒加载（不在视口内不加载）
function LazyVideo({ src, poster }: { src: string; poster: string }) {
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting && videoRef.current) {
        videoRef.current.src = src;
        videoRef.current.load();
        observer.disconnect();
      }
    });

    if (videoRef.current) observer.observe(videoRef.current);
    return () => observer.disconnect();
  }, [src]);

  return (
    <video
      ref={videoRef}
      poster={poster}
      autoPlay
      loop
      muted
      playsInline
    />
  );
}
```

---

## 关键图片预加载

```typescript
// LCP 图片需要预加载（在 HTML head 中添加 link rel="preload"）
// Next.js 中 <Image priority> 自动处理
// 手动方式：
function App() {
  return (
    <Head>
      <link
        rel="preload"
        as="image"
        href="/hero.webp"
        // 响应式预加载（根据视口选择）
        imageSrcSet="/hero-400.webp 400w, /hero-800.webp 800w"
        imageSizes="(max-width: 768px) 100vw, 50vw"
      />
    </Head>
  );
}
```

---

## 面试常见追问

**Q: WebP 和 AVIF 都支持了，为什么还要保留 JPEG 回退？**
A: AVIF 是 Chrome 85+ / Safari 16+，WebP 是 Chrome 23+ / Safari 14+。仍有少量用户用旧 Safari 或 IE，`<picture>` 的 `<source>` 按顺序匹配，不支持则自动回退到 `<img src>` 的 JPEG。如果不在意旧浏览器（如 B 端管理系统），可以直接 WebP。

**Q: `width` 和 `height` 属性对 CLS 有什么影响？**
A: 浏览器在图片加载完成前不知道高度，会先渲染 0 高度占位，图片加载后撑开，导致布局偏移（CLS 扣分）。设置 `width` + `height` 后，浏览器提前计算宽高比保留空间，消除 CLS。Next/Image 静态导入自动设置，动态 URL 需手动指定或用 `fill`。

**Q: 图片 CDN 和普通 CDN 有什么区别？**
A: 普通 CDN 只缓存和分发静态资源（不改变内容）。图片 CDN（Cloudinary/Imgix）在 CDN 节点上**实时变换**图片（resize/格式转换/质量压缩），首次请求时处理并缓存，后续请求直接返回缓存。这样只需上传一份原图，CDN 按需生成所有变体。

**Q: `loading="lazy"` 有什么缺点？**
A: 首屏图片不能用（会延迟 LCP）；SSR 环境下服务端不知道视口位置，首屏判断可能不准；部分爬虫不执行 JS 也不滚动，可能无法爬取懒加载图片（SEO 风险，重要图片不要懒加载）。
