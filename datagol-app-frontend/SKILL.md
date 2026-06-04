---
name: datagol-app-frontend
description: Frontend (Vite + React + TypeScript) rules for a single-port DataGOL app — file structure, routing (one route per entity), responsive + mandatory light/dark theming, tables, and the hard rule that the client calls ONLY same-origin /api/* (never DataGOL directly, never holds the service token). Child of datagol-app-development. Read this whenever writing anything under client/.
---

# DataGOL App — Frontend (React UI)

Child of **`datagol-app-development`** (read the parent first for the single-port
architecture and the `client/` + `server/` monorepo layout). This skill governs
everything under **`client/`**.

> **Also invoke `datagol-frontend-design`** before writing UI files (aesthetic
> direction) and **`react-vite-best-practices`** for the React/Vite conventions
> this skill assumes. If those conflict with this skill, they win on their topic.

## The one hard rule: the client never touches DataGOL directly — and never uses an ABSOLUTE `/api`

- The client calls the app's API **through the `BASE_URL`-relative `api()`
  wrapper only** — never `*.datagol.ai`, and **never a leading-slash `/api/...`**.
- The client **never holds `DATAGOL_SERVICE_TOKEN`** and never imports a token
  constant. All DataGOL access happens in the Express API (`datagol-app-backend`).
- Mode 3 only: the client may hold a per-user bearer in `localStorage` and send
  it to the app's API; the server forwards it. Still no service token in the bundle.

> ⚠️ **Why not absolute `/api`.** In the codex preview the app is served under
> `/preview/<id>/` and the iframe's origin is the **codex**. An absolute
> `fetch('/api/auth/login')` resolves to `http://<codex>/api/auth/login` — the
> **codex's own** API (it really has `/api/auth/login`, `/api/apps`,
> `/api/workspaces`, …), NOT your app. Your login then talks to the codex and
> fails in confusing ways. **Always go through `import.meta.env.BASE_URL`** so the
> call lands at `/preview/<id>/api/...`, which the codex proxies to your app. The
> same wrapper works when published (BASE_URL is `/`, so it's just `/api/...`).
> A grep for `` fetch('/api `` or `` fetch(`/api `` in `client/` must be empty.

```ts
// client/src/lib/api.ts — the ONLY data entry point.
// BASE_URL is the Vite base: '/preview/<id>/' in the codex preview, '/' when
// published. Prepending it makes every call land on THIS app, never the codex.
const BASE = import.meta.env.BASE_URL;

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(BASE + 'api' + path, {   // e.g. /preview/<id>/api/products
    ...init,
    headers: { 'Content-Type': 'application/json', ...(init?.headers ?? {}) },
  });
  if (!res.ok) throw new Error(`${path} → HTTP ${res.status}: ${await res.text().catch(() => '')}`);
  return res.json() as Promise<T>;
}
```

## File structure (`client/`)

```text
client/
├── index.html
├── vite.config.ts        # dev proxy '/api' → the app's server (8787); see parent. NOT 3001 (codex)
├── tsconfig.json         # MUST include "types": ["vite/client"] in compilerOptions
├── package.json
└── src/
    ├── main.tsx          # React DOM bootstrap
    ├── App.tsx           # shell + router
    ├── index.css         # CSS variables (incl. light/dark tokens), resets, responsive
    ├── pages/            # one file per route (Products.tsx, CustomerDetail.tsx, …)
    ├── components/       # Icons.tsx (SVG, never emoji), ui/, forms/, tables
    └── lib/
        ├── api.ts        # the BASE_URL-relative api() wrapper above — the only data layer
        └── utils.ts      # formatting/helpers
```

- All data/fetch logic lives in `src/lib/`. No workspace IDs, table IDs, base
  URLs, or tokens in the client — those are server-side now.

## Routing — every distinct page gets its own URL route

Use **React Router** when the app has >1 page. As soon as there's more than one
entity/section, **each must live at its own path** — never view-switch with
React state across entities (it breaks deep-linking, Back, refresh, bookmarks).

- One route per entity, named after it: `/products`, `/customers`, `/orders`,
  `/dashboard`, `/settings` (plural preferred; be consistent).
- List + detail split: `/products` and `/products/:id` (`useParams`); nested
  relations extend the path (`/customers/:id/orders`).
- One route → one file in `src/pages/`; nav links via `<Link>`; active state from
  `useLocation()`.
- **Iframe-safe (codex preview):** wrap the router with
  `basename={import.meta.env.BASE_URL}` so routes match the Vite `--base` prefix.
  Use fully-qualified URLs for any raw `<a>`; never absolute paths.
- Routerless only for a genuinely single-purpose app (one entity, no detail view).

> `tsconfig.json` must include `"types": ["vite/client"]` in `compilerOptions`
> so `import.meta.env` typechecks; otherwise `tsc` (and the Docker build) fail.

## UI quality rules

1. **Responsive-first is mandatory** — usable on desktop, tablet, phone before
   done. Fluid containers (`width:100%`, `max-width`, `minmax(0,1fr)`),
   breakpoints at ~900px and ~560px, collapse multi-column shells to one column
   on mobile, ≥44px touch targets, data grids become cards/stacked rows on small
   screens.
2. **Light/dark mode is mandatory** — every app supports both, no light-only.
   - Define colors as CSS custom properties on `:root` and a `[data-theme="dark"]`
     (or `.dark`) selector; **never hard-code hex in components** — use the tokens.
   - Initialize from `prefers-color-scheme`, then allow override.
   - Provide a visible toggle in the chrome and **persist** the choice to
     `localStorage`, applied before first paint (no flash).
   - Verify contrast/legibility in **both** modes (text, borders, inputs, cards,
     disabled states).
3. Write **complete files** — never truncate, never leave `TODO`.
4. Every list needs loading, empty, and error states.
5. Every form needs Cancel/back behavior and required-field validation.
6. Detail pages need edit/delete when deletion is in scope; confirm destructive
   deletes (inline confirmation UI, not `window.confirm()` in sandboxed iframes).
7. **Never use emoji as UI icons** — inline SVG icon components instead.
8. Keep all config/data access in `src/lib/` (the `/api` wrapper) — don't scatter
   `fetch` calls across components.

## Tables

Native tables for simple apps; `@tanstack/react-table` only when the app needs
spreadsheet-grade sorting/filtering/column-sizing/selection/pagination. Tables
must fit their container (no horizontal scroll on small screens — become cards or
stacked rows).

## Verify (frontend)

- `tsc --noEmit` clean in `client/`.
- A `grep` for `datagol.ai` / `DATAGOL_SERVICE_TOKEN` in `client/src` returns
  **nothing** — all data goes through `/api/*`.
- App renders in the codex preview (Vite `--base` prefix) and in both themes.
