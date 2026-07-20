---
name: perf-astro
description: "Astro-specific performance optimizations for 95+ Lighthouse scores. Covers critical CSS inlining, compression, font loading, and LCP optimization. Triggers on: astro performance, astro lighthouse, astro optimization, astro-critters."
---

# Astro Performance Playbook

Astro-specific optimizations for 95+ Lighthouse scores.

## Quick Setup

```bash
npm install astro-critters @playform/compress
```

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import critters from 'astro-critters';
import compress from '@playform/compress';

export default defineConfig({
  integrations: [
    critters(),
    compress({
      CSS: true,
      HTML: true,
      JavaScript: true,
      Image: false,
      SVG: false,
    }),
  ],
});
```

## Integrations

### astro-critters

Automatically extracts and inlines critical CSS. No configuration needed.

What it does:
- Scans rendered HTML for above-the-fold elements
- Inlines only the CSS those elements need
- Lazy-loads the rest

Build output shows what it inlined:
```
Inlined 40.70 kB (80% of original 50.50 kB) of _astro/index.xxx.css.
```

**Verified against Astro 7.1.0 (2026-07)**: works cleanly, drops
render-blocking-CSS Lighthouse findings to a clean pass (local Lighthouse
went 88 → 94 in one real-world case, purely from this + the non-blocking
font pattern below). One caveat worth knowing before it surprises you in a
build log: `astro-critters` (last published Jan 2025, wraps Google's now-
archived `critters` library, no explicit Astro peer-dependency declared)
silently skips CSS rules its parser can't handle rather than failing the
build — e.g. Tailwind v4's `:has()` selectors:

```
1 rules skipped due to selector errors:
  .has-disabled\:opacity-50:has() -> Empty sub-selector
```

This is harmless (that one rule just isn't inlined, it's still in the
deferred stylesheet), but re-check the build log after any Tailwind or
Astro version bump in case the skipped-selector list grows, and don't
assume a clean build means every rule was actually inlined. **Verify no
FOUC** after adding this integration: screenshot the page at
`domcontentloaded` and again ~1.5s later (Playwright) — they should be
pixel-identical, since the whole point is that everything visible on first
paint is covered by the inlined critical CSS.

**Confusing chunk names are cosmetic, not a scoping bug**: Astro/Vite may
name a page's entire CSS bundle after an unrelated shared import it
happens to pick as the dependency graph's chunk-naming entry point (e.g. a
file called `_astro/event-info.<hash>.css` that's actually the *whole
site's* Tailwind output, not scoped to anything about "event info" — the
name came from a JSON config file imported by one of the components on
that page). Don't chase a "why is unrelated CSS loading on this page"
theory based on the filename alone; open the file and check what selectors
are actually in it.

### @playform/compress

Minifies HTML, CSS, and JavaScript in the final build.

Options:
```js
compress({
  CSS: true,      // Minify CSS
  HTML: true,     // Minify HTML
  JavaScript: true, // Minify JS
  Image: false,   // Skip if using external image optimization
  SVG: false,     // Skip if SVGs are already optimized
})
```

## Layout Pattern

Structure your `Layout.astro` for performance:

```astro
---
import '../styles/global.css'
---

<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- Font fallback (prevents FOIT) -->
    <style>
      @font-face {
        font-family: 'Inter';
        font-display: swap;
        src: local('Inter');
      }
    </style>

    <!-- Non-blocking Google Fonts -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link
      rel="stylesheet"
      href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap"
      media="print"
      onload="this.media='all'"
    />
    <noscript>
      <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap">
    </noscript>

    <!-- Preload LCP images -->
    <link rel="preload" as="image" href="/hero.png" fetchpriority="high">

    <title>{title}</title>

    <!-- Defer third-party scripts -->
    <script>
      let loaded = false;
      function loadAnalytics() {
        if (loaded) return;
        loaded = true;
        // Load GTM, analytics, etc.
      }
      ['scroll', 'click', 'touchstart'].forEach(e => {
        document.addEventListener(e, loadAnalytics, { once: true, passive: true });
      });
      setTimeout(loadAnalytics, 5000);
    </script>
  </head>
  <body>
    <slot />
  </body>
</html>
```

## Client directive choice affects the critical path, not just hydration timing

`network-dependency-tree-insight`/"reduce unused JavaScript" findings that
trace back to a component with `client:load` are often a directive choice,
not a bundling problem. `client:load` hydrates (and therefore fetches its
whole JS chain — React runtime included, if it's a React island) as part of
the initial navigation, competing for bandwidth/priority with whatever's
actually on the LCP path (hero image, fonts). A header/nav island is the
classic case: it's already fully visible from Astro's SSR'd HTML, and its
interactivity (mobile-nav toggle, a dropdown) isn't needed until the user
actually reaches for it. Switching it to `client:idle` defers the fetch +
hydration until the main thread is idle instead of blocking on it from
navigation start — on a real site this dropped a react/react-dom (43KB) +
a dozen chunks off the reported critical path with no user-visible change,
since nothing about first paint depended on that island being hydrated yet.
`client:visible` is the right call instead when the island is genuinely
below the fold (defer until scrolled into view); `client:idle` is right
when it's above the fold but not interaction-critical on load. Verify no
regression the same way as any hydration-timing change: run the full
Playwright suite, not just a visual check — an above-the-fold island can
still have tests that click it immediately after `page.goto`.

## Measuring

```bash
npx lighthouse https://your-site.com --preset=perf --form-factor=mobile
```

See also:
- **perf-lighthouse** - Running audits, reading reports, setting budgets, matching PageSpeed Insights' exact device/throttling settings
- **perf-web-optimization** - Core Web Vitals, bundle size, caching strategies (including the immutable-cache-headers gotcha for `public/`-style static assets)
- **core-web-vitals** - responsive-image `srcset` descriptor pitfalls (density vs. width) that show up constantly on Astro's `public/` static assets since Astro doesn't auto-generate `srcset` for them the way `astro:assets`'s `<Image>` does
- **seo** - static-export/SSG framework notes (locale-aware `<html lang>`, per-page metadata, sitemap generation) for the same Astro static-build context this skill optimizes

## Checklist

- [ ] `astro-critters` installed and configured
- [ ] `@playform/compress` installed and configured
- [ ] Google Fonts use `media="print" onload` pattern (not just moved out
      of a CSS `@import` into a plain `<link>` — a plain `<link
      rel="stylesheet">` is still just as render-blocking as the `@import`
      chain it replaced; the `media`/`onload` trick is the part that
      actually removes it from the critical path)
- [ ] Third-party scripts deferred to user interaction
- [ ] LCP images preloaded in `<head>`
- [ ] After adding critical-CSS inlining, screenshot-diffed
      `domcontentloaded` vs. settled to confirm no FOUC
- [ ] Re-ran Lighthouse and confirmed `render-blocking-insight` (or
      Lighthouse ≤12's `render-blocking-resources`) actually scores clean
      — don't rely on "the integration is configured" as a proxy for "it
      worked"
