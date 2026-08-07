# Upgrading to maquina-components 0.6

> Verified against maquina-components 0.6.1.

Everything 0.6.0 changed is **behavioral**. No partial, helper, or component was renamed or
removed, so nothing raises — the failures are silent and visual. Read this before upgrading an
app that is on 0.5.x.

```bash
bundle update maquina-components
bin/rails maquina:doctor
```

`maquina:doctor` reads your CSS, views and JavaScript and prints file:line for every pattern
this release changes, grouped BREAKING / REVIEW / CLEANUP. It never edits anything and never
fails a build.

## The doctor's rules

| Rule id | Severity | What it finds |
|---------|----------|---------------|
| `unlayered-universal-rule` | breaking | An unlayered `*` rule — the `theme.css` border shim (§1) |
| `data-active-presence` | breaking | `[data-active]` presence selectors and `data-[active]:` utilities (§5) |
| `restated-svg-uri` | breaking | An app-restated control mark SVG that should read a `*-image` token |
| `hardcoded-radius` | review | A literal radius where a role token now applies (§3) |
| `hardcoded-shadow` | review | A literal shadow where an elevation token now applies (§3) |
| `dark-twin-rule` | review | A `.dark`-duplicated rule that a token value would cover |
| `unlayered-component-rule` | cleanup | App CSS targeting a component from outside a layer |
| `inline-shape-utility` | cleanup | A shape utility inline in a view that belongs in a token |

---

## 1. The theme.css shim now flattens alert and toast borders

**This affects every existing app.** The `theme.css` shipped by earlier installers ends with an
unlayered universal rule. Engine rules now live in `@layer components`, and unlayered CSS
outranks every layer at any specificity — so that one rule wins over the tinted borders on all
alert and toast variants. A destructive alert's border measures plain `--border`
(`oklch(0.928 0.006 264)`) where 0.5.1 painted `oklch(0.92 0.05 25)`.

The generator template is fixed, but the rule lives in *your* file. Wrap it:

```css
/* 0.5.1 — as installed */
* { border-color: var(--color-border); }

/* 0.6.0 — one line of nesting */
@layer base {
  * { border-color: var(--color-border); }
}
```

The same applies to any other unlayered `*` rule you have added.

## 2. Utilities passed through `css_classes:` now win

Every engine rule is flattened to specificity 0,1,0 and layered, so a Tailwind utility passed as
`css_classes:` finally takes effect. It used to be silently swallowed — which means utilities
you already pass may start applying.

| Site | 0.5.1 | 0.6.0 |
|------|-------|-------|
| Input with a width utility | 448px | 137px |
| Form actions with a hidden utility | `display: flex` | `display: none` |
| Form with a flex utility | `display: grid` | `display: flex` |

**Search your views for `css_classes:` before upgrading.** Anything passed as decoration and
never seen is now live; delete what you did not mean.

## 3. Radius and elevation normalize onto role tokens

Eight radius sites move:

| Component / part | 0.5.1 | 0.6.0 |
|------------------|-------|-------|
| `[data-component="card"]` | 12px | 8px |
| `[data-sidebar-part="inset"]` (variant inset) | 12px | 8px |
| `[data-sidebar-part="inset"] [data-component="header"]` top corners | 12px | 8px |
| `[data-combobox-part="content"]` popover | 6px | 8px |
| `[data-dropdown-menu-part="content"]` popover | 6px | 8px |
| `[data-combobox-part="option"]` | 4px | 6px |
| `[data-dropdown-menu-part="item"]` | 4px | 6px |
| `[data-toast-part="close"]` | 4px | 6px |

Four elevation sites collapse `shadow-lg` → `--elevation-overlay` (= `shadow-md`): the toast,
the toast on hover, the drawer panel and the date-picker popover.

Each site keeps a component-level **escape hatch**, so any one can be pinned without redefining
a role token. See the token layer in [installation-guide.md](installation-guide.md).

## 4. Focus rings are outlines, and buttons finally have them

```css
/* 0.5.1 — a box-shadow ring, on :focus as well as :focus-visible */
[data-component="input"]:focus,
[data-component="input"]:focus-visible {
  box-shadow: 0 0 0 2px var(--background), 0 0 0 4px var(--ring);
}

/* 0.6.0 — an outline, keyboard focus only, from tokens */
[data-component="input"]:focus-visible {
  outline: var(--focus-ring-width) var(--focus-ring-style) var(--focus-ring-color);
  outline-offset: var(--focus-ring-offset);
}
```

- **Form fields ring on keyboard focus only.** The bare `:focus` half of each pair is gone, so
  a mouse click no longer rings. This is intentional — review checklists should expect it.
- **Rings are `outline` + `outline-offset`, uniformly 3px at offset 0.** Sites that faked a
  backdrop band with `0 0 0 2px var(--background)` lose the band. An outline cannot be clipped
  by an ancestor's `overflow` and never affects layout, which is why the drawer and sidebar
  could not ring before.
- **Six button variants gain a ring they never had.** `:focus-visible` used to be declared
  before the variant rules at equal specificity, so every variant setting a background
  overwrote it — 2 of 16 buttons on the specimen page actually ringed. If your app restated a
  ring on buttons to work around this, delete it.

Custom components should read the tokens rather than the engine's rule: `--focus-ring-width`,
`--focus-ring-offset`, `--focus-ring-style`, `--focus-ring-color`.

## 5. `merge_component_data` precedence narrows

The component used to win every key it set. Now it wins only its **identity keys**:
`:component`, `:variant`, `:size`, and any key ending in `_part` or `-part`. `:controller` and
`:action` still concatenate — component tokens first, then yours. The caller wins everything
else.

```erb
<%# 0.5.1: the toast's own state won, this did nothing %>
<%# 0.6.0: renders data-state="exiting" %>
<%= render "components/toast", title: "Saved", data: { state: "exiting" } %>
```

The merged hash is `.compact`ed, so a `nil` value emits no attribute where it used to emit an
empty one. `false` still renders `"false"` — that is a value, not an absence.

Related: a sidebar item now **omits** `data-active` when inactive instead of writing
`data-active="false"`. Presence selectors no longer match:

```css
/* before */ [data-sidebar-part="menu-button"][data-active] { }
/* after  */ [data-sidebar-part="menu-button"][data-active="true"] { }
```

```erb
<%# before %> <a class="data-[active]:bg-accent">
<%# after  %> <a class="data-[active=true]:bg-accent">
```

Active items also now carry `aria-current="page"`.

## 6. Surfaces above the page stop painting the page color

An alert, a calendar and the date-picker popover painted `--background` — the page. Anything
floating above the page is a surface, so they now paint `--card` or `--popover`.

If your theme sets those to the same value, nothing moves. That is why it went unnoticed: in
the default light theme all three are white. In the default dark theme they separate.

```
alert, calendar background (dark)   oklch(0.13 0.028 261) → oklch(0.178 0.032 260)
```

The `outline` and `ghost` buttons and the active pagination link now paint `transparent`
instead of `--background`, so they work inside a card — which they previously did not.

To pin the old behavior, point the surface tokens at the page:

```css
:root {
  --popover: var(--background);
  --card: var(--background);
}
```

> Checking surface-against-surface contrast? Use ΔL on the CIE L\* axis, not a WCAG ratio.
> WCAG contrast is a text metric; on two adjacent large surfaces it reads a misleading ~1.1.

## 7. Tinted badges lose a stray hairline

Badge's `success` / `warning` / `destructive` variants have always set
`border-color: transparent`. The unlayered `*` shim from §1 was overriding it with `--border`,
so those badges carried a grey 1px outline they were never meant to have. Once the shim is
layered, the intended transparent border shows through. Nothing to do — but if you compensated
for the hairline elsewhere, remove the compensation.

---

## Also new in 0.6.0

- **`MaquinaComponents.strict_icons`** — defaults to `Rails.env.local?`. An unresolvable icon
  name now raises `MaquinaComponents::UnknownIconError` in development and test instead of
  rendering nothing. Audit `icon_for` calls before upgrading; see the icon roster in
  [installation-guide.md](installation-guide.md).
- **`components/label` partial**, alongside the existing `data-component="label"` form.
- **Six partials promoted from CSS-only hooks**: `drawer/section`, `drawer/separator`,
  `sidebar/group_action`, `sidebar/menu_action`, `sidebar/menu_badge`, `sidebar/separator`.
- **Multi-drawer support** — `drawer/trigger` takes `for_id:`, `drawer/provider` takes `name:`.
- **`Toast.destructive(title, options)`** in the JS API; `variant: "destructive"` normalizes to
  `"error"`.
- **Two fixed helpers**: `dropdown_menu_simple` raised `NoMethodError` on its first item, and
  `combobox_simple` rendered an empty popover. Both work in 0.6.0.
- **`[data-variant="bordered"]` on tables now matches** — it was emitted as an escaped attribute
  string and was unreachable.
- **The install generator is idempotent.** Re-running
  `bin/rails generate maquina_components:install` appends the token block once and never
  rewrites your palette.

## Appendix: keeping the 0.5.1 look

Everything above is a value, so one token block reverts the visual changes. Drop this into
`theme.css` and delete the lines you do not want.

```css
:root {
  /* Radius — the eight sites that moved */
  --card-radius: 0.75rem;
  --inset-radius: 0.75rem;
  --combobox-radius: 0.375rem;
  --dropdown-menu-radius: 0.375rem;
  --combobox-item-radius: 0.25rem;
  --dropdown-menu-item-radius: 0.25rem;
  --toast-close-radius: 0.25rem;

  /* Elevation — the four sites that collapsed shadow-lg → shadow-md */
  --toast-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
  --toast-hover-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
  --drawer-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
  --date-picker-popover-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);

  /* Focus ring — the closest outline equivalent of the old two-step ring */
  --focus-ring-width: 2px;
  --focus-ring-offset: 2px;
}
```

Two things this cannot bring back, because they are not values: the **backdrop band** (the old
ring drew `--background` under `--ring` inside one `box-shadow`; `--focus-ring-offset: 2px`
leaves the same gap, showing whatever is actually behind the control), and the **mouse-click
ring** on form fields plus the **absent ring** on five button variants. Both were
`:focus-visible` bugs, and both are fixed on purpose.
