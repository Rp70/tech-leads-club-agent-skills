# Image Optimization

## Table of Contents
- [Modern Formats](#modern-formats)
- [Responsive Images](#responsive-images)
- [Sharp Script](#sharp-script)
- [Framework Components](#framework-components)

---

## Modern Formats

| Format | Use Case | Savings |
|--------|----------|---------|
| AVIF | Best compression, modern browsers | 50-80% vs JPEG |
| WebP | Good compression, wide support | 25-35% vs JPEG |
| JPEG | Fallback for old browsers | baseline |

### Picture Element with Fallbacks

```html
<picture>
  <source srcset="/image.avif" type="image/avif">
  <source srcset="/image.webp" type="image/webp">
  <img src="/image.jpg" alt="Description" width="800" height="600">
</picture>
```

---

## Responsive Images

### srcset with width descriptors

```html
<img
  src="/image-800.jpg"
  srcset="
    /image-400.jpg 400w,
    /image-800.jpg 800w,
    /image-1200.jpg 1200w
  "
  sizes="(max-width: 600px) 100vw, 50vw"
  alt="Description"
  width="800"
  height="600"
  loading="lazy"
>
```

### Density (`x`) vs. width (`w`) descriptors — pick the one that matches the layout

These are not interchangeable, and using the wrong one silently defeats
responsive image selection even when everything else about the markup is
correct:

- **Density descriptors (`1x`, `2x`)** — for an image whose **CSS display
  size doesn't change** across viewports (a logo in a fixed-height header,
  an avatar, a fixed-size icon). The browser picks a candidate purely from
  its own `devicePixelRatio` — it has no way to know (and doesn't need to
  know) how wide the image renders, because that's constant.

  ```html
  <img src="/logo.webp" srcset="/logo-1x.webp 1x, /logo-2x.webp 2x"
       alt="Company logo" width="200" height="50">
  ```

- **Width descriptors (`400w`, `800w`, `1200w`) + `sizes`** — for an image
  inside a **fluid/responsive container** whose rendered width genuinely
  varies (a hero image, a card in a grid). The browser computes the
  candidate it needs as `(sizes-resolved slot width) × devicePixelRatio`
  and picks the smallest sufficient one — but only if `sizes` is accurate.

**The failure mode that's easy to miss**: putting density descriptors on a
fluid-width image (or width descriptors with a `sizes` value that
over-declares the real slot width) makes the browser fetch a larger
candidate than necessary on any screen with `devicePixelRatio > 1` — which
is most phones. Concretely: an image in a container with `sizes="(max-
width: 768px) 100vw, 50vw"`, where the *actual* rendered width on mobile is
smaller than 100vw because of container padding the `sizes` string doesn't
account for — the browser trusts the **declared** `sizes` value, not the
true layout, so it computes a higher pixel need than what's really used
and fetches a bigger image than the page actually displays. Lighthouse's
`image-delivery-insight`/"oversized images" audit will flag this as
"larger than it needs to be for its displayed dimensions" — and the fix
is **not** to recompress the image (it's already correctly sized for a
different, smaller viewport at a lower `devicePixelRatio`), it's to
correct `sizes` to reflect what the container actually resolves to at each
breakpoint (subtracting padding/margins, accounting for any `max-width`).
Verify by checking the *reported* "displayed dimensions" in the audit
against `(declared sizes value) × devicePixelRatio` — if they don't match,
`sizes` is the bug, not the image file.

### Full responsive picture

```html
<picture>
  <source
    srcset="/hero-400.avif 400w, /hero-800.avif 800w, /hero-1200.avif 1200w"
    sizes="100vw"
    type="image/avif"
  >
  <source
    srcset="/hero-400.webp 400w, /hero-800.webp 800w, /hero-1200.webp 1200w"
    sizes="100vw"
    type="image/webp"
  >
  <img
    src="/hero-800.jpg"
    srcset="/hero-400.jpg 400w, /hero-800.jpg 800w, /hero-1200.jpg 1200w"
    sizes="100vw"
    alt="Hero"
    width="1200"
    height="600"
  >
</picture>
```

### Transparent WebP logos: the alpha channel, not the RGB data, is usually the size

When Lighthouse's `image-delivery-insight` flags a small flat-color/
transparent logo for "increasing the compression factor," lowering `-q`
alone often barely moves the file size — because `cwebp` defaults to a
**lossless** alpha channel regardless of the lossy RGB quality setting, and
for a logo (flat colors, sharp text edges, transparent background) the
alpha plane can be 40-60% of the total bytes. Check with `cwebp -q <N>
input.png -o out.webp` (verbose output) — look for a `transparency:` line
under "bytes used" and a "Lossless-alpha compressed size" line; if that
number barely changes as you sweep `-q`, the RGB quality isn't your
bottleneck. Add `-alpha_q <N>` (0-100, default 100) to make the alpha plane
lossy too:

```bash
cwebp -q 80 -alpha_q 60 -m 6 logo.png -o logo.webp
```

On a real 269x84 transparent logo this cut the file from 12.4KB to 9.7KB
(vs. ~12KB from `-q` alone) with no visible difference at 4x zoom.
Recompress from the original source raster (PNG/SVG export), not by
re-encoding an already-lossy `.webp` — decoding and re-encoding a lossy
source compounds artifacts for no size benefit, since the alpha plane is
what's actually costing bytes either way. Verify visually before shipping:
decode both candidates (`dwebp old.webp -o a.png`, `dwebp new.webp -o
b.png`), upscale 3-4x (`convert a.png -resize 400% a-big.png`), and
side-by-side compare — text edges and antialiasing are where quality loss
shows first on a logo.

---

## Sharp Script

Batch convert images to modern formats and sizes:

```javascript
// scripts/optimize-images.js
const sharp = require('sharp');
const fs = require('fs');
const path = require('path');

const SIZES = [400, 800, 1200];
const INPUT_DIR = './images/original';
const OUTPUT_DIR = './public/images';

async function optimizeImage(inputPath) {
  const filename = path.basename(inputPath, path.extname(inputPath));

  for (const size of SIZES) {
    const resized = sharp(inputPath).resize(size);

    // AVIF
    await resized
      .avif({ quality: 70 })
      .toFile(`${OUTPUT_DIR}/${filename}-${size}.avif`);

    // WebP
    await resized
      .webp({ quality: 80 })
      .toFile(`${OUTPUT_DIR}/${filename}-${size}.webp`);

    // JPEG fallback
    await resized
      .jpeg({ quality: 80, progressive: true })
      .toFile(`${OUTPUT_DIR}/${filename}-${size}.jpg`);
  }
}

// Process all images
fs.readdirSync(INPUT_DIR)
  .filter(f => /\.(jpg|jpeg|png)$/i.test(f))
  .forEach(f => optimizeImage(path.join(INPUT_DIR, f)));
```

Run: `node scripts/optimize-images.js`

---

## Framework Components

### Next.js Image

```javascript
import Image from 'next/image';

// Automatic optimization
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority // For LCP images
/>

// Fill container
<div style={{ position: 'relative', height: 400 }}>
  <Image src="/bg.jpg" alt="Background" fill style={{ objectFit: 'cover' }} />
</div>
```

### Astro Image

```astro
---
import { Image } from 'astro:assets';
import hero from '../assets/hero.png';
---

<Image src={hero} alt="Hero" width={1200} height={600} />
```

### Vite imagetools

```javascript
// vite.config.js
import { imagetools } from 'vite-imagetools';

export default {
  plugins: [imagetools()]
};

// Usage in code
import heroSrcset from './hero.jpg?w=400;800;1200&format=webp&as=srcset';
```
