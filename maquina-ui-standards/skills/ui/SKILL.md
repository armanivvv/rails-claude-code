---
name: ui
description: Build consistent, accessible UIs in Rails using maquina_components. Use this skill whenever implementing UI for features, creating views, building forms, designing layouts, or reviewing UI specs. Triggers on view creation, UI implementation, form building, layout design, or mentions of maquina_components.
---

# Maquina UI Standards

Build production-quality Rails UIs with maquina_components — ERB partials styled with Tailwind CSS 4 and data attributes, inspired by shadcn/ui.

> Verified against maquina-components 0.7.0. Upgrading an app from 0.5.x? Read
> [upgrading-0.6.md](../../references/upgrading-0.6.md) first — 0.6.0 changed behavior silently.
> 0.7.0 broke nothing, but it deprecated `collapse_after` on `responsive_breadcrumbs` and fixed
> focus-ring timing; never transition `outline-color` in a component of your own.

**Official documentation:** https://maquina.app/documentation/components/

## Core Rules

1. **Composition first** — build screens from component partials and helpers; wrap repeated compositions into app-specific partials that encode your conventions.
2. **Data-attribute styling** — components style through `data-component` / `data-*-part` attributes; the engine CSS handles appearance. Reach for a variant or a role token first, and use Tailwind utilities for layout — grids, spacing, placement.
3. **The layered engine** — every engine rule lives in `@layer components` flattened to specificity 0,1,0, so a utility passed as `css_classes:` **applies**. That makes rule 2 a convention you keep, not a mechanism that keeps itself: an unintended utility silently reshapes the component.
4. **Semantic variants** — map meaning to variants (`:success`, `:warning`, `:destructive`), one mapping per status domain. The danger variant is `:destructive` everywhere; sizes are `:sm` / `:default` / `:lg` (badge's middle size is `:md`; both accept the other's name as an alias). This vocabulary is canonical — the catalog defers to it.
5. **Helpers for interactive components** — `dropdown_menu`, `combobox`, `toggle_group`, `simple_table`, `pagination_nav` (Pagy), `empty_state`, `toast_flash_messages`, `breadcrumbs`. Use `_simple` variants for data-driven one-liners, block builders for custom content, partials for structural components (Card, Alert, Badge, Sidebar, Drawer).
6. **Inline errors** — field errors render next to their input (`data-form-part="error"`), with a brief flash summary. Complete inputs: every field carries `type`, `required`, `maxlength`, `autocomplete`, and `inputmode` where they apply.
7. **Handle the zero state** — every list renders an `empty_state` (or `empty_search_state` / `empty_list_state`) when the collection is empty.
8. **Icons via `icon_for`** — one icon system, delegating to the app's `main_icon_svg_for` override with built-in SVG fallbacks. Under `strict_icons` (on by default in development and test) an unresolvable name raises `MaquinaComponents::UnknownIconError`, so use a name from the roster in [installation-guide.md](../../references/installation-guide.md).
9. **Tokens carry the design** — color, shape, elevation, focus and weight are all CSS variables. Use **role tokens** (`--surface-radius`, `--elevation-overlay`, `--focus-ring-width`) to restyle a whole class of component, and a per-component **escape hatch** (`--card-radius`, `--toast-shadow`) to pin exactly one. A theme changes values, not selectors. One accent color moment per screen; neutrals elsewhere.

## Component Selection

| Need | Component | Helper |
|------|-----------|--------|
| Container with header/content/footer | Card | — |
| Important message | Alert | — |
| Status indicator | Badge | — |
| Data display | Table | `simple_table` |
| Zero-data state | Empty | `empty_state` |
| Actions menu | Dropdown Menu | `dropdown_menu` / `dropdown_menu_simple` |
| Option selection | Toggle Group | `toggle_group_simple` |
| Page location | Breadcrumbs | `breadcrumbs` / `responsive_breadcrumbs` |
| Paginated collections | Pagination | `pagination_nav` (Pagy) |
| App navigation | Sidebar | — |
| Slide-out panel | Drawer | `drawer_state` / `drawer_open?` |
| Form inputs and labels | Form components (data attributes), Label | — |
| Inline date selection | Calendar | — |
| Date input field | Date Picker | — |
| Searchable selection | Combobox | `combobox` / `combobox_simple` |
| Temporary feedback | Toast | `toast_flash_messages`, `toast_success` … |
| Dashboard metrics | Stats | — |
| Content divider | Separator | — |

## Universal Component API

Every partial accepts `css_classes:` (additional classes — these apply, see rule 3) and `**html_options` (id, aria, data, title — any HTML attribute).

- **Container partials** (card, table, drawer, …) take a block. **Leaf partials** (titles, descriptions, cells, …) take `text:` for strings or `content:` for captured HTML, with a block as fallback.
- **Your `data:` merges with the component's.** The component wins only its **identity keys** — `component`, `variant`, `size`, and any key ending in `_part`/`-part`. `controller` and `action` **concatenate** (component tokens first), so `data: { controller: "analytics" }` on a combobox renders `data-controller="combobox analytics"`. You win every other key. The merged hash is compacted: `nil` emits no attribute, `false` still renders `"false"`.
- Components generate **deterministic ids** (derived from name/side/title), safe under Turbo morphs.

```erb
<%= render "components/card", id: "profile", data: { controller: "collapsible" } do %>
  <%= render "components/card/header" do %>
    <%= render "components/card/title", text: @user.name %>
  <% end %>
<% end %>

<%= f.email_field :email, data: { component: "input" },
      required: true, maxlength: 254, autocomplete: "email" %>
<%= f.submit "Save", data: { component: "button", variant: "primary" } %>
```

## Quality Bar

Before marking a screen complete: every list renders its empty state, one accent moment per screen with neutrals elsewhere, and `bin/rails maquina:doctor` reports nothing at BREAKING severity. [spec-checklist.md](../../references/spec-checklist.md) carries the full pass.

## Workflow

1. **Map the spec to components** using the selection table above.
2. **Plan layout structure** (grid, stacking, breakpoints) before writing components — read [layout-patterns.md](../../references/layout-patterns.md) when the page has more than one region.
3. **Build**, reaching for references as needed (below).
4. **Verify** against [spec-checklist.md](../../references/spec-checklist.md), then run `bin/rails maquina:doctor`.

## References — read when the task touches them

| Read | When |
|------|------|
| [component-catalog.md](../../references/component-catalog.md) | Rendering any component — props, variants, composition examples |
| [helpers-reference.md](../../references/helpers-reference.md) | Using builder helpers or `_simple` methods |
| [form-patterns.md](../../references/form-patterns.md) | Building or reviewing forms |
| [layout-patterns.md](../../references/layout-patterns.md) | Page structure, grids, responsive design |
| [turbo-integration.md](../../references/turbo-integration.md) | Frames, Streams, or morph interacting with components |
| [stimulus-controllers.md](../../references/stimulus-controllers.md) | Extending or debugging component JavaScript |
| [installation-guide.md](../../references/installation-guide.md) | Setup, theme tokens, icon roster, `maquina:doctor` |
| [upgrading-0.6.md](../../references/upgrading-0.6.md) | Moving an app from 0.5.x, or a component looks subtly wrong after an upgrade |
| [spec-checklist.md](../../references/spec-checklist.md) | Final verification before completion |
