# Figma SVG Icon Extraction — Reference Guide

## Decision Flowchart

```
User mentions icons / implements Figma design with icons
    │
    ▼
Does the Figma design contain icon components/sets?
    ├─ YES → Use this guide (Figma MCP icon extraction)
    └─ NO  → Check if icons exist in project's icon library
                ├─ YES → Reuse existing icons
                └─ NO  → Ask user for icon source; prefer Figma
```

---

## MCP Tool Sequence for Icon Extraction

```
1. get_metadata(fileKey, iconSetNodeId)
   → Returns all icon variant node IDs

2. For each icon node:
   get_design_context(fileKey, iconNodeId)
   → Returns SVG paths, fill colors, viewBox, dimensions

3. get_screenshot(fileKey, iconNodeId)
   → Visual reference for validation

4. Assets endpoint (auto-provided in response)
   → localhost URLs for SVG files — use directly
```

---

## Naming Convention

| Figma Layer Name | React Component Name | File Name |
|---|---|---|
| `Icon/Arrow/Right` | `ArrowRightIcon` | `ArrowRightIcon.tsx` |
| `ic_close` | `CloseIcon` | `CloseIcon.tsx` |
| `icon-search-24` | `SearchIcon` | `SearchIcon.tsx` |
| `Chevron Down` | `ChevronDownIcon` | `ChevronDownIcon.tsx` |
| `Star/Filled` | `StarFilledIcon` | `StarFilledIcon.tsx` |

**Rules:**
- Strip prefixes: `Icon/`, `ic_`, `icon-`
- Suffix with `Icon` always
- PascalCase for component name; same casing for file (minus `.tsx`)
- Include size variant in name only when multiple sizes exported: `ArrowRight16Icon`, `ArrowRight24Icon`

---

## SVG Optimization Checklist (Manual)

Apply before creating the React component:

- [ ] Remove `width` and `height` from `<svg>` root
- [ ] Verify `viewBox="0 0 W H"` exists (add if missing, using original Figma width/height)
- [ ] Replace any static hex `fill` with `currentColor`
- [ ] Replace any static hex `stroke` with `currentColor`
- [ ] Remove `fill="none"` on paths that should inherit (keep intentional `none` fills)
- [ ] Remove Figma-generated IDs or rename them with component-name prefix to avoid clashes in the DOM
- [ ] Remove all `data-*` attributes
- [ ] Remove `<defs>` if they only contain empty gradients or unused clip paths
- [ ] Ensure no `transform` on the root `<svg>` (should be on child groups if needed)

---

## SVGO Config (if `svgo` is available)

Create `.svgorc.json` in the icons directory:

```json
{
  "plugins": [
    "preset-default",
    {
      "name": "removeAttrs",
      "params": {
        "attrs": ["data-.*", "inkscape:.*", "sodipodi:.*"]
      }
    },
    {
      "name": "removeViewBox",
      "active": false
    },
    {
      "name": "convertColors",
      "params": {
        "currentColor": true
      }
    }
  ]
}
```

Run: `pnpm exec svgo --config .svgorc.json icon.svg -o icon.optimized.svg`

---

## SVGR Integration (Automated Generation)

For projects using `@svgr/webpack` or `@svgr/rollup`:

```js
// svgr.config.js
module.exports = {
  typescript: true,
  ref: true,                        // enables forwardRef
  svgProps: { role: '{role}' },     // custom prop injection
  template: require('./svgr-template.js'),
};
```

**Custom SVGR template** (`svgr-template.js`):

```js
const template = (variables, { tpl }) => tpl`
  import { forwardRef } from 'react';
  import type { SVGProps } from 'react';

  export interface ${variables.componentName}Props extends SVGProps<SVGSVGElement> {
    size?: number;
    title?: string;
  }

  const ${variables.componentName} = forwardRef<SVGSVGElement, ${variables.componentName}Props>(
    ({ size = 24, title, 'aria-label': ariaLabel, ...props }, ref) => {
      const isDecorative = !title && !ariaLabel;
      return (
        ${variables.jsx({
          ref: 'ref',
          width: '{size}',
          height: '{size}',
          'aria-hidden': '{isDecorative ? true : undefined}',
          role: '{isDecorative ? undefined : "img"}',
          ...variables.props,
        })}
      );
    }
  );
  ${variables.componentName}.displayName = '${variables.componentName}';
  export default ${variables.componentName};
`;

module.exports = template;
```

Generation command:
```bash
pnpm exec svgr --out-dir src/components/icons src/assets/icons/*.svg
```

---

## Multi-Size Icons

When Figma provides icons in multiple sizes (16, 20, 24, 32), use a single component with a `size` prop:

```tsx
// Single component, multiple sizes
<SearchIcon size={16} />   // small — for dense UIs
<SearchIcon size={20} />   // medium — for buttons  
<SearchIcon size={24} />   // default — for most contexts
<SearchIcon size={32} />   // large — for empty states, illustrations
```

If different sizes have meaningfully different path data (detail changes), create per-size variants and use a wrapper:

```tsx
const SearchIcon = ({ size = 24, ...props }) => {
  if (size <= 16) return <SearchIcon16 {...props} size={size} />;
  return <SearchIcon24 {...props} size={size} />;
};
```

---

## Accessibility: When to Use `title` vs `aria-label`

| Usage | Recommendation |
|---|---|
| Icon inside labelled button (`<button>Click me <CloseIcon /></button>`) | `aria-hidden="true"` (default, decorative) |
| Icon-only button (`<button><DeleteIcon /></button>`) | Use `aria-label` on the button, not the icon |
| Standalone icon conveying meaning (e.g., status indicator) | `<StatusIcon title="Error" />` |
| Icon with visible adjacent label | `aria-hidden="true"` (decorative, label covers it) |

**Rule:** Set accessible name on the interactive element (button, link), NOT on the icon itself, unless the icon IS the entire content.

---

## Figma Component Sets → Storybook Stories

When icons are extracted from a Figma component set, generate a Storybook story for documentation:

```tsx
// ArrowRightIcon.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import ArrowRightIcon from './ArrowRightIcon';

const meta: Meta<typeof ArrowRightIcon> = {
  title: 'Icons/ArrowRightIcon',
  component: ArrowRightIcon,
  argTypes: {
    size: { control: { type: 'select' }, options: [12, 16, 20, 24, 32] },
    color: { control: 'color' },
  },
};
export default meta;

type Story = StoryObj<typeof ArrowRightIcon>;

export const Default: Story = { args: { size: 24 } };
export const Small: Story = { args: { size: 16 } };
export const Colored: Story = {
  args: { size: 24, style: { color: 'var(--color-primary-600)' } },
};
```

---

## Troubleshooting

### Icon looks different from Figma

1. Check `viewBox` — Figma may export at 2x (48px) for a 24px icon. Set to `0 0 24 24`.
2. Check if Figma `clip-path` was dropped by SVGO — add `cleanupIds: false` to SVGO config.
3. Stroke-based icons may lose width: verify `stroke-width` is preserverd and `stroke="currentColor"`.

### Icon is clipped/cropped

The Figma component likely has overflow:hidden or a clip mask. Check `get_design_context` for `clipsContent: true`. Either:
- Fetch the icon's child frame instead of the outer frame
- Add padding to `viewBox`: `viewBox="-2 -2 28 28"` for a 24px icon with 2px overflow

### Paths are wrong / distorted

Never modify path `d` attribute values. If paths look wrong, re-fetch with `get_design_context` at the exact vector/path node level (not the frame).

### IDs clash when multiple icons on same page

All Figma-exported SVGs use generic IDs like `clip0`, `filter0`. Prefix with component name:
- `clip0` → `ArrowRightIcon-clip0`
- Also update references: `url(#clip0)` → `url(#ArrowRightIcon-clip0)`
