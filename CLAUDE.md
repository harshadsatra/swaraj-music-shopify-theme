# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Shopify store theme built on **Horizon** (Shopify's theme, v4.2.0) for swarajmusic.co.uk. There is no
build step, package manager, or bundler — this is the raw theme file structure that the Shopify CLI
serves and deploys directly.

## Commands

Requires the Shopify CLI (`shopify`, already installed globally). All commands run from the repo root.
Store is `swaraj-music.myshopify.com`; theme ID `201404744007` is the theme these scripts target.
`package.json` wraps the common ones as npm scripts:

- `npm run dev` — `shopify theme dev --store=swaraj-music.myshopify.com`; starts a local dev server with
  hot reload. Not pinned to a theme ID, so it spins up its own temporary preview theme rather than
  live-editing theme `201404744007` — pass `--theme=201404744007` explicitly if you want to preview
  against that theme's live data instead.
- `npm run check` — `shopify theme check`; runs Theme Check (Liquid/JSON linter) against local files only,
  no store/theme needed. No custom `.theme-check.yml` exists, so defaults apply.
- `npm run push` — `shopify theme push --store=... --theme=201404744007`; uploads local files straight
  into theme `201404744007` on the store. **Confirm that theme isn't the currently published/live theme
  before running this** — pushing overwrites it immediately for anyone viewing it. Use
  `shopify theme push --unpublished` (drop `--theme`) instead if you want to push to a brand-new
  unpublished theme for review first.
- `npm run pull` — `shopify theme pull --store=... --theme=201404744007`; pulls that theme's current
  state down over local files (careful: overwrites local changes).

There are no JS/CSS test suites or build scripts — validation is via `shopify theme check` and manual
verification with `shopify theme dev`.

`.shopify/` holds CLI-local state (theme metafield definitions cache, etc.) and is entirely gitignored.

## Architecture

### Directory roles (standard Shopify Online Store 2.0 + theme blocks)

- `layout/` — `theme.liquid` (main HTML shell) and `password.liquid`
- `templates/` — one JSON file per page type (e.g. `product.json`, `collection.json`) defining which
  sections render on that route, plus `gift_card.liquid` as a liquid-only template
- `sections/` — top-level layout units placed by templates (`.liquid` with an embedded `{% schema %}`,
  or `.json` for section groups like `header-group.json`/`footer-group.json`)
- `blocks/` — theme blocks, the composable unit nested inside sections (or inside other blocks)
- `snippets/` — reusable Liquid partials rendered via `{% render %}`; no schema, pure logic/markup
- `assets/` — JS (ES modules), CSS, and static files
- `config/` — `settings_schema.json` (theme editor settings definitions) and `settings_data.json`
  (current values for those settings)
- `locales/` — one `<lang>.json` (storefront-facing strings) and `<lang>.schema.json` (theme-editor
  strings) per supported language

### Theme blocks: public vs. private

`blocks/` mixes two kinds of file, distinguished by filename:

- **Public blocks** (no leading underscore, e.g. `blocks/button.liquid`, `blocks/product-card.liquid`) —
  addable directly by merchants in the theme editor, and referenceable in a section's schema via
  `"type": "@theme"` in its `blocks` array.
- **Private blocks** (leading underscore, e.g. `blocks/_card.liquid`, `blocks/_content.liquid`) — building
  blocks only relevant when nested inside a specific parent; not offered standalone in the editor.

A block/section that wants to accept nested theme blocks declares this in its `{% schema %}` with entries
like `{"type": "@theme"}` (any theme block) and `{"type": "@app"}` (app blocks), and renders the child
content via `{% content_for 'blocks' %}`. Presets in a section's schema can pre-populate nested blocks
with full settings and a `block_order` array — see `sections/hero.liquid` for a deeply nested example.

Use `{% doc %} ... {% enddoc %}` at the top of snippets/blocks to document expected params (already the
convention in ~30 files) — follow it when adding new reusable snippets/blocks.

### JavaScript: import maps + web components, no bundler

All theme JS is loaded as native ES modules via a browser import map defined in `snippets/scripts.liquid`,
mapping bare specifiers like `@theme/component`, `@theme/events`, `@theme/utilities` to
`{{ 'x.js' | asset_url }}`. There is no bundling — files in `assets/*.js` are shipped as-is and import
each other using these `@theme/...` (and `@shopify/events` for Shopify's standard analytics events)
specifiers, so a browser (or Theme Check) resolving imports needs the map from `scripts.liquid`, not
node_modules resolution.

Custom elements extend the shared base class in `assets/component.js` (`Component`, built on
`DeclarativeShadowElement`), which auto-wires `ref`-attributed child elements into `this.refs` and
supports declarative event listeners. Cross-component communication goes through DOM events defined
centrally in `assets/events.js` (`ThemeEvents` enum + typed `Event` subclasses like
`QuantitySelectorUpdateEvent`) — dispatch/listen via these rather than inventing new ad hoc event names.

When adding a script that a section/block needs, register it in the import map (if imported by specifier)
and/or add a `<script type="module" src="...">` / `<link rel="modulepreload">` tag in
`snippets/scripts.liquid`, following the existing `fetchpriority="low"` and conditional-load
(`{% if settings.x %}`) patterns used there for optional features.

### CSS

Single global stylesheet at `assets/base.css`, themed entirely through CSS custom properties (e.g.
`--color-foreground`, `--color-background`) set on `:root` and overridden per-section/block (see
`snippets/contrast-override.liquid`, used by blocks/sections to compute per-instance color overrides from
merchant-picked background colors). Component-specific styles live in dedicated snippets like
`*-styles.liquid` (e.g. `accordion-styles.liquid`, `cart-typography-styles.liquid`) rendered inline where
needed rather than one monolithic file per component.

### Metafields & translations

- Custom metafield definitions the theme depends on (product `custom.description`, `custom.upsell`,
  Google Shopping fields, etc.) are cached in `.shopify/metafields.json` for CLI reference — the
  authoritative source is the store's admin.
- `locales/<lang>.json` = storefront copy (used via the `t` filter in Liquid). `locales/<lang>.schema.json`
  = theme-editor-only copy (setting labels/help text referenced as `t:` keys in schemas). Keep both in
  sync per language when adding new translatable strings/settings.

### Events

Events are a metaobject (type handle `events`, defined in the store admin under Settings → Custom data →
Metaobjects — not in theme code) with fields `status` (choice: `Past`/`Upcoming`, merchant-set — not
computed from the date), `title`, `tag_line`, `meta_description`, `event_date`, `content` (rich text),
`gallary` (list of files — **yes, that's a typo in the field key**, kept as-is in the Liquid code to
match; only rename both together if you ever fix it in admin), `ticket_link`, `featured_image` (single
file — the lead image shown in the detail hero and event cards; falls back to the first `gallary` entry
via `| default:` for entries saved before this field existed), `seo_title`, and `seo_description` (these
two back the page's `<title>`/meta description via the metaobject's "Publish as web page" mapping — not
`title`/`meta_description`, which are for on-page display copy). The metaobject has "Web pages" enabled
with URL handle `events`, backing a Page (title "Events",
handle `events`, template `page.events`) so entries live at `/pages/events/<handle>`, rendered by
`templates/metaobject/events.json` → `sections/event-detail.liquid` (+ `sections/masonry-gallery.liquid`
for the gallery). The listing page (`/pages/events`, `templates/page.events.json`) uses
`sections/events-list.liquid` + `snippets/event-card.liquid`, which bucket entries into Upcoming/Past tabs
by the `status` field, sorting within each via a "date|handle" sort-key trick (Liquid's `sort` filter can't
sort by a nested `field.value`) resolved back to the full object with `shop.metaobjects.events[handle]`.
`sections/masonry-gallery.liquid` reads `metaobject.gallary` first, falling back to the page metafield
`custom.gallery_images` so the older gallery page keeps working.

Two non-obvious Liquid quirks discovered building this (verified against this theme's actual metaobject
rendering, not just docs — re-check if Shopify changes this):
- A metaobject drop's own handle/id/url live under `.system` (`event.system.handle`, `event.system.url`),
  not directly on the object — presumably so a custom field can be named `handle` without colliding.
- A rich text metaobject field (`content`) does **not** auto-render as HTML like a rich-text *metafield*
  does. Printing it directly (with or without `.value`) dumps its raw JSON structure — pipe it through
  `| metafield_tag` to get HTML (see `sections/event-detail.liquid`).
