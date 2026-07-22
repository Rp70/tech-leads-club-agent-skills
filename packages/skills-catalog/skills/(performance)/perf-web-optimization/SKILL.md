---
name: perf-web-optimization
description: "Optimize web performance: Core Web Vitals (LCP, CLS, INP), bundle size, images, caching. Use when site is slow, optimizing for Lighthouse scores, reducing bundle size, fixing layout shifts, or improving Time to Interactive. Triggers on: web performance, Core Web Vitals, LCP, CLS, INP, FID, bundle size, page speed, slow site."
---

# Web Performance Optimization

Systematic approach: Measure → Identify → Prioritize → Implement → Verify.

## Target Metrics

| Metric | Good | Needs Work | Poor |
|--------|------|------------|------|
| LCP | < 2.5s | 2.5-4s | > 4s |
| INP | < 200ms | 200-500ms | > 500ms |
| CLS | < 0.1 | 0.1-0.25 | > 0.25 |
| TTFB | < 800ms | 800ms-1.8s | > 1.8s |

## Quick Wins

### 1. Images (usually biggest impact on LCP)

```html
<!-- Hero/LCP image: eager + high priority -->
<img src="/hero.webp" alt="Hero" width="1200" height="600"
     loading="eager" fetchpriority="high" decoding="async">

<!-- Below fold: lazy load -->
<img src="/product.webp" alt="Product" width="400" height="300"
     loading="lazy" decoding="async">
```

Always set `width` and `height` to prevent CLS.

### 2. Fonts (common LCP/CLS culprit)

```html
<!-- Preconnect to font origin -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<!-- Non-blocking font load -->
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter&display=swap"
      media="print" onload="this.media='all'">
```

**Self-hosted `@font-face` files: check what glyphs they actually contain
before assuming the file size is fixed.** A `network-dependency-tree-
insight`/critical-path finding pointing at a `.ttf`/`.otf` file isn't
always fixable only by loading strategy (preload/swap/subset via a
webfont service) — the file itself may be far larger than the page
needs. Inspect it directly:

```python
from fontTools.ttLib import TTFont
f = TTFont('font.ttf')
cmap = f.getBestCmap()
print(len(cmap), 'glyphs')  # compare against what the site's content actually uses
```

A real case: a Spanish/English-only site's self-hosted "Futura PT"
family turned out to be the full multi-script "Futura **Cyrillic**"
variant — 416 glyphs per weight, 175 of them Cyrillic, on a site with
zero Cyrillic content anywhere. Subsetting to Latin + common
punctuation with `fonttools`' subsetter cut every weight file by
~67% (66-71KB → 21-23KB) with zero visual change:

```bash
pip install fonttools
python3 -m fontTools.subset font.ttf --output-file=font-latin.ttf \
  --unicodes="U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FB01-FB02,U+FEFF,U+FFFD" \
  --layout-features='*' --glyph-names --symbol-cmap --legacy-cmap \
  --notdef-glyph --notdef-outline --recommended-glyphs \
  --name-IDs='*' --name-legacy --name-languages='*'
```

Before trusting any subset in production: diff the codepoints dropped
against every codepoint your content actually needs (Latin-1 range
alone is not a safe assumption — check the *original* file for gaps
too; a purchased/licensed font can be missing glyphs, like accented
Latin characters, that were never there to begin with, in which case
subsetting doesn't regress anything new). Then verify visually —
render representative strings (uppercase tracked-out labels, ligature
pairs like "fi"/"fl", any digits/punctuation actually used) in both
the original and subsetted file side by side (a Playwright page with
two `@font-face` rules pointed at data-URI-embedded fonts works well
for this) and confirm pixel-for-pixel identical output before
shipping. Never assume "Latin subset = safe" without checking the
specific font's actual glyph coverage first.

### 3. Third-party Scripts (common INP killer)

```html
<!-- Defer to user interaction -->
<script>
  function loadThirdParty() {
    // Load analytics, chat widgets, etc.
  }
  ['scroll','click','touchstart'].forEach(e =>
    addEventListener(e, loadThirdParty, {once:true, passive:true})
  );
  setTimeout(loadThirdParty, 5000);
</script>
```

### 4. Critical CSS

Inline critical CSS in `<head>`, defer the rest:

```html
<style>/* critical styles */</style>
<link rel="preload" href="/styles.css" as="style" onload="this.rel='stylesheet'">
```

## Bundle Analysis

```bash
# Webpack
npx webpack-bundle-analyzer dist/stats.json

# Vite
npx vite-bundle-visualizer

# Check package size before installing
npx bundlephobia <package-name>
```

Common heavy packages to replace:
- `moment` (67KB) → `date-fns` (12KB) or `dayjs` (2KB)
- `lodash` (72KB) → cherry-pick imports or native methods

## Code Splitting Patterns

```javascript
// React lazy
const Chart = lazy(() => import('./Chart'));

// Next.js dynamic
const Admin = dynamic(() => import('./Admin'), { ssr: false });

// Vite/Rollup manual chunks
build: {
  rollupOptions: {
    output: {
      manualChunks: { vendor: ['react', 'react-dom'] }
    }
  }
}
```

## Caching Headers

```
# Static assets (immutable hash in filename)
Cache-Control: public, max-age=31536000, immutable

# HTML (revalidate)
Cache-Control: no-cache

# API responses
Cache-Control: private, max-age=0, must-revalidate
```

⚠️ **`immutable` is a promise about the URL, not the file** — it tells every
cache in the path (CDN edge, browser) to never revalidate that exact URL,
ever. That's exactly right for hashed build output (`main.a3f8c9.js`) where
a content change always produces a new filename. It's a trap for anything
under a *stable* path — a `public/`-style static asset referenced by a
fixed name (`logo.webp`, `hero-bg.png`) that a future change might
recompress or replace **in place**. If that happens, every cache that
already has the old bytes under that URL keeps serving them indefinitely —
the origin is fixed, the deploy succeeded, and a re-audit (Lighthouse, PSI,
even just eyeballing the page) will keep reporting the *old*, pre-fix file
with no obvious error anywhere. This isn't hypothetical: it cost multiple
full audit-fix-reaudit cycles on a real static site before being caught by
directly `curl -I`-ing the live asset URL and comparing `content-length`
against the just-committed file. **The fix, in order of preference:**
1. Give the changed asset a new filename (any cache-busting suffix, or a
   proper content hash) and update every reference — guaranteed fresh
   fetch, no cache purge needed, works even without CDN dashboard access.
2. If the filename truly can't change, purge the CDN's cache for that
   specific URL after deploying.

Never assume a "fixed" static asset reached users just because the build/
deploy succeeded — verify with `curl -sI <url> | grep -i content-length`
(or your CDN's cache-status header, e.g. Cloudflare's `cf-cache-status:
HIT`/`MISS`) and compare against the actual file size on disk.

⚠️ **Even a brand-new immutable-cached URL can get poisoned — by your own
verification request.** This is a distinct failure mode from the one
above (that one is about *editing an old file in place*; this one can
hit a URL that has *never been requested before*, on a genuinely fresh
deploy). Right after a deploy goes live, there's a brief window where a
CDN's edge can still be propagating the new static assets while the
HTML/routing layer is already serving the new build — a request that
lands in that gap can get back a fallback response (e.g. a Pages/SPA
catch-all serving `index.html`, `200 text/html`, for a static asset path
that isn't fully replicated yet) instead of 404. If that path is under an
`immutable` cache-control rule, the CDN edge caches that **wrong**
response for that exact URL for the full TTL — often a year — and every
subsequent real visitor gets it too, indefinitely, with the browser
silently falling back (e.g. an invalid font file just makes text render
in the fallback font, no visible error). This is easy to trigger by being
diligent: checking a URL immediately after merging/deploying is exactly
the kind of request that can land in the propagation gap.

**Verify safely**: append a throwaway cache-busting query string
(`?verify=<timestamp>`) to the *first* check after a deploy — most CDNs
(confirmed on Cloudflare) key the cache on the full URL including query
string, so a bust query always forces a fresh fetch from origin without
ever touching (or risking poisoning) the canonical URL real users
actually request. Only check the plain canonical URL once the bust-query
check confirms the origin is serving correctly, and even then treat a
single bad-looking response with suspicion — re-check 2-3 times before
concluding either way, since the propagation gap is transient. If a
canonical URL does turn out poisoned, the fix is the same as above:
rename to a fresh URL, don't fight the cache.

## Measurement

```bash
# Quick audit
npx lighthouse https://site.com --preset=perf --form-factor=mobile
```

For running audits, reading reports, and setting budgets, see **perf-lighthouse**.

## Checklist

### Images
- [ ] Modern formats (WebP/AVIF)
- [ ] Responsive `srcset` — using the right **descriptor type** for the
      layout (density `x` for fixed-size images, width `w` + accurate
      `sizes` for fluid containers — see
      [references/image-optimization.md](references/image-optimization.md#responsive-images))
- [ ] `width`/`height` attributes
- [ ] `loading="lazy"` below fold
- [ ] `fetchpriority="high"` on LCP image
- [ ] Any static/non-hashed image path that got recompressed was verified
      live (`curl -I`), not just rebuilt — see the caching-headers caveat
      above

### JavaScript
- [ ] Bundle < 200KB gzipped
- [ ] Code splitting by route
- [ ] Third-party scripts deferred
- [ ] No unused dependencies

### CSS
- [ ] Critical CSS inlined
- [ ] Non-critical CSS deferred
- [ ] No unused CSS

### Fonts
- [ ] `font-display: swap`
- [ ] Preconnect to font origin
- [ ] Subset if possible

## Detailed Examples

For in-depth optimization patterns, see:
- [references/core-web-vitals.md](references/core-web-vitals.md) - Fixing LCP, CLS, INP issues
- [references/bundle-optimization.md](references/bundle-optimization.md) - Reducing JS bundle size
- [references/image-optimization.md](references/image-optimization.md) - Image formats, responsive images, sharp scripts
