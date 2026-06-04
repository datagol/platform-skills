---
name: datagol-create-dashboard
description: Create a dashboard page in a generated React app by embedding an existing DataGOL BI dashboard via the embed.js script + window.DataGOL.init. Use this skill whenever the user wants to "create a dashboard", "build a dashboard page", "add a dashboard", "embed a dashboard", "show a BI dashboard", "add analytics/charts from DataGOL", or wire a DataGOL BI app/page into a CRM/internal-tool/site. Covers embed.js loading, window.DataGOL.init, token sourcing, and filters.
---

# Create a Dashboard Page

> **Runtime skill — DataGOL access goes through the app's API.** This embeds a
> DataGOL BI dashboard in the **generated app**, so it is **runtime**. The
> **embed token must be obtained from the app's `/api`** (the Express server
> mints/returns it using the server-side service token) — **never bake a service
> token into the client** for `window.DataGOL.init`. `embed.js` itself runs in the
> browser, but the token it receives comes from a same-origin `/api/*` call. Any
> other DataGOL data the dashboard page needs goes through `/api/*` too. See
> `datagol-app-development` §Build-time vs runtime.

Use this when the user asks to add a dashboard to a generated app. The skill dynamically loads `embed.js` and calls `window.DataGOL.init(...)` with the user-supplied config. **Always run the interview below first** to collect the required values before writing any files.

> **In an existing user codebase?** If the sandbox was opened via "Open repo from GitHub" (project config has `source: 'github'`), read the **`datagol-integrate`** skill first. The Dashboard component shape below stays the same; the routing and styling wiring follows the host project's conventions.

---

## Step 1 — Interview (mandatory, run before writing any files)

Ask the user **one message** with these questions:

> To set up your dashboard I need a few details:
>
> 1. **Config** — paste your full embed script or JSON config, or provide these values individually:
>    - `appId`
>    - `pageId`
>    - `baseUrl` (defaults to `https://app.datagol.ai` if not set)
>    - Any filters — each as a `name` + `value` pair
>
> 2. **Token** — how should the dashboard get its `userToken`? Options:
>    - **Hardcoded** — paste the token value directly (fine for prototypes)
>    - **From an env variable** — specify the variable name (e.g. `VITE_DASHBOARD_TOKEN`)
>    - **From an API call** — provide the endpoint URL (e.g. `/api/get-token`) and the response field that holds the token

Parse the user's answer and extract:
- `appId`, `pageId`, `baseUrl`, `filters[]`
- `tokenStrategy`: one of `hardcoded | envVar | apiCall`
- `tokenValue` (for `hardcoded`), `tokenEnvVar` (for `envVar`), `tokenApiUrl` + `tokenApiField` (for `apiCall`)

If the user pastes a script or JSON block, extract all values from it automatically — do not ask follow-up questions for fields that are already present.

---

## Step 2 — Integration choice (pick without asking)

If the project already has other pages (Contacts, Companies, Leads, etc.), add a new **"Dashboard"** entry to the existing nav and route to `<Dashboard />`. If the project is brand-new with no other pages, render `<Dashboard />` directly from `App.tsx` as the only screen.

---

## Step 3 — Files to write

### `src/lib/dashboard-config.ts`

Populate with the values collected in the interview. Do **not** echo this file's contents in chat.

```ts
// src/lib/dashboard-config.ts
export const DASHBOARD_APP_ID  = '<appId from interview>';
export const DASHBOARD_PAGE_ID = '<pageId from interview>';
export const DASHBOARD_BASE_URL = '<baseUrl from interview, default https://app.datagol.ai>';
export const DASHBOARD_FILTERS: { name: string; value: string }[] = [
  // one entry per filter collected, e.g.:
  // { name: 'tenant_name', value: 'acme' },
];
```

### `src/components/Dashboard.tsx`

Dynamically loads `embed.js` once, then calls `window.DataGOL.init(...)`. Token sourcing varies by strategy — see the four patterns below. All config values come from `dashboard-config.ts`; nothing is inlined.

```tsx
// src/components/Dashboard.tsx
import { useEffect, useRef } from 'react';
import {
  DASHBOARD_APP_ID,
  DASHBOARD_PAGE_ID,
  DASHBOARD_BASE_URL,
  DASHBOARD_FILTERS,
} from '../lib/dashboard-config';

declare global {
  interface Window {
    DataGOL?: {
      init: (config: {
        elementId: string;
        appId: string;
        pageId: string;
        baseUrl: string;
        userToken: string;
        filters?: { name: string; value: string }[];
      }) => void;
    };
  }
}

function loadEmbedScript(): Promise<void> {
  return new Promise((resolve, reject) => {
    if (document.getElementById('datagol-embed-script')) { resolve(); return; }
    const script = document.createElement('script');
    script.id = 'datagol-embed-script';
    script.src = 'https://datagol-prod-public-bucket.s3.us-east-1.amazonaws.com/embed.js';
    script.onload = () => resolve();
    script.onerror = () => reject(new Error('Failed to load embed.js'));
    document.head.appendChild(script);
  });
}

// ─── TOKEN SOURCING — pick the pattern that matches tokenStrategy ──────────

// Strategy A — hardcoded
// const TOKEN = '<value>';
// loadEmbedScript().then(() => init(TOKEN));

// Strategy B — envVar
// const TOKEN = import.meta.env.VITE_DASHBOARD_TOKEN as string;
// loadEmbedScript().then(() => init(TOKEN));

// Strategy C — apiCall
// loadEmbedScript()
//   .then(() => fetch('<tokenApiUrl>').then(r => r.json()))
//   .then(data => init(data.<tokenApiField>));

function init(userToken: string) {
  window.DataGOL!.init({
    elementId: 'dashboard',
    appId: DASHBOARD_APP_ID,
    pageId: DASHBOARD_PAGE_ID,
    baseUrl: DASHBOARD_BASE_URL,
    userToken,
    filters: DASHBOARD_FILTERS,
  });
}

export default function Dashboard() {
  const initialised = useRef(false);

  useEffect(() => {
    if (initialised.current) return;
    initialised.current = true;
    // <insert resolved token strategy here>
  }, []);

  return (
    <div
      id="dashboard"
      style={{ width: '100%', height: '100%', minHeight: '100vh' }}
    />
  );
}
```

Replace the comment `// <insert resolved token strategy here>` with the concrete implementation for the chosen strategy. Do not leave strategy comments in the final file — emit only the chosen pattern, cleanly.

### `src/App.tsx` (or wherever the route lives)

**Standalone (fresh project):**

```tsx
import Dashboard from './components/Dashboard';

export default function App() {
  return <Dashboard />;
}
```

**Existing multi-page app:**

Add `'dashboard'` to the nav state type and render `<Dashboard />` when that nav item is active.

---

## File-writing order

1. `src/lib/dashboard-config.ts`
2. `src/components/Dashboard.tsx` (with the chosen token strategy)
3. Update `src/App.tsx` (or the existing router/nav) to render `<Dashboard />`

---

## Token strategy reference

| Strategy | How to resolve `userToken` in `useEffect` |
|---|---|
| `hardcoded` | `const TOKEN = '<value>'; loadEmbedScript().then(() => init(TOKEN));` |
| `envVar` | `const TOKEN = import.meta.env.<VAR_NAME> as string;` then `loadEmbedScript().then(() => init(TOKEN));` |
| `apiCall` | `loadEmbedScript().then(() => fetch('<url>').then(r => r.json())).then(data => init(data.<field>));` |

---

## After writing, tell the user

Keep it short:

> Dashboard page added — click **Dashboard** in the nav (or refresh the preview if it's a standalone app).

Do not mention `appId`, `pageId`, token values, embed script URLs, or UUIDs.

---

## Hard Rules

- **Always run the interview first.** Never write files before collecting `appId`, `pageId`, and the token strategy.
- **Use `embed.js` + `window.DataGOL.init`, not an iframe.** Do not build a raw iframe URL.
- **Guard against double-init** with a `useRef(false)` flag — React StrictMode calls effects twice in dev.
- **Load `embed.js` lazily** via a dynamic `<script>` tag (the `loadEmbedScript()` helper above), not via a `<script>` in `index.html`.
- **Never leave strategy comments in the final output.** Emit only the chosen token pattern.
- **Don't ask "where to put it?"** Default to a new "Dashboard" nav entry in projects with existing nav, or a standalone screen in fresh projects.
- **Never echo `dashboard-config.ts` contents in chat** — it's an implementation detail.

---

## Cross-references

- **`datagol-agent-chat-ui`** — conversational UI pattern for DataGOL agents.
- **`datagol-app-development`** — full CRUD apps over workbooks; use when the user wants to *edit* data, not just view a dashboard.
- **`datagol-context`** — DataGOL data model and platform terminology.
- **`datagol-app-auth`** — service token pattern and environment configuration for DataGOL apps.
