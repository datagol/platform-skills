---
name: datagol-app-auth
description: Authentication and environment configuration for any DataGOL-generated app. Triggered whenever scaffolding code that calls DataGOL APIs (workbooks, connectors, agents). Establishes the service-token pattern (`x-auth-token` header), the test-vs-prod base-URL inheritance from the codex environment (`DATAGOL_BASE_URL` / `DATAGOL_ENV` env vars set by the cli-server), and the one-time service-account provisioning + workspace-access grant the codex performs at scaffold time. Every other generated-app skill (workbook reads/writes, connectors, chat UIs that call DataGOL) sits on top of this — read it first.
---

# DataGOL App Auth & Environment

This skill is the foundation every DataGOL-generated app stands on. It answers two questions every other skill assumes:

1. **Which DataGOL backend does this app talk to?** Inherited from the codex environment — the cli-server sets `DATAGOL_BASE_URL` and `DATAGOL_ENV` in your process env. You read those and bake them in. **Don't ask the user.**
2. **How does it authenticate?** Depends on the auth mode — see "Choose your auth mode first" below.

If you're scaffolding code that calls *any* DataGOL endpoint — workbooks, connectors, agents, RAG, anything — apply the patterns here before writing the first `fetch`.

> ## ⚠️ Topology: DataGOL calls are SERVER-SIDE now
>
> Apps are **single-port full-stack** (see `datagol-app-development`): a React
> `client/` and an Express `server/` served from one process. **The service
> token and every DataGOL call live in `server/`, never the browser.**
>
> ```
> client/ ──fetch('/api/...')──► server/ (Express) ──dgFetch──► DataGOL REST
>   no token       same origin         x-auth-token + x-appid (from process.env)
> ```
>
> - The **service token is provisioned exactly as before** (Steps A/B/C below) —
>   that part is unchanged. What changes is **where it's stored**: in the
>   **server's env** (`process.env.DATAGOL_SERVICE_TOKEN`), not in a client
>   `src/config.ts`. A grep for the token in `client/dist` must return nothing.
> - The **`dgFetch` helper lives in `server/src/lib/datagol.ts`** — see
>   `datagol-app-backend`. The client's only data entry point is a `/api`
>   wrapper (see `datagol-app-frontend`).
> - **Mode 3:** the per-user bearer is sent by the client to `/api/*` and the
>   Express server forwards it to DataGOL — still no token persisted in the bundle.
>
> Wherever this skill below says "`src/config.ts`" / "the generated app's bundle"
> / "direct `fetch()` from the browser works", read it as **the server config /
> the server process** under the single-port model. The provisioning,
> env-inheritance, `x-appid`, and base-URL rules are all unchanged; only the
> client↔server boundary moved.

## Choose your auth mode first

The interview's §3 answer locks the auth mode. Confirm it before doing anything else; **never infer or default it.**

> **Single-port note:** in all modes the app is a Vite client + Express server.
> The **browser only calls the app's `/api/*`**; the **server** is what calls
> DataGOL. The "DataGOL header" column below is what the **server** sends to
> DataGOL, not what the browser sends.

| Mode | When | App-session credential (browser → `/api/*`) | DataGOL header (server → DataGOL) | Service-token provisioning (Steps A/B/C below) |
|------|------|----------------------------------------------|----------------------------------|------------------------------------------------|
| **1. No auth** | Single-user demo | none | `x-auth-token: <SERVICE_TOKEN>` (server env) | **Required** |
| **2. B2B/B2C** | App owns its own user table | app bearer JWT issued by `/api/auth/login` (see `datagol-user-auth`) | `x-auth-token: <SERVICE_TOKEN>` (server env) | **Required** |
| **3. Internal / DataGOL login** | Internal tool, users sign in with their DataGOL credentials | the user's DataGOL bearer, sent to `/api/*` | `Authorization: Bearer <user bearer>` (server forwards it) | **Skip — no service account** |

**If you're on Mode 3**, the §Environment / §`x-appid` rules in this skill still apply, but **stop reading at the end of §Environment** and switch to `datagol-internal-app-auth` for the rest. The service-token provisioning (Steps A/B/C), the `DATAGOL_SERVICE_TOKEN` constant in `src/config.ts`, the `x-auth-token` header in `dgFetch`, and the service-account workspace-grant rules do NOT apply to Mode 3.

**If you're on Mode 1 or Mode 2**, continue below — the entire skill applies. Mode 2 additionally layers `datagol-user-auth` on top of everything described here.

## Mandatory scaffolding checklist — do these first

> ⚠️ **Before writing the app's data code, run these steps in order. The service token does not exist until Step B — you (the agent) must create it. Then you MUST populate `server/.env` (Step D) with the real token + every table id — an empty `.env` is why apps ship that 401 on every call while `/api/health` looks fine. The workspace id is NOT baked: it's resolved at runtime from the service token (which is scoped to exactly one workspace).**

0. **Confirm the auth mode** (Mode 1, 2, or 3 from the table above). The interview's §3 answer is the source of truth; the detailed-plan's §1 Architecture must restate it. **If Mode 3, switch to `datagol-internal-app-auth` now** — Steps 3, 4, and 5 below do not apply to you.
1. **Read the env** — `echo "DATAGOL_ENV=${DATAGOL_ENV}" && echo "DATAGOL_BASE_URL=${DATAGOL_BASE_URL}"`. Details in §Environment.
2. **Probe the user's bearer** against `${DATAGOL_BASE_URL}` — must return HTTP 200, else STOP and surface the env mismatch. Details in §Environment.
3. **Step A — create the service account record** *(Modes 1 + 2 only)*: `POST /idp/api/v1/company/serviceAccount`. Capture `SVC_ID` and `SVC_EMAIL`. Details in §Create the service token.
4. **Step B — generate the service token** *(Modes 1 + 2 only)*: `GET /idp/api/v1/company/serviceAccount/${SVC_ID}/token`. Capture `SERVICE_TOKEN`. **This is the only place the token comes from. There is no pre-provisioned token, no platform-supplied value, no fallback.** Details in §Create the service token.
5. **Step C — grant the service account `CREATOR`** *(Modes 1 + 2 only)*: `POST /noCo/api/v2/workspaces/${WS_ID}/bulkUsers`. Details in §Create the service token.
6. **Step D — POPULATE `server/.env`** *(Modes 1 + 2 — single-port apps)*: write the **real, non-empty** values into the app's runtime env file `server/.env` (NOT just the empty `.env.example` the template ships, and NOT `src/config.ts`):
   - `DATAGOL_SERVICE_TOKEN=<token from Step B>` — the provisioned service token. **NOT** `DATAGOL_TOKEN` (that's the codex's *user* bearer and returns 401 when sent as `x-auth-token`).
   - `DATAGOL_BASE_URL=<from env>`, `DATAGOL_APP_ID=<UUID from DATAGOL_PREVIEW_URL>`.
   - **Do NOT set `DATAGOL_WORKSPACE_ID`.** The server resolves it at boot from the service token via `getWorkspaceId()` (the token is scoped to exactly one workspace — see `datagol-app-backend`). Baking or client-passing a workspace id is the entire "wrong / stale / agent-id-as-workspace-id" bug class; let the credential define the scope. (The var still exists as an optional local-debug override only.)
   - `<RESOURCE>_TABLE_ID=<id>` for **every** workbook/table the server routes read (get them from `datagol_get_workspace_schema`). An `undefined` table id → `undefined` in the request URL → failure.
   - (Mode 2 also: `APP_JWT_SECRET=<random>` — see `datagol-user-auth`.)

   **Then verify it's actually populated** — `server/.env` with only `PORT` set is the #1 cause of "every call 401s / `undefined` in URL while `/api/health` is green". The `datagol-self-test` gate's Check 3 (a real `/api/<resource>` must return 200) exists to catch exactly this; do not declare the app ready until it passes.

Only after all steps succeed: write `server/src/config.ts` (reading from
`process.env`), `server/src/lib/datagol.ts`, the `/api/<resource>` routes, then
the client. The service token lives in `server/.env` only — never in `client/`,
never in the bundle.

## Why a service token, not the user's bearer

The generated app runs on its own — embedded in a sandbox preview, deployed to Vercel, or grafted into the user's existing repo. It can't depend on the user's session, can't prompt them for credentials at runtime, and can't ship the user's bearer JWT (which is short-lived and personal). A **service-account token** scoped to a company solves all three: it's long-lived, it's a real identity in DataGOL's IDP, and it can be granted access to specific workspaces without leaking anything user-specific.

> **Old skills still document `Authorization: Bearer ${DATAGOL_TOKEN}` from `localStorage`.** That's the legacy convention. This skill supersedes it for newly-scaffolded apps.

## Environment — inherited from the codex, not asked

DataGOL has two backends:

| Env  | Backend                          |
|------|----------------------------------|
| prod | `https://be.datagol.ai`          |
| test | `https://testing-gcp-be.datagol.ai`  |

All paths (`/noCo/api/v2/*`, `/idp/api/v1/*`, `/customAgents/api/v1`, `/connector/api/v1/*`, `/integrations/*`) are identical between envs — only the host changes.

**Don't ask the user which environment to target.** The cli-server is started with `DATAGOL_ENV=prod` or `DATAGOL_ENV=test` (its operator decides), and exposes both `DATAGOL_BASE_URL` and `DATAGOL_ENV` to every agent process. The user already sees a TEST badge in the codex header when they're in the test environment, so they know which backend their app is being built against.

Read these from `process.env` and bake them straight into the generated app's `src/config.ts`. Always emit a literal absolute URL — **never** gate on `import.meta.env.DEV`, **never** use a relative path like `/datagol`, **never** scaffold a Vite proxy.

> ⚠️ **MANDATORY — run BOTH commands before continuing to provisioning:**
>
> **Step 1 — read the env:**
> ```bash
> echo "DATAGOL_ENV=${DATAGOL_ENV}" && echo "DATAGOL_BASE_URL=${DATAGOL_BASE_URL}"
> ```
>
> **Step 2 — verify the user's bearer actually authenticates against that URL:**
> ```bash
> curl -s -o /dev/null -w "%{http_code}" \
>   -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
>   "${DATAGOL_BASE_URL}/noCo/api/v2/workspaces"
> ```
> The response **must be 200**. If it returns 401, **STOP immediately** — do not provision a service account, do not write `src/config.ts`, do not try a different backend. Tell the user:
> > "Your session token does not authenticate against `$DATAGOL_BASE_URL`. This is a cli-server misconfiguration — `DATAGOL_TOKEN` and `DATAGOL_BASE_URL` are pointing at different environments. Please restart the codex with a consistent env and token, then try again."
>
> **Never silently fall back to prod when the env says test, or vice versa.** Switching backends mid-session means all provisioned resources (workspace, service account, agent) end up in a different environment than the one the user's session targets — exactly the bug this check prevents.

**Mention the inherited env in the scaffold summary** so the user knows which backend the new app talks to:

> Built against the **prod** backend (`be.datagol.ai`) — the same env this codex is running in. To target test, restart the codex with `DATAGOL_ENV=test` and rebuild.

State the consequence: workbooks, service accounts, and connector data live in *that* env only. Switching envs later means re-provisioning everything.

## Create the service token — 3 curl calls, run before writing any code

> ⚠️ **Applies to Mode 1 and Mode 2 only.** If you're on **Mode 3 (internal app / DataGOL login)**, skip this entire section — there is no service account to create. The generated app authenticates each user with their own DataGOL bearer captured at sign-in time; see `datagol-internal-app-auth`.

> ⚠️ **MANDATORY — the service token does not exist until you run Step B below.**
>
> The `<SERVICE_TOKEN from Step B>` placeholder you'll see in the `src/config.ts` template (next section) is **YOUR Step B output to fill in**, not a pre-provisioned value supplied by the platform. There is no default, no env var to read, no fallback. If you try to write `src/config.ts` without first running these three calls, the generated app will have no working token and every API request will 401.
>
> Run Steps A, B, and C in order. All three are required.

Run these from the Pi extension at scaffold time. The codex has the user's bearer at `process.env.DATAGOL_TOKEN` (set by the cli-server) and the backend host at `process.env.DATAGOL_BASE_URL` (also set by the cli-server). The bearer is the *only* place a user JWT ever appears in this flow.

```bash
BE="${DATAGOL_BASE_URL}"   # set by the cli-server; do not hard-code
WS_ID="<workspace UUID>"    # from datagol_list_workspaces / datagol_create_workspace
APP_NAME="<app name slug>"  # e.g. "gmail-sync"

# Step A — create the service account record (no token yet)
curl -X POST "${BE}/idp/api/v1/company/serviceAccount" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"${APP_NAME}-svc\",\"description\":\"Generated app service account\"}"
# Response: { id: 283, email: "datagol-xxx@inncretech.iam.serviceaccount.com", ... }
# Capture THREE values from this response — carry them through the rest of
# provisioning AND through any later agent / dashboard / element creation:
#   SVC_ID                   = response.id  ← the service account's *userId*;
#                              required later when granting per-element
#                              permissions via POST /noCo/api/v2/elementPermissions/bulk
#                              (see datagol-create-agent for the canonical use)
#   SVC_EMAIL                = response.email
#   SERVICE_ACCOUNT_USER_ID  = response.id  ← alias used by other skills;
#                              same value as SVC_ID

# Step B — retrieve the service account token
# ⚠️  Use GET — POST/PUT/PATCH on this endpoint all return 405.
# The token has a ~1-year expiry. To rotate it later, call this endpoint again.
curl -X GET "${BE}/idp/api/v1/company/serviceAccount/${SVC_ID}/token" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}"
# Response: { token: "14a09...", expiryInSeconds: 31536000, expiryDate: "..." }
# Capture: SERVICE_TOKEN = response.token

# Step C — grant CREATOR on every workspace the app needs
curl -X POST "${BE}/noCo/api/v2/workspaces/${WS_ID}/bulkUsers" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"workSpaceUsers\":[{\"email\":\"${SVC_EMAIL}\",\"role\":\"CREATOR\"}],\"message\":\"\"}"
```

The token has a ~1-year expiry. If you ever need to rotate it, re-run Step B (the GET call returns the current active token; the platform manages rotation server-side).

If the app touches multiple workspaces, repeat Step C for each one before scaffolding any code that reads/writes those workspaces.

> **Don't lose `SVC_ID`.** Workspace-level access (Step C) is the *baseline* — it gives the service account read/write on every workbook in the workspace. But **agents, dashboards, and other element-scoped resources** also need a per-element grant via `POST /noCo/api/v2/elementPermissions/bulk`, which keys on the service account's numeric `userId` (= `SVC_ID`), not its email. Without that grant, the generated app will hit `400 "User does not have access to ..."` even though the service token is valid and the workspace grant is in place. `datagol-create-agent` documents the canonical pattern; other element-creating skills (dashboards, custom views) follow the same shape.

## Writing the config (now `server/` env + `server/src/config.ts`)

> ⚠️ **Single-port model:** this goes in the **server**, not the client. Put
> `DATAGOL_SERVICE_TOKEN`, `DATAGOL_BASE_URL`, `DATAGOL_ENV`, `DATAGOL_APP_ID`,
> and `DATAGOL_AI_URL` in **`server/.env`** (gitignored, runtime-injected) and
> read them via `process.env` in `server/src/config.ts`. The `client/` has **no**
> token and **no** DataGOL URL — it calls `/api/*`. The values + provisioning
> below are unchanged; only the destination file is the server, not `src/config.ts`.
> The example block below shows the *values* to capture — write them as
> `server/.env` keys (`DATAGOL_SERVICE_TOKEN=...`), not a client TS constant.

> ⚠️ **For Mode 3, see `datagol-internal-app-auth` instead.** The Mode 3 `src/config.ts` does NOT export `DATAGOL_SERVICE_TOKEN` (the token is runtime-only, held in `localStorage`). Everything else in this section (DATAGOL_ENV, DATAGOL_BASE_URL, DATAGOL_APP_ID, DATAGOL_AI_URL) is still required for Mode 3 — only the service-token constant is omitted.

Only after Steps A, B, and C have all succeeded. Capture the actual values from the env-read step (`DATAGOL_ENV`, `DATAGOL_BASE_URL`) and Step B (`SERVICE_TOKEN`) and substitute them below. **Never guess. Never default to prod.** If `DATAGOL_ENV=test`, the app must say `test` and point to `testing-gcp-be.datagol.ai`.

```ts
// src/config.ts (generated)
// ⚠️  These values are baked in at scaffold time.
//     DATAGOL_ENV, DATAGOL_BASE_URL, and DATAGOL_PREVIEW_URL come from
//     process.env (cli-server sets them). DATAGOL_SERVICE_TOKEN comes
//     from Step B of §Create the service token.
//     Run `echo $DATAGOL_ENV && echo $DATAGOL_BASE_URL && echo $DATAGOL_PREVIEW_URL`
//     FIRST, then substitute the real values below — do NOT default to prod.
export const DATAGOL_ENV = '<DATAGOL_ENV from process.env>' as const;   // e.g. 'test' or 'prod'
export const DATAGOL_BASE_URL = '<DATAGOL_BASE_URL from process.env>';   // e.g. 'https://testing-gcp-be.datagol.ai'

// The codex app id. Required as the `x-appid` header on every DataGOL API
// call from a generated app — DataGOL uses it to attribute requests to the
// originating codex app for audit, rate-limiting, and per-app permissions.
// Derived from DATAGOL_PREVIEW_URL: the path segment after `/preview/` is
// the appId (a UUID). Example: from
//   DATAGOL_PREVIEW_URL=http://localhost:3001/preview/d2a4c0f8-0b07-4d95-8e7e-326d7665ed37/
// extract `d2a4c0f8-0b07-4d95-8e7e-326d7665ed37` and bake it below.
export const DATAGOL_APP_ID = '<UUID extracted from DATAGOL_PREVIEW_URL>';

// Streaming for agents (chat UIs) lives on a *separate* host. Use this
// for POST /ai/api/v2/messages/streaming. Conversation creation (POST
// /ai/api/v2/conversations) still goes to DATAGOL_BASE_URL. Workbook
// reads/writes also use DATAGOL_BASE_URL.
//   prod: https://ai.datagol.ai
//   test: https://testing-core-ai.datagol.ai
// ⚠️ The prod and test hosts do NOT share a naming pattern (prod is
// `ai.`, test is `testing-core-ai.`) — do not try to derive one from
// the other. Hard-code the branch on DATAGOL_ENV like below.
export const DATAGOL_AI_URL =
  DATAGOL_ENV === 'test' ? 'https://testing-core-ai.datagol.ai' : 'https://ai.datagol.ai';

// Service token — hardcoded after provisioning (Step B above).
// Do NOT use import.meta.env here: the sandbox tsconfig lacks vite/client
// types, causing a TypeScript error, and the env var won't be set anyway
// unless you also write .env.local. Just bake the literal token string.
export const DATAGOL_SERVICE_TOKEN = '<SERVICE_TOKEN from Step B>';
```

For non-Vite frameworks the token sourcing may differ (e.g. `process.env.DATAGOL_SERVICE_TOKEN` on a Node server), but the env/URL pattern is identical.

> **Don't invent a CORS workaround.** The DataGOL backend allows browser origins for the API endpoints used by generated apps — direct `fetch()` from `localhost:<sandbox-port>` works without a proxy. The pattern `import.meta.env.DEV ? '/datagol' : '<backend>'` is wrong: in dev it makes calls land on the Vite dev server (which has no proxy rule and 404s), and in prod it diverges from dev (which makes the dev iframe useless for testing). Always emit one absolute URL string.

**After writing `src/config.ts`, run this sanity check:**
```bash
grep 'DATAGOL_BASE_URL\|DATAGOL_ENV' src/config.ts
```
Verify the baked URL matches what `echo $DATAGOL_BASE_URL` returned. If they differ, correct the file before proceeding.

## Storing the service token in the generated app

> ⚠️ **Mode 3 has no service token to store — skip this whole section.** Mode 3 stores the per-user bearer in `localStorage` under `DATAGOL_USER_TOKEN_KEY`; see `datagol-internal-app-auth`.

**For the Vite sandbox (standard case):** hardcode the token literal directly in `src/config.ts` after Step B. Do **not** use `import.meta.env.VITE_DATAGOL_SERVICE_TOKEN` in the sandbox:

- The sandbox `tsconfig.json` does not include `vite/client` types, so `import.meta.env` is a TypeScript error.
- Even with the types, the env var won't be defined unless you also write `.env.local` — one more file to manage for no benefit in the sandbox context.
- Hardcoding is safe here: the token is a service-account key (not a user credential), the sandbox is ephemeral, and the file is never committed to git.

```ts
// src/config.ts — correct pattern for the Vite sandbox
export const DATAGOL_SERVICE_TOKEN = '14a09daaf3139ae470f9203a67f4adf7504089741524a8674445dc083aaf9d06';
```

**For non-sandbox / production deployments** (Vercel, Netlify, self-hosted):
- Write `.env.local` with `VITE_DATAGOL_SERVICE_TOKEN=<token>` and add `/// <reference types="vite/client" />` to `src/vite-env.d.ts` so TypeScript recognises `import.meta.env`.
- Add `.env.local` to `.gitignore`.
- For static hosts, set the env var in the deploy dashboard instead.
- For plain Node servers, use `.env` + `process.env.DATAGOL_SERVICE_TOKEN`.

In all cases: don't echo the token in any UI, don't log it, don't include it in error messages.

## The `dgFetch` helper

> ⚠️ **Single-port model:** `dgFetch` lives in **`server/src/lib/datagol.ts`**
> and reads the token + `x-appid` from `process.env` — see `datagol-app-backend`
> for the canonical server-side implementation and the `/api/<resource>` routes
> that wrap it. The client never imports `dgFetch`; it calls the `/api` wrapper
> in `datagol-app-frontend`. The header rules below are unchanged (they're just
> applied server-side now).

> ⚠️ **For Mode 3, the body of `dgFetch` differs**: it reads the bearer from `localStorage` and sends `Authorization: Bearer <token>` instead of `x-auth-token: <SERVICE_TOKEN>`, and on 401 it clears storage and hard-reloads to the login page. The Mode 3 `dgFetch` lives in `datagol-internal-app-auth`. The `x-appid` header rides on every request in **all three modes**.

Every DataGOL API call from the generated app routes through one helper. Drop this into `src/api/datagol.ts` (or the framework-equivalent path):

```ts
// src/api/datagol.ts (generated)
import { DATAGOL_BASE_URL, DATAGOL_SERVICE_TOKEN, DATAGOL_APP_ID } from '../config';

export async function dgFetch<T>(
  path: string,
  init?: RequestInit,
): Promise<T> {
  const res = await fetch(`${DATAGOL_BASE_URL}${path}`, {
    ...init,
    headers: {
      'Content-Type': 'application/json',
      'x-auth-token': DATAGOL_SERVICE_TOKEN,
      'x-appid': DATAGOL_APP_ID,
      ...(init?.headers ?? {}),
    },
  });
  if (!res.ok) {
    const body = await res.text().catch(() => '');
    throw new Error(`DataGOL ${path} → HTTP ${res.status}${body ? `: ${body}` : ''}`);
  }
  // Some endpoints return empty bodies (PATCH /rows, DELETE /rows). Guard for that.
  const text = await res.text();
  return (text ? JSON.parse(text) : undefined) as T;
}
```

`x-appid` rides on **every** DataGOL request from the generated app — workbook reads/writes, agent invocations, dashboard fetches, the streaming AI host (`DATAGOL_AI_URL`), everything that hits `*.datagol.ai`. Putting it in `dgFetch` means callers never have to think about it; if you write a one-off `fetch()` that bypasses `dgFetch`, add the header manually.

Components, hooks, and feature modules all call into wrappers built on `dgFetch` (`src/api/workbooks.ts`, `src/api/connectors.ts`, etc.) — never raw `fetch` against `be.datagol.ai`.

## Hard rules

- **(Modes 1 + 2 only.) Always provision the service account before scaffolding any DataGOL API call.** Steps A, B, and C run in order; if any fails, surface the failure to the user — don't silently proceed with a broken auth setup. The token is the agent's output, not a platform-supplied value. **In Mode 3 there is no service account; do not run Steps A/B/C.**
- **(Modes 1 + 2 only.) Never put the *codex operator's* bearer JWT into a generated app.** It's short-lived, personal, and over-privileged. The codex operator's bearer is used only inside the codex / Pi extension for one-time service-account provisioning. **In Mode 3, the bearer baked into `localStorage` is a different identity entirely — the *signed-in app user's* token, captured at runtime from `/idp/api/v1/user/login`. See `datagol-internal-app-auth`.**
- **(Mode 3 only.) Never bake `DATAGOL_SERVICE_TOKEN` into a Mode 3 app's `src/config.ts`.** Mode 3 apps authenticate each user with their own bearer at sign-in; a service token in the bundle would silently let the app fall back to a shared identity. A grep for `DATAGOL_SERVICE_TOKEN` or `x-auth-token` in a Mode 3 generated tree must return nothing.
- **Never hard-code `be.datagol.ai` or `testing-gcp-be.datagol.ai` outside `src/config.ts`.** Every API wrapper imports `DATAGOL_BASE_URL`. A grep for the literal hostnames anywhere else in the generated tree should return nothing.
- **Every DataGOL request MUST send the `x-appid` header** with the codex app id (the UUID from `DATAGOL_PREVIEW_URL`). This is non-negotiable — DataGOL uses it for per-app permissions and audit. `dgFetch` sets it for you; any raw `fetch()` you write directly must add it too. A grep for `'x-appid'` should match every API helper in the generated tree.
- **`DATAGOL_BASE_URL` lives in the server env and is a single absolute URL derived solely from `DATAGOL_ENV`.** Never gate it on a "dev vs prod" flag. The **client** does scaffold a Vite proxy — but for **`/api` → Express** (`server.proxy['/api']`), NOT for DataGOL. The client never holds `DATAGOL_BASE_URL` and never calls `*.datagol.ai`; only the server does. (Legacy guidance about the client calling DataGOL directly / not proxying is superseded by the single-port topology banner at the top.)
- **Always grant the service account `CREATOR` on every workspace the app touches.** If the user later asks the app to touch a new workspace, regrant before adding code that reads/writes it.
- **For element-scoped resources (agents, dashboards, custom views), workspace grant alone is not enough.** Right after creating the element, also `POST /noCo/api/v2/elementPermissions/bulk` with `{elementId, elementType, userId: SVC_ID, permission: "CREATOR"}` so the service account can access *that specific element*. Otherwise the generated app will get `400 "User does not have access to ..."` despite a valid service token. See `datagol-create-agent` for the agent-specific recipe.
- **Always add `.env.local` to `.gitignore`.** If it's already there, fine. If not, add it.
- **Never echo the service token** — not in chat, not in UI, not in scaffold summaries, not in error messages. Reference it as `<service token>` if you must mention it.
- **Inherit the env, don't ask — and run bash first.** Before writing `src/config.ts`, always execute `echo "DATAGOL_ENV=${DATAGOL_ENV}" && echo "DATAGOL_BASE_URL=${DATAGOL_BASE_URL}"` and use the actual output. Never default to prod values. If `DATAGOL_BASE_URL` is somehow unset (shouldn't happen — the cli-server sets it), fail loudly rather than guessing. The user picks the env by how they started the codex; they see it in the header badge.
- **Probe the token before provisioning anything.** After reading the env, run `curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer ${DATAGOL_TOKEN}" "${DATAGOL_BASE_URL}/noCo/api/v2/workspaces"`. If the result is not 200, stop and tell the user — never silently switch to a different backend. A 401 here means `DATAGOL_TOKEN` and `DATAGOL_BASE_URL` are mismatched (different environments) and is a fatal session misconfiguration, not something the agent should work around.
- **Never provision on a different backend than `DATAGOL_BASE_URL`.** If the token only works on prod but `DATAGOL_BASE_URL` says test, the correct action is to surface the mismatch — not to provision on prod and bake prod into the config. Resources provisioned on the wrong backend are unreachable from the user's session.
- **Verify `src/config.ts` after writing it.** `grep 'DATAGOL_BASE_URL\|DATAGOL_ENV' src/config.ts` must show the same URL that `echo $DATAGOL_BASE_URL` returned. If they differ, fix the file before moving on.
- **Mention the inherited env** in the scaffold summary so the user knows which backend their new app talks to.

## Cross-references

- **`datagol-internal-app-auth`** — the **Mode 3** alternative to this skill. Used when the generated app's users are DataGOL users signing in with their own credentials. Replaces the service-token provisioning and the `dgFetch` body; the §Environment / §`x-appid` / §base-URL rules in this skill still apply.
- **`datagol-user-auth`** — the **Mode 2** layer on top of this skill. Adds a Users workbook + PBKDF2 + sign-up/sign-in pages; the service-token pattern below still runs underneath every DataGOL API call.
- **`datagol-app-connector`** — Composio-backed connector for OAuth-based SaaS integrations (Gmail, Slack, Notion, GitHub, …). Uses the same service-token auth pattern this skill establishes.
- **`datagol-workbook-operations`** — workbook read/write API reference. The `dgFetch` helper here replaces the bearer-based examples in that skill for newly-scaffolded apps.
- **`datagol-app-development`** — the parent skill: single-port full-stack architecture (`client/` + `server/`, one process/port). Read it first; it forks on the auth mode (1/2/3) and routes to the two children below.
- **`datagol-app-backend`** — where the service token + `dgFetch` live (server-side), plus the `/api/<resource>` routes and the Express serving trick.
- **`datagol-app-frontend`** — the client's `/api` wrapper and the hard rule that the browser never calls DataGOL or holds the token.
- **`datagol-agent-chat-ui`** — chat UI scaffold. Same auth pattern when the chat calls DataGOL endpoints.
- **`datagol-create-dashboard`** — dashboard scaffold. Same.
- **`datagol-integrate`** — when grafting any of these into an existing user repo. Apply that skill's mounting/styling rules; this skill's auth rules still hold.
