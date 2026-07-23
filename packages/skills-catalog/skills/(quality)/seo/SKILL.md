---
name: seo
description: Make pages discoverable, rankable, and understandable — by search engines *and* AI systems. Covers metadata, Open Graph/Twitter cards, JSON-LD structured data, rich snippet eligibility, robots.txt/llms.txt for AI crawlers, sitemaps, Lighthouse SEO/accessibility validation, the "Agentic browsing" audit category (ARIA-validity for AI-agent navigation), and static-export/SSG framework specifics (Next.js App Router, Astro) for locale-aware `<html lang>`, per-page metadata, and trailing-slash-consistent sitemaps. Use when asked to "improve SEO", "optimize for search", "fix meta tags", "add structured data", "add Open Graph tags", "set up llms.txt", "fix rich snippets", "audit SEO", "run a Lighthouse SEO check", "fix agentic browsing"/"fix accessibility tree not well-formed", or "fix duplicate canonical"/"fix hreflang on a static export".
license: MIT
metadata:
  author: web-quality-skills
  version: '2.5'
  last_reviewed: '2026-07'
---

# SEO, metadata & AI discoverability

Technical and on-page SEO based on Google Search Central guidelines, Lighthouse's SEO audit, schema.org, the Open Graph protocol, and emerging conventions for AI crawlers (llms.txt, GEO). The goal isn't just ranking in blue links anymore — it's being correctly parsed, previewed, and cited by search engines, social platforms, *and* LLM-based answer engines (Google AI Overviews, ChatGPT, Perplexity, Claude).

## How search (and AI) visibility breaks down

| Factor                             | Influence | This skill                                                                |
| ----------------------------------- | --------- | -------------------------------------------------------------------------- |
| Content quality, relevance, E-E-A-T | ~35%      | Partial — structure & signals, not writing quality                       |
| Backlinks & authority               | ~20%      | ✗ (off-page)                                                              |
| Technical SEO (crawl/index/render)  | ~15%      | ✓                                                                         |
| Page experience (Core Web Vitals)   | ~10%      | See [Core Web Vitals](../core-web-vitals/SKILL.md)                       |
| On-page SEO & metadata              | ~10%      | ✓                                                                         |
| Structured data & rich results      | ~5%       | ✓                                                                         |
| AI/LLM discoverability (GEO)        | growing   | ✓ (llms.txt, crawler access, extractable content)                        |

Google's own position: optimizing for AI Overviews and LLM answer engines is not a separate discipline — it's the same fundamentals (crawlable, fast, well-structured, authoritative) plus a few new surfaces (AI crawler access, extractable answers).

---

## 1. Technical SEO

### Crawlability — robots.txt

```text
# /robots.txt
User-agent: *
Allow: /

# Block admin/private areas
Disallow: /admin/
Disallow: /api/
Disallow: /private/

# Don't block resources needed for rendering
# ❌ Disallow: /static/
# ❌ Disallow: /*.js$

Sitemap: https://example.com/sitemap.xml
```

**Meta robots:**

```html
<!-- Default: indexable, followable -->
<meta name="robots" content="index, follow" />

<!-- Noindex specific pages (thank-you pages, internal search, filters) -->
<meta name="robots" content="noindex, nofollow" />

<!-- Indexable but don't follow links (e.g. user-generated content pages) -->
<meta name="robots" content="index, nofollow" />

<!-- Control how Google renders snippets/previews -->
<meta name="robots" content="max-snippet:150, max-image-preview:large, max-video-preview:-1" />
```

**Canonical URLs:**

```html
<link rel="canonical" href="https://example.com/current-page" />
```

- Self-referencing canonical on every indexable page (recommended, even if it looks redundant)
- Absolute URLs only — relative canonicals are a common source of bugs
- Structured data must describe the canonical page; Google can't award rich results to a non-canonical URL

### AI crawlers & robots.txt

Traditional `robots.txt` now also gates AI systems. Each company uses **separate user-agent tokens** for training vs. retrieval — blocking one does not block the others.

| User-agent            | Company    | Purpose                                  |
| ---------------------- | ---------- | ------------------------------------------ |
| `GPTBot`                | OpenAI     | Training data collection                  |
| `OAI-SearchBot`         | OpenAI     | ChatGPT search indexing                   |
| `ChatGPT-User`          | OpenAI     | Live browsing triggered by a user          |
| `ClaudeBot`             | Anthropic  | Training data collection                  |
| `Claude-SearchBot`      | Anthropic  | Search indexing                           |
| `Claude-User`           | Anthropic  | Live browsing triggered by a user          |
| `Google-Extended`       | Google     | Gemini / Vertex AI training (separate from Googlebot) |
| `Applebot-Extended`     | Apple      | Apple Intelligence training                |
| `PerplexityBot`         | Perplexity | Retrieval for cited answers                |
| `CCBot`                 | Common Crawl | Public dataset used to train many LLMs   |
| `Bytespider`            | ByteDance  | Training (mixed compliance record)        |

```text
# Allow answer-engine retrieval (helps citations), block training scrapers
User-agent: GPTBot
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: Google-Extended
Disallow: /

User-agent: CCBot
Disallow: /

User-agent: Bytespider
Disallow: /
```

There's no universally "correct" policy — it's a business decision (traffic/citations vs. training-data control). Decide per bot, don't lump them under `User-agent: *`.

### XML sitemap

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/</loc>
    <lastmod>2026-07-01</lastmod>
    <changefreq>daily</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://example.com/products</loc>
    <lastmod>2026-06-28</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.8</priority>
  </url>
</urlset>
```

- Max 50,000 URLs / 50MB uncompressed per file — use a sitemap index for larger sites
- Include **only** canonical, indexable (200, non-noindex) URLs
- Update `lastmod` accurately — Google ignores it if it doesn't correlate with real changes
- Reference it in `robots.txt` and submit it in Search Console / Bing Webmaster Tools

### URL structure

```
✅ Good:  https://example.com/products/blue-widget
✅ Good:  https://example.com/blog/how-to-use-widgets

❌ Poor:  https://example.com/p?id=12345
❌ Poor:  https://example.com/products/item/category/subcategory/blue-widget-2024-sale-discount
```

- Hyphens, not underscores; lowercase only; keep under ~75 characters
- Keywords naturally, not stuffed; avoid unnecessary parameters
- HTTPS everywhere, including every asset (`img`, `script`, `link` sources) — mixed content erodes trust signals

**Security headers that double as SEO trust signals:**

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
```

---

## 2. Metadata & on-page SEO

### Title tags

```html
<!-- ❌ -->
<title>Page</title>
<title>Home</title>

<!-- ✅ Descriptive, primary keyword near the front, brand at the end -->
<title>Blue Widgets for Sale | Premium Quality | Example Store</title>
```

- 50–60 characters (Google truncates around there in pixels, not strict chars)
- Unique per page, matches search intent, no keyword stuffing

### Meta descriptions

```html
<meta
  name="description"
  content="Shop premium blue widgets with free shipping. 30-day returns. Rated 4.9/5 by 10,000+ customers. Order today and save 20%."
/>
```

- 150–160 characters, unique per page, a real call-to-action
- Google frequently rewrites descriptions it judges low-quality — make it worth keeping

### Heading structure

```html
<!-- ✅ -->
<h1>Blue Widgets - Premium Quality</h1>
<h2>Product Features</h2>
<h3>Durability</h3>
<h3>Design</h3>
<h2>Customer Reviews</h2>
```

Single `<h1>` describing the page's main topic, no skipped levels, keywords used naturally — not stuffed. (This also drives most of Lighthouse's accessibility "heading order" audit — see [§8](#8-accessibility--seo-overlap).)

### Image SEO

```html
<img
  src="blue-widget-product-photo.webp"
  alt="Blue widget with chrome finish, side view showing control panel"
  width="800"
  height="600"
  loading="lazy"
/>
```

Descriptive filenames and alt text, explicit `width`/`height` (prevents CLS), modern formats (WebP/AVIF) with fallbacks, lazy-load below the fold, eager-load/`fetchpriority="high"` for the LCP image.

### Internal linking

```html
<!-- ❌ -->
<a href="/products">Click here</a>

<!-- ✅ Descriptive anchor text -->
<a href="/products/blue-widgets">Browse our blue widget collection</a>
```

Descriptive anchors (they're also an extractable signal for AI answer engines), breadcrumbs for hierarchy, fix broken links promptly.

---

## 3. Open Graph, Twitter/X Cards & icons

Metadata that controls how a page looks when shared — Slack, iMessage, Facebook, LinkedIn, Discord, X, and increasingly what an LLM sees when it renders a link preview.

### Core Open Graph tags

```html
<meta property="og:type" content="website" />
<meta property="og:title" content="Blue Widgets for Sale | Example Store" />
<meta property="og:description" content="Premium blue widgets with free shipping and 30-day returns." />
<meta property="og:url" content="https://example.com/products/blue-widget" />
<meta property="og:site_name" content="Example Store" />
<meta property="og:locale" content="en_US" />

<!-- Image: always pair with explicit width/height so platforms don't have to
     re-fetch and recompute — this is what actually speeds up cache generation -->
<meta property="og:image" content="https://example.com/og/blue-widget.jpg" />
<meta property="og:image:secure_url" content="https://example.com/og/blue-widget.jpg" />
<meta property="og:image:width" content="1200" />
<meta property="og:image:height" content="630" />
<meta property="og:image:type" content="image/jpeg" />
<meta property="og:image:alt" content="Blue widget with chrome finish on a white background" />
```

**og:image sizing (2026 consensus):**

| Platform            | Recommended size | Aspect ratio |
| -------------------- | ----------------- | ------------- |
| Facebook / LinkedIn / Slack / iMessage | 1200×630 | 1.91:1 |
| X (Twitter) `summary_large_image`      | 1200×675 (1200×630 also works) | 16:9 |
| Pinterest (reads og:image, but built for portrait) | 1000×1500 | 2:3 |

Use **1200×630** as the single default — it's accepted everywhere and only Pinterest prefers portrait. Keep text/logos ≥90px from every edge (X and LinkedIn crop the frame slightly differently); file <1MB, JPG for photos, PNG if you need crisp text; always set `og:image:width`/`og:image:height` — platforms cache faster and skip a re-fetch when they're present.

### Type-specific Open Graph (article/product)

```html
<!-- Article -->
<meta property="og:type" content="article" />
<meta property="article:published_time" content="2026-01-15T08:00:00+00:00" />
<meta property="article:modified_time" content="2026-06-20T10:30:00+00:00" />
<meta property="article:author" content="https://example.com/authors/jane-smith" />
<meta property="article:section" content="Guides" />
<meta property="article:tag" content="widgets" />

<!-- Product -->
<meta property="og:type" content="product" />
<meta property="product:price:amount" content="49.99" />
<meta property="product:price:currency" content="USD" />
<meta property="product:availability" content="in stock" />
```

### Twitter/X Cards

```html
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:site" content="@examplestore" />
<meta name="twitter:creator" content="@janesmith" />
<meta name="twitter:title" content="Blue Widgets for Sale | Example Store" />
<meta name="twitter:description" content="Premium blue widgets with free shipping and 30-day returns." />
<meta name="twitter:image" content="https://example.com/og/blue-widget.jpg" />
<meta name="twitter:image:alt" content="Blue widget with chrome finish on a white background" />
```

Always set `twitter:card` explicitly — if omitted, X silently falls back to the small `summary` card, which gets meaningfully less engagement. The `og:*` tags are used as a fallback by X if `twitter:*` equivalents are missing, but don't rely on that — set both.

### Icons, theme color & manifest

```html
<link rel="icon" href="/favicon.ico" sizes="32x32" />
<link rel="icon" href="/icon.svg" type="image/svg+xml" />
<link rel="apple-touch-icon" href="/apple-touch-icon.png" /> <!-- 180x180 -->
<link rel="manifest" href="/manifest.webmanifest" />
<meta name="theme-color" content="#0b5fff" />
```

### Preview & validation

- [Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/) — also force-refreshes Facebook/Instagram's cache
- [LinkedIn Post Inspector](https://www.linkedin.com/post-inspector/)
- [Card Validator (X)](https://cards-dev.x.com/validator) *(inconsistent availability; a local OG preview tool or `curl` + manual check is a reliable fallback)*

---

## 4. Structured data (JSON-LD)

Google explicitly recommends **JSON-LD only** (over Microdata/RDFa) — easiest to maintain, least error-prone, no interference with visible markup. Rules that apply to *every* type below:

- Markup must describe **visible, on-page content** — don't mark up content the reader can't see
- Data must be accurate and current — stale prices/availability get types demoted, not just the specific item
- No self-serving ratings/reviews (e.g. a business writing its own "reviews" of itself)
- Structured-data pages must **not** be blocked by `robots.txt` or `noindex` — Google can't evaluate what it can't crawl
- Violations risk a manual action that strips rich-result eligibility (search ranking itself is unaffected)

### ⚠️ 2026 rich-result eligibility changes

Schema markup is still valid and useful (for AI Overviews / other engines) even where Google no longer renders a *visual* rich result:

| Type                      | Status                                              |
| -------------------------- | ---------------------------------------------------- |
| **FAQPage**                 | Rich result **discontinued May 2026** for standard web search |
| **HowTo**                    | Rich result discontinued (2023)                     |
| **Sitelinks Search Box**     | Discontinued Jan 2026                              |
| **SpecialAnnouncement, Q&A, Practice Problem** | Discontinued Jan 2026            |
| **Dataset** (general search) | No longer a general-search rich result (Dataset Search product only) |
| **Article, Product, Review, BreadcrumbList, Video, Event, LocalBusiness, Organization, Course** | Still active, still worth implementing |

Keep FAQ/HowTo markup only if it also serves other consumers (site search, AI answer engines); don't invest new effort chasing a rich snippet that no longer renders.

### Organization

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Example Company",
  "url": "https://example.com",
  "logo": "https://example.com/logo.png",
  "sameAs": ["https://twitter.com/example", "https://linkedin.com/company/example"],
  "contactPoint": {
    "@type": "ContactPoint",
    "telephone": "+1-555-123-4567",
    "contactType": "customer service"
  }
}
</script>
```

### WebSite

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "Example Store",
  "url": "https://example.com"
}
</script>
```

### Article

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "How to Choose the Right Widget",
  "description": "Complete guide to selecting widgets for your needs.",
  "image": "https://example.com/article-image.jpg",
  "author": { "@type": "Person", "name": "Jane Smith", "url": "https://example.com/authors/jane-smith" },
  "publisher": {
    "@type": "Organization",
    "name": "Example Blog",
    "logo": { "@type": "ImageObject", "url": "https://example.com/logo.png" }
  },
  "datePublished": "2026-01-15",
  "dateModified": "2026-06-20"
}
</script>
```

### Product (with review + availability)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Blue Widget Pro",
  "image": "https://example.com/blue-widget.jpg",
  "description": "Premium blue widget with advanced features.",
  "brand": { "@type": "Brand", "name": "WidgetCo" },
  "offers": {
    "@type": "Offer",
    "price": "49.99",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "priceValidUntil": "2026-12-31",
    "url": "https://example.com/products/blue-widget"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "reviewCount": "1250"
  }
}
</script>
```

Keep `price`/`availability`/`priceValidUntil` fresh — Google increasingly cross-checks Product structured data against live page content and Merchant Center feeds; stale data is a common cause of lost rich results.

### FAQ (schema still valid; no longer a Google rich result)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What colors are available?",
      "acceptedAnswer": { "@type": "Answer", "text": "Our widgets come in blue, red, and green." }
    }
  ]
}
</script>
```

### Breadcrumbs

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Home", "item": "https://example.com" },
    { "@type": "ListItem", "position": 2, "name": "Products", "item": "https://example.com/products" },
    { "@type": "ListItem", "position": 3, "name": "Blue Widgets", "item": "https://example.com/products/blue-widgets" }
  ]
}
</script>
```

### LocalBusiness

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "LocalBusiness",
  "name": "Example Store — Downtown",
  "image": "https://example.com/store-front.jpg",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "123 Main St",
    "addressLocality": "Vancouver",
    "addressRegion": "BC",
    "postalCode": "V6B 1A1",
    "addressCountry": "CA"
  },
  "telephone": "+1-604-555-0100",
  "openingHoursSpecification": [
    { "@type": "OpeningHoursSpecification", "dayOfWeek": ["Monday","Tuesday","Wednesday","Thursday","Friday"], "opens": "09:00", "closes": "18:00" }
  ]
}
</script>
```

### Event

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Event",
  "name": "Vancouver Microsoft 365 Summit",
  "startDate": "2026-09-15T09:00:00-07:00",
  "endDate": "2026-09-16T17:00:00-07:00",
  "eventAttendanceMode": "https://schema.org/MixedEventAttendanceMode",
  "eventStatus": "https://schema.org/EventScheduled",
  "location": {
    "@type": "Place",
    "name": "Vancouver Convention Centre",
    "address": { "@type": "PostalAddress", "addressLocality": "Vancouver", "addressRegion": "BC", "addressCountry": "CA" }
  },
  "image": ["https://example.com/event-image.jpg"],
  "organizer": { "@type": "Organization", "name": "Example Events", "url": "https://example.com" },
  "offers": { "@type": "Offer", "url": "https://example.com/tickets", "price": "199.00", "priceCurrency": "CAD", "availability": "https://schema.org/InStock" }
}
</script>
```

### VideoObject

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "VideoObject",
  "name": "Widget assembly walkthrough",
  "description": "Step-by-step assembly guide for the Blue Widget Pro.",
  "thumbnailUrl": ["https://example.com/video-thumb.jpg"],
  "uploadDate": "2026-03-10T08:00:00+00:00",
  "duration": "PT4M32S",
  "contentUrl": "https://example.com/videos/widget-assembly.mp4"
}
</script>
```

### Validation

- [Google Rich Results Test](https://search.google.com/test/rich-results) — confirms actual rich-result eligibility, not just valid JSON
- [Schema Markup Validator](https://validator.schema.org/) — spec-conformance for any schema.org type, not just Google-eligible ones
- Google Search Console → *Enhancements* — production-monitoring for structured data errors at scale

---

## 5. llms.txt & AI/LLM discoverability

A proposed convention (Jeremy Howard, Answer.AI, Sept 2024) for helping LLM-based tools find and parse the *important* parts of a site — like a curated index, not a raw crawl. `robots.txt` is access control ("what you may fetch"); `llms.txt` is a routing hint ("what's worth fetching among what you're allowed to").

**Reality check before implementing:** this is *not* a ratified web standard, and no major AI crawler (OpenAI, Anthropic, Google) has publicly confirmed it drives their retrieval — Google's John Mueller has said none of them do yet, and independent testing (Search Engine Land) found most sites saw no measurable traffic change after adding it. Treat it as a low-cost, low-risk addition — not a substitute for the fundamentals in the rest of this doc (crawlable HTML, fast pages, clean structured data).

### Format (per the spec)

```markdown
# Project or Site Name

> One-sentence summary of what this site/project is.

Optional longer context paragraph — audience, scope, anything an LLM
needs before reading further.

## Docs

- [Getting Started](https://example.com/docs/start): Setup and first steps
- [API Reference](https://example.com/docs/api): Full endpoint reference

## Examples

- [Quickstart repo](https://example.com/examples/quickstart)

## Optional

- [Changelog](https://example.com/changelog): Only needed for deep context
```

- H1 (site/project name) is the only strictly required section
- Blockquote summary directly under the H1
- H2 sections are markdown link lists: `[title](url): optional note`
- An `## Optional` section signals links that can be skipped when a consumer wants a shorter context window

### llms-full.txt (common companion, not part of the formal spec)

Many doc sites (Anthropic, Mintlify-generated docs, Vercel) additionally publish `llms-full.txt` — a single file with the *entire* concatenated documentation body, for tools that want full context in one fetch rather than following links. Add it once your `llms.txt` index stabilizes; skip it for small sites where the index alone is enough.

### Generative Engine Optimization (GEO) — content tactics

For content that AI Overviews/ChatGPT/Perplexity actually quote, independent of any special markup:

- **Definition-first sentences** ("X is...") outperform narrative lead-ins for citation rate
- **Concrete statistics and named sources** get cited more than vague claims
- **Direct, extractable answers** near the top of the page/section — the same content that ranks organically tends to get cited in AI Overviews, so this reinforces rather than replaces SEO
- Clean semantic HTML and heading structure (an LLM parses structure the same way a crawler does)
- Fresh, dated content — Perplexity in particular favors recency

### Agentic browsing — a distinct, newer audit category

Lighthouse (≥13.x) and PageSpeed Insights now report a separate **"Agentic
browsing"** category alongside Performance/Accessibility/Best
Practices/SEO — pass/fail-scored like SEO, and explicitly described by
Google as "still under development and subject to change." It checks
whether a page is *browsable and operable* by an AI agent (not just
whether an LLM can find/cite its content, which is what the rest of this
section covers), and validates WebMCP integrations where present. It's
reported as its own category, not folded into `--only-categories=seo`
or `=accessibility` — run a full/default Lighthouse pass (no category
filter) or check the PageSpeed Insights UI directly to see it; the exact
CLI category id isn't stable yet given Google's own "subject to change"
caveat.

The audit worth knowing in detail — **"Accessibility tree is not
well-formed" / "ARIA attributes must conform to valid values"**: it
fails whenever an ARIA attribute's value doesn't resolve to something
real, most commonly `aria-controls`/`aria-labelledby`/`aria-describedby`
pointing at an element **ID that isn't in the DOM at evaluation time**.
This is easy to trip on legitimately, and Lighthouse evaluates the page
in whatever state it loads in — usually "closed":

```html
<!-- ❌ Fails "Agentic browsing" (and is a real screen-reader bug too) —
     evaluated while the menu is closed, #mobileNav doesn't exist yet -->
<button aria-controls="mobileNav" aria-expanded="false">Menu</button>
{isOpen && <nav id="mobileNav">…</nav>}
```

**Two fixes — pick the second one:**

1. **Always render the target, toggle visibility with CSS** (e.g.
   `translate-x-full`/`opacity-0` + `pointer-events-none` +
   `aria-hidden={!isOpen}` instead of conditional mount/unmount). This
   keeps the ID always resolvable, but it's a **DOM-structure change**:
   anything that assumed the element doesn't exist while closed —
   test-suite locators expecting a single match (`page.locator('nav')`
   in strict mode), duplicate landmark/accessible-name collisions,
   analytics/crawlers seeing "hidden" content differently — can silently
   break. This is not hypothetical: fixing this exact pattern on a
   production site this way broke ~7 existing Playwright specs that
   located `header nav` expecting exactly one element, once a
   conditionally-mounted mobile-nav `<nav>` became permanently mounted.
2. **Make the ARIA attribute itself conditional instead** — omit it
   entirely when there's nothing valid to point at. Per the WAI-ARIA
   spec `aria-controls` is optional; a disclosure widget without it,
   paired with `aria-expanded`, is fully conformant:

   ```jsx
   // React/JSX
   <button
     aria-controls={isOpen ? 'mobileNav' : undefined}
     aria-expanded={isOpen}
   >
     Menu
   </button>
   {isOpen && <nav id="mobileNav">…</nav>}
   ```

   ```vue
   <!-- Vue -->
   <button :aria-controls="isOpen ? 'mobileNav' : null" :aria-expanded="isOpen">Menu</button>
   <nav v-if="isOpen" id="mobileNav">…</nav>
   ```

   ```svelte
   <!-- Svelte -->
   <button aria-controls={isOpen ? 'mobileNav' : undefined} aria-expanded={isOpen}>Menu</button>
   {#if isOpen}<nav id="mobileNav">…</nav>{/if}
   ```

   ```js
   // Vanilla JS
   function setOpen(open) {
     button.toggleAttribute('aria-controls', open); // removes it entirely when false
     if (open) button.setAttribute('aria-controls', 'mobileNav');
     button.setAttribute('aria-expanded', String(open));
     nav.hidden = !open; // or mount/unmount the node
   }
   ```

   This has **zero DOM-structure impact** — no new elements, no test
   regressions, no accessible-name collisions — and for SSR'd/static
   frameworks (Next.js, Astro, Nuxt) it's correct out of the box as long
   as the *initial* server-rendered state already omits the attribute
   when closed (true by default whenever "closed" is the initial render
   state, which it almost always is) — no client-only hydration fix-up
   needed.

---

## 6. Mobile SEO

```html
<!-- ✅ -->
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

```css
/* ✅ Adequate tap target */
.mobile-friendly-link {
  padding: 12px;
  font-size: 16px;
  min-height: 48px;
  min-width: 48px;
}

body {
  font-size: 16px; /* readable without zooming */
  line-height: 1.5;
}
```

Mobile-first indexing means Googlebot evaluates the mobile version of the page — verify metadata and structured data are present there too, not just on desktop.

---

## 7. International SEO

```html
<link rel="alternate" hreflang="en" href="https://example.com/page" />
<link rel="alternate" hreflang="es" href="https://example.com/es/page" />
<link rel="alternate" hreflang="fr" href="https://example.com/fr/page" />
<link rel="alternate" hreflang="x-default" href="https://example.com/page" />
```

```html
<html lang="en">
<!-- or -->
<html lang="es-MX">
```

Every hreflang set must be reciprocal (each page links back to all others, including itself) or Google ignores the whole cluster.

---

## 8. Accessibility & SEO overlap

Accessibility and SEO share more signals than most checklists admit — both Lighthouse categories score several of the *same* underlying markup. Fixing one commonly fixes the other:

| Signal                          | SEO impact                          | Accessibility impact                |
| --------------------------------- | -------------------------------------- | -------------------------------------- |
| `alt` text on images              | Image search, context for crawlers    | Screen reader content                |
| Heading hierarchy (single `h1`, no skipped levels) | Topic signal, featured-snippet eligibility | Screen reader navigation ("jump to heading") |
| Descriptive link text (not "click here") | Anchor-text ranking signal      | Screen reader "links list" navigation |
| `lang` attribute                  | Correct language targeting            | Correct pronunciation for assistive tech |
| Semantic HTML (`<nav>`, `<main>`, `<button>`) | Easier for crawlers to parse structure | Landmark navigation, correct roles |
| Color contrast                    | Indirect (bounce rate / dwell time)   | WCAG 1.4.3 requirement               |
| Valid, well-formed HTML            | Reliable parsing/rendering            | WCAG 4.1.1 Robust                    |

For the full WCAG-based checklist (keyboard nav, ARIA, focus management, screen-reader testing), see [Accessibility](../web-accessibility/SKILL.md) — don't duplicate that work here, just remember an accessibility pass is usually also an SEO pass. This includes two Lighthouse audits that trip up image-heavy pages specifically: **`image-redundant-alt`** (an image's `alt` duplicating adjacent visible text — common on card/list layouts where the whole card, image included, is one link) and **identical accessible names on different links** (WCAG 2.4.4) — both are image-SEO/semantic-HTML overlaps, and both are covered in detail in [Accessibility § Text alternatives](../web-accessibility/SKILL.md#text-alternatives-11).

**"Descriptive link text" (row above) and `aria-label` are two separate signals — fixing one doesn't fix the other.** Lighthouse's `link-text` audit (the SEO check behind that table row) reads `node.innerText` and fails on an exact match against a per-language generic-phrase blocklist ("learn more", "click here", "read more", etc.) — it does not look at `aria-label` at all (confirmed from `lighthouse/core/audits/seo/link-text.js`: it filters on `link.text`, sourced from `innerText`, never the accessible-name computation). So `<a aria-label="Learn more">Learn more</a>` — a same-value `aria-label` that a shared button component sets defensively — still fails this specific audit even though the accessible name computation "exists." Two different fixes for two different consumers: `aria-label` (or better, a *distinct* aria-label value) is what screen readers use; the audit needs the *rendered* text itself to not be generic. If the visible text must stay short for design reasons, add a `.sr-only` span *inside* the link with the differentiating context (event/product/article title) — `.sr-only` text (absolute-positioned + clipped, not `display:none`/`visibility:hidden`) is still included in `innerText`, so it satisfies the audit without changing what sighted users see. See [Accessibility § Text alternatives](../web-accessibility/SKILL.md#text-alternatives-11) for the parallel same-value-aria-label pitfall on the accessibility side.

---

## 9. Lighthouse testing & validation

### Run the SEO category alone

```bash
npx lighthouse https://example.com --only-categories=seo --output=json --output-path=./seo-report.json
```

### Run SEO + accessibility + best-practices together (skip performance for a quick content/markup pass)

```bash
npx lighthouse https://example.com --only-categories=seo,accessibility,best-practices
```

### What the SEO category checks (pass/fail, no partial credit)

- Document has a `<title>` element and a meta description
- Page has a successful HTTP status code (no 4xx/5xx)
- Links have descriptive text (not "click here")
- Links are crawlable (real `href`, not JS-only handlers)
- Page isn't blocked from indexing (meta robots, `X-Robots-Tag`, robots.txt)
- `robots.txt` is valid (no syntax errors)
- Document has a valid `rel=canonical`
- `<html>` has a `lang` attribute
- Viewport meta tag is present and correctly configured
- Structured data is valid (no parse errors — this does *not* confirm rich-result eligibility, only well-formed JSON-LD)

A single failing audit can drop the whole category score noticeably since there's no partial credit — fix the failing audit, don't chase the score directly.

**Agentic browsing** is a separate category from SEO/Accessibility (see
[§5](#5-llmstxt--ai-llm-discoverability) for the ARIA-validity audit that
drives it) — don't assume `--only-categories=seo,accessibility` covers
it; run the full/default report or check PageSpeed Insights directly.

### A re-audit that "doesn't reflect the fix" is usually a caching problem, not a failed deploy

If a metadata/markup fix (canonical, `hreflang`, structured data, `og:image`
dimensions) was deployed but a re-run of Lighthouse/PSI still shows the old
value, don't assume the deploy failed — `curl -sI <url>` the specific
resource first and check `cache-control`/`cf-cache-status` (or your host's
equivalent header) before re-diagnosing the code. A long-lived/immutable
cache in front of the origin (CDN edge cache, not just the browser) can
keep serving pre-fix bytes at the *same URL* indefinitely, even after a
correct deploy — this bit an image-optimization pass hard enough to be
worth its own writeup: see [Core Web Vitals § Verifying a fix actually
reached production](../core-web-vitals/SKILL.md#verifying-a-fix-actually-reached-production).
The same risk applies to any SEO artifact served from a static/cached path
— an `og:image` file, a hand-generated `sitemap.xml`, a static
`robots.txt` — if you edit its bytes in place without changing the URL,
re-validate by checking the live response, not just the rebuilt output.

### One locale variant scoring lower than another isn't necessarily a real gap

On a multi-locale site, it's common to audit `/es/` and `/en/` (or any
other locale pair) and see one score noticeably lower in a single PSI/
Lighthouse snapshot. Resist diagnosing that locale's content/markup as
the cause before running the audit 3+ times per locale — a real case
showed an 8-point gap in one snapshot pair that completely disappeared
across three fresh runs each (both landed in the same high-90s band,
order reversed between runs). Cross-check with metrics that aren't
sensitive to network jitter — DOM element count, total byte weight — to
confirm whether the locales are actually structurally different before
concluding one has a real defect. See [perf-lighthouse's variance
section](../perf-lighthouse/SKILL.md#matching-pagespeed-insights-exactly)
for the full methodology and a worked example.

### CI integration

```bash
npm install -D @lhci/cli
```

```json
// lighthouserc.json
{
  "ci": {
    "collect": { "url": ["https://example.com/"], "numberOfRuns": 3 },
    "assert": {
      "assertions": {
        "categories:seo": ["error", { "minScore": 0.95 }],
        "categories:accessibility": ["error", { "minScore": 0.9 }]
      }
    }
  }
}
```

```bash
npx lhci autorun
```

For deeper Lighthouse CLI/CI setup (budgets, throttling, parsing reports), see [Lighthouse Audits](../perf-lighthouse/SKILL.md). For LCP/INP/CLS specifics, see [Core Web Vitals](../core-web-vitals/SKILL.md).

### Beyond Lighthouse

| Check                          | Tool                                                                 |
| --------------------------------- | ----------------------------------------------------------------------- |
| Rich-result eligibility (not just valid JSON) | [Rich Results Test](https://search.google.com/test/rich-results) |
| schema.org spec conformance        | [Schema Markup Validator](https://validator.schema.org/)              |
| Social preview rendering           | [Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/), [LinkedIn Post Inspector](https://www.linkedin.com/post-inspector/) |
| Indexing status, crawl errors, Core Web Vitals field data | Google Search Console                        |
| Sitewide crawl audit               | Screaming Frog                                                        |

---

## 10. SEO audit checklist

### Critical

- [ ] HTTPS enabled everywhere, no mixed content
- [ ] `robots.txt` allows crawling of important paths
- [ ] No unintended `noindex` on indexable pages
- [ ] Title tags present and **unique per page** — check more than the
      homepage; a shared root layout/template silently serving the same
      static title/description/canonical to every route is the single most
      common finding on multi-page sites (see [§11](#11-static-export--ssg-framework-notes))
- [ ] Single `<h1>` per page, and it actually renders in the **built/shipped
      HTML** — grep the output, not just the component source. A commented-out
      or otherwise unwired `<h1>` render path is a common false negative:
      the "correct" JSX exists in the file, just never reached at runtime.

### High priority

- [ ] Meta descriptions present and unique
- [ ] Self-referencing canonical on every page
- [ ] Canonical, `hreflang` alternates, and every sitemap.xml `<loc>` agree on
      the exact URL form (trailing slash or not) — a framework's metadata
      API may auto-normalize the canonical it renders while a hand-built
      sitemap entry or JSON-LD `url` field does not, so the sitemap silently
      contradicts the canonical it's supposed to describe
- [ ] Sitemap present, valid, and submitted to Search Console
- [ ] Open Graph + Twitter Card tags with a properly sized `og:image` —
      verify the **actual file dimensions** match the declared
      `og:image:width`/`height`, not just that the tags are present
      (`file`/`identify og-image.png`, or `Image.open(...).size` in Python)
- [ ] Mobile-responsive, tap targets ≥ 48px
- [ ] Core Web Vitals passing at the 75th percentile

### Medium priority

- [ ] JSON-LD structured data implemented for applicable types (Organization, Article/Product/Event, Breadcrumb)
- [ ] Structured data passes Rich Results Test with no errors
- [ ] Image alt text and descriptive filenames
- [ ] Internal linking with descriptive anchor text
- [ ] `hreflang` set up correctly (if multi-region/language)
- [ ] `<html lang>` matches the actual page locale on **every** localized
      route, not just the default one — on i18n-routed static/SSG sites this
      is easy to get structurally wrong (see [§11](#11-static-export--ssg-framework-notes))

### Emerging / AI discoverability

- [ ] AI crawler policy in `robots.txt` is a deliberate choice, not an accidental blanket block
- [ ] `llms.txt` present (optional, low-cost) if the site is documentation-heavy or developer-facing
- [ ] Key answers/definitions are extractable near the top of the relevant section
- [ ] "Agentic browsing" Lighthouse category checked — `aria-controls`/`aria-labelledby`/`aria-describedby` on disclosure widgets (menus, accordions, modals) don't reference IDs that are conditionally absent from the DOM in the widget's default (closed) state

### Ongoing

- [ ] Fix crawl errors flagged in Search Console
- [ ] Update sitemap `lastmod` when content actually changes
- [ ] Re-validate structured data after schema/template changes
- [ ] Monitor ranking + AI Overview citation changes
- [ ] Check for broken internal/external links

---

## 11. Static export & SSG framework notes

Patterns from implementing/porting the same SEO behavior across a Next.js
App Router static export (`output: 'export'`) and an Astro static build for
the same i18n site. The underlying bugs are generic — they'll recur on any
statically-generated, multi-locale, multi-page site — the fixes are
framework-specific. Since Core Web Vitals/page-experience is itself a
ranking factor (see the table at the top of this doc), an SEO pass on an
Astro site should generally also apply [Astro Performance
Playbook](../perf-astro/SKILL.md)'s render-blocking-CSS/font fixes — a slow
page undermines the metadata work in this section.

### Locale-aware `<html lang>` on i18n-routed sites

The most common way this goes wrong: the layout/template that knows the
current locale isn't the same one that renders `<html>`.

**Next.js App Router**: a route like `app/[locale]/page.tsx` puts `locale`
in a dynamic segment. If `<html>`/`<body>` live in a *parent* `app/layout.tsx`
(above `[locale]` in the tree), that layout structurally cannot read
`params.locale` — Next.js only passes a segment's params to layouts at or
below it. The result is `<html lang>` hardcoded to one language, silently
wrong on every other locale. Fix with Next's documented **multiple root
layouts** pattern: split the bare/non-locale routes into their own route
group (`app/(root)/layout.tsx`, its own minimal `<html>`, typically
`noindex` since it's just a redirect stub) and move the *real*
`<html lang={locale}>`/`<body>` into `app/[locale]/layout.tsx` itself, which
does receive the param. Every route in `app/` must resolve to exactly one
html-bearing layout; audit for stray routes outside both trees before
shipping.

**Astro**: locale is just a prop threaded from `Astro.params` into
`BaseLayout.astro`, which renders `<html lang={locale}>` directly — no
provider/context indirection, no layout-nesting constraint, "just works" for
every localized page. The equivalent gotcha instead shows up as a **separate,
non-localized bare route** (e.g. `src/pages/index.astro` redirecting `/` to
the default locale) needing its own minimal layout/redirect page — same
shape as Next's route-group split, just for a different reason (local
dev/preview parity with a hosting-platform redirect rule, not a
locale-plumbing limitation).

**Generic check, any framework**: build the site, open the generated HTML
for *every* locale (not just the default), and diff the `<html lang="...">`
value against the URL. Don't trust that "it's a locale-aware framework" means
this is automatic.

### Per-page metadata — the shared-canonical trap

A single static `metadata`/`<head>` object at the root layout is the single
most common multi-page SEO bug: every route inherits the *same* title,
description, and — critically — the *same* `canonical`. On a site with N
indexable routes, this tells search engines all N pages are duplicates of
whichever URL the canonical happens to hardcode (usually `/`), and only one
of them can ever rank.

- **Next.js App Router**: give every route its own `generateMetadata`
  (page-level, not just layout-level) returning a per-route
  `alternates.canonical`, `title`, `description`, and `openGraph`/`twitter`
  overrides. The layout can still provide sitewide defaults (icons, robots,
  author) — those merge with, and get overridden by, whatever the page-level
  `generateMetadata` returns for overlapping keys.
- **Astro**: the equivalent is per-page frontmatter building a metadata
  object (title/description/canonical/og-image) and passing it as props into
  a shared `<Seo.astro>`/`<head>` component — the component renders the tags,
  but the *values* must come from the page, not be hardcoded inside the
  component.
- **Any framework**: after implementing, diff the rendered `<title>` and
  `<link rel="canonical">` across at least 3 different routes. If they're
  identical, the bug is still there regardless of how confident the
  implementation looked.

A small shared helper (`absoluteUrl(path)`, `localizedPath(locale, path)`,
`buildLanguageAlternates(path)`) that every page/component routes URLs
through is worth the abstraction the moment more than one place needs to
build a URL — canonical, hreflang, sitemap, and JSON-LD `url`/`item` fields
all have to agree, and they will silently drift if each one is built by hand
in its own file.

### robots.txt / sitemap.xml as generated code, not static files

Prefer generating both from the framework's own routing/content source
(locales list, CMS/content-collection query, event/product config) over
hand-writing `public/robots.txt` and `public/sitemap.xml` — a hand-written
sitemap goes stale the moment a route is added or removed and nothing
reminds you to update it.

- **Next.js**: `app/robots.ts` / `app/sitemap.ts` (the metadata-route
  convention) work under `output: 'export'`, but **only** with
  `export const dynamic = 'force-static'` added to each file — without it,
  the build fails outright (`Error: export const dynamic = "force-static"
  ... not configured on route "/robots.txt" with "output: export"`), because
  Next treats these as dynamic server routes by default.
- **Astro**: the equivalent is an API route (`src/pages/robots.txt.ts`,
  `src/pages/sitemap.xml.ts`) exporting `export const prerender = true` so it
  emits a static file at build time instead of requiring a server at
  request time.

### The trailing-slash consistency trap

Static-export configs frequently set a trailing-slash mode
(`trailingSlash: true` in Next.js, `trailingSlash: 'ignore'` +
`build.format: 'directory'` in Astro) because every route is physically
served as `folder/index.html`. Frameworks' own metadata APIs tend to
auto-normalize URLs to match this (e.g. Next resolves `alternates.canonical`
with the trailing slash even if you pass a path without one) — but anything
you build by hand outside that API (a sitemap generator's `url` field, a
JSON-LD `item`/`url` string) does **not** get that normalization for free.
The result: a canonical with a trailing slash and a sitemap entry without
one, describing the same page as two different URLs. Route every hand-built
URL through the one shared helper mentioned above, and verify by grepping
the built output for the same path across the canonical tag, hreflang
alternates, `og:url`, the sitemap, and any JSON-LD blocks — they must be
byte-for-byte identical.

### Verifying a static export locally

Before running Lighthouse (or Playwright) against a static export, serve it
and manually curl/view-source **each locale's route**, not just `/`:

```bash
npx serve out          # or dist/, build/ — whatever the framework outputs
```

**Do not add `-s`/`--single`** (SPA fallback mode) — it rewrites *every*
path to the same `index.html`, so `/es/` and `/en/events/foo/` all silently
serve the site's root page. This produces a convincing false negative: the
server responds 200, the page "loads," but Lighthouse/Playwright is
auditing the wrong content entirely (symptoms: an empty `<title>`, or every
locale reporting identical results). `-s` exists for client-side-routed SPAs
with no real per-route files on disk — a multi-page static export is the
opposite case, and needs the plain per-directory file resolution instead.

Running headless Chrome inside a container/CI sandbox commonly needs
explicit flags and sometimes an explicit binary path:

```bash
CHROME_PATH=/usr/bin/chromium npx lighthouse http://localhost:3000/ \
  --chrome-flags="--headless=new --no-sandbox --disable-gpu --disable-dev-shm-usage"
```

If Lighthouse still reports `NO_FCP` ("the page did not paint any content")
after adding these flags, suspect the SPA-fallback issue above before
assuming it's a sandboxing problem — check what's actually being served
first (`curl -s <url> | grep '<title>'`).

---

## Tools

| Tool                      | Use                                             |
| ------------------------- | ------------------------------------------------ |
| Google Search Console     | Indexing status, rich-result errors, CWV field data |
| Google PageSpeed Insights | Performance + Core Web Vitals (lab + field)     |
| Rich Results Test         | Validate structured data → rich-result eligibility |
| Schema Markup Validator   | Validate structured data → spec conformance     |
| Lighthouse / Lighthouse CI | SEO, accessibility, performance, best-practices audit |
| Facebook Sharing Debugger / LinkedIn Post Inspector | Social preview rendering & cache refresh |
| Screaming Frog            | Sitewide crawl analysis                         |

## References

- [Google Search Central](https://developers.google.com/search)
- [Structured data general guidelines](https://developers.google.com/search/docs/appearance/structured-data/sd-policies)
- [Google structured data gallery](https://developers.google.com/search/docs/appearance/structured-data/search-gallery)
- [Schema.org](https://schema.org/)
- [The Open Graph Protocol](https://ogp.me/)
- [llmstxt.org — the llms.txt spec](https://llmstxt.org/)
- [Next.js Metadata API — generateMetadata](https://nextjs.org/docs/app/building-your-application/optimizing/metadata)
- [Next.js Multiple Root Layouts](https://nextjs.org/docs/app/building-your-application/routing/route-groups#opting-specific-segments-into-a-layout)
- [Astro Endpoints (robots.txt/sitemap.xml as routes)](https://docs.astro.build/en/guides/endpoints/)
- [Core Web Vitals](../core-web-vitals/SKILL.md)
- [Accessibility](../web-accessibility/SKILL.md)
- [Lighthouse Audits](../perf-lighthouse/SKILL.md)
- [Web Quality Audit](../web-quality-audit/SKILL.md)
