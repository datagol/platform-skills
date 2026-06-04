---
name: datagol-app-development
description: How to build a complete DataGOL web application as a single-port full-stack app — a Vite + React + TypeScript client and an Express + TypeScript API served from ONE Node process / ONE container, with all DataGOL workbook calls made server-side from the app's API (never the browser). Use this when scaffolding CRUD apps, internal tools, dashboards, or workbook-driven apps backed by DataGOL. Routes to child skills datagol-app-frontend (UI rules) and datagol-app-backend (API rules).
---

# DataGOL App Development Guide

When a user asks to build, generate, or scaffold a web application from their
DataGOL workspace, follow these instructions precisely. This skill owns the
**architecture**; the two child skills own the details:

- **`datagol-app-frontend`** — everything under `client/` (React UI, routing,
  theming, the `/api` data wrapper).
- **`datagol-app-backend`** — everything under `server/` (Express serving trick,
  the server-side DataGOL proxy, env, auth modes).

Read this parent, then read the child for whichever side you're writing.

> **Prerequisites — both must already have happened:**
> 1. **`datagol-interview`** has run (purpose, user-management model, entities,
>    flows, output shape).
> 2. **`datagol-detailed-plan`** has been presented and approved. Its §1
>    Architecture (auth mode) decides sign-in scaffolding; §2 UX decides pages;
>    §3 Data Model gives the IDs to wire in (now via **server env**, not client).
>
> Don't write files from a one-line request — route through
> `datagol-interview` → `datagol-detailed-plan` first.

## Core principle — single port, full stack

**In production, ONE Express process serves BOTH the built React static files AND
the `/api/*` routes on a single port (one Docker container).** There is no
separate UI server.

**In development**, Vite runs its own dev server (5173, hot reload) and proxies
`/api` → Express (3001); the Vite server does not exist in production. This
dev-vs-prod split is orthogonal to Docker — Docker just wraps the prod command.

**Data flow (the rule that ties everything together):**

```
browser (client/)  ──fetch('/api/...')──►  Express (server/)  ──dgFetch──►  DataGOL REST
   no token                  same origin        service token + x-appid        workbooks
```

- The **client calls only same-origin `/api/*`** — never `*.datagol.ai`.
- The **Express API is the only caller of DataGOL**; the **service token +
  `x-appid` live in server env only**, never in the client bundle.
- Data layer is **DataGOL workspace/workbook** — there is no SQL database.

### Build-time vs runtime — which DataGOL calls go where

This is the rule every app-development skill follows. Decide by **who makes the
call**, not which endpoint:

| | Caller | Goes to |
|---|---|---|
| **Build-time / info-gathering** — schema discovery (`datagol_get_workspace_schema`), service-account provisioning, workbook/agent/pipeline creation, the read-only verification smoke-test | the **codex agent** (operator bearer / Pi tools) | **direct to `*.datagol.ai`** — this code never ships |
| **Runtime** — the shipped app reading/writing data for end users | the **app** | **client → `/api/*` → Express → DataGOL** |

- If the agent is gathering info to *develop* the app, it calls DataGOL
  directly — correct, unchanged.
- If the code *ships and runs for end users*, its DataGOL access goes through
  the app's own Express API; the browser never calls DataGOL or holds the token.
- Any skill whose code runs **in the generated app** (data reads/writes, auth,
  agent chat, connectors, dashboards) is a **runtime** skill and must route
  through `/api/*`. Skills the **agent** runs to scaffold/verify are
  **build-time** and stay direct.

> **Auth mode — fork before scaffolding** (from the approved plan's §1):
> - **Mode 1 (no auth / demo)** and **Mode 2 (own user table)** → service token,
>   server-side. Read `datagol-app-auth` (+ `datagol-user-auth` for Mode 2).
> - **Mode 3 (DataGOL login)** → read `datagol-internal-app-auth`. The client
>   holds a per-user bearer and sends it to `/api/*`; Express forwards it. No
>   service token in the app.
> Full per-mode contract lives in `datagol-app-backend` §Auth + `datagol-app-auth`.

## Layout — npm workspaces monorepo

```
.
├── package.json          # workspaces: [client, server]; scripts below
├── client/               # Vite + React + TS   → see datagol-app-frontend
├── server/               # Express + TS        → see datagol-app-backend
├── Dockerfile            # multi-stage: build → slim single-port runtime
├── .env / .env.example   # server config (DataGOL base URL, service token, app id, port)
└── .dockerignore         # node_modules, dist, .env, .git
```

Root scripts:
```json
{
  "dev":   "concurrently \"npm:dev -w client\" \"npm:dev -w server\"",
  "build": "npm run build -w client && npm run build -w server",
  "start": "node server/dist/index.js"
}
```

Dev proxy (`client/vite.config.ts`): `server: { proxy: { '/api': 'http://localhost:3001' } }`.
The UI always calls same-origin `/api/*`, so the same code works in dev (proxied)
and prod (same server). No CORS config.

## Dockerfile (multi-stage, single-port)

```dockerfile
FROM node:24-alpine AS build
WORKDIR /app
COPY package*.json ./
COPY client/package*.json ./client/
COPY server/package*.json ./server/
RUN npm ci
COPY . .
RUN npm run build

FROM node:24-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
COPY client/package*.json ./client/
COPY server/package*.json ./server/
RUN npm ci --omit=dev
COPY --from=build /app/server/dist ./server/dist
COPY --from=build /app/client/dist ./client/dist
EXPOSE 3001
CMD ["node", "server/dist/index.js"]
```

Run with `docker run --env-file .env -p 3001:3001 <image>` — credentials are
passed at **runtime**, never baked into the image. Ship `.env.example`, gitignore
`.env`.

## Before you start

Decide whether the app needs DataGOL data backing:
- **Pure UI / local-state apps** (calculator, localStorage todo) — skip the
  workspace steps; the `server/` API may be trivial or just serve static.
- **Data-backed apps** — resolve the workspace `id` per `datagol-interview` §2
  (`datagol_create_workspace` for new, `datagol_list_workspaces` for existing),
  call `datagol_get_workspace_schema`, and wire the workspace/table IDs into
  **server env/config** (not the client).

## Build order

1. Root `package.json` (workspaces + scripts), `Dockerfile`, `.dockerignore`,
   `.env.example`.
2. **`server/`** — read `datagol-app-backend`: `index.ts` (serving trick),
   `lib/datagol.ts` (proxy), `routes/<resource>.ts`, `config.ts`.
3. **`client/`** — read `datagol-app-frontend`: `lib/api.ts`, pages, components,
   theming, routing.
4. Wire the auth gate (Mode 2/3) last.

> **You will read `datagol-self-test` after writing files.** Every build turn
> that writes app code must end with a Tier-1 self-test (see Verification).

## Codex preview — ONE instance, don't run the app yourself

The codex sandbox **builds the app and runs a single instance** of it
(`node server/dist/index.js` on one assigned port), then the preview **proxies
`/preview/<id>/*` to that instance**. That proxied instance is the **source of
truth** — it's what the user sees and what `datagol-self-test` probes.

- **Do NOT run the app yourself** — no `npm run dev`, no `npm start`, no
  `node server/dist/index.js`. That spawns a *second* instance on a different
  port; you'd then be testing the wrong server while the preview shows another,
  and a duplicate can fight the codex for a port. The manager already runs it; a
  rebuild happens automatically after your turn.
- **The client must address its API prefix-relative** (`import.meta.env.BASE_URL
  + 'api/...'`, per `datagol-app-frontend`). Inside the preview the iframe's
  origin is the **codex**, so an absolute `/api/*` hits the **codex's** routes
  (`/api/auth/login`, `/api/apps`, …), not your app. Prefix-relative → the call
  goes to `/preview/<id>/api/*` → the codex proxies it to your app. Same wrapper
  works when published (BASE_URL is `/`).
- To verify, hit the **preview URL** (`${API_URL}/preview/<id>/…`), not a
  hand-run port. See `datagol-self-test`.

## Verification — *MANDATORY before declaring done*

```bash
npm run build && npm start
curl localhost:3001/api/health           # {"status":"ok"}
curl -s localhost:3001/ | grep '<title>' # UI served by Express
curl -s localhost:3001/api/<resource>    # JSON from DataGOL via the API layer
! grep -rq "$DATAGOL_SERVICE_TOKEN" client/dist   # token must NOT be in the client bundle
# Docker: docker build -t app . && docker run --env-file .env -p 3001:3001 app
```

`tsc` must be clean in **both** `client/` and `server/`.

> ⛔ **You may NOT tell the user the app is "ready" / "done" / "working" until
> the `datagol-self-test` Tier-1 GATE has run this turn and passed, and you've
> pasted its evidence block.** A green `/api/health` is **not** enough — a real
> `/api/<resource>` must return **200** (proves the server's DataGOL wiring +
> `server/.env` are actually populated; see `datagol-app-auth` Step D). Health
> can pass while every data call 401s on an empty `.env` — that is the bug the
> gate exists to catch. No passed, shown gate → do not claim ready.
