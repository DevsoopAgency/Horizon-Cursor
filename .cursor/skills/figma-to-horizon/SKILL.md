---
name: figma-to-horizon
description: >-
  Translates a Figma section into Shopify Horizon theme sections/blocks and
  updates the matching template JSON with Figma copy and Shopify image
  placeholders. When desktop and mobile Figma refs are both provided, builds one
  adaptive section that matches both layouts (stock responsive settings first,
  additive schema only if needed). Use when the user pastes a Figma section URL,
  asks to build a Figma design in Horizon, map a Figma frame to theme sections,
  or populate templates/index.json (or other templates) from Figma.
disable-model-invocation: false
---

# Figma → Horizon Section

## Core directive (verbatim)

I’ll provide you with the Figma section URL. Review the design and translate it into the Horizon theme using the existing sections that best match the design. Always try to reuse existing sections first. If the design can’t be matched without structural changes, update the existing section and add new settings following the established rules.
also update the json of that template and popluate the same info you got in the figma sepcailly copy add the placeholder images from shopify any i will update it later manually

When I provide both desktop and mobile Figma references, build the section so it adapts to both layouts. Match Figma’s mobile design — don’t assume a simple stack. Prefer stock Horizon responsive settings (`vertical_on_mobile`, column direction, alignment, padding, gap, section height) before structural Liquid/CSS changes. Only extend the section with additive settings or markup when existing options can’t reproduce the desktop↔mobile difference.

## Mandatory context

1. Follow [horizon-theme-build](../../rules/horizon-theme-build.mdc) for architecture, reuse-first, additive settings, and safety.
2. Before `get_design_context`, load the Figma MCP skill `figma-design-to-code` (resource).
3. Pass `skillNames` including `resource:figma-design-to-code` and this skill’s intent when calling `get_design_context`.

## Workflow

### 1. Parse the Figma URL(s)

- Extract `fileKey` and `nodeId` (`node-id=1-2` → `1:2`).
- If there is no `node-id`, ask for a section/frame URL — do not guess.
- If the user gives **desktop and mobile** frames/URLs, pull both. Treat them as one responsive section, not two separate sections (unless Figma clearly uses different content per breakpoint).

### 2. Pull design context

- Call `get_design_context` on the target node(s). For dual refs, run both desktop and mobile nodes.
- Use `get_screenshot` only to validate layout after mapping (desktop + mobile viewports when dual refs exist), not as a substitute for design context.
- Capture: structure, hierarchy, copy (headings, body, CTAs), spacing/alignment intent, media slots, colors only when they differ from theme palette.
- **Responsive delta (required when both refs exist):** note what changes between desktop and mobile — layout direction, order, alignment, type scale, padding, media aspect/crop, visibility of elements, CTA placement. Build to those deltas explicitly.

### 3. Map to stock Horizon first

Pick the closest existing section(s) from the inventory in the theme-build rule. Prefer composing via `templates/*.json` + nested `@theme` blocks over new Liquid.

Always try existing sections first. Only make structural changes or add new settings when stock composition + existing responsive settings cannot match the design (including mobile).

Common mappings (start here, not exhaustive):

| Figma pattern | Try first |
|---|---|
| Full-bleed media + text/CTA | `hero`, `media-with-content`, `slideshow` |
| Multi-slide / layered media | `slideshow`, `layered-slideshow` |
| Product grid / collection | `product-list`, `featured-collection` / product card blocks |
| Collection tiles / links | `collection-list`, `collection-links` |
| Marquee / ticker | `marquee` |
| FAQ / stacked panels | `section` + `accordion` / `group` / `text` |
| Blog row | `featured-blog-posts` |
| Generic content band | `section` + `text` / `button` / `image` / `group` |

Announce which stock section(s) you chose and why (one short line). If dual refs: one line on how desktop↔mobile will adapt (settings used or additive change required).

### 4. Only then extend stock sections

If structure cannot match without hacks (including a desktop/mobile layout that stock settings can’t express):

- **Do not** hard-code CSS overrides on stock classes.
- **Do** add additive schema settings on the existing section/block (new `id`s, defaults = current Horizon look). Prefer settings that drive CSS variables / conditional classes for mobile vs desktop.
- Never rename/remove/repurpose existing setting `id`s.
- New sections/blocks only as last resort — and they must use `{% content_for 'blocks' %}` + `"blocks": [{ "type": "@theme" }, { "type": "@app" }]` and Horizon-standard setting IDs.

### 5. Update the template JSON

- Target the template the user names (`templates/index.json`, `product.json`, etc.). If unspecified, ask once — do not assume.
- Preserve the Shopify auto-generated comment header.
- Insert or update the section instance: `type`, `blocks`, `block_order`, `settings`, and parent `order` array.
- **Copy:** populate every text/richtext/button label from Figma (same wording). Use real Figma strings, not lorem.
- **Links:** use sensible Shopify defaults when Figma has no URL (`shopify://collections/all`, `#`, or leave blank per setting type).
- **Images / media (Horizon override of Figma asset fidelity):**
  - Do **not** commit Figma MCP export URLs (they expire).
  - Leave image/video picker settings **empty / unset** so Horizon’s built-in Shopify placeholders render.
  - Do **not** download Figma assets into `assets/` unless the user explicitly asks.
  - Merchant will replace placeholders in the theme editor later.
- Match layout settings to design intent using existing IDs (`section_width`, `section_height`, alignment, gap, padding-block-*, overlay, `vertical_on_mobile`, column flex direction, etc.) — prefer settings over custom CSS.
- When dual refs exist, set those responsive IDs so mobile matches the mobile frame (not a generic collapse).
- Keep block nesting valid for Horizon theme blocks (public types like `text`, `button`, `image`, `group`, plus private `_*.liquid` only when that section already uses them).

### 6. Validate before finishing

- Defaults unchanged for unrelated sections.
- New settings optional; blank/default = prior Horizon look.
- No global CSS restyling unrelated components.
- Template JSON is valid and the section appears in `order`.
- Desktop and mobile (when both refs provided) visually match Figma intent — check via screenshot or theme preview breakpoints.
- Optional: Shopify theme MCP `validate_theme` on touched files when available.

## Output style

Be terse. Expert audience. Lead with: section chosen → responsive approach (stock settings vs additive) → files touched → what still needs a manual image swap in the editor.

## Anti-patterns

- Forking/gutting core Horizon sections when settings + JSON composition would work
- Dawn-style local-only section blocks (`"type": "heading"`) instead of `@theme` blocks
- Pasting Figma React/Tailwind output into the theme
- Hard-coded one-off CSS instead of additive schema settings
- Inventing parallel layout systems instead of Horizon setting IDs + `section` snippet patterns
- Building desktop-only and “hoping” mobile stacks correctly when a mobile Figma frame was provided
- Duplicating the same section once for desktop and once for mobile when one adaptive section can cover both
- Structural Liquid/CSS changes before exhausting stock responsive settings (`vertical_on_mobile`, alignment, gap, padding, height)
