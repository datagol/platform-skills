---
name: datagol-ui-generation
description: How to generate a complete Next.js + React + TypeScript DataGOL web application with App Router, reusable components, DataGOL API helpers, workbook/table views, and production-grade UI. Use this when scaffolding CRUD apps, internal tools, dashboards, or workbook-driven apps backed by DataGOL.
---

# UI Generation Guide

When a user asks to build, generate, or scaffold a web application from their DataGOL workspace, follow these instructions precisely.

> **Prerequisites — both must already have happened:**
>
> 1. **`datagol-interview`** has run and gathered: purpose, user-management model
>    (multi-user vs single-user demo — mandatory question), entities, key
>    flows, output shape.
> 2. **`datagol-detailed-plan`** has been presented to the user and they've said
>    "yes, proceed". The plan's §1 Architecture (auth model) tells you
>    whether to scaffold sign-in screens; §2 UX tells you which pages to
>    write; §3 Data Model gives you the IDs to hardcode in the app.
>
> Don't skip ahead and start writing files based on a one-line request —
> route it through `datagol-interview` → `datagol-detailed-plan` first.
>
> **If the plan calls for multi-user login, also read `datagol-user-auth`** before
> writing any code. It adds the Users workbook + auth scaffold (sign-in /
> sign-up pages, PBKDF2 hashing, session hook) on top of the standard scaffold
> this skill describes.
>
> **Always invoke `datagol-frontend-design` before writing any UI files.** This
> skill (`datagol-ui-generation`) gives you the layout, data wiring, and file
> structure. `datagol-frontend-design` gives you the aesthetic direction.

## CRITICAL: Use Next.js + React + TypeScript

Generate DataGOL apps with **Next.js, React, and TypeScript** by default.

- Use **Next.js App Router**.
- Use **React client components** (`"use client"`) only where hooks, browser APIs,
  interactive state, or client-side DataGOL fetching are needed.
- Do **not** scaffold a Vite app unless the user explicitly asks for Vite or the
  existing repository is already Vite-only and the user wants to keep it that way.
- Treat `app/`, `components/`, and `lib/` as the primary application folders.

### Dev server / port rule

Do **not** bind the app directly to the platform `PORT` variable. Pi may already
use it. Use `APP_PORT` for the Next.js dev server:

```json
{
  "scripts": {
    "dev": "next dev -H 0.0.0.0 -p ${APP_PORT:-3000}",
    "build": "next build",
    "start": "next start",
    "typecheck": "tsc --noEmit"
  }
}
```

If preview fails, check for port conflicts before changing app logic.

### Package/dependency rule

Prefer the dependencies already present in the workspace. If a new package is
needed, first inspect `package.json` and explain the dependency addition. Do not
silently add packages for simple UI effects that can be implemented with CSS,
React hooks, or browser-native APIs.

For a fresh generated app, the default package stack is:

```json
{
  "scripts": {
    "dev": "next dev -H 0.0.0.0 -p ${APP_PORT:-3000}",
    "build": "next build",
    "start": "next start",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "next": "^15.3.2",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "typescript": "^5.7.3",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19"
  }
}
```

Add `@tanstack/react-table` only when the app needs spreadsheet/workbook-grade
sorting, filtering, column sizing, row selection, pagination, or column
visibility.

## Before You Start

1. Call `datagol_get_workspace_schema` to load the current schema.
2. Identify workbooks, columns, data types, required fields, and LINK relationships.
3. Ask the user how each workbook should be displayed if not specified.
4. If integrating into an existing repo, inspect the repo first and preserve its
   framework, style system, and conventions unless the user requested a migration.

## Tech Stack

Default generated app stack:

- **Framework**: Next.js 15+
- **UI library**: React 19+
- **Language**: TypeScript
- **Routing**: Next.js App Router
- **Styling**: `app/globals.css`, CSS variables, component-level styles, and inline
  styles where appropriate
- **Tables**: native tables for simple apps; `@tanstack/react-table` for complex
  workbook/table UIs when available
- **HTTP**: native `fetch`
- **Data layer**: reusable helpers in `lib/`
- **Data source**: DataGOL REST API
- **DataGOL auth**: `x-auth-token` service-token header
- **State**: React hooks; avoid unnecessary global state unless the app needs it

## Canonical File Structure

Use this structure for new Next.js DataGOL apps:

```text
app/
├── globals.css          — global CSS variables, resets, responsive rules
├── layout.tsx           — root metadata, html/body, global providers if needed
├── page.tsx             — dashboard, primary workbook shell, or landing route
└── {route}/
    └── page.tsx         — additional top-level routes

components/
├── Icons.tsx            — reusable SVG icons; never emoji for UI chrome
├── Sidebar.tsx          — optional app navigation
├── Topbar.tsx           — optional app header/search/user context
├── WorkbookTabs.tsx     — optional workbook/entity tab navigation
├── WorkbookTable.tsx    — optional DataGOL workbook table shell
├── GenericTable.tsx     — optional reusable table renderer
├── forms/               — reusable form components
└── ui/                  — Button, Modal, Badge, EmptyState, etc.

lib/
├── config.ts            — DataGOL base URL, workspace ID, service token, env
├── datagol.ts           — dgFetch, row CRUD helpers, schema helpers
├── schema.ts            — workspace/table/column aliases and IDs
├── workbook-columns.ts  — UI-only column display metadata when needed
├── useWorkbookData.ts   — client hook for row loading/mutation when needed
└── utils.ts             — formatting and shared helpers

public/
└── optional static assets

package.json
next.config.ts
tsconfig.json
```

Placement rules:

- **`lib/`** — every DataGOL API helper and env/config constant lives here.
- **`components/`** — reusable UI elements and icons live here.
- **`app/`** — route entry points live here.
- Do not duplicate workspace IDs, table IDs, service-token logic, or base URLs in
  random components.
- Prefer aliases such as `companies`, `tasks`, or `deals` over scattering raw IDs.

## Routing

Use **Next.js App Router**, not manual `history.pushState` routing.

- `/` should normally be the dashboard, workbook list, or primary app shell.
- Use `app/{route}/page.tsx` for top-level routes.
- Use `next/link` for navigation.
- Use `usePathname`, `useRouter`, and `useSearchParams` from `next/navigation`
  inside client components.
- Add `"use client"` only when a component uses hooks, browser APIs, client-side
  interactivity, or DataGOL client fetches.

### Iframe-safe links (CRITICAL)

The generated app runs inside the DataGOL Codex preview iframe served at
`/preview/<sessionId>/`. Hard-coded **absolute** anchor `href`s in raw
`<a>` tags break out of that prefix — clicking them gives "Cannot GET
/foo" because the request hits the cli-server, not the per-session dev
server.

Rules:

- For Next.js (App Router): always use `<Link href="/foo">` from
  `next/link`, never `<a href="/foo">`. `next/link` uses client-side
  routing — no full page navigation, no prefix loss.
- If you must use a raw `<a>` (external links, mailto, etc.), use a
  fully-qualified URL — never an absolute path that the browser would
  resolve against the iframe's current origin.
- For React Router (only if you're NOT using Next.js for some reason —
  rare; the rest of this skill targets Next.js): wrap your router with
  `basename={import.meta.env.BASE_URL}` so route matching agrees with
  the Vite `--base` prefix:

  ```tsx
  import { BrowserRouter } from 'react-router-dom';
  // import.meta.env.BASE_URL is set by Vite (e.g. "/preview/<sessionId>/")
  <BrowserRouter basename={import.meta.env.BASE_URL}>
    {/* routes written naturally as /workbooks/companies, etc. */}
  </BrowserRouter>
  ```

  Without the `basename`, React Router sees the full prefixed path and
  none of your `<Route path="/...">` declarations match.

## DataGOL API Pattern

Use the DataGOL REST API. Hardcode real workspace ID and table IDs from the
schema response so the app works immediately.

### `lib/config.ts`

```ts
export const DATAGOL_ENV = 'test' as const; // or 'prod' if provided by context
export const DATAGOL_BASE_URL = 'https://testing-be.datagol.ai';
export const DATAGOL_WORKSPACE_ID = 'ACTUAL_WORKSPACE_ID';
export const DATAGOL_SERVICE_TOKEN = 'ACTUAL_SERVICE_TOKEN';
```

### `lib/datagol.ts`

```ts
import { DATAGOL_BASE_URL, DATAGOL_SERVICE_TOKEN, DATAGOL_WORKSPACE_ID } from './config';

export type DGRow = Record<string, unknown>;

export async function dgFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), 30_000);
  try {
    const res = await fetch(`${DATAGOL_BASE_URL}${path}`, {
      ...init,
      signal: controller.signal,
      headers: {
        'Content-Type': 'application/json',
        'x-auth-token': DATAGOL_SERVICE_TOKEN,
        ...(init?.headers ?? {}),
      },
    });

    if (!res.ok) {
      const body = await res.text().catch(() => '');
      throw new Error(`DataGOL ${path} → HTTP ${res.status}${body ? `: ${body}` : ''}`);
    }

    const text = await res.text();
    return (text ? JSON.parse(text) : undefined) as T;
  } finally {
    clearTimeout(timer);
  }
}
```

### Row shape and cursor endpoint

The cursor endpoint returns rows as `{ id, cellValues: { ... } }`. Always flatten
rows before handing them to UI components:

```ts
interface RawRow {
  id: number | string;
  cellValues?: Record<string, unknown>;
  [key: string]: unknown;
}

interface CursorResponse {
  rows?: RawRow[];
  content?: RawRow[];
  list?: RawRow[];
  totalNumberOfRecords?: number;
  totalNumberOfPages?: number;
  totalElements?: number;
  totalPages?: number;
  isLastPage?: boolean;
}

function flattenRow(raw: RawRow): DGRow {
  return { id: raw.id, ...(raw.cellValues ?? {}) };
}

export async function listRows(tableId: string, page = 1, pageSize = 100) {
  const data = await dgFetch<CursorResponse>(
    `/noCo/api/v2/workspaces/${DATAGOL_WORKSPACE_ID}/tables/${tableId}/cursor`,
    {
      method: 'POST',
      body: JSON.stringify({ requestPageDetails: { pageNumber: page, pageSize } }),
    },
  );

  const rawRows = data.rows ?? data.content ?? data.list ?? [];
  return {
    rows: rawRows.map(flattenRow),
    totalPages: data.totalNumberOfPages ?? data.totalPages ?? 1,
    totalElements: data.totalNumberOfRecords ?? data.totalElements ?? rawRows.length,
    isLastPage: data.isLastPage ?? page >= (data.totalNumberOfPages ?? data.totalPages ?? 1),
  };
}
```

### Writes must use `cellValues`

- `POST /rows` body must be `{ cellValues: ... }`.
- `PUT /rows` uses the row id in the body: `{ id: Number(id), cellValues: ... }`.
- Do not send a flat body.
- Strip server-managed fields before writes.
- Coerce DATE values to DataGOL's required timestamp format.

```ts
export function toDataGolDate(value: unknown): unknown {
  if (value == null || value === '') return value;
  if (value instanceof Date) return value.toISOString().replace(/\.\d{3}Z$/, '+00:00');
  if (typeof value === 'number') return new Date(value).toISOString().replace(/\.\d{3}Z$/, '+00:00');
  if (typeof value !== 'string') return value;
  if (/^\d{4}-\d{2}-\d{2}$/.test(value)) return `${value}T00:00:00+00:00`;
  if (/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}$/.test(value)) return `${value}:00+00:00`;
  if (/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}$/.test(value)) return `${value}+00:00`;
  if (/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$/.test(value)) return value.replace(/\.\d{3}Z$/, '+00:00');
  return value;
}

export function sanitizeForWrite(data: Record<string, unknown>): Record<string, unknown> {
  const out: Record<string, unknown> = {};
  for (const [k, v] of Object.entries(data)) {
    if (k === 'id') continue;
    if (k.startsWith('datagol_')) continue;
    if (k.startsWith('noco_')) continue;
    if (k.startsWith('_position')) continue;
    out[k] = toDataGolDate(v);
  }
  return out;
}

export async function createRow(tableId: string, data: Record<string, unknown>) {
  const raw = await dgFetch<RawRow>(
    `/noCo/api/v2/workspaces/${DATAGOL_WORKSPACE_ID}/tables/${tableId}/rows`,
    { method: 'POST', body: JSON.stringify({ cellValues: sanitizeForWrite(data) }) },
  );
  return flattenRow(raw);
}

export async function updateRow(tableId: string, id: string | number, data: Record<string, unknown>) {
  await dgFetch<void>(
    `/noCo/api/v2/workspaces/${DATAGOL_WORKSPACE_ID}/tables/${tableId}/rows`,
    { method: 'PUT', body: JSON.stringify({ id: Number(id), cellValues: sanitizeForWrite(data) }) },
  );
}

export async function deleteRow(tableId: string, id: string | number) {
  await dgFetch<void>(
    `/noCo/api/v2/workspaces/${DATAGOL_WORKSPACE_ID}/tables/${tableId}/rows/${id}`,
    { method: 'DELETE' },
  );
}
```

### Where clause safety

Never use `JSON.stringify(value)` inside a `whereClause`. It produces a
double-quoted string that the DataGOL SQL engine treats as a column identifier.
Use single-quoted SQL string literals and escape embedded single quotes by
doubling them:

```ts
// ❌ WRONG
whereClause: `\`owner\` = ${JSON.stringify(username)}`;

// ✅ CORRECT
const safe = username.replace(/'/g, "''");
whereClause: `\`owner\` = '${safe}'`;
```

## TypeScript Interfaces

Column type → TypeScript type mapping:

- `SINGLE_LINE_TEXT`, `MULTI_LINE_TEXT`, `EMAIL`, `PHONE_NUMBER`, `URL`, `USER` → `string`
- `NUMBER`, `CURRENCY`, `PERCENT`, `RATING` → `number`
- `CHECKBOX` → `boolean`
- `DATE` → `string`
- `LINK` → linked row id(s) or flattened relationship payload depending on DataGOL response

Only include user-facing columns in entity types. Skip system-generated fields.

## Workbook/Table UI Pattern

For workspace-backed CRUD apps, prefer a workbook-style interface when it matches
the user's requested workflow:

- Sidebar for high-level navigation.
- Topbar for search, user context, breadcrumbs, or global actions.
- Tab strip for workbook/entity switching.
- Table body with loading, error, empty, pagination, and refresh states.
- Formatted cells for dates, currency, percentages, URLs, checkboxes, status
  badges, linked records, and users.
- Reusable table renderer driven by schema/config where possible.

Use `@tanstack/react-table` when available and when the app requires:

- Sorting
- Column sizing
- Column visibility
- Row selection
- Client-side filtering
- Workbook/spreadsheet interactions

Simple CRUD apps may use native table markup with well-designed reusable cells.

## UI Quality Rules

1. **Responsive-first is mandatory.** Every generated app must be usable on
   desktop, tablet, and phone before the build is considered complete.
   - In Next.js, put viewport metadata in `app/layout.tsx` using the Metadata API
     or an exported `viewport` object when needed.
   - Use fluid containers (`width: 100%`, `max-width`, `minmax(0, 1fr)`) and avoid
     hard-coded layouts that create horizontal overflow.
   - Add CSS breakpoints at minimum for tablet (`~900px`) and phone (`~560px`).
   - On mobile: collapse multi-column shells to one column, hide or move
     secondary sidebars, make navigation compact/sticky or bottom-style, stack
     page headers/actions, make forms full-width, and ensure modals fit within
     `100svh` with internal scrolling.
   - Touch targets must be at least ~44px high for primary buttons/nav items.
   - Data-heavy grids/tables must become cards, horizontal scroll regions, or
     stacked rows on small screens.
2. Write **complete files** — never truncate and never leave `TODO` placeholders.
3. Every list needs loading, empty, and error states.
4. Every form needs Cancel/back behavior and basic required-field validation.
5. Every detail page needs edit and delete actions when deletion is in scope;
   use `window.confirm()` before destructive deletes in simple prototypes.
6. Never use emoji as UI icons. Emoji render inconsistently and look
   unprofessional in app chrome. Use inline SVG icon components instead.
7. Use real workspace and table IDs from the schema so the app works immediately.
8. Keep secrets centralized in `lib/config.ts` or the auth pattern supplied by
   `datagol-app-auth`; do not scatter tokens across components.

## File Writing Order

For a fresh app, write files in this order so imports resolve:

1. `package.json`
2. `tsconfig.json`
3. `next.config.ts`
4. `app/globals.css`
5. `app/layout.tsx`
6. `lib/config.ts`
7. `lib/schema.ts`
8. `lib/datagol.ts`
9. shared `components/` files
10. route files under `app/`

For existing apps, inspect first and preserve the existing structure unless the
user explicitly approves a migration.

## Verification

After scaffolding or substantial changes:

```bash
npm run typecheck
npm run build
```

For local preview/dev:

```bash
APP_PORT=3000 npm run dev
```

If the preview fails, check whether the port is occupied before changing code.
