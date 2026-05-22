---
name: datagol-connector
description: The shared pattern for building any 3rd-party OAuth connector inside a DataGOL-generated app — Gmail, Calendar, Outlook, Slack, Salesforce, Drive, etc. Covers the `Integration Connectors` workbook, the OAuth handoff via the existing DataGOL backend redirect (`/idp/api/v1/oauth2/{service}/authorize`), persisting the `connectorId`, the token-broker contract (`/connector/api/v1/instance/byId/{id}`), the Connections page UI shell, the in-page polling loop, and the generated-app API layer structure. Triggered by "build a connector", "connect <service>", "sync my <inbox|calendar|drive|messages>", or any 3rd-party integration request — including specific phrasings like "build a gmail connector", "connect google", "sync my inbox", "build a google calendar integration", "sync my calendar", "build a google integration", "build a slack connector", "connect slack", "sync my slack", "build a slack integration", "index slack messages". Provider-specific details (scopes, schemas, sync logic) live in reference files alongside this skill: `reference/google.md` (Gmail + Calendar), `reference/slack.md` (channels + messages). Depends on `datagol-app-auth`.
---

# DataGOL Connector (parent pattern)

This skill defines the *shape* of every DataGOL connector — what every Gmail, Calendar, Outlook, Slack, or Salesforce integration looks like before you fill in provider-specific details. It pairs with a **provider reference file** alongside this skill (e.g. `reference/google.md`) that supplies the provider's URLs, scopes, data schemas, and sync logic.

> **Read `datagol-app-auth` first.** This skill assumes a service token in `x-auth-token`, the `DATAGOL_BASE_URL` config constant, and the `dgFetch` helper. None of that is repeated here.

## When to use

- The user asks for a 3rd-party data sync — Gmail, Calendar, Outlook, Drive, Slack, Salesforce, etc.
- The user is wiring a "Connections" or "Integrations" page (like the screenshot with Email / Calendar / Storage groups).
- A reference file exists for the provider (`reference/google.md`, `reference/slack.md`, future `microsoft.md`, etc.) — **read it immediately after this SKILL.md.** The reference file supplies provider-specific bits; this parent supplies the shared mechanics.
- No reference file exists yet — use this skill alone, work off placeholders for provider-specific parts (OAuth scopes, API URLs, row schemas), and **write a new reference file afterwards** so the next person doesn't reinvent the same shape.

## Provider references

When the user names a provider, immediately read the matching reference
**before doing anything else** — it carries the scopes, paths, schemas,
backfill logic, polling semantics, and parser bits this parent doesn't
repeat. Use the `Read` tool with the relative path from this folder.

| User mentions | Read |
| --- | --- |
| Gmail, Calendar, "sync my inbox", "sync my calendar", "google integration", `gmail.google.com`, `calendar.google.com` | `reference/google.md` |
| Slack, Slack channels/messages, slack workspace, slack.com URL, "index slack" | `reference/slack.md` |

If the user names a provider not in this table (Outlook, Drive,
Salesforce, etc.), confirm with them — we may not have a reference yet
and should fall back to the parent's generic flow plus writing a new
reference file at the end.

## The connector loop (one-page architecture)

```
[Connections page]
        │
        │  user clicks "Connect <Provider>"
        ▼
GET /idp/api/v1/oauth2/{service}/authorize?clientRedirectUri=<this URL>
        │
        │  302 to Google / Microsoft / etc. consent screen
        ▼
[provider OAuth]
        │
        │  user approves
        ▼
[DataGOL backend redirect URI — already built]
        │
        │  exchanges code with server-held client_secret,
        │  persists tokens, mints connectorId
        ▼
302 to <clientRedirectUri>?connectorId=<id>&success=true&redirectUri=
        │  (Note: param is `success=true`, NOT `status=success`)
        ▼
[Connections page on mount]
   1. PERSIST CONNECTOR ROW to `Integration Connectors` workbook (FIRST)
   2. history.replaceState — scrub query
   3. getTokens(connectorId) → { accessToken, refreshToken, expiresAt, accountEmail, ... }
   4. PATCH connector row with account_email
   5. hand off to provider reference: backfill (last 30 days)
        │
        ▼
[60s poller, while page is open]
   reads `Integration Connectors` workbook every cycle
   for each connected row → provider reference's syncOne(connectorRow):
     - getTokens(connectorRow.connector_id)
     - call provider API for delta since cursor_json
     - dedupe + bulk-insert into the data workbook
     - advance cursor_json AFTER successful insert
```

The provider reference's job is to supply: scopes, provider API URLs, data-row schemas, `syncOne()` implementation, and `cursor_json` shape. Everything else here applies unchanged.

## Pre-flight checklist

Before scaffolding any connector code, in this order:

1. **Run `datagol-app-auth` provisioning** — create service account, mint token, drop it into `.env.local`. The provider reference can't bulk-insert workbook rows without it.
2. **Always ask the user which workspace to use before creating any workbooks.** Call `datagol_list_workspaces` to show the user their available workspaces, then explicitly ask: *"Which workspace should I create the workbooks in?"* Never silently pick the default workspace returned by `datagol_get_workspace_schema` — the user must confirm. If they want a new workspace, create it first with `datagol_create_workspace` and use that ID.
3. **Grant the service account `CREATOR` on the confirmed workspace** — re-run Step C of `datagol-app-auth` provisioning if the workspace differs from the one already granted.
4. **Discover existing workbooks** with `datagol_get_workspace_schema` on the confirmed workspace. If `Integration Connectors` already exists in this workspace, **reuse it** — don't create a second one. Same for the provider data workbooks (the provider reference's responsibility to check).
5. **Announce the polling caveat** to the user upfront, don't bury it: *"sync only runs while this page is open. For always-on sync we'd need a backend job — out of scope for now."*

## The `Integration Connectors` workbook

One workbook per workspace. One row per `(account_email, service_type)` pair. Created once and reused across every connector the user adds in that workspace.

| internal name              | type      | notes                                                       |
|----------------------------|-----------|-------------------------------------------------------------|
| `connector_id`             | LONG_TEXT | unique; opaque id from backend redirect                     |
| `service_type`             | LONG_TEXT | `gmail` \| `calendar` \| `drive` \| `outlook` \| `slack` \| ... |
| `provider`                 | LONG_TEXT | `google` \| `microsoft` \| `slack` \| ...                   |
| `account_email`            | LONG_TEXT |                                                             |
| `account_display_name`     | LONG_TEXT |                                                             |
| `oauth_status`             | LONG_TEXT | `connected` \| `disconnected` \| `error`                    |
| `connected_at`             | DATE      | ISO 8601 with tz                                            |
| `disconnected_at`          | DATE      | nullable                                                    |
| `last_synced_at`           | DATE      |                                                             |
| `cursor_json`              | LONG_TEXT | JSON-encoded; provider reference defines per-service shape         |
| `last_error`               | LONG_TEXT | nullable                                                    |

Use `datagol_create_workbook` to create it with these columns. The cursor is a JSON blob so each provider can encode whatever it needs — Gmail uses `{"history_id":"..."}`, Calendar uses `{"sync_token":"...","calendar_id":"primary"}`, future providers will use whatever fits their API. Don't add a per-provider cursor column to the workbook; it'd make schema migrations brittle.

## OAuth handoff — the 5 shared steps

Every child reuses these. Don't reimplement them in provider references; just call into `src/api/connectors.ts`.

### 1. Start

The Connect button calls:

```
GET ${DATAGOL_BASE_URL}/idp/api/v1/oauth2/<servicePath>/authorize
    ?x-auth-token=<encodeURIComponent(DATAGOL_SERVICE_TOKEN)>
    &sourceType=connector
    &clientRedirectUri=<encodeURIComponent(window.location.href)>
```

This is a full-page navigation (not a `dgFetch` call) — the browser follows the redirect chain to the provider's consent screen, so we can't set request headers. Identity is carried in the **`?x-auth-token=`** query param (same name as the header used elsewhere). `<servicePath>` is per-provider — the provider reference knows which (`gmail`, `gcalendar`, `slack`, `outlook`, ...).

> **`?x-auth-token=` MUST be the service token, never a user JWT.** The backend ties each connector to the identity that initiates OAuth. The broker (`GET /connector/api/v1/instance/byId/{id}`) — which the generated app calls later with the `x-auth-token` *header* set to the service token — only returns tokens for connectors owned by that same identity. Mixing a user JWT in OAuth start with a service token in the broker call produces `errorCodes: ["REAUTHENTICATION_REQUIRED"]` from the broker, even immediately after a fresh OAuth flow. **Both calls must use the same identity.** Never scaffold a `DATAGOL_USER_JWT` constant in the generated app's `config.ts` — no flow needs it.

> **Param name is `x-auth-token`, not `jwtToken`.** Older example curls used `?jwtToken=`; the canonical name today matches the header name. If a provider's OAuth start endpoint redirects to `auth_failed` or returns an empty `connectorId`, double-check the param name is `x-auth-token`.

#### ⚠️ Sandbox / iframe constraint — always use `window.open(_blank)` for OAuth

The generated app is often served inside a sandbox iframe (e.g. the DataGOL codex preview). OAuth providers (Slack, Google, Microsoft, etc.) set `X-Frame-Options: sameorigin` on their consent screens, which causes two cascading failures when the OAuth URL is navigated to inside an iframe:

1. The iframe refuses to display the consent screen → browser shows a chrome-error page.
2. `window.location.href = url` inside an iframe only navigates the iframe, not the top frame.
3. `window.open(url, '_top')` is blocked by cross-origin iframe policy when the sandbox and the top frame are on different origins.

**The correct pattern is always `window.open(url, '_blank')`** — open OAuth in a new tab. After the user approves, the new tab lands on `clientRedirectUri?connectorId=...`, processes the callback, then uses a `localStorage` signal to notify the original tab (still in the iframe) that a new connector is ready. The new tab then calls `window.close()` to clean up.

Complete implementation of `startOAuth` and the cross-tab callback handshake:

```ts
// src/api/connectors.ts
export function startOAuthUrl(servicePath: string): string {
  const token = encodeURIComponent(DATAGOL_SERVICE_TOKEN);
  const back = encodeURIComponent(window.location.href.split('?')[0]);
  return (
    `${DATAGOL_BASE_URL}/idp/api/v1/oauth2/${servicePath}/authorize` +
    `?x-auth-token=${token}&sourceType=connector&clientRedirectUri=${back}`
  );
}

export function startOAuth(servicePath: string): void {
  const url = startOAuthUrl(servicePath);
  const popup = window.open(url, '_blank');
  if (!popup) {
    // Popup blocked — fall back to navigating the current frame directly.
    // The OAuth flow will still work; the user returns to a standalone app URL.
    window.location.href = url;
  }
}
```

In the **Connections page**, after the OAuth callback is fully processed (connector row persisted + backfill complete), emit a localStorage signal and close the tab:

```ts
// At the end of handleOAuthReturn(), after backfill finishes:
localStorage.setItem('oauth_done', Date.now().toString());
await new Promise((r) => setTimeout(r, 300)); // let storage event fire
window.close(); // works because this tab was opened by script
```

In the **same Connections page** (which is still alive in the original iframe), add a `storage` event listener so it refreshes automatically when the new tab signals completion:

```ts
useEffect(() => {
  const onStorage = (e: StorageEvent) => {
    if (e.key === 'oauth_done') loadConnectors();
  };
  window.addEventListener('storage', onStorage);
  return () => window.removeEventListener('storage', onStorage);
}, []);
```

The full cross-tab flow:

```
Original tab (in iframe)
  └─ click Connect
  └─ window.open(oauthUrl, '_blank')  ← new tab opens

New tab
  └─ navigates to provider consent    ← no iframe restriction
  └─ user approves
  └─ lands on clientRedirectUri?connectorId=xxx&status=success
  └─ persists row, backfills data
  └─ localStorage.setItem('oauth_done', ...)
  └─ window.close()

Original tab (storage event)
  └─ loadConnectors()                 ← list refreshes automatically
```

> If the browser blocks popups, `window.open` returns `null` and the fallback (`window.location.href`) kicks in — this navigates the iframe to the OAuth URL which will fail with the X-Frame-Options error. Surface a UI hint: *"If the Slack window didn't open, allow popups for this site in your browser's address bar and try again."*

### 2. User approves on the provider

Out of our hands.

### 3. Backend redirect

The DataGOL backend completes OAuth (using its server-held `client_secret`), persists tokens against a new `connectorId`, then 302s the browser to:

```
<clientRedirectUri>?connectorId=<id>&status=success
```

Or `&status=error&error=<msg>` on failure. The user lands back on whatever URL they started from — that's why `clientRedirectUri = window.location.href` at start.

### 4. PERSIST CONNECTOR ROW (immediately, before anything else)

This is the most important step in the whole flow. On mount, the Connections page reads `URLSearchParams`:

```ts
const params = new URLSearchParams(window.location.search);
const connectorId = params.get('connectorId');
// ⚠️ DataGOL returns ?success=true, NOT ?status=success.
// Always check both for forward-compatibility.
const isSuccess = params.get('success') === 'true' || params.get('status') === 'success';

if (connectorId && isSuccess) {
  // STEP 4 — persist FIRST. Before token fetch, before backfill, before anything.
  await insertRow('Integration Connectors', {
    connector_id: connectorId,
    service_type: <service>,        // provider reference knows this
    provider: <provider>,           // provider reference knows this
    oauth_status: 'connected',
    connected_at: new Date().toISOString(),
  });

  // STEP 4.5 — scrub the query string so refreshes don't reprocess
  // and so connectorId doesn't leak into history / referrer.
  history.replaceState({}, '', window.location.pathname);

  // ... continue with steps 5+
}
```

`<service>` and `<provider>` are determined by which Connect button was clicked. The page can encode that in `clientRedirectUri` (e.g. `?_pendingService=gmail`) so it's still in scope when the redirect comes back, or read it from a per-button `localStorage` flag set right before navigating.

### 5. Hydrate metadata + tokens

Now that the row is durable, fetch tokens and connector metadata:

```ts
const { accessToken, refreshToken, expiresAt, accountEmail, accountDisplayName }
  = await getTokens(connectorId);

await updateRow('Integration Connectors', { connector_id: connectorId }, {
  account_email: accountEmail,
  account_display_name: accountDisplayName,
  last_synced_at: new Date().toISOString(),
});
```

Then call the provider reference's backfill function (`backfillGmail(connectorId)`, `backfillCalendar(connectorId)`, etc.) — that's where the provider-specific work begins.

## Token broker — `getTokens(connectorId)`

The single entry point for getting an access token, anywhere in the app. Lives in `src/api/connectors.ts`.

```ts
// src/api/connectors.ts (generated)
import { dgFetch } from './datagol';

interface Tokens {
  accessToken: string;
  refreshToken: string;
  expiresAt: number;          // unix ms
  accountEmail: string;
  accountDisplayName: string;
  serviceType: string;
}

const tokenCache = new Map<string, Tokens>();

export async function getTokens(connectorId: string): Promise<Tokens> {
  const cached = tokenCache.get(connectorId);
  // 60-second safety margin — refresh just before the broker would say "expired"
  if (cached && cached.expiresAt > Date.now() + 60_000) return cached;

  const raw = await dgFetch<any>(`/connector/api/v1/instance/byId/${connectorId}`);

  // Actual broker response shape (confirmed):
  // {
  //   success: true,
  //   data: {
  //     id: 7246387,
  //     name: 'oauth2Connector',
  //     connectorType: 'SLACK',
  //     credentialType: 'OAUTH2',
  //     userId: 1012,
  //     config: [{
  //       type: 'header',
  //       data: {
  //         accessToken: 'xoxb-...',
  //         tokenType: 'bot',
  //         expiresAt: null,
  //         refreshToken: null,
  //         scope: 'channels:read,...'
  //       }
  //     }]
  //   }
  // }
  const configData = raw.data?.config?.[0]?.data ?? {};
  const tokens: Tokens = {
    accessToken:        configData.accessToken  ?? raw.accessToken  ?? raw.access_token,
    refreshToken:       configData.refreshToken ?? raw.refreshToken ?? raw.refresh_token,
    expiresAt:          configData.expiresAt    ?? raw.expiresAt    ?? Date.now() + 50 * 60_000,
    accountEmail:       raw.data?.accountEmail  ?? raw.accountEmail ?? raw.account_email ?? raw.email ?? '',
    accountDisplayName: raw.data?.name          ?? raw.accountDisplayName ?? '',
    serviceType:        raw.data?.connectorType ?? raw.serviceType ?? raw.service_type,
  };

  tokenCache.set(connectorId, tokens);
  return tokens;
}
```

The cache is **module-level memory only**. Never persist tokens — not to a workbook, not to `localStorage`, not to logs. On page reload, the cache is empty and the broker is re-called for every connector; that's fine.

When a Google API returns `401`, **bypass the cache once** (force-refresh) and retry; if it still 401s, mark the connector `oauth_status: 'error'` and surface a "Reconnect" CTA in the UI.

## The polling loop

`setInterval(syncAll, 60_000)` on Connections page mount, `clearInterval` on unmount. Lives next to the Connections page component (or in a `useEffect` if the framework is React).

```ts
async function syncAll() {
  const connectors = await listRows('Integration Connectors', {
    whereClause: "`oauth_status` = 'connected'",
  });

  for (const row of connectors) {
    try {
      // syncOne is provided by the provider reference, dispatched on row.service_type.
      await syncOne(row);
    } catch (err) {
      console.warn(`[sync] ${row.service_type}/${row.account_email} failed:`, err);
      // Don't crash the loop — one bad connector shouldn't stop the others.
    }
  }
}
```

Why state lives in the workbook and not in memory: page reloads (and tab restores, and browser quits-and-reopens) all need to resume cleanly. The workbook is the single source of truth — every poll cycle re-reads the connector rows. New connectors added by clicking Connect appear in the next cycle automatically.

## Generated-app API layer

Components don't call `fetch` directly. Three files in `src/api/`:

```
src/api/datagol.ts      — dgFetch (from datagol-app-auth)
src/api/workbooks.ts    — listRows, insertRow, bulkInsertRows, updateRow
src/api/connectors.ts   — startOAuth, getTokens, disconnectConnector, listConnectors
```

Plus per-provider client modules added by provider references (`src/api/google.ts` for Gmail/Calendar, future `src/api/microsoft.ts`, etc.).

Sketch of `connectors.ts`:

```ts
// src/api/connectors.ts
import { dgFetch } from './datagol';
import { DATAGOL_BASE_URL, DATAGOL_SERVICE_TOKEN } from '../config';

export function startOAuthUrl(servicePath: string): string {
  const token = encodeURIComponent(DATAGOL_SERVICE_TOKEN);
  const back = encodeURIComponent(window.location.href.split('?')[0]);
  return (
    `${DATAGOL_BASE_URL}/idp/api/v1/oauth2/${servicePath}/authorize` +
    `?x-auth-token=${token}&sourceType=connector&clientRedirectUri=${back}`
  );
}

// IMPORTANT: always open in a new tab (_blank), never navigate the current
// frame. OAuth providers set X-Frame-Options: sameorigin and will be blocked
// inside the sandbox iframe. See "Sandbox / iframe constraint" section above.
export function startOAuth(servicePath: string): void {
  const url = startOAuthUrl(servicePath);
  const popup = window.open(url, '_blank');
  if (!popup) {
    // Popup blocked — fall back (may show X-Frame-Options error in sandbox).
    window.location.href = url;
  }
}

export async function getTokens(connectorId: string) { /* see above */ }

export async function disconnectConnector(_connectorId: string): Promise<void> {
  // No server-side revocation endpoint exists on DataGOL.
  // DELETE /instance/byId/{id} → 405. POST /instance/{id}/disconnect → 404.
  // Disconnect is handled entirely by updating the workbook row to
  // oauth_status = 'disconnected'. This function is intentionally a no-op.
}
```

Sketch of `workbooks.ts`:

```ts
// src/api/workbooks.ts
import { dgFetch } from './datagol';

const WS = import.meta.env.VITE_DATAGOL_WORKSPACE_ID as string;

// Resolve workbook id by name. Cache in module memory.
const workbookIdCache = new Map<string, string>();

async function workbookId(name: string): Promise<string> {
  if (workbookIdCache.has(name)) return workbookIdCache.get(name)!;
  const schema = await dgFetch<any>(`/noCo/api/v2/workspaces/${WS}/schema`);
  const wb = (schema.workbooks ?? schema.tables ?? [])
    .find((w: any) => w.name === name || w.displayName === name);
  if (!wb) throw new Error(`workbook "${name}" not found`);
  workbookIdCache.set(name, wb.id);
  return wb.id;
}

export async function listRows(
  workbook: string,
  opts: { whereClause?: string; pageSize?: number } = {},
): Promise<any[]> {
  const wbId = await workbookId(workbook);
  const body: any = {
    requestPageDetails: { pageNumber: 1, pageSize: opts.pageSize ?? 1000 },
  };
  if (opts.whereClause) body.whereClause = opts.whereClause;
  const data = await dgFetch<any>(
    `/noCo/api/v2/workspaces/${WS}/tables/${wbId}/cursor`,
    { method: 'POST', body: JSON.stringify(body) },
  );
  return (data.rows ?? []).map((r: any) => ({ id: r.id, ...r.cellValues }));
}

export async function insertRow(workbook: string, cellValues: Record<string, unknown>): Promise<void> {
  const wbId = await workbookId(workbook);
  await dgFetch(`/noCo/api/v2/workspaces/${WS}/tables/${wbId}/rows`, {
    method: 'POST',
    body: JSON.stringify({ cellValues }),
  });
}

export async function bulkInsertRows(workbook: string, rows: Record<string, unknown>[]): Promise<void> {
  if (rows.length === 0) return;
  const wbId = await workbookId(workbook);
  await dgFetch(`/noCo/api/v2/workspaces/${WS}/tables/${wbId}/rows/bulk`, {
    method: 'POST',
    body: JSON.stringify(rows.map((cellValues) => ({ cellValues }))),
  });
}

export async function updateRow(
  workbook: string,
  match: Record<string, unknown>,
  cellValues: Record<string, unknown>,
): Promise<void> {
  // Find row by match clause, then PATCH.
  const matchKey = Object.keys(match)[0];
  const matchVal = String(Object.values(match)[0]).replace(/'/g, "''");
  const where = `\`${matchKey}\` = '${matchVal}'`;
  const rows = await listRows(workbook, { whereClause: where, pageSize: 1 });
  if (rows.length === 0) throw new Error(`updateRow: no row matched ${where}`);
  const wbId = await workbookId(workbook);
  await dgFetch(`/noCo/api/v2/workspaces/${WS}/tables/${wbId}/rows/${rows[0].id}`, {
    method: 'PATCH',
    body: JSON.stringify({ cellValues }),
  });
}
```

Date columns (`connected_at`, `last_synced_at`, etc.) need ISO 8601 with timezone. `new Date().toISOString()` produces `2026-04-30T15:23:00.000Z` which DataGOL accepts; if you build a date manually, see `datagol-workbook-operations` for the `normalizeDate()` helper.

## Connections page UI shell

Mirror the screenshot the user provided: three sections (Email / Calendar / Storage), each with a subtitle and a row of provider buttons.

Layout:

```
┌─ Connections ────────────────────────────────────────────┐
│                                                          │
│  Email                                                   │
│  Connect Google or Microsoft for sending and full        │
│  inbox sync                                              │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │ ● Connect Google │  │ ■ Connect Microsoft (disabled) │ │
│  └──────────────────┘  └──────────────────┘              │
│                                                          │
│  Calendar                                                │
│  Connect Google or Microsoft calendars                   │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │ ● Connect Google │  │ ■ Connect Microsoft (disabled) │ │
│  └──────────────────┘  └──────────────────┘              │
│                                                          │
│  Storage                                                 │
│  One storage account — connect Google Drive or OneDrive  │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │ ● Connect Google │  │ ■ Connect Microsoft (disabled) │ │
│  └──────────────────┘  └──────────────────┘              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

Provider button states:

| State        | Display                                                              |
|--------------|----------------------------------------------------------------------|
| not-connected | `Connect <Provider>`                                                 |
| connected    | `account@domain — Disconnect` plus `Last sync 2m ago • 1,247 items`  |
| coming soon  | button rendered `disabled` with tooltip `"coming soon"` (Microsoft)  |

State per (section × provider) is derived from the `Integration Connectors` workbook on every render. Keep the UI dumb — just project the workbook into the layout. New connectors appearing mid-session show up automatically at the next render.

Styling follows the host app's convention via `datagol-integrate`. Don't introduce Tailwind into a non-Tailwind project; don't add styled-components if they're using CSS Modules; etc.

## Disconnect flow

When the user clicks Disconnect on a connected row:

1. **Show an in-page confirmation UI — never `window.confirm()`.** Render a small inline confirmation state (e.g. the button label changes to "Are you sure? Yes / Cancel", or a compact inline alert replaces the button) directly in the connection card. `window.confirm()` is a native browser dialog that is blocked in sandboxed iframes, produces inconsistent UX across browsers, and cannot be styled. Always use React state to manage the confirmation step.
2. **There is no server-side revocation endpoint on DataGOL.** Do not call `DELETE /connector/api/v1/instance/byId/{id}` (405) or `POST /instance/{id}/disconnect` (404). The `disconnectConnector()` function in `src/api/connectors.ts` should be a no-op — disconnect is handled entirely by the workbook row update in the next step.
3. Update the connector row: `oauth_status = 'disconnected'`, `disconnected_at = now` via **`PUT /rows`** (row `id` in the body, not the URL). Never use `PATCH /rows/:id` — it returns 405. Don't delete the row — keeping it gives the user a history of past connections, and re-connecting later can re-use the row by `account_email`.
4. Call the `onDisconnect()` callback to reset UI state.
5. The next polling cycle's `whereClause` filter (`\`oauth_status\` = 'connected'`) automatically skips disconnected rows.

Example in-page confirmation pattern (React):

```tsx
const [confirmDisconnect, setConfirmDisconnect] = useState(false);
const [disconnecting, setDisconnecting] = useState(false);

const handleDisconnect = async () => {
  setDisconnecting(true);
  setConfirmDisconnect(false);
  try {
    try {
      await disconnectConnector(connector.connector_id);
    } catch (e) {
      console.warn('DELETE connector API failed (continuing local disconnect):', e);
    }
    await connectors.update(connector.id, {
      oauth_status: 'disconnected',
      disconnected_at: new Date().toISOString(),
    });
    onDisconnect();
  } catch (e) {
    console.error(e);
  } finally {
    setDisconnecting(false);
  }
};

// In JSX — replace the disconnect button with an inline confirm when clicked:
{!confirmDisconnect ? (
  <button onClick={() => setConfirmDisconnect(true)}>
    <UnlinkIcon /> Disconnect
  </button>
) : (
  <div style={{ display: 'flex', alignItems: 'center', gap: 8 }}>
    <span style={{ fontSize: 13, color: 'var(--text-muted)' }}>Disconnect?</span>
    <button onClick={handleDisconnect} disabled={disconnecting}
      style={{ color: '#EA4335', border: '1px solid rgba(234,67,53,0.35)', background: 'transparent' }}>
      {disconnecting ? <Spinner /> : 'Yes, disconnect'}
    </button>
    <button onClick={() => setConfirmDisconnect(false)}
      style={{ color: 'var(--text-muted)', border: '1px solid var(--border)', background: 'transparent' }}>
      Cancel
    </button>
  </div>
)}
```

## Hard rules

- **Always ask the user which workspace to use before creating any workbooks.** Show the workspace list, get explicit confirmation, then create. Never silently use whatever `datagol_get_workspace_schema` returns by default.
- **All text columns in connector data workbooks must use `LONG_TEXT` as the `uiDataType`.** Never use `SINGLE_LINE_TEXT` for any field in `Integration Connectors` or any provider data workbook (Gmail Messages, Calendar Events, Slack Messages, etc.). This applies to every field that is not a DATE, NUMBER, BOOLEAN, or ID — including short values like `connector_id`, `oauth_status`, and `service_type`. `LONG_TEXT` avoids silent truncation of long connector IDs, email addresses, OAuth tokens, and cursor blobs.
- **Persist `connectorId` to `Integration Connectors` *before* anything else.** It's the sole durable handle to the OAuth grant. If the page reloads (or crashes) before persistence, the user has to re-OAuth — and they will, more often than you think.
- **Never persist `accessToken` or `refreshToken`** to a workbook, `localStorage`, `IndexedDB`, or any logs. Memory-only via the module-level cache.
- **The polling loop reads connectors from the workbook on every cycle, not from in-memory state.** A page refresh or tab restore must resume cleanly without manual intervention.
- **All DataGOL calls go through `dgFetch`** (which sets `x-auth-token`). No raw `fetch` to `be.datagol.ai`.
- **All provider calls use `Authorization: Bearer <accessToken>`** — that's the standard for Google, Microsoft, etc. Don't try `x-auth-token` against a Google endpoint; it'll fail with a confusing error.
- **Always advance the cursor *after* a successful insert batch, never before.** If insert fails, the next poll re-tries from the same cursor and the dedupe step keeps it idempotent. If you advance first and insert fails, you've lost data forever.
- **Always dedupe** by the provider's stable id (Gmail `message_id`, Calendar `event_id`, etc.) before inserting. Use a small `whereClause` query against the data workbook before bulk-inserting; only insert ids not already present.
- **Don't backfill more than 30 days without explicit user confirmation.** Provider rate limits are real (Gmail throttles at ~250 quota-units/user/sec) and so is workbook bloat. If the user explicitly asks for "all my mail" or "everything since 2020", confirm out loud, then page through carefully.
- **Provider buttons that don't have a provider reference yet must render as `disabled`** with a "coming soon" tooltip. Don't build a half-working Microsoft button just because the Google one works.
- **Announce the polling caveat upfront.** "Sync only runs while this page is open" is non-obvious to users used to enterprise integrations. Tell them at scaffold time and surface it in the UI (e.g. footer text).
- **Always use `window.open(url, '_blank')` for OAuth — never `window.location.href` or `window.open(url, '_top')`.** The generated app runs inside a sandbox iframe; OAuth providers block iframe display with `X-Frame-Options: sameorigin`. `_top` navigation is blocked by cross-origin iframe policy. The only working pattern is a new tab (`_blank`) + `localStorage` signal back to the original tab. See the "Sandbox / iframe constraint" section in Step 1 for the complete implementation.
- **Don't re-emit a status summary after every intermediate step.** Connector scaffolding is a multi-step flow (provision service token → create `Integration Connectors` workbook → wire Connections page → start polling loop → …) and each step may want to confirm. Use **one short prose line per step** (`Created Integration Connectors workbook.`, `Connections page wired.`, `Polling loop scheduled.`) and reserve any `Parameter | Value` summary table for the **final** message — emitted exactly once at the end. Each intermediate text emission lives in its own message item (because tool calls split text items in the chat store), so a re-emitted table renders the whole table again every time — stacking four near-identical tables in the chat. One table at the end, prose in between.

## Cross-references

- **`datagol-app-auth`** — service-token + env-switching foundations. Required reading.
- **`reference/google.md`** (alongside this skill) — Gmail + Calendar provider reference. Read on demand.
- **`reference/slack.md`** (alongside this skill) — Slack channels + messages provider reference. Read on demand.
- **`datagol-integrate`** — when grafting the Connections page into an existing user repo. Follows that skill's mounting and styling rules.
- **`datagol-workbook-operations`** — full reference for the workbook read/write APIs the `workbooks.ts` wrappers call.
- **`datagol-workbook-design`** — when you need to design or extend the data-row workbooks beyond what a provider reference specifies.
- **`datagol-frontend-design`** — Connections page styling, button states, dropdowns.
- **`datagol-context`** — DataGOL data model (Workspace → Workbook → Column / Row).
