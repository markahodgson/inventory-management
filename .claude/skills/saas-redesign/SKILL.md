---
name: saas-redesign
description: Redesign this inventory app's Vue 3 UI into a modern SaaS layout with a left vertical sidebar (replacing the top nav), design tokens, consistent spacing and a polished professional look. Use when asked to redesign the UI, add sidebar navigation, modernize the layout, or apply/maintain the SaaS look in client/src.
---

# SaaS Redesign (Factory Inventory Management)

Converts `client/src` from a top-nav layout to a left-sidebar SaaS shell, built on CSS custom properties so every view and component shares one spacing, color, radius and shadow system. The skill is idempotent: if the sidebar shell already exists, skip to the token migration and audit steps.

## Ground rules

- **Delegate every `.vue` edit to the `vue-expert` agent** (CLAUDE.md rule). Plain `.js` files (locales) can be edited directly.
- Behavior must not change: same routes, same filters, same API calls, same modals and emitted events. This is a visual/structural change only.
- No new npm dependencies. Icons are inline SVG (24x24 viewBox, `stroke="currentColor"`, `stroke-width="1.75"`, `fill="none"`, rendered at 18px).
- No emojis anywhere in the UI.
- Unique `v-for` keys (`item.path`, `sku`, `id`), never `index`.
- Comment non-obvious logic, e.g. the watcher that closes the mobile drawer on route change.
- Don't touch chart *data* logic in views; only colors and spacing.

## Step 1: Design tokens (`client/src/App.vue`, global `<style>`)

Add a `:root` block at the top of the unscoped style:

```css
:root {
  /* Neutrals (slate) */
  --color-bg: #f8fafc;
  --color-surface: #ffffff;
  --color-surface-muted: #f1f5f9;
  --color-border: #e2e8f0;
  --color-border-strong: #cbd5e1;
  --color-text: #0f172a;
  --color-text-secondary: #475569;
  --color-text-muted: #64748b;
  --color-text-subtle: #94a3b8;

  /* Brand */
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;
  --color-primary-soft: #eff6ff;
  --color-primary-border: #bfdbfe;

  /* Status: fg / soft bg / strong text */
  --color-success: #10b981;  --color-success-soft: #d1fae5;  --color-success-strong: #065f46;
  --color-info: #3b82f6;     --color-info-soft: #dbeafe;     --color-info-strong: #1e40af;
  --color-warning: #f59e0b;  --color-warning-soft: #fef3c7;  --color-warning-strong: #92400e;
  --color-danger: #ef4444;   --color-danger-soft: #fee2e2;   --color-danger-strong: #991b1b;

  /* Sidebar */
  --sidebar-bg: #0f172a;
  --sidebar-text: #94a3b8;
  --sidebar-text-active: #ffffff;
  --sidebar-item-hover: rgba(148, 163, 184, 0.12);
  --sidebar-item-active: rgba(59, 130, 246, 0.18);

  /* Spacing (4px base) */
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px; --space-4: 16px;
  --space-5: 24px; --space-6: 32px; --space-7: 48px;

  /* Radius, shadow, type */
  --radius-sm: 6px; --radius-md: 8px; --radius-lg: 12px; --radius-full: 999px;
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.05);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);
  --shadow-lg: 0 12px 32px rgba(15, 23, 42, 0.12);
  --font-xs: 0.75rem; --font-sm: 0.8125rem; --font-base: 0.875rem;
  --font-md: 1rem; --font-lg: 1.25rem; --font-xl: 1.5rem;

  /* Layout */
  --sidebar-width: 248px;
  --sidebar-collapsed: 72px;
  --topbar-height: 64px;
  --content-max: 1440px;
  --transition: 150ms ease;
}
```

## Step 2: Layout shell (`client/src/App.vue`)

Target template structure:

```
.app-shell  (CSS Grid: var(--sidebar-width) 1fr; min-height 100vh)
├── aside.sidebar            (sticky, height 100vh, dark --sidebar-bg)
│   ├── .sidebar-brand       company name + subtitle (t('nav.companyName'), t('nav.subtitle'))
│   ├── nav.sidebar-nav      v-for navItems -> router-link.sidebar-link (icon + label)
│   └── .sidebar-footer      <LanguageSwitcher /> + <ProfileMenu ... />
├── .sidebar-backdrop        (mobile only, v-if drawer open, click closes)
└── .app-main
    ├── header.topbar        sticky top 0, height --topbar-height, white, bottom border
    │   ├── button.menu-toggle (mobile only, aria-label t('nav.toggleNavigation'))
    │   ├── h1.topbar-title  current nav item's label
    │   └── <FilterBar />    inline, right-aligned
    └── main.main-content    max-width --content-max, padding --space-6, <router-view />
```

Keep `ProfileDetailsModal`, `TasksModal` and all task handlers (`addTask`, `deleteTask`, `toggleTask`, `loadTasks`) exactly as they are.

**Nav items** come from an array in `setup()`, not repeated `router-link`s:

```js
const navItems = [
  { path: '/', labelKey: 'nav.overview', icon: 'grid' },
  { path: '/inventory', labelKey: 'nav.inventory', icon: 'box' },
  { path: '/orders', labelKey: 'nav.orders', icon: 'cart' },
  { path: '/spending', labelKey: 'nav.finance', icon: 'dollar' },
  { path: '/demand', labelKey: 'nav.demandForecast', icon: 'trending' },
  { path: '/reports', labelKey: 'nav.reports', icon: 'file' }
]
```

Render icons via a small `icons` map of SVG path strings (`v-html` is not needed; use `<path :d="...">` elements, or several paths per icon as an array). Active state uses `$route.path === item.path` as it does today. `topbar-title` is a computed that finds the active item.

**Sidebar link styling:** `display:flex; gap: var(--space-3); padding: var(--space-2) var(--space-3); border-radius: var(--radius-md); color: var(--sidebar-text); font-size: var(--font-base); font-weight: 500`. Hover uses `--sidebar-item-hover`. Active uses `--sidebar-item-active`, white text, and a 3px `--color-primary` bar on the left via `::before`.

**Collapsible sidebar (icons-only mode):**
- `const sidebarCollapsed = ref(...)` toggled by a `button.sidebar-collapse` at the bottom of the sidebar, above the footer. The button shows a chevron icon that rotates 180deg when collapsed, and its `aria-label`/`title` switch between `t('nav.collapseSidebar')` and `t('nav.expandSidebar')`. Set `aria-expanded` on it.
- Persist the choice in `localStorage` under the key `sidebarCollapsed`. Wrap the reads and writes in try/catch so blocked storage can't break the app.
- When collapsed, apply `.app-shell.is-collapsed`: the grid column becomes `var(--sidebar-collapsed)`, `transition: grid-template-columns var(--transition)`. Labels, the brand subtitle and the language switcher text are hidden, the brand shows a monogram (first letter of the company name), and the icons are centered.
- Every `.sidebar-link` gets `:title="t(item.labelKey)"` so the label still shows as a tooltip in icon mode.
- The icons-only styles live under one shared selector list, `.app-shell.is-collapsed, ...` plus the 1024px media query below, so the manual and automatic modes look identical.

**Responsive:**
- `max-width: 1024px`: icons-only mode applies automatically, whatever the stored preference. Hide the collapse button here, because there is nothing to expand into.
- `max-width: 768px`: the grid becomes one column. The sidebar is `position: fixed`, `transform: translateX(-100%)`, and switches to `translateX(0)` when `sidebarOpen` is true; show the backdrop and the menu toggle. A `watch(() => route.path, ...)` closes the drawer, so `useRoute()` is needed in `setup()`.
- The topbar wraps FilterBar onto a second row when space is tight (`flex-wrap: wrap`). Use `min-height` instead of a fixed height so wrapping doesn't overflow.

## Step 3: FilterBar (`client/src/components/FilterBar.vue`)

- Remove its own sticky positioning (`position: sticky; top: 70px`), its full-width background, the bottom border and the `max-width` container. It now lives inside the sticky topbar.
- Style the selects as compact pills: height 34px, `border: 1px solid var(--color-border)`, `border-radius: var(--radius-md)`, `padding: 0 var(--space-3)`, `font-size: var(--font-sm)`, and a focus ring `box-shadow: 0 0 0 3px var(--color-primary-soft)` with `border-color: var(--color-primary)`.
- Keep all `v-model`s, options and the reset logic unchanged.

## Step 4: Global component styles (`client/src/App.vue`)

Restyle these with tokens only, without renaming the classes (views depend on them):

| Class | Spec |
|---|---|
| `.page-header` | margin-bottom `--space-5`; h2 `--font-xl`, 700, `--color-text`; p `--font-base`, `--color-text-muted` |
| `.stats-grid` | gap `--space-4`, margin-bottom `--space-5` |
| `.stat-card` / `.card` | `--color-surface`, 1px `--color-border`, `--radius-lg`, `--shadow-sm`, padding `--space-5`; hover on stat-card raises to `--shadow-md` |
| `.card-header` | flex space-between, padding-bottom `--space-4`, margin-bottom `--space-4`, bottom border |
| `.card-title` | `--font-md`, 600 |
| `.table-container` table | th: `--font-xs`, uppercase, letter-spacing .04em, `--color-text-muted`, `--color-surface-muted` bg, sticky top 0; td: padding `--space-3` `--space-4`; row hover `--color-surface-muted` |
| `.badge.*` | `--radius-full`, `--font-xs`, 600, padding 2px `--space-2`; success/warning/danger/info use `-soft` bg + `-strong` text; keep all existing modifiers (increasing, decreasing, stable, high, medium, low) mapped to status tokens |
| `.loading` / `.error` | centered padding `--space-7`; error uses danger-soft/strong with `--radius-md` |

Body background `--color-bg`, text `--color-text`, font stack unchanged.

## Step 5: Token migration in views and components

Replace hard-coded colors in `client/src/views/*.vue` and `client/src/components/*.vue` `<style>` blocks (and inline `fill`/`stroke` in SVG templates where they're static) using this map:

| Hex | Token |
|---|---|
| `#0f172a` | `--color-text` |
| `#1e293b`, `#334155` | `--color-text` (or `--color-text-secondary` for body copy) |
| `#475569` | `--color-text-secondary` |
| `#64748b` | `--color-text-muted` |
| `#94a3b8` | `--color-text-subtle` |
| `#cbd5e1` | `--color-border-strong` |
| `#e2e8f0` | `--color-border` |
| `#f1f5f9` | `--color-surface-muted` |
| `#f8fafc` | `--color-bg` |
| `#ffffff`, `white` (as a surface) | `--color-surface` |
| `#2563eb` | `--color-primary` |
| `#1d4ed8` | `--color-primary-hover` |
| `#eff6ff` | `--color-primary-soft` |
| `#3b82f6` | `--color-info` |
| `#dbeafe` | `--color-info-soft` |
| `#1e40af` | `--color-info-strong` |
| `#10b981`, `#059669` | `--color-success` |
| `#d1fae5`, `#dcfce7`, `#f0fdf4` | `--color-success-soft` |
| `#065f46` | `--color-success-strong` |
| `#f59e0b`, `#d97706` | `--color-warning` |
| `#fef3c7`, `#fffbeb`, `#fed7aa` | `--color-warning-soft` |
| `#92400e` | `--color-warning-strong` |
| `#ef4444`, `#dc2626` | `--color-danger` |
| `#fee2e2`, `#fef2f2`, `#fecaca` | `--color-danger-soft` |
| `#991b1b` | `--color-danger-strong` |

Leave categorical chart series colors (purples such as `#8b5cf6` and `#667eea`, orange `#ea580c`) and gradients as they are. Note them in the summary rather than inventing tokens.

Also normalize spacing while in each file: padding, margin and gap values snap to the nearest `--space-*`. Border-radius values snap to `--radius-*`. Modal shadows use `--shadow-lg`.

Work file by file, largest first (`Dashboard.vue`, `Spending.vue`, `TasksModal.vue`, `Reports.vue`, `InventoryDetailModal.vue`...). Split the work across several vue-expert calls if needed.

## Step 6: Docked menus (`ProfileMenu.vue`, `LanguageSwitcher.vue`)

Both now sit at the sidebar bottom on a dark background:
- The trigger buttons use `--sidebar-text` and the hover background `--sidebar-item-hover`, and span the full sidebar width (icon-only in the collapsed rail).
- Dropdowns open **upward** (`bottom: calc(100% + var(--space-2))`, `left: 0`) with a `--color-surface` panel, `--shadow-lg` and `--radius-lg`, and stay above the sidebar (z-index higher than the sidebar).

## Step 7: i18n (`client/src/locales/en.js`, `ja.js`)

Add these to the `nav` block in both files:
- `reports`: `'Reports'` / `'レポート'`
- `toggleNavigation`: `'Toggle navigation'` / `'ナビゲーション切替'`
- `collapseSidebar`: `'Collapse sidebar'` / `'サイドバーを折りたたむ'`
- `expandSidebar`: `'Expand sidebar'` / `'サイドバーを展開'`

Replace the hard-coded "Reports" label with `t('nav.reports')`.

## Step 8: Verify

1. `cd client && npm run build` passes.
2. With the dev servers running (`/start`), check http://localhost:3000 in a browser (Playwright MCP if connected, otherwise Claude in Chrome):
   - All 6 routes load from the sidebar, and the active state follows the route.
   - Changing the filters changes the data on Overview and Orders.
   - The profile menu, tasks modal, language switch (sidebar labels update, including Reports) and detail modals all work.
   - On desktop the collapse button switches between the full sidebar and the icon rail, tooltips show the labels, and the choice survives a reload.
   - At 1000px width the sidebar is an icon rail. At 600px width the hamburger opens the drawer, the drawer closes on navigation and the backdrop click works, and there is no horizontal scroll.
3. `grep -rnoiE "#[0-9a-f]{3,6}\b" client/src/views client/src/components` lists only the intentional chart/gradient colors.
4. Run the `code-reviewer` agent on the diff.
