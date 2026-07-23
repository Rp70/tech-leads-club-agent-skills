---
name: perf-lighthouse
description: "Run Lighthouse audits locally via CLI or Node API, parse and interpret reports, set performance budgets. Use when measuring site performance, understanding Lighthouse scores, setting up budgets, or integrating audits into CI. Triggers on: lighthouse, run lighthouse, lighthouse score, performance audit, performance budget."
---

# Lighthouse Audits

## CLI Quick Start

```bash
# Install
npm install -g lighthouse

# Or run without installing anything (no global install needed/wanted in a
# one-off sandbox or CI job):
npx lighthouse https://example.com
pnpm dlx lighthouse https://example.com

# Basic audit
lighthouse https://example.com

# Mobile performance only (faster)
lighthouse https://example.com --preset=perf --form-factor=mobile

# Output JSON for parsing
lighthouse https://example.com --output=json --output-path=./report.json

# Output HTML report
lighthouse https://example.com --output=html --output-path=./report.html
```

**Two-output gotcha**: `--output=json --output=html` writes to
`<output-path>.report.json` / `.report.html` (suffix appended). A *single*
`--output=json` writes to the literal `--output-path` you gave with no
suffix — `require('./report.json')` will then fail with `MODULE_NOT_FOUND`
unless you either request both formats or rename the file yourself before
parsing it.

## Matching PageSpeed Insights exactly

PSI's mobile report uses a specific device + network profile — Lighthouse's
own `--form-factor=mobile` default is close but not byte-identical unless
you pin every value. To reproduce PSI's lab numbers locally (so a local
Lighthouse score is a trustworthy proxy for what PSI will report before you
spend a deploy cycle finding out):

```bash
lighthouse https://example.com \
  --form-factor=mobile \
  --screenEmulation.mobile \
  --screenEmulation.width=412 \
  --screenEmulation.height=823 \
  --screenEmulation.deviceScaleFactor=1.75 \
  --throttling-method=simulate \
  --throttling.rttMs=150 \
  --throttling.throughputKbps=1638.4 \
  --throttling.cpuSlowdownMultiplier=4
```

This is Lighthouse's "Moto G Power" emulation + "Slow 4G" throttling
profile, the same one PSI's mobile report is built on. `--throttling-
method=simulate` (not `devtools`) matches PSI's lab methodology — it
extrapolates timing from an observed trace rather than literally
rate-limiting the connection, which is why a page can show a real
absolute load time of ~150ms in `lcp-breakdown-insight`'s subparts while
the *reported*, simulated LCP is several seconds: the simulated number
models total critical-path weight under throttled conditions, not a
literal stopwatch measurement. Don't be alarmed by the gap between the two
— cross-check against the `*-breakdown-insight`/`*-insight` audits'
`details` before concluding a metric is "wrong."

Even with identical settings, **run at least twice and expect real
run-to-run variance** (network/CPU jitter, especially against a live
production origin rather than localhost) — a single run's score is a
sample, not a fact; two runs landing on the same side of your target
threshold is much stronger evidence than one.

### Diagnosing a suspected asymmetry between two pages/variants needs *more* samples per side, not a single side-by-side

A single-snapshot PSI/Lighthouse report showing page A at 88 and page B at
96 (e.g. two locale variants of the same site, `/es/` vs `/en/`) reads like
a real, fixable gap — and it's tempting to start diagnosing the lower one
immediately. Before writing any code, run **3+ independent samples per
side** with identical settings and look at the spread, not just one pair:
a real case showed `/es/` at 99, 98, 97 and `/en/` at 97, 96, 98 across
three fresh runs each — i.e. the two pages were statistically
indistinguishable, and the original 88-vs-96 reading was pure run-to-run
noise on PSI's shared crawling infrastructure, not a persistent defect.
Confirm with structural comparisons that don't have this noise problem —
`dom-size`/`optimize-dom-size` element count, `total-byte-weight`, and
per-audit `details` (e.g. `long-tasks-insight`'s attributed URL and
duration) are stable properties of the built page, not measurements of a
single throttled network trace, so comparing them directly between
variants is a much more reliable "are these actually different" check
than comparing category scores. **If the structural comparison comes back
identical (or the direction of the difference flips run to run), don't
invent a fix** — a code change for a gap that doesn't reproduce is scope
creep dressed up as diligence, and it risks introducing the very
regression you were trying to prevent. Report the investigation and the
"no structural difference found" conclusion; that's a complete, honest
answer to "why is X slower than Y," not a non-answer.

## Common Flags

```bash
--preset=perf           # Performance only (skip accessibility, SEO, etc.)
--form-factor=mobile    # Mobile device emulation (default)
--form-factor=desktop   # Desktop
--throttling-method=devtools  # More accurate throttling
--only-categories=performance,accessibility  # Specific categories
--chrome-flags="--headless"   # Headless Chrome
```

## Performance Budgets

Create `budget.json`:

```json
[
  {
    "resourceSizes": [
      { "resourceType": "script", "budget": 200 },
      { "resourceType": "image", "budget": 300 },
      { "resourceType": "stylesheet", "budget": 50 },
      { "resourceType": "total", "budget": 500 }
    ],
    "resourceCounts": [
      { "resourceType": "third-party", "budget": 5 }
    ],
    "timings": [
      { "metric": "interactive", "budget": 3000 },
      { "metric": "first-contentful-paint", "budget": 1500 },
      { "metric": "largest-contentful-paint", "budget": 2500 }
    ]
  }
]
```

Run with budget:

```bash
lighthouse https://example.com --budget-path=./budget.json
```

## Node API

```javascript
import lighthouse from 'lighthouse';
import * as chromeLauncher from 'chrome-launcher';

async function runAudit(url) {
  const chrome = await chromeLauncher.launch({ chromeFlags: ['--headless'] });

  const result = await lighthouse(url, {
    port: chrome.port,
    onlyCategories: ['performance'],
    formFactor: 'mobile',
    throttling: {
      cpuSlowdownMultiplier: 4,
    },
  });

  await chrome.kill();

  const { performance } = result.lhr.categories;
  const { 'largest-contentful-paint': lcp } = result.lhr.audits;

  return {
    score: Math.round(performance.score * 100),
    lcp: lcp.numericValue,
  };
}
```

## GitHub Actions

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse

on:
  pull_request:
  push:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build site
        run: npm ci && npm run build

      - name: Run Lighthouse
        uses: treosh/lighthouse-ci-action@v11
        with:
          urls: |
            http://localhost:3000
            http://localhost:3000/about
          budgetPath: ./budget.json
          uploadArtifacts: true
          temporaryPublicStorage: true
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

## Lighthouse CI (LHCI)

For full CI integration with historical tracking:

```bash
# Install
npm install -g @lhci/cli

# Initialize config
lhci wizard
```

Creates `lighthouserc.js`:

```javascript
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/about'],
      startServerCommand: 'npm run start',
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['warn', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 1500 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
      },
    },
    upload: {
      target: 'temporary-public-storage', // or 'lhci' for self-hosted
    },
  },
};
```

Run:

```bash
lhci autorun
```

## Parse JSON Report

```javascript
import fs from 'fs';

const report = JSON.parse(fs.readFileSync('./report.json'));

// Overall scores (0-1, multiply by 100 for percentage)
const scores = {
  performance: report.categories.performance.score,
  accessibility: report.categories.accessibility.score,
  seo: report.categories.seo.score,
};

// Core Web Vitals
const vitals = {
  lcp: report.audits['largest-contentful-paint'].numericValue,
  cls: report.audits['cumulative-layout-shift'].numericValue,
  fcp: report.audits['first-contentful-paint'].numericValue,
  tbt: report.audits['total-blocking-time'].numericValue,
};

// Failed audits
const failed = Object.values(report.audits)
  .filter(a => a.score !== null && a.score < 0.9)
  .map(a => ({ id: a.id, score: a.score, title: a.title }));
```

## Compare Builds

```bash
# Save baseline
lighthouse https://prod.example.com --output=json --output-path=baseline.json

# Run on PR
lighthouse https://preview.example.com --output=json --output-path=pr.json

# Compare (custom script)
node compare-reports.js baseline.json pr.json
```

Simple comparison script:

```javascript
const baseline = JSON.parse(fs.readFileSync(process.argv[2]));
const pr = JSON.parse(fs.readFileSync(process.argv[3]));

const metrics = ['largest-contentful-paint', 'cumulative-layout-shift', 'total-blocking-time'];

metrics.forEach(metric => {
  const base = baseline.audits[metric].numericValue;
  const current = pr.audits[metric].numericValue;
  const diff = ((current - base) / base * 100).toFixed(1);
  const emoji = current <= base ? '✅' : '❌';
  console.log(`${emoji} ${metric}: ${diff}% (${base.toFixed(0)} → ${current.toFixed(0)})`);
});
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Inconsistent scores | Run multiple times (`--number-of-runs=3`), use median |
| Chrome not found | Set `CHROME_PATH` env var |
| Timeouts | Increase with `--max-wait-for-load=60000` |
| Auth required | Use `--extra-headers` or puppeteer script |
| `Runtime error: Browser tab has unexpectedly crashed` (`TARGET_CRASHED`) in a container/CI sandbox | Add `--chrome-flags="--headless=new --no-sandbox --disable-dev-shm-usage --disable-gpu"` — `--disable-dev-shm-usage` in particular matters in containers with a small `/dev/shm` |
| No system Chrome/Chromium installed at all | Point `CHROME_PATH` at any already-installed Chromium — a Playwright or Puppeteer install already has one: `find ~/.cache/ms-playwright -maxdepth 2 -iname "chrome" -type f` (or the Puppeteer equivalent under `~/.cache/puppeteer`) rather than installing a second browser just for Lighthouse |
| `browserType.launch: Executable doesn't exist` (Playwright, not Lighthouse) | Different problem, same neighborhood: run `npx playwright install` (or `pnpm exec playwright install`) — a sandbox/container that's had its cache reset needs this even if it worked before; check which specific browser is missing (chromium vs. its headless-shell vs. webkit vs. firefox — a config using multiple device "projects" can each depend on a different one) before assuming the whole install is broken |

## See also

- **perf-astro** / **perf-web-optimization** — the actual fixes for whatever this audit surfaces (render-blocking CSS/fonts, image delivery, bundle size)
- **core-web-vitals** — deep-dive on LCP/INP/CLS, including why a `lcp-breakdown-insight`'s absolute subpart durations can look tiny while the simulated LCP metric is high (see "Matching PageSpeed Insights exactly" above)
- **seo** — Lighthouse's SEO and "Agentic browsing" categories aren't covered by `--only-categories=performance`; run a full/default report (or check PSI directly) if either matters for the task at hand
