# Installation Guide

How to install and configure maquina_components in a Rails application.

> Verified against maquina-components 0.7.0.

---

## Prerequisites

- Ruby on Rails 7.2+
- [tailwindcss-rails](https://github.com/rails/tailwindcss-rails) `~> 4.2`, with `app/assets/tailwind/application.css` present
- importmap-rails `>= 1.2` and stimulus-rails `>= 1.3` (pulled in as gem dependencies)

The gem is named with a hyphen (`maquina-components`) even though the module and repository use
the underscore (`MaquinaComponents`, `maquina_components`).

---

## Installation

### 1. Add the Gem

```ruby
# Gemfile
gem "maquina-components"
```

```bash
bundle install
```

### 2. Run the Generator

```bash
bin/rails generate maquina_components:install
```

**Generator Options:**

| Option | Description |
|--------|-------------|
| `--skip-theme` | Skip adding theme CSS variables |
| `--skip-helper` | Skip creating the icon helper file |

The generator is **idempotent** — re-run it after a gem upgrade to pick up new token blocks. It
appends each marker-guarded block once and leaves an existing palette untouched.

### 3. What the Generator Creates

1. **CSS import** — Injects after `@import "tailwindcss"` in `app/assets/tailwind/application.css`:
   ```css
   @import "../builds/tailwind/maquina_components_engine.css";
   ```

2. **Theme variables** — Appends `:root` and `@theme` blocks with OKLCH color variables (unless `--skip-theme`)

3. **Shape and state tokens** — Appends the marker-guarded `shape_state_tokens.css` block with
   the radius, focus, elevation, mark and weight tokens (unless `--skip-theme`)

4. **Icon helper** — Creates `app/helpers/maquina_components_helper.rb` with `main_icon_svg_for` override point (unless `--skip-helper`)

---

## CSS Architecture

maquina_components uses **data-attribute selectors** for all component styling:

```css
/* Component root */
[data-component="card"] { ... }

/* Component parts */
[data-card-part="header"] { ... }
[data-card-part="content"] { ... }

/* Variants */
[data-component="badge"][data-variant="success"] { ... }

/* States */
[data-component="sidebar"][data-state="expanded"] { ... }
```

**Key patterns:**
- `@apply` for spacing and typography utilities
- Explicit CSS for theme variable colors, resolved from variables rather than utility classes
- Data attributes carry structure, variants, and state; classes stay free for layout
- Dark mode switches CSS variable values — author one rule and let the theme resolve it

### The layered engine

As of 0.6.0 every engine rule lives in `@layer components`, flattened to specificity 0,1,0 via
`:where()`. Two consequences shape how you write app CSS:

**A utility passed as `css_classes:` applies.** It used to be silently swallowed. Audit what
your views already pass — decoration you never saw is now live.

**Unlayered CSS outranks every layer, at any specificity.** One unlayered `*` rule beats every
component rule in the engine. The generator-installed border shim must therefore be layered:

```css
/* Wrong — flattens the tinted borders on every alert and toast variant */
* { border-color: var(--color-border); }

/* Right */
@layer base {
  * { border-color: var(--color-border); }
}
```

App CSS that intentionally overrides a component belongs in a layer declared after
`components`, or unlayered if you mean it to win unconditionally. `bin/rails maquina:doctor`
reports unlayered rules that look accidental.

### Presence selectors on state attributes

A component omits a state attribute rather than writing a falsy value, so match on the value:

```css
/* Matches nothing since 0.6.0 */ [data-sidebar-part="menu-button"][data-active] { }
/* Correct */                     [data-sidebar-part="menu-button"][data-active="true"] { }
```

The Tailwind form is `data-[active=true]:` rather than `data-[active]:`.

---

## Theme System

The theme follows [shadcn/ui](https://ui.shadcn.com/) conventions using OKLCH color space.

### Color Variables

Variables are defined in two places for Tailwind CSS 4 compatibility:

1. **`:root` block** — CSS custom properties for runtime use
2. **`@theme` block** — Tailwind theme bindings for utility class generation

**Core Variables:**

| Variable | Purpose | Tailwind Class |
|----------|---------|---------------|
| `--background` / `--foreground` | Page background and text | `bg-background`, `text-foreground` |
| `--card` / `--card-foreground` | Card surfaces | `bg-card`, `text-card-foreground` |
| `--popover` / `--popover-foreground` | Popover surfaces | `bg-popover` |
| `--primary` / `--primary-foreground` | Brand/CTA | `bg-primary`, `text-primary` |
| `--secondary` / `--secondary-foreground` | Secondary elements | `bg-secondary` |
| `--muted` / `--muted-foreground` | Subdued content | `bg-muted`, `text-muted-foreground` |
| `--accent` / `--accent-foreground` | Highlights | `bg-accent` |
| `--destructive` / `--destructive-foreground` | Dangerous actions | `bg-destructive` |
| `--success` / `--success-foreground` | Positive feedback | `bg-success` |
| `--warning` / `--warning-foreground` | Caution | `bg-warning` |
| `--border` | Default border color | `border-border` |
| `--input` | Input border color | `border-input` |
| `--ring` | Focus ring color | `ring-ring` |
| `--chart-1` through `--chart-5` | Chart colors | `bg-chart-1` |

**Layout Variables:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `--header-height` | `3.5rem` | Top header height |
| `--sidebar-width` | `16rem` | Sidebar expanded width |
| `--sidebar-width-icon` | `3rem` | Sidebar icon-only width |

**Sidebar Variables:** `--sidebar`, `--sidebar-foreground`, `--sidebar-primary`, `--sidebar-primary-foreground`, `--sidebar-accent`, `--sidebar-accent-foreground`, `--sidebar-border`, `--sidebar-ring`

### Shape, Focus and Elevation Tokens

As of 0.6.0 shape, focus rings, elevation and weight are variables too — which is the whole of
what used to require override CSS. **A theme changes values, not selectors.**

**Role tokens** restyle a whole class of component at once:

| Token | Default | Applies to |
|-------|---------|------------|
| `--control-radius` | `0.375rem` | Buttons, inputs, selects, textareas, badges, menu items, pagination links, calendar days, sidebar items |
| `--surface-radius` | `0.5rem` | Cards, alerts, popovers, toasts, tables, stats, empty, calendar, drawer, sidebar inset |
| `--mark-radius` | `4px` | The checkbox box |
| `--pill-radius` | `calc(infinity * 1px)` | Radio, switch track |
| `--focus-ring-width` | `3px` | Every focus ring |
| `--focus-ring-offset` | `0px` | Every focus ring |
| `--focus-ring-style` | `solid` | Every focus ring |
| `--focus-ring-color` | *per family, see below* | Every focus ring; invalid fields and destructive buttons override with the destructive tint |
| `--elevation-control` | `shadow-xs` | Inputs, selects, textareas, checkbox, radio |
| `--elevation-raised` | `shadow-sm` | Cards, stats cards, floating sidebar, every filled button |
| `--elevation-overlay` | `shadow-md` | Dropdown and combobox popovers, date-picker popover, toasts, drawer panel |
| `--elevation-none` | `none` | Ghost and link buttons, the inset sidebar |
| `--label-weight` | `500` | Labels, buttons |
| `--value-weight` | `700` | Stat values |
| `--control-fill` | `transparent` | Field background; re-set under `.dark` |


`--focus-ring-color` is the one token with no single default. It is declared nowhere; each rule
supplies its own fallback, because the right resting colour differs by family:

| Family | Default when the token is unset |
|---|---|
| Buttons, cards, badges, toasts, drawer, pagination, calendar, toggle group, date picker | `var(--ring)` |
| Everything inside the sidebar, and the menu button | `var(--sidebar-ring, var(--ring))` |
| Form fields — input, textarea, select, checkbox, radio | `color-mix(in oklch, var(--ring) 50%, transparent)` |

Setting it once at `:root` overrides all three, since a declared token means no fallback fires.
Two states outrank a `:root` override on purpose — an `aria-invalid` field and a
`data-variant="destructive"` button declare it on the element itself, and an element's own custom
property beats an inherited one.

**Never transition `outline-color`** in a component of your own, and that means never using
Tailwind's `transition-colors`, which folds `outline-color` in as of v4. A transitioned ring
animates from its pre-focus value — the initial `currentColor`, i.e. the control's own text colour —
so on a filled variant it is a near-white ring for the first 150ms, which is no focus indicator at
all. It also makes a `getComputedStyle` read taken right after a `Tab` press report the previous
colour. Name the properties instead:
`transition-property: color, background-color, border-color, text-decoration-color`.

**Mark tokens** carry the control glyphs: `--checkbox-mark-image`,
`--checkbox-indeterminate-image`, `--radio-mark-image`, `--switch-thumb-image`,
`--select-chevron-image`. Point one at your own SVG data URI rather than restating the rule.

**Escape hatches** pin exactly one component without redefining a role — `--card-radius`,
`--alert-radius`, `--badge-radius`, `--button-radius`, `--input-radius`, `--inset-radius`,
`--combobox-radius`, `--combobox-item-radius`, `--dropdown-menu-radius`,
`--dropdown-menu-item-radius`, `--toast-radius`, `--toast-close-radius`, `--table-radius`,
`--sidebar-radius`, `--sidebar-item-radius`, `--pagination-radius`, `--stats-radius`, and the
matching `*-shadow` set (`--card-shadow`, `--toast-shadow`, `--toast-hover-shadow`,
`--drawer-shadow`, `--date-picker-popover-shadow`, `--combobox-shadow`,
`--dropdown-menu-shadow`, `--stats-shadow`).

A flat theme is six lines:

```css
:root {
  --elevation-control: none;
  --elevation-raised: none;
  --elevation-overlay: none;
  --elevation-none: none;
  --control-radius: 0.25rem;
  --surface-radius: 0.25rem;
}
```

Declare these in a plain, unlayered `:root` block — that is what the installer generates, and
unlayered CSS wins over the engine's `@theme` defaults whatever the import order. Wrapping them
in `@theme` emits into `@layer theme` alongside the engine's own defaults, where source order
becomes the only tie-breaker. Keep the names as they are: renaming them into Tailwind's
`--radius-*` / `--shadow-*` namespaces means an app-side `@theme { --radius-*: initial }` wipes
them.

### Dark Mode

Dark mode is handled via CSS variables — switching the `.dark` class on `<html>` changes all variables automatically. Author one rule and let the theme resolve it, rather than pairing a light rule with a `dark:` twin.

```css
:root {
  --background: oklch(1 0 0);        /* white */
  --foreground: oklch(0.145 0 0);    /* near-black */
}

.dark {
  --background: oklch(0.145 0 0);    /* near-black */
  --foreground: oklch(0.985 0 0);    /* near-white */
}
```

### Customizing Colors

Override in your `app/assets/tailwind/application.css`:

```css
:root {
  --primary: oklch(0.467 0.175 3.95);
  --primary-foreground: oklch(0.985 0 0);
}
```

---

## Stimulus Controllers

All Stimulus controllers are **auto-registered** via importmap. No manual registration or import needed. The engine handles:

1. Pin declarations in the engine's initializer
2. Controller registration through Stimulus autoload conventions

Controllers become available as `data-controller="sidebar"`, `data-controller="combobox"`, etc.

---

## Icon System

### Default: Built-in Icons

The gem ships 56 built-in SVG icons (Lucide-style). Use `icon_for`:

```erb
<%= icon_for :check, class: "size-4" %>
```

**This list is the single source of truth for icon names.** Under `strict_icons` a name outside
it raises rather than rendering nothing, so check here before using one:

`activity`, `align_center`, `align_left`, `align_right`, `arrow_down`, `arrow_left`,
`arrow_right`, `arrow_up`, `bold`, `briefcase`, `calendar`, `chart_bar`, `check`,
`check_circle`, `chevron_left`, `chevron_right`, `chevron_up_down`, `circle_alert`,
`circle_check`, `circle_x`, `clipboard_list`, `clock`, `credit_card`, `dollar`, `download`,
`ellipsis`, `folder`, `grid`, `home`, `inbox`, `info`, `italic`, `layout_dashboard`,
`left_panel`, `lightning_bolt`, `line_chart`, `list`, `log_out`, `logout`, `mail`,
`message_square`, `money`, `more_horizontal`, `pencil`, `piggy_bank`, `search`,
`select_chevron`, `settings`, `slash`, `trash`, `trend_down`, `trend_up`, `triangle_alert`,
`underline`, `upload`, `user`, `users`, `x`

`:alert_triangle` is accepted as an alias of `:triangle_alert`.

**Common names that are *not* built in** and need an app-side `main_icon_svg_for` entry:
`plus`, `sun`, `moon`, `copy`, `edit`, `panel_left`, `trending_up`, `folder_open`. Several are
Lucide names whose engine spelling differs — `pencil` for edit, `left_panel` for panel_left,
`trend_up` for trending_up.

> `empty_list_state` defaults to `icon: :folder_open`, which is not in the roster. Under
> `strict_icons` that default raises unless your app's `main_icon_svg_for` resolves it — pass an
> explicit `icon:` or define `folder_open` in the override.

### Strict Icons

```ruby
# config/initializers/maquina_components.rb
MaquinaComponents.strict_icons = false  # fall back to rendering nothing
```

`strict_icons` defaults to `Rails.env.local?` — on in development and test, off in production.
When on, `icon_for` raises `MaquinaComponents::UnknownIconError` for a name that resolves to no
SVG, instead of silently rendering nothing. Anything the app's `main_icon_svg_for` resolves
counts, so a custom icon system satisfies it.

### Custom Icon System

Override `main_icon_svg_for` in `app/helpers/maquina_components_helper.rb`:

```ruby
module MaquinaComponentsHelper
  def main_icon_svg_for(name)
    # Return SVG string for your icon system
    # Return nil to fall back to built-in icons

    # Example: Heroicons via heroicon gem
    heroicon(name.to_s)

    # Example: Lucide from node_modules
    File.read(Rails.root.join("node_modules/lucide-static/icons/#{name}.svg")).html_safe

    # Example: app/assets/icons directory
    file = Rails.root.join("app/assets/icons/#{name}.svg")
    file.exist? ? file.read.html_safe : nil
  end
end
```

### Sidebar State Helpers

The generated helper also re-exports sidebar state methods for convenience:

```ruby
def app_sidebar_state(cookie_name = "sidebar_state")
  sidebar_state(cookie_name)
end

def app_sidebar_open?(cookie_name = "sidebar_state")
  sidebar_open?(cookie_name)
end

def app_sidebar_closed?(cookie_name = "sidebar_state")
  sidebar_closed?(cookie_name)
end
```

---

## Verifying an Install: `maquina:doctor`

```bash
bin/rails maquina:doctor
```

Reads your CSS, views and JavaScript and prints file:line for every pattern the current release
changes, grouped BREAKING / REVIEW / CLEANUP. It never edits anything and never fails a build.

| Rule id | Severity | What it finds |
|---------|----------|---------------|
| `unlayered-universal-rule` | breaking | An unlayered `*` rule — typically the `theme.css` border shim |
| `data-active-presence` | breaking | `[data-active]` presence selectors and `data-[active]:` utilities |
| `restated-svg-uri` | breaking | An app-restated control mark SVG that should read a `*-image` token |
| `hardcoded-radius` | review | A literal radius where a role token now applies |
| `hardcoded-shadow` | review | A literal shadow where an elevation token now applies |
| `dark-twin-rule` | review | A `.dark`-duplicated rule that a token value would cover |
| `unlayered-component-rule` | cleanup | App CSS targeting a component from outside a layer |
| `inline-shape-utility` | cleanup | A shape utility inline in a view that belongs in a token |

Every rule id is explained in [upgrading-0.6.md](upgrading-0.6.md).
