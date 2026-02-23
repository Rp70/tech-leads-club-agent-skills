---
name: figma-implement-design
description: Translate Figma nodes into production-ready code with 1:1 visual fidelity using the Figma MCP workflow (design context, screenshots, assets, and project-convention translation). Trigger when the user provides Figma URLs or node IDs, or asks to implement designs or components that must match Figma specs. Requires a working Figma MCP server connection.
metadata:
  author: github.com/openai/skills
  version: '1.0.0'
---

# Implement Design

## Overview

This skill provides a structured workflow for translating Figma designs into production-ready code with pixel-perfect accuracy. It ensures consistent integration with the Figma MCP server, proper use of design tokens, and 1:1 visual parity with designs.

## Prerequisites

- Figma MCP server must be connected and accessible
- User must provide a Figma URL in the format: `https://figma.com/design/:fileKey/:fileName?node-id=1-2`
  - `:fileKey` is the file key
  - `1-2` is the node ID (the specific component or frame to implement)
- **OR** when using `figma-desktop` MCP: User can select a node directly in the Figma desktop app (no URL required)
- Project should have an established design system or component library (preferred)

## Required Workflow

**Follow these steps in order. Do not skip steps.**

### Step 0: Set up Figma MCP (if not already configured)

If any MCP call fails because Figma MCP is not connected, pause and set it up:

1. Add the Figma MCP server to your agent's MCP configuration:
   - URL: `https://mcp.figma.com/mcp`
2. Enable remote MCP client if required by your agent.
3. Log in with OAuth following your agent's authentication flow.

After successful login, the user will have to restart their agent. You should finish your answer and tell them so when they try again they can continue with Step 1.

### Step 1: Get Node ID

#### Option A: Parse from Figma URL

When the user provides a Figma URL, extract the file key and node ID to pass as arguments to MCP tools.

**URL format:** `https://figma.com/design/:fileKey/:fileName?node-id=1-2`

**Extract:**

- **File key:** `:fileKey` (the segment after `/design/`)
- **Node ID:** `1-2` (the value of the `node-id` query parameter)

**Note:** When using the local desktop MCP (`figma-desktop`), `fileKey` is not passed as a parameter to tool calls. The server automatically uses the currently open file, so only `nodeId` is needed.

**Example:**

- URL: `https://figma.com/design/kL9xQn2VwM8pYrTb4ZcHjF/DesignSystem?node-id=42-15`
- File key: `kL9xQn2VwM8pYrTb4ZcHjF`
- Node ID: `42-15`

#### Option B: Use Current Selection from Figma Desktop App (figma-desktop MCP only)

When using the `figma-desktop` MCP and the user has NOT provided a URL, the tools automatically use the currently selected node from the open Figma file in the desktop app.

**Note:** Selection-based prompting only works with the `figma-desktop` MCP server. The remote server requires a link to a frame or layer to extract context. The user must have the Figma desktop app open with a node selected.

### Step 2: Fetch Design Context

Run `get_design_context` with the extracted file key and node ID.

```
get_design_context(fileKey=":fileKey", nodeId="1-2")
```

This provides the structured data including:

- Layout properties (Auto Layout, constraints, sizing)
- Typography specifications
- Color values and design tokens
- Component structure and variants
- Spacing and padding values

**If the response is too large or truncated:**

1. Run `get_metadata(fileKey=":fileKey", nodeId="1-2")` to get the high-level node map
2. Identify the specific child nodes needed from the metadata
3. Fetch individual child nodes with `get_design_context(fileKey=":fileKey", nodeId=":childNodeId")`

### Step 3: Capture Visual Reference

Run `get_screenshot` with the same file key and node ID for a visual reference.

```
get_screenshot(fileKey=":fileKey", nodeId="1-2")
```

This screenshot serves as the source of truth for visual validation. Keep it accessible throughout implementation.

### Step 4: Download Required Assets

Download any assets (images, icons, SVGs) returned by the Figma MCP server.

**IMPORTANT:** Follow these asset rules:

- If the Figma MCP server returns a `localhost` source for an image or SVG, use that source directly
- DO NOT import or add new icon packages - all assets should come from the Figma payload
- DO NOT use or create placeholders if a `localhost` source is provided
- Assets are served through the Figma MCP server's built-in assets endpoint

### Step 5: Translate to Project Conventions

Translate the Figma output into this project's framework, styles, and conventions.

**Key principles:**

- Treat the Figma MCP output (typically React + Tailwind) as a representation of design and behavior, not as final code style
- Replace Tailwind utility classes with the project's preferred utilities or design system tokens
- Reuse existing components (buttons, inputs, typography, icon wrappers) instead of duplicating functionality
- Use the project's color system, typography scale, and spacing tokens consistently
- Respect existing routing, state management, and data-fetch patterns

### Step 6: Achieve 1:1 Visual Parity

Strive for pixel-perfect visual parity with the Figma design.

**Guidelines:**

- Prioritize Figma fidelity to match designs exactly
- Avoid hardcoded values - use design tokens from Figma where available
- When conflicts arise between design system tokens and Figma specs, prefer design system tokens but adjust spacing or sizes minimally to match visuals
- Follow WCAG requirements for accessibility
- Add component documentation as needed

### Step 7: Validate Against Figma

Before marking complete, validate the final UI against the Figma screenshot.

**Validation checklist:**

- [ ] Layout matches (spacing, alignment, sizing)
- [ ] Typography matches (font, size, weight, line height)
- [ ] Colors match exactly
- [ ] Interactive states work as designed (hover, active, disabled)
- [ ] Responsive behavior follows Figma constraints
- [ ] Assets render correctly
- [ ] Accessibility standards met

## Implementation Rules

### Component Organization

- Place UI components in the project's designated design system directory
- Follow the project's component naming conventions
- Avoid inline styles unless truly necessary for dynamic values

### Design System Integration

- ALWAYS use components from the project's design system when possible
- Map Figma design tokens to project design tokens
- When a matching component exists, extend it rather than creating a new one
- Document any new components added to the design system

### Code Quality

- Avoid hardcoded values - extract to constants or design tokens
- Keep components composable and reusable
- Add TypeScript types for component props
- Include JSDoc comments for exported components

## Examples

### Example 1: Implementing a Button Component

User says: "Implement this Figma button component: https://figma.com/design/kL9xQn2VwM8pYrTb4ZcHjF/DesignSystem?node-id=42-15"

**Actions:**

1. Parse URL to extract fileKey=`kL9xQn2VwM8pYrTb4ZcHjF` and nodeId=`42-15`
2. Run `get_design_context(fileKey="kL9xQn2VwM8pYrTb4ZcHjF", nodeId="42-15")`
3. Run `get_screenshot(fileKey="kL9xQn2VwM8pYrTb4ZcHjF", nodeId="42-15")` for visual reference
4. Download any button icons from the assets endpoint
5. Check if project has existing button component
6. If yes, extend it with new variant; if no, create new component using project conventions
7. Map Figma colors to project design tokens (e.g., `primary-500`, `primary-hover`)
8. Validate against screenshot for padding, border radius, typography

**Result:** Button component matching Figma design, integrated with project design system.

### Example 2: Building a Dashboard Layout

User says: "Build this dashboard: https://figma.com/design/pR8mNv5KqXzGwY2JtCfL4D/Dashboard?node-id=10-5"

**Actions:**

1. Parse URL to extract fileKey=`pR8mNv5KqXzGwY2JtCfL4D` and nodeId=`10-5`
2. Run `get_metadata(fileKey="pR8mNv5KqXzGwY2JtCfL4D", nodeId="10-5")` to understand the page structure
3. Identify main sections from metadata (header, sidebar, content area, cards) and their child node IDs
4. Run `get_design_context(fileKey="pR8mNv5KqXzGwY2JtCfL4D", nodeId=":childNodeId")` for each major section
5. Run `get_screenshot(fileKey="pR8mNv5KqXzGwY2JtCfL4D", nodeId="10-5")` for the full page
6. Download all assets (logos, icons, charts)
7. Build layout using project's layout primitives
8. Implement each section using existing components where possible
9. Validate responsive behavior against Figma constraints

**Result:** Complete dashboard matching Figma design with responsive layout.

## Best Practices

### Always Start with Context

Never implement based on assumptions. Always fetch `get_design_context` and `get_screenshot` first.

### Incremental Validation

Validate frequently during implementation, not just at the end. This catches issues early.

### Document Deviations

If you must deviate from the Figma design (e.g., for accessibility or technical constraints), document why in code comments.

### Reuse Over Recreation

Always check for existing components before creating new ones. Consistency across the codebase is more important than exact Figma replication.

### Design System First

When in doubt, prefer the project's design system patterns over literal Figma translation.

---

## SVG Icon Extraction Workflow

### Overview

When implementing Figma designs that contain icons, ALWAYS extract them as standalone React SVG components — never use raster images or third-party icon libraries unless the icon does not exist in Figma. This ensures exact pixel-fidelity, tree-shaking, and universal reuse.

### Step-by-Step: Icon Extraction

#### 1. Identify Icon Nodes

In the `get_design_context` or `get_metadata` response, look for:
- Nodes of type `COMPONENT`, `COMPONENT_SET`, or `FRAME` with names matching patterns like `Icon/`, `icon-`, `ic_`, or containing common icon names (arrow, close, search, etc.)
- Vector nodes (`VECTOR`, `BOOLEAN_OPERATION`, `ELLIPSE`, `POLYGON`) that represent icon paths

When a component set is dedicated to icons (e.g. `Icons/16`, `Icons/24`), fetch ALL its children:

```
get_metadata(fileKey=":fileKey", nodeId=":iconSetNodeId")
```

Then fetch each variant individually if needed.

#### 2. Fetch SVG Source via MCP

For each icon node, use `get_design_context` which returns SVG data via the localhost assets endpoint:

```
get_design_context(fileKey=":fileKey", nodeId=":iconNodeId")
```

**Rules:**
- If the MCP response includes a `localhost` URL for an SVG asset, use it **directly** — do not re-download or transform the URL
- If the response contains inline SVG `<path>` data in the design context, extract the raw SVG markup
- Never substitute with an icon font or npm icon package

#### 3. Clean and Optimize the SVG

Before creating React components, normalize the raw SVG:

```
MANDATORY cleanup steps (apply in order):
1. Remove width/height attributes from <svg> — these are controlled via props
2. Add viewBox if missing — derive from width/height (e.g., "0 0 24 24")
3. Replace any hardcoded fill/stroke color (e.g., fill="#1A1A1A") with currentColor
   EXCEPTION: Multi-color icons — keep distinct colors, expose them as CSS custom properties
4. Remove any <title>, <desc>, <defs> with IDs that may clash (rename IDs to use component name prefix)
5. Remove data-* and Figma-specific attributes (e.g., data-figma-*, inkscape:*)
6. Merge redundant transform="translate(0,0)" or equivalent no-ops
7. Keep all path data exactly as-is — never manually simplify paths
```

If `svgo` is available in the project: `pnpm exec svgo --multipass icon.svg`

#### 4. React SVG Component Pattern

Use this canonical pattern for every exported React SVG icon:

```tsx
// src/components/icons/ArrowRightIcon.tsx
import { forwardRef, SVGProps } from 'react';

export interface ArrowRightIconProps extends SVGProps<SVGSVGElement> {
  /** Size in pixels. Overrides width and height. Defaults to 24. */
  size?: number;
  /** Title for accessibility when icon is used standalone (not decorative). */
  title?: string;
}

const ArrowRightIcon = forwardRef<SVGSVGElement, ArrowRightIconProps>(
  ({ size = 24, title, className, 'aria-label': ariaLabel, ...props }, ref) => {
    const isDecorative = !title && !ariaLabel;
    return (
      <svg
        ref={ref}
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 24 24"
        width={size}
        height={size}
        fill="currentColor"
        className={className}
        aria-hidden={isDecorative ? true : undefined}
        aria-label={ariaLabel}
        role={isDecorative ? undefined : 'img'}
        {...props}
      >
        {title && <title>{title}</title>}
        {/* Paste exact paths from Figma SVG here */}
        <path d="M..." />
      </svg>
    );
  }
);
ArrowRightIcon.displayName = 'ArrowRightIcon';

export default ArrowRightIcon;
```

**Key rules:**
- `forwardRef` ALWAYS — enables consumers to attach refs (e.g., tooltips, popovers)
- `size` prop controls both `width` and `height` uniformly; individual `width`/`height` props are still passable for non-square icons
- `currentColor` fill — icon color inherits from CSS `color` property
- Decorative icons (default): `aria-hidden="true"`, no role
- Standalone meaningful icons: require `title` or `aria-label`; set `role="img"`
- `SVGProps<SVGSVGElement>` spread — allows `className`, `style`, `onClick`, etc.
- `displayName` for React DevTools

#### 5. Multi-Color Icon Pattern

When Figma uses multiple intentional colors (e.g., a brand logo or status indicator):

```tsx
// Example: brand logo with 2 intentional colors
interface TwoColorIconProps extends Omit<SVGProps<SVGSVGElement>, 'fill'> {
  size?: number;
  primaryColor?: string;   // defaults to CSS --icon-color-primary or currentColor
  accentColor?: string;    // defaults to CSS --icon-color-accent or project token
}
```

Map Figma layer fill colors to CSS custom property defaults. Never hardcode hex values.

#### 6. File Organization

```
src/
  components/
    icons/
      ArrowRightIcon.tsx
      CloseIcon.tsx
      SearchIcon.tsx
      CheckIcon.tsx
      ...
      index.ts              ← barrel export for all icons
```

`index.ts` barrel:
```ts
// src/components/icons/index.ts
export { default as ArrowRightIcon } from './ArrowRightIcon';
export type { ArrowRightIconProps } from './ArrowRightIcon';
export { default as CloseIcon } from './CloseIcon';
// ... one export per icon
```

**Naming convention:** `{FigmaLayerName}Icon.tsx` using PascalCase. Strip Figma prefixes like `ic_`, `icon/`, `Icon/`.

#### 7. Icon Size System

Figma icons typically come in multiple sizes (16, 20, 24, 32). Check the Figma component set for available sizes.

```tsx
// src/components/icons/types.ts
export type IconSize = 12 | 16 | 20 | 24 | 32 | 48;
// Use the size that matches the Figma frame/component frame, not the artboard size
```

Default size should match the most common usage in the Figma design.

#### 8. Validation Checklist for Icons

Before marking icon extraction complete:

- [ ] SVG paths match Figma exactly (compare `get_screenshot` output side by side)
- [ ] `viewBox` is correct — no cropping or overflow
- [ ] Fill/stroke uses `currentColor` (or explicit multi-color props)
- [ ] No hardcoded color hex values in `fill` or `stroke`
- [ ] No width/height attributes on `<svg>` root (controlled via `size` prop)
- [ ] `forwardRef` implemented
- [ ] `aria-hidden` set for decorative, `title`/`aria-label` accessible for standalone
- [ ] Barrel export added to `index.ts`
- [ ] TypeScript compiles with zero errors

### Icon Reuse in Parent Components

When using an extracted icon in another component, always import from the icon barrel:

```tsx
// ✅ Correct — import from barrel
import { ArrowRightIcon, CloseIcon } from '@/components/icons';

// ❌ Wrong — deep import that bypasses barrel
import ArrowRightIcon from '@/components/icons/ArrowRightIcon';
```

Set icon color via the parent's CSS `color` property (leverages `currentColor`):

```tsx
<button className="text-primary-600 hover:text-primary-700">
  <ArrowRightIcon size={16} />
  Continue
</button>
```

---

## Exact Figma Replication: Advanced Rules

### Design Token Extraction

Before coding any component, extract Figma variables/styles into project tokens:

1. Run `get_variable_defs(fileKey, nodeId)` on the root frame or style guide node
2. Map each Figma variable to the project's existing token (CSS variable or JS constant)
3. Create new tokens in the project's token file only if no mapping exists
4. Token mapping table (maintain this per project):

| Figma Variable | Project Token | Notes |
|---|---|---|
| `color/brand/primary` | `--color-primary-600` | from Figma design context |
| `spacing/4` | `--spacing-1` | 4px base unit |
| `typography/body-md` | `text-base` | Tailwind class or CSS rule |

### Auto-Layout → Flexbox/Grid Translation

Figma Auto Layout properties map directly to CSS:

| Figma Auto Layout | CSS Equivalent |
|---|---|
| Direction: Horizontal | `display: flex; flex-direction: row` |
| Direction: Vertical | `display: flex; flex-direction: column` |
| Spacing: between items (gap) | `gap: {value}px` |
| Padding: {top} {right} {bottom} {left} | `padding: {t}px {r}px {b}px {l}px` |
| Alignment: Top Left | `align-items: flex-start; justify-content: flex-start` |
| Size: Fill container | `flex: 1` or `width: 100%` |
| Size: Hug contents | `width: fit-content` or no explicit width |
| Size: Fixed | `width: {value}px; flex-shrink: 0` |

Always check `get_design_context` for the `layoutMode`, `primaryAxisAlignItems`, `counterAxisAlignItems`, `paddingTop/Right/Bottom/Left`, and `itemSpacing` fields.

### Typography: Figma → Code

```
Figma Text Properties → CSS
─────────────────────────────────────────────────────
fontFamily        → font-family (map to project font stack)
fontSize          → font-size (convert to rem: px / 16)
fontWeight        → font-weight (100–900)
lineHeightPx      → line-height (convert to unitless: lineHeightPx / fontSize)
letterSpacing     → letter-spacing (convert to em: ls_px / fontSize)
textAlignHorizontal → text-align (LEFT→left, CENTER→center, RIGHT→right)
textDecoration    → text-decoration
textTransform     → text-transform (UPPER→uppercase, LOWER→lowercase)
```

### Shadow: Figma → CSS box-shadow

```
Figma Effect: DROP_SHADOW
  x, y, blur, spread, color(r,g,b,a)
→ CSS: box-shadow: {x}px {y}px {blur}px {spread}px rgba({r},{g},{b},{a})

Figma Effect: INNER_SHADOW
→ CSS: box-shadow: inset {x}px {y}px {blur}px {spread}px rgba(...)
```

### Border Radius

Figma uses individual corner radii. Always extract all four values:
- If all four equal → `border-radius: {value}px`
- If mixed → `border-radius: {tl}px {tr}px {br}px {bl}px`
- Figma "percent" radius (smooth corners) → add `border-radius` with same px value (CSS doesn't support smooth corners natively; use `squircle` polyfill if required)

### Pixel-Perfect Validation Technique

After implementation, perform a visual diff:

1. Open the implementation in browser (dev server)
2. Take a screenshot at `1x` device pixel ratio (avoid Retina/HiDPI for comparison)
3. Layer the Figma screenshot on top using Chrome DevTools overlay or Figma mirror
4. Check for deviations >1px in spacing, >1px in sizing, or any color mismatch
5. Fix deviations by adjusting to exact Figma values, then re-validate

**Common causes of mismatch:**
- Forgetting to apply Figma's `gap` vs `padding` correctly
- Text `line-height` being browser-default vs Figma value
- Missing `box-sizing: border-box` on elements with padding + fixed width
- Incorrect `overflow: visible` vs `overflow: hidden` on containers

---

## Common Issues and Solutions

### Issue: Figma output is truncated

**Cause:** The design is too complex or has too many nested layers to return in a single response.
**Solution:** Use `get_metadata` to get the node structure, then fetch specific nodes individually with `get_design_context`.

### Issue: Design doesn't match after implementation

**Cause:** Visual discrepancies between the implemented code and the original Figma design.
**Solution:** Compare side-by-side with the screenshot from Step 3. Check spacing, colors, and typography values in the design context data.

### Issue: Assets not loading

**Cause:** The Figma MCP server's assets endpoint is not accessible or the URLs are being modified.
**Solution:** Verify the Figma MCP server's assets endpoint is accessible. The server serves assets at `localhost` URLs. Use these directly without modification.

### Issue: Design token values differ from Figma

**Cause:** The project's design system tokens have different values than those specified in the Figma design.
**Solution:** When project tokens differ from Figma values, prefer project tokens for consistency but adjust spacing/sizing to maintain visual fidelity.

## Understanding Design Implementation

The Figma implementation workflow establishes a reliable process for translating designs to code:

**For designers:** Confidence that implementations will match their designs with pixel-perfect accuracy.
**For developers:** A structured approach that eliminates guesswork and reduces back-and-forth revisions.
**For teams:** Consistent, high-quality implementations that maintain design system integrity.

By following this workflow, you ensure that every Figma design is implemented with the same level of care and attention to detail.

## Additional Resources

- [Figma MCP Server Documentation](https://developers.figma.com/docs/figma-mcp-server/)
- [Figma MCP Server Tools and Prompts](https://developers.figma.com/docs/figma-mcp-server/tools-and-prompts/)
- [Figma Variables and Design Tokens](https://help.figma.com/hc/en-us/articles/15339657135383-Guide-to-variables-in-Figma)
