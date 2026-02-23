---
name: figma
description: Use the Figma MCP server to fetch design context, screenshots, variables, and assets from Figma, and to translate Figma nodes into production code. Also handles SVG icon extraction from Figma: export icons as SVG, create React SVG icon components for reuse. Trigger when a task involves Figma URLs, node IDs, design-to-code implementation, icon extraction, or Figma MCP setup and troubleshooting.
metadata:
  author: github.com/openai/skills
  version: '2.0.0'
---

# Figma MCP

Use the Figma MCP server for Figma-driven implementation. For setup and debugging details (env vars, config, verification), see `references/figma-mcp-config.md`. For SVG icon extraction, see `references/figma-svg-icons.md`.

## Figma MCP Integration Rules

These rules define how to translate Figma inputs into code for this project and must be followed for every Figma-driven change.

### Required flow (do not skip)

1. Run `get_design_context` first to fetch the structured representation for the exact node(s).
2. If the response is too large or truncated, run `get_metadata` to get the high-level node map and then re-fetch only the required node(s) with `get_design_context`.
3. Run `get_screenshot` for a visual reference of the node variant being implemented.
4. Run `get_variable_defs` on the root frame or style guide node to extract Figma design tokens — map them to the project's existing CSS variables / design tokens before writing any code.
5. Only after you have `get_design_context`, `get_screenshot`, and design token mapping, download any assets needed and start implementation.
6. Translate the output (usually React + Tailwind) into this project's conventions, styles and framework. Reuse the project's color tokens, components, and typography wherever possible.
7. Validate against Figma for 1:1 look and behavior before marking complete.

### Implementation rules

- Treat the Figma MCP output (React + Tailwind) as a representation of design and behavior, not as final code style.
- Replace Tailwind utility classes with the project's preferred utilities/design-system tokens when applicable.
- Reuse existing components (e.g., buttons, inputs, typography, icon wrappers) instead of duplicating functionality.
- Use the project's color system, typography scale, and spacing tokens consistently.
- Respect existing routing, state management, and data-fetch patterns already adopted in the repo.
- Strive for 1:1 visual parity with the Figma design. When conflicts arise, prefer design-system tokens and adjust spacing or sizes minimally to match visuals.
- Validate the final UI against the Figma screenshot for both look and behavior.

### Auto-Layout → CSS Translation (mandatory)

Always read `layoutMode`, `primaryAxisAlignItems`, `counterAxisAlignItems`, `paddingTop/Right/Bottom/Left`, and `itemSpacing` from `get_design_context` and translate directly:

| Figma Property | CSS Output |
|---|---|
| `layoutMode: HORIZONTAL` | `display:flex; flex-direction:row` |
| `layoutMode: VERTICAL` | `display:flex; flex-direction:column` |
| `itemSpacing` | `gap: {value}px` |
| `paddingTop/Right/Bottom/Left` | `padding: {t}px {r}px {b}px {l}px` |
| `layoutSizingHorizontal: FILL` | `flex:1` or `width:100%` |
| `layoutSizingHorizontal: HUG` | `width:fit-content` |
| `layoutSizingHorizontal: FIXED` | `width:{value}px; flex-shrink:0` |

### Asset handling

- The Figma MCP Server provides an assets endpoint which can serve image and SVG assets.
- IMPORTANT: If the Figma MCP Server returns a localhost source for an image or an SVG, use that image or SVG source directly.
- IMPORTANT: DO NOT import/add new icon packages — all assets must come from the Figma payload.
- IMPORTANT: DO NOT use or create placeholders if a localhost source is provided.

### SVG Icon Extraction (triggered automatically when icons are detected)

When any of the following are true, apply the full icon extraction workflow from `references/figma-svg-icons.md`:

- The design context contains node types `VECTOR`, `BOOLEAN_OPERATION`, `COMPONENT_SET` named like icons
- The Figma layer names match patterns: `Icon/`, `icon-`, `ic_`, or common icon names (arrow, close, search, chevron, star, check, etc.)
- User explicitly requests: "export icons", "create icon components", "extract SVG icons"

**Icon extraction rules (non-negotiable):**
1. Extract icons as React SVG components — NEVER as `<img src="...">` or from an npm icon package
2. Clean SVG: remove hardcoded colors → `currentColor`, remove `width`/`height` from `<svg>`, ensure `viewBox` is present
3. Use `forwardRef` + `SVGProps<SVGSVGElement>` pattern with `size` prop (full template in `figma-implement-design` skill)
4. Decorative icons: `aria-hidden="true"` (default). Standalone meaningful icons: `title` prop or parent `aria-label`
5. Save to `src/components/icons/{IconName}Icon.tsx`; re-export from `src/components/icons/index.ts`
6. Validate visually against `get_screenshot` before marking done

### Design Token Extraction (run before implementation)

```
get_variable_defs(fileKey=":fileKey", nodeId=":styleGuideNodeId")
```

Map Figma variables → project tokens before writing any component code. Document deviations in a comment at the top of the component file.

### Link-based prompting

- The server is link-based: copy the Figma frame/layer link and give that URL to the MCP client when asking for implementation help.
- The client cannot browse the URL but extracts the node ID from the link; always ensure the link points to the exact node/variant you want.

## References

- `references/figma-mcp-config.md` — setup, verification, troubleshooting, and link-based usage reminders.
- `references/figma-tools-and-prompts.md` — tool catalog and prompt patterns for selecting frameworks/components and fetching metadata.
- `references/figma-svg-icons.md` — complete SVG icon extraction guide: naming, optimization, React component patterns, SVGR config, accessibility.
