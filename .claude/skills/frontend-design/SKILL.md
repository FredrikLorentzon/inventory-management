---
name: frontend-design
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface with a vertical navigation sidebar on the left, a slim top bar, consistent spacing, clean card layouts and a polished professional look, while keeping the project's own colors and design rules. Use this skill whenever the user asks to redesign, restyle, modernize, polish or "make it look like a SaaS app", wants a sidebar instead of a top nav, a dashboard or admin panel look, cleaner cards, better spacing, or a more professional UI for a Vue app, even when they do not say "frontend-design". Also use it for follow-ups on such a redesign, such as a collapsible or icon-only sidebar, a mobile navigation drawer, or tightening the layout of a single view.
---

# Frontend Design: SaaS-Style Vue 3 Redesign

This skill turns a Vue 3 app with a basic layout into a polished SaaS-style interface: a fixed vertical sidebar on the left, a slim top bar, and content laid out in consistent cards on a calm background. The goal is a layout and polish upgrade, not a rebrand. The app should look like the same product after a good designer spent a week on it.

## Principles

- **Restructure the layout, keep the brand.** Reuse the colors, status colors, fonts and rules the project already defines (CLAUDE.md, a design system section, existing CSS variables). Introduce new tokens only for what is missing, such as a spacing scale, radii and shadows, and derive any new color from the existing palette. Users and stakeholders recognize a product by its colors, so changing them makes a redesign feel like a different app.
- **Change presentation, not behavior.** Routes, data fetching, filters, i18n keys, props and emitted events stay as they are. A redesign that breaks a filter or a translation is a regression, however good it looks. If a view's logic is tangled with its markup, restyle around it instead of refactoring it.
- **One system, applied everywhere.** Consistency is what makes an interface look professional. Every page should use the same page header, the same card, the same table and the same spacing steps. Define these once and reuse them rather than styling each view by hand.
- **No new dependencies.** Use plain CSS (custom properties, Grid, Flexbox) and inline SVG icons. This keeps the bundle small, works offline and avoids version churn. Add a UI library only if the user asks for one.
- **Follow the project's own rules.** If the project says to delegate .vue work to a specific subagent, forbids emojis, or prescribes a component structure (Options API with `setup()`, `<script setup>`, scoped styles), follow it.

## Workflow

### 1. Audit before touching anything

Read enough to answer these questions, and keep the answers in your plan:

- Where is the app shell? Usually `App.vue`. Find the current top nav, global styles and any global components such as a filter bar, language switcher or profile menu.
- What are the routes? Read the router setup (often `main.js` or `router/index.js`) and list every route with its nav label. Note views that exist but are not routed, and do not add routes the user did not ask for.
- What design rules exist? Look in CLAUDE.md files, a design system section, and existing `:root` variables. Note the colors, status colors and fonts.
- Which shared pieces repeat across views? Page titles, KPI or stat cards, chart cards, tables, filter bars, badges, modals. These become the shared patterns.
- How does active navigation work today? Keep that behavior, usually `router-link` with an active class.

### 2. Present a plan

List the files you will change, the token set you will add, the shell structure, and the shared patterns. Call out anything you will deliberately leave alone. If the user is in plan mode, this is the plan you submit.

### 3. Add the design tokens

Put tokens on `:root` in the global stylesheet. Start from the project's existing colors and add what is missing:

```css
:root {
  /* existing project colors are kept, then mapped to roles */
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-border: #e2e8f0;
  --color-bg: #f8fafc;
  --color-surface: #ffffff;
  --color-sidebar: #0f172a;

  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-5: 24px;
  --space-6: 32px;
  --space-7: 48px;

  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 14px;

  --shadow-card: 0 1px 2px rgb(15 23 42 / 0.04), 0 1px 3px rgb(15 23 42 / 0.06);
  --shadow-raised: 0 4px 12px rgb(15 23 42 / 0.08);

  --sidebar-width: 240px;
  --topbar-height: 60px;
}
```

Replace hard-coded paddings, margins and gaps with the spacing steps as you restyle. Stray values such as 13px or 22px are the main reason a layout looks almost right but not quite.

### 4. Build the shell

Use a two-column grid: a fixed sidebar and a main column with a sticky top bar and a scrollable content area.

```
+-----------+-------------------------------------+
| Logo      | Top bar: page title, global filters |
|           | language, profile menu              |
| Nav item  +-------------------------------------+
| Nav item  |                                     |
| Nav item  |  Page content                       |
| ...       |  (max width ~1400px, padded)        |
|           |                                     |
| Footer    |                                     |
+-----------+-------------------------------------+
```

Sidebar guidance:

- Brand at the top: logo mark plus product name, the same height as the top bar so their bottom edges line up.
- Nav items: icon plus label, 40px tall, `--radius-sm` corners, `--space-3` horizontal padding. Group items under small uppercase section labels when there are more than about six.
- Active item: a clearly filled background and a stronger text color. Hover: a subtle background. Keep a visible focus ring for keyboard users.
- A dark sidebar (the darkest project color) with a light content area is the classic SaaS look and works well with slate palettes. A light sidebar with a border is the alternative when the brand is light.
- Global controls that sat in the old top nav move to the top bar. Filters that apply to every page belong in the top bar or directly under it, not inside the sidebar.

Small screens: below about 768px the sidebar becomes an off-canvas drawer opened by a menu button in the top bar, with a backdrop that closes it, and it closes on navigation. Below about 1024px you may offer an icon-only collapsed sidebar (labels hidden, tooltips via `title` or a small popover, width about 72px). Store the collapsed preference in `localStorage` inside a try/catch.

### 5. Icons

Write small inline SVG icons, one per nav item, in a single icon component or a map of paths so they stay consistent:

```vue
<svg viewBox="0 0 24 24" width="20" height="20" fill="none"
     stroke="currentColor" stroke-width="1.75"
     stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
  <path d="M3 12l9-9 9 9M5 10v10h14V10" />
</svg>
```

Use one stroke width, one size and `currentColor` throughout so icons inherit the nav item's color. Icons are decorative next to a visible label, so mark them `aria-hidden`. When labels are hidden in collapsed mode, give the link an `aria-label`.

### 6. Shared content patterns

Define these once in the global stylesheet (or as small components if the project already uses them) and apply them to every view:

- **Page header:** title, optional one-line description, actions aligned right. `--space-5` below it.
- **Card:** `--color-surface`, 1px `--color-border`, `--radius-md`, `--shadow-card`, `--space-5` padding. Card header with title and optional actions, separated by spacing rather than a heavy rule.
- **Stat or KPI card:** small muted uppercase label, large value, a secondary line for goal or trend, status colors only for meaning (good, warning, bad), never for decoration.
- **Grid:** CSS Grid with `repeat(auto-fit, minmax(240px, 1fr))` for stat rows and a 12-column or two-column grid for charts. Gaps are `--space-5`.
- **Table:** full width inside a card, sticky header, muted uppercase header text, 44 to 52px rows, row hover, numbers right aligned with `font-variant-numeric: tabular-nums`. Wrap in a horizontally scrollable container on small screens.
- **Badges:** pill shape, soft tinted background with darker text in the matching status color.
- **Forms and filters:** consistent control height (36 to 40px), `--radius-sm`, visible focus ring, labels above or inline but the same everywhere.
- **Modals:** reuse the card surface, `--radius-lg`, `--shadow-raised`, a dimmed backdrop.

### 7. Verify

- Build or run the dev server and fix any console errors.
- Visit every route in the browser. Check that navigation, the active state, filters, language switching, modals and charts all still work.
- Check widths around 1440px, 1024px and 375px. There should be no horizontal page scroll, and the drawer or collapsed sidebar should work.
- Search the changed files for leftover hard-coded spacing and colors that should be tokens.
- Report what changed, what was deliberately left alone, and anything that looked broken before the redesign so it is not mistaken for a regression.

## Common mistakes

- Restyling the shell but leaving each view with its own padding and card styles, so pages feel inconsistent.
- Moving global filters into the sidebar, where they crowd navigation and are easy to miss.
- Replacing the project's colors with a generic indigo or purple accent.
- Forgetting the mobile drawer, so the sidebar eats half of a phone screen.
- Breaking `router-link` active classes by wrapping or renaming links.
- Touching data logic, computed properties or API calls while restyling.
