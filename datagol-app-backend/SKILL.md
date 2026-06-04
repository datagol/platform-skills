---
name: datagol-app-backend
description: Backend (Express + TypeScript) rules for a single-port DataGOL app — the serving trick (one process serves the React build AND /api/*), strict route order, the server-side DataGOL proxy layer (dgFetch with the service token + x-appid from env), env config, and per-mode auth. Child of datagol-app-development. Read this whenever writing anything under server/.
---

# DataGOL App — Backend (Express API)

Child of **`datagol-app-development`** (read the parent first for the single-port
architecture and the `client/` + `server/` monorepo layout). This skill governs
everything under **`server/`**.

The backend has two jobs:
1. In production, **serve the built React client AND the `/api/*` routes from one
   Express process on one port**.
2. Be the **only** place that talks to DataGOL — the browser never holds the
   service token or calls `*.datagol.ai` directly.

## The serving trick — ORDER MATTERS (`server/src/index.ts`)

```ts
import express from 'express';
import path from 'path';
import { fileURLToPath } from 'url';
import { films } from './routes/films.js';   // NodeNext: .js specifiers

const app = express();
app.use(express.json());

// 1. API routes FIRST
app.use('/api/films', films);
app.get('/api/health', (_req, res) => res.json({ status: 'ok' }));

// 2. Static React build SECOND
const __dirname = path.dirname(fileURLToPath(import.meta.url)); // = server/dist
const clientDist = path.resolve(__dirname, '../../client/dist');
app.use(express.static(clientDist));

// 3. SPA fallback LAST — non-API GETs return index.html (client-side routing)
app.use((req, res, next) => {
  if (req.method !== 'GET' || req.path.startsWith('/api/')) return next();
  res.sendFile(path.join(clientDist, 'index.html'));
});

app.listen(Number(process.env.PORT ?? 3001));
```

- **API → `express.static` → SPA fallback.** If static/fallback comes first it
  swallows `/api/*`.
- **The SPA fallback MUST skip `/api/`**, or an unknown API path returns HTML
  instead of a JSON 404.
- **`__dirname` is `server/dist` at runtime** → `../../client/dist` resolves on
  host and in the container (same relative layout).
- **NodeNext TS**: use `.js` import specifiers in server source.

## The DataGOL proxy layer (`server/src/lib/datagol.ts`)

This is the old client `lib/datagol.ts`, moved server-side. The service token and
`x-appid` come from **`process.env`** — never from a bundled constant. The client
reaches this only through `/api/*`.

```ts
const DATAGOL_BASE_URL = process.env.DATAGOL_BASE_URL!;
const DATAGOL_SERVICE_TOKEN = process.env.DATAGOL_SERVICE_TOKEN!;
const DATAGOL_APP_ID = process.env.DATAGOL_APP_ID!;
// Optional override only — leave UNSET. The workspace is RESOLVED from the
// service token (getWorkspaceId below). Set this only to force a specific id
// during local debugging.
const DATAGOL_WORKSPACE_ID_OVERRIDE = process.env.DATAGOL_WORKSPACE_ID;

export type DGRow = Record<string, unknown>;

export async function dgFetch<T>(p: string, init?: RequestInit): Promise<T> {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), 30_000);
  try {
    const res = await fetch(`${DATAGOL_BASE_URL}${p}`, {
      ...init,
      signal: controller.signal,
      headers: {
        'Content-Type': 'application/json',
        'x-auth-token': DATAGOL_SERVICE_TOKEN,   // service token: SERVER ONLY
        'x-appid': DATAGOL_APP_ID,               // required on EVERY DataGOL call
        ...(init?.headers ?? {}),
      },
    });
    if (!res.ok) {
      const body = await res.text().catch(() => '');
      throw new Error(`DataGOL ${p} → HTTP ${res.status}${body ? `: ${body}` : ''}`);
    }
    const text = await res.text();
    return (text ? JSON.parse(text) : undefined) as T;
  } finally {
    clearTimeout(timer);
  }
}

// The service token is scoped to EXACTLY ONE workspace — listing workspaces
// with it returns that single workspace, and any other workspace id 403s.
// So resolve the workspace id from the token itself, once, and cache it.
// NEVER accept a workspace id from the client and NEVER hardcode one: doing so
// is the entire "wrong / stale / agent-id-as-workspace-id" bug class. The
// credential IS the scope.
let _workspaceId: string | undefined;
export async function getWorkspaceId(): Promise<string> {
  if (DATAGOL_WORKSPACE_ID_OVERRIDE) return DATAGOL_WORKSPACE_ID_OVERRIDE;
  if (_workspaceId) return _workspaceId;
  const list = await dgFetch<Array<{ id: string }>>('/noCo/api/v2/workspaces');
  if (!Array.isArray(list) || list.length === 0) {
    throw new Error('Service token resolves to no workspace — re-check provisioning Step C (workspace grant).');
  }
  _workspaceId = list[0]!.id;            // scoped to exactly one
  return _workspaceId;
}

// Every workbook/table call derives the workspace id from the token:
//   const ws = await getWorkspaceId();
//   await dgFetch(`/noCo/api/v2/workspaces/${ws}/tables/${tableId}/cursor`, …)
```

> `x-appid` (`DATAGOL_APP_ID`) rides on **every** DataGOL request. See
> `datagol-app-auth`.
>
> **Workspace id: resolve, never receive.** Always get it from `getWorkspaceId()`
> (derived from the scoped service token). Never read it from a client request
> param/body and never bake a literal id — the token only grants one workspace,
> so resolution is unambiguous and tamper-proof (the backend 403s any other id).

### Expose data through REST endpoints, not a passthrough

Each resource gets typed `/api/<resource>` routes (`server/src/routes/<resource>.ts`).
**Validate/whitelist inputs** and map DataGOL errors to clean JSON status codes —
never blindly forward a raw DataGOL response or let a `whereClause` take raw
client input.

```ts
// server/src/routes/films.ts
import { Router } from 'express';
import { listRows, createRow } from '../lib/datagol.js';
export const films = Router();
const FILMS_TABLE = process.env.FILMS_TABLE_ID!;   // table ids from the schema, via env/config

films.get('/', async (req, res) => {
  try { res.json(await listRows(FILMS_TABLE, Number(req.query.page ?? 1))); }
  catch (e) { res.status(502).json({ error: e instanceof Error ? e.message : 'upstream error' }); }
});
films.post('/', async (req, res) => {
  try { res.status(201).json(await createRow(FILMS_TABLE, req.body)); }
  catch (e) { res.status(502).json({ error: e instanceof Error ? e.message : 'upstream error' }); }
});
```

### Row shape, writes, where-clause safety (unchanged — now server-side)

These rules are identical to before; they just live in `server/`:

- The cursor endpoint returns `{ id, cellValues: {...} }` — **flatten** rows
  (`{ id, ...cellValues }`) before returning them to the client.
- **Writes use `cellValues`**: `POST /rows` → `{ cellValues }`; `PUT /rows` →
  `{ id: Number(id), cellValues }`. Never a flat body. Strip `id` / `datagol_*`
  / `noco_*` / `_position*` and coerce DATE values to DataGOL's
  `YYYY-MM-DDTHH:mm:ss+00:00` format before writing.
- **`whereClause` safety**: never `JSON.stringify(value)` (it becomes a column
  identifier). Use single-quoted SQL literals and double embedded quotes:
  `` `owner` = '${username.replace(/'/g, "''")}'``.
- **TS interfaces**: map DataGOL column types → TS (`NUMBER/CURRENCY/PERCENT/
  RATING`→`number`, `CHECKBOX`→`boolean`, text/`DATE`/`USER`→`string`, `LINK`→
  linked id(s)). Only user-facing columns; skip system fields.

(The full `listRows`/`createRow`/`updateRow`/`deleteRow`/`sanitizeForWrite`/
`toDataGolDate` helpers are the canonical implementations — carry them verbatim
into `server/src/lib/datagol.ts`.)

## API docs (Swagger) — keep `/api/docs` in sync

The template mounts Swagger UI at **`/api/docs`** from an OpenAPI spec in
`server/src/openapi.ts`:

```ts
import swaggerUi from 'swagger-ui-express'
import { openapiSpec } from './openapi.js'
app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(openapiSpec))   // after express.json(), with the other /api routes
```

**Whenever you add or change an `/api/<resource>` route, add/update its entry in
`server/src/openapi.ts`** (path, method, params, request body, responses) so the
docs match the API. Treat the spec as part of the route — don't add a route
without documenting it.

- Deps are already in the template (`swagger-ui-express` + `@types/swagger-ui-express`).
- Access: directly at `http://localhost:<port>/api/docs/` (cleanest). In the
  codex preview, Swagger-UI's own assets can be finicky behind the
  `/preview/<id>/` prefix-strip, so prefer the direct port for the docs UI.
- Keep it under `/api/` so the SPA fallback doesn't swallow it.

## Env config — runtime-injected, never baked

`server/src/config.ts` reads everything from `process.env`. Ship `.env.example`,
gitignore `.env`, and never bake secrets into the image (pass `--env-file .env`
at `docker run`).

| Var | Purpose |
|---|---|
| `PORT` | the single port (default 3001) |
| `DATAGOL_BASE_URL` | DataGOL REST host (prod/test) |
| `DATAGOL_SERVICE_TOKEN` | service token — **server only** |
| `DATAGOL_APP_ID` | `x-appid` on every DataGOL call |
| `DATAGOL_AI_URL` | streaming/agent host (if the app uses agents) |
| `DATAGOL_WORKSPACE_ID` | **optional override only** — normally UNSET; the workspace is resolved from the service token via `getWorkspaceId()` |
| table ids | per-workbook ids the app reads/writes (the workspace itself is resolved, not baked) |

## Auth modes (see `datagol-app-auth` for the full contract)

- **Mode 1 / Mode 2 (service token):** the token lives only in the server env.
  The client calls `/api/*` with no DataGOL credential at all. For Mode 2, the
  Users workbook + row-scoping logic runs **in the API** (the server knows the
  signed-in user from the app's own session, then scopes the `whereClause`).
- **Mode 3 (DataGOL login):** the client sends its per-user bearer to `/api/*`
  (e.g. `Authorization: Bearer <token>` from `localStorage`); Express **forwards**
  it to DataGOL as the per-request credential instead of the service token. The
  bearer is still never persisted in the bundle.

## Verify (backend)

```bash
npm run build && npm start
curl localhost:3001/api/health            # {"status":"ok"}
curl -o/dev/null -w '%{http_code}\n' localhost:3001/api/docs/  # 200 (Swagger UI)
curl -s localhost:3001/ | grep '<title>'  # UI served by Express
curl -s localhost:3001/api/<resource>     # JSON from DataGOL via the API layer
curl -o/dev/null -w '%{http_code}\n' localhost:3001/api/nope   # 404 JSON, not HTML
! grep -rq "$DATAGOL_SERVICE_TOKEN" client/dist   # token must NOT be in the client bundle
```
