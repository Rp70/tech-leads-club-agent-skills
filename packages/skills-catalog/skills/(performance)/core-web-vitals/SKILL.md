---
name: core-web-vitals
description: Optimize Core Web Vitals (LCP, INP, CLS) for better page experience and search ranking. Use when asked to "improve Core Web Vitals", "fix LCP", "reduce CLS", "optimize INP", "page experience optimization", or "fix layout shifts".
license: MIT
metadata:
  author: web-quality-skills
  version: '1.1'
  last_reviewed: '2026-07'
---

# Core Web Vitals optimization

Targeted optimization for the three Core Web Vitals metrics that affect Google Search ranking and user experience.

## The three metrics

| Metric  | Measures         | Good    | Needs work    | Poor    |
| ------- | ---------------- | ------- | ------------- | ------- |
| **LCP** | Loading          | ≤ 2.5s  | 2.5s – 4s     | > 4s    |
| **INP** | Interactivity    | ≤ 200ms | 200ms – 500ms | > 500ms |
| **CLS** | Visual Stability | ≤ 0.1   | 0.1 – 0.25    | > 0.25  |

Google measures at the **75th percentile** — 75% of page visits must meet "Good" thresholds.

---

## LCP: Largest Contentful Paint

LCP measures when the largest visible content element renders. Usually this is:

- Hero image or video
- Large text block
- Background image
- `<svg>` element

### Common LCP issues

**1. Slow server response (TTFB > 800ms)**

```
Fix: CDN, caching, optimized backend, edge rendering
```

**2. Render-blocking resources**

```html
<!-- ❌ Blocks rendering -->
<link rel="stylesheet" href="/all-styles.css" />

<!-- ✅ Critical CSS inlined, rest deferred -->
<style>
  /* Critical above-fold CSS */
</style>
<link rel="preload" href="/styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'" />
```

**3. Slow resource load times**

```html
<!-- ❌ No hints, discovered late -->
<img src="/hero.jpg" alt="Hero" />

<!-- ✅ Preloaded with high priority -->
<link rel="preload" href="/hero.webp" as="image" fetchpriority="high" />
<img src="/hero.webp" alt="Hero" fetchpriority="high" />
```

**4. Client-side rendering delays**

```javascript
// ❌ Content loads after JavaScript
useEffect(() => {
  fetch('/api/hero-text')
    .then((r) => r.json())
    .then(setHeroText)
}, [])

// ✅ Server-side or static rendering
// Use SSR, SSG, or streaming to send HTML with content
export async function getServerSideProps() {
  const heroText = await fetchHeroText()
  return { props: { heroText } }
}
```

### LCP optimization checklist

```markdown
- [ ] TTFB < 800ms (use CDN, edge caching)
- [ ] LCP image preloaded with fetchpriority="high"
- [ ] LCP image optimized (WebP/AVIF, correct size)
- [ ] Responsive images use the right srcset descriptor for the layout
      (density `x` for fixed-size, width `w` + accurate `sizes` for
      fluid — see perf-web-optimization's image-optimization reference)
- [ ] Critical CSS inlined (< 14KB)
- [ ] No render-blocking JavaScript in <head>
- [ ] Fonts don't block text rendering (font-display: swap)
- [ ] LCP element in initial HTML (not JS-rendered)
```

**The LCP element is often a CSS `background-image`, not an `<img>`** —
`fetchpriority` can't be set on a CSS property, and a plain
`background-image: url(...)` is discovered late (only once the browser has
parsed the CSS that references it). If Lighthouse's `lcp-discovery-insight`
flags "fetchpriority=high should be applied" for an element that's
visually a background image, the fix isn't on the (nonexistent) `<img>`
tag — add a `<link rel="preload" as="image" href="..." fetchpriority="high">`
in `<head>` pointing at the same URL. This makes the resource discoverable
immediately from the initial HTML regardless of how the CSS eventually
paints it.

### LCP element identification

```javascript
// Find your LCP element
new PerformanceObserver((list) => {
  const entries = list.getEntries()
  const lastEntry = entries[entries.length - 1]
  console.log('LCP element:', lastEntry.element)
  console.log('LCP time:', lastEntry.startTime)
}).observe({ type: 'largest-contentful-paint', buffered: true })
```

---

## INP: Interaction to Next Paint

INP measures responsiveness across ALL interactions (clicks, taps, key presses) during a page visit. It reports the worst interaction (at 98th percentile for high-traffic pages).

### INP breakdown

Total INP = **Input Delay** + **Processing Time** + **Presentation Delay**

| Phase        | Target  | Optimization                |
| ------------ | ------- | --------------------------- |
| Input Delay  | < 50ms  | Reduce main thread blocking |
| Processing   | < 100ms | Optimize event handlers     |
| Presentation | < 50ms  | Minimize rendering work     |

### Common INP issues

**1. Long tasks blocking main thread**

```javascript
// ❌ Long synchronous task
function processLargeArray(items) {
  items.forEach((item) => expensiveOperation(item))
}

// ✅ Break into chunks with yielding
async function processLargeArray(items) {
  const CHUNK_SIZE = 100
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE)
    chunk.forEach((item) => expensiveOperation(item))

    // Yield to main thread
    await new Promise((r) => setTimeout(r, 0))
    // Or use scheduler.yield() when available
  }
}
```

**2. Heavy event handlers**

```javascript
// ❌ All work in handler
button.addEventListener('click', () => {
  // Heavy computation
  const result = calculateComplexThing()
  // DOM updates
  updateUI(result)
  // Analytics
  trackEvent('click')
})

// ✅ Prioritize visual feedback
button.addEventListener('click', () => {
  // Immediate visual feedback
  button.classList.add('loading')

  // Defer non-critical work
  requestAnimationFrame(() => {
    const result = calculateComplexThing()
    updateUI(result)
  })

  // Use requestIdleCallback for analytics
  requestIdleCallback(() => trackEvent('click'))
})
```

**3. Third-party scripts**

```javascript
// ❌ Eagerly loaded, blocks interactions
;<script src="https://heavy-widget.com/widget.js"></script>

// ✅ Lazy loaded on interaction or visibility
const loadWidget = () => {
  import('https://heavy-widget.com/widget.js').then((widget) => widget.init())
}
button.addEventListener('click', loadWidget, { once: true })
```

**4. Excessive re-renders (React/Vue)**

```javascript
// ❌ Re-renders entire tree
function App() {
  const [count, setCount] = useState(0)
  return (
    <div>
      <Counter count={count} />
      <ExpensiveComponent /> {/* Re-renders on every count change */}
    </div>
  )
}

// ✅ Memoized expensive components
const MemoizedExpensive = React.memo(ExpensiveComponent)

function App() {
  const [count, setCount] = useState(0)
  return (
    <div>
      <Counter count={count} />
      <MemoizedExpensive />
    </div>
  )
}
```

### INP optimization checklist

```markdown
- [ ] No tasks > 50ms on main thread
- [ ] Event handlers complete quickly (< 100ms)
- [ ] Visual feedback provided immediately
- [ ] Heavy work deferred with requestIdleCallback
- [ ] Third-party scripts don't block interactions
- [ ] Debounced input handlers where appropriate
- [ ] Web Workers for CPU-intensive operations
```

### INP debugging

```javascript
// Identify slow interactions
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 200) {
      console.warn('Slow interaction:', {
        type: entry.name,
        duration: entry.duration,
        processingStart: entry.processingStart,
        processingEnd: entry.processingEnd,
        target: entry.target,
      })
    }
  }
}).observe({ type: 'event', buffered: true, durationThreshold: 16 })
```

---

## CLS: Cumulative Layout Shift

CLS measures unexpected layout shifts. A shift occurs when a visible element changes position between frames without user interaction.

**CLS Formula:** `impact fraction × distance fraction`

### Common CLS causes

**1. Images without dimensions**

```html
<!-- ❌ Causes layout shift when loaded -->
<img src="photo.jpg" alt="Photo" />

<!-- ✅ Space reserved -->
<img src="photo.jpg" alt="Photo" width="800" height="600" />

<!-- ✅ Or use aspect-ratio -->
<img src="photo.jpg" alt="Photo" style="aspect-ratio: 4/3; width: 100%;" />
```

**2. Ads, embeds, and iframes**

```html
<!-- ❌ Unknown size until loaded -->
<iframe src="https://ad-network.com/ad"></iframe>

<!-- ✅ Reserve space with min-height -->
<div style="min-height: 250px;">
  <iframe src="https://ad-network.com/ad" height="250"></iframe>
</div>

<!-- ✅ Or use aspect-ratio container -->
<div style="aspect-ratio: 16/9;">
  <iframe src="https://youtube.com/embed/..." style="width: 100%; height: 100%;"></iframe>
</div>
```

**3. Dynamically injected content**

```javascript
// ❌ Inserts content above viewport
notifications.prepend(newNotification)

// ✅ Insert below viewport or use transform
const insertBelow = viewport.bottom < newNotification.top
if (insertBelow) {
  notifications.prepend(newNotification)
} else {
  // Animate in without shifting
  newNotification.style.transform = 'translateY(-100%)'
  notifications.prepend(newNotification)
  requestAnimationFrame(() => {
    newNotification.style.transform = ''
  })
}
```

**4. Web fonts causing FOUT**

```css
/* ❌ Font swap shifts text */
@font-face {
  font-family: 'Custom';
  src: url('custom.woff2') format('woff2');
}

/* ✅ Optional font (no shift if slow) */
@font-face {
  font-family: 'Custom';
  src: url('custom.woff2') format('woff2');
  font-display: optional;
}

/* ✅ Or match fallback metrics */
@font-face {
  font-family: 'Custom';
  src: url('custom.woff2') format('woff2');
  font-display: swap;
  size-adjust: 105%; /* Match fallback size */
  ascent-override: 95%;
  descent-override: 20%;
}
```

**5. Animations triggering layout**

```css
/* ❌ Animates layout properties */
.animate {
  transition:
    height 0.3s,
    width 0.3s;
}

/* ✅ Use transform instead */
.animate {
  transition: transform 0.3s;
}
.animate.expanded {
  transform: scale(1.2);
}
```

### CLS optimization checklist

```markdown
- [ ] All images have width/height or aspect-ratio
- [ ] All videos/embeds have reserved space
- [ ] Ads have min-height containers
- [ ] Fonts use font-display: optional or matched metrics
- [ ] Dynamic content inserted below viewport
- [ ] Animations use transform/opacity only
- [ ] No content injected above existing content
```

### CLS debugging

```javascript
// Track layout shifts
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      console.log('Layout shift:', entry.value)
      entry.sources?.forEach((source) => {
        console.log('  Shifted element:', source.node)
        console.log('  Previous rect:', source.previousRect)
        console.log('  Current rect:', source.currentRect)
      })
    }
  }
}).observe({ type: 'layout-shift', buffered: true })
```

---

## Measurement tools

### Lab testing

- **Chrome DevTools** → Performance panel, Lighthouse
- **WebPageTest** → Detailed waterfall, filmstrip
- **Lighthouse CLI** → `npx lighthouse <url>`

### Field data (real users)

- **Chrome User Experience Report (CrUX)** → BigQuery or API
- **Search Console** → Core Web Vitals report
- **web-vitals library** → Send to your analytics

```javascript
import { onLCP, onINP, onCLS } from 'web-vitals'

function sendToAnalytics({ name, value, rating }) {
  gtag('event', name, {
    event_category: 'Web Vitals',
    value: Math.round(name === 'CLS' ? value * 1000 : value),
    event_label: rating,
  })
}

onLCP(sendToAnalytics)
onINP(sendToAnalytics)
onCLS(sendToAnalytics)
```

---

## Framework quick fixes

### Next.js

```jsx
// LCP: Use next/image with priority
import Image from 'next/image'
;<Image src="/hero.jpg" priority fill alt="Hero" />

// INP: Use dynamic imports
const HeavyComponent = dynamic(() => import('./Heavy'), { ssr: false })

// CLS: Image component handles dimensions automatically
```

### React

```jsx
// LCP: Preload in head
;<link rel="preload" href="/hero.jpg" as="image" fetchpriority="high" />

// INP: Memoize and useTransition
const [isPending, startTransition] = useTransition()
startTransition(() => setExpensiveState(newValue))

// CLS: Always specify dimensions in img tags
```

### Vue/Nuxt

```vue
<!-- LCP: Use nuxt/image with preload -->
<NuxtImg src="/hero.jpg" preload loading="eager" />

<!-- INP: Use async components -->
<component :is="() => import('./Heavy.vue')" />

<!-- CLS: Use aspect-ratio CSS -->
<img :style="{ aspectRatio: '16/9' }" />
```

### Astro

```astro
---
// LCP: astro:assets' <Image> generates width/height + modern formats
// automatically, but for a CSS background-image (a common hero pattern
// that astro:assets doesn't cover) you need a manual preload:
---
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" slot="head" />
<section style={`background-image: url(${heroUrl})`}>...</section>
```

```astro
<!-- INP: client directives control JS shipping per-component — the
     biggest lever Astro gives you that other frameworks don't. Match the
     directive to the component's actual above-the-fold/interactivity
     need, don't default every interactive island to client:load: -->
<Header client:load />        <!-- always-visible, needed immediately -->
<GallerySection client:visible />  <!-- below the fold, defer until scrolled to -->
<SearchWidget client:idle />   <!-- present but not urgent -->
```

For the full Astro-specific playbook (critical CSS inlining via
`astro-critters`, the non-blocking Google Fonts pattern, and why a plain
`<link rel="stylesheet">` is just as render-blocking as the `@import`
chain it might be replacing), see [Astro Performance
Playbook](../perf-astro/SKILL.md).

## Verifying a fix actually reached production

A re-run of Lighthouse/PSI that still shows a metric you already fixed
doesn't always mean the fix is wrong or the deploy failed — check the
*live* resource before re-diagnosing the code:

```bash
curl -sI https://example.com/assets/hero.webp | grep -iE "content-length|cache-control|cf-cache-status"
```

If `content-length` matches the *old*, pre-fix file size, and there's a
long-lived/`immutable` `Cache-Control` on that path, the origin has the
fix but a CDN edge cache (or even just the browser) is still serving
bytes it cached before the fix shipped — the URL didn't change, so nothing
told the cache to revalidate. This is a completely different problem from
"the fix didn't work" or "the deploy didn't happen," and it wastes a real
amount of time if you don't check the live response first. See
[perf-web-optimization's Caching Headers
section](../perf-web-optimization/SKILL.md#caching-headers) for the fix
(rename the asset rather than editing bytes in place under an `immutable`
path, or purge the CDN cache for that URL).

This applies just as much to non-image assets an SEO/CWV pass might touch
— a regenerated `og:image`, a hand-edited `robots.txt`, any static file
served from a stable path — not just recompressed images.

## References

- [web.dev LCP](https://web.dev/articles/lcp)
- [web.dev INP](https://web.dev/articles/inp)
- [web.dev CLS](https://web.dev/articles/cls)
- [Web Performance Optimization](../perf-web-optimization/SKILL.md) — bundle size, caching, the immutable-cache staleness caveat
- [Astro Performance Playbook](../perf-astro/SKILL.md) — critical CSS, non-blocking fonts, client directive tuning
- [Lighthouse Audits](../perf-lighthouse/SKILL.md) — running audits, matching PageSpeed Insights' exact device/throttling profile
- [SEO](../seo/SKILL.md) — Core Web Vitals as a ranking factor, Agentic Browsing category
