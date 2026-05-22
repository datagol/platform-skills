---
name: datagol-app-auth
description: Authentication and environment configuration for any DataGOL-generated app. Triggered whenever scaffolding code that calls DataGOL APIs (workbooks, connectors, agents). Establishes the service-token pattern (`x-auth-token` header), the test-vs-prod base-URL inheritance from the codex environment (`DATAGOL_BASE_URL` / `DATAGOL_ENV` env vars set by the cli-server), and the one-time service-account provisioning + workspace-access grant the codex performs at scaffold time. Every other generated-app skill (workbook reads/writes, connectors, chat UIs that call DataGOL) sits on top of this — read it first.
---

# DataGOL App Auth & Environment

This skill is the foundation every DataGOL-generated app stands on. It answers two questions every other skill assumes:

1. **Which DataGOL backend does this app talk to?** Inherited from the codex environment — the cli-server sets `DATAGOL_BASE_URL` and `DATAGOL_ENV` in your process env. You read those and bake them in. **Don't ask the user.**
2. **How does it authenticate?** (`x-auth-token: <SERVICE_TOKEN>`, never the user's bearer.)

If you're scaffolding code that calls *any* DataGOL endpoint — workbooks, connectors, agents, RAG, anything — apply the patterns here before writing the first `fetch`.

## Why a service token, not the user's bearer

The generated app runs on its own — embedded in a sandbox preview, deployed to Vercel, or grafted into the user's existing repo. It can't depend on the user's session, can't prompt them for credentials at runtime, and can't ship the user's bearer JWT (which is short-lived and personal). A **service-account token** scoped to a company solves all three: it's long-lived, it's a real identity in DataGOL's IDP, and it can be granted access to specific workspaces without leaking anything user-specific.

> **Old skills still document `Authorization: Bearer ${DATAGOL_TOKEN}` from `localStorage`.** That's the legacy convention. This skill supersedes it for newly-scaffolded apps.

## Environment — inherited from the codex, not asked

DataGOL has two backends:

| Env  | Backend                          |
|------|----------------------------------|
| prod | `https://be.datagol.ai`          |
| test | `https://testing-be.datagol.ai`  |

All paths (`/noCo/api/v2/*`, `/idp/api/v1/*`, `/customAgents/api/v1`, `/connector/api/v1/*`, `/integrations/*`) are identical between envs — only the host changes.

**Don't ask the user which environment to target.** The cli-server is started with `DATAGOL_ENV=prod` or `DATAGOL_ENV=test` (its operator decides), and exposes both `DATAGOL_BASE_URL` and `DATAGOL_ENV` to every agent process. The user already sees a TEST badge in the codex header when they're in the test environment, so they know which backend their app is being built against.

Read these from `process.env` and bake them straight into the generated app's `src/config.ts`. Always emit a literal absolute URL — **never** gate on `import.meta.env.DEV`, **never** use a relative path like `/datagol`, **never** scaffold a Vite proxy.

> ⚠️ **MANDATORY — run BOTH commands before writing `src/config.ts` or provisioning anything:**
>
> **Step 1 — read the env:**
> ```bash
> echo "DATAGOL_ENV=${DATAGOL_ENV}" && echo "DATAGOL_BASE_URL=${DATAGOL_BASE_URL}"
> ```
>
> **Step 2 — verify the token actually authenticates against that URL:**
> ```bash
> curl -s -o /dev/null -w "%{http_code}" \
>   -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
>   "${DATAGOL_BASE_URL}/noCo/api/v2/workspaces"
> ```
> The response **must be 200**. If it returns 401, **STOP immediately** — do not write `src/config.ts`, do not create a workspace, do not provision a service account, do not try a different backend. Tell the user:
> > "Your session token does not authenticate against `$DATAGOL_BASE_URL`. This is a cli-server misconfiguration — `DATAGOL_TOKEN` and `DATAGOL_BASE_URL` are pointing at different environments. Please restart the codex with a consistent env and token, then try again."
>
> **Never silently fall back to prod when the env says test, or vice versa.** Switching backends mid-session means all provisioned resources (workspace, service account, agent) end up in a different environment than the one the user's session targets — exactly the bug this check prevents.
>
> Capture the actual values from Step 1 output and substitute them below.
> **Never guess. Never default to prod.** If `DATAGOL_ENV=test`, the app must say `test` and point to `testing-be.datagol.ai`.

```ts
// src/config.ts (generated)
// ⚠️  These values are baked in at scaffold time from process.env.
//     DATAGOL_ENV and DATAGOL_BASE_URL are set by the cli-server.
//     Run `echo $DATAGOL_ENV && echo $DATAGOL_BASE_URL` FIRST,
//     then substitute the real values below — do NOT default to prod.
export const DATAGOL_ENV = '<DATAGOL_ENV from process.env>' as const;   // e.g. 'test' or 'prod'
export const DATAGOL_BASE_URL = '<DATAGOL_BASE_URL from process.env>';   // e.g. 'https://testing-be.datagol.ai'

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

// Service token — hardcoded after provisioning (see below).
// Do NOT use import.meta.env here: the sandbox tsconfig lacks vite/client
// types, causing a TypeScript error, and the env var won't be set anyway
// unless you also write .env.local. Just bake the literal token string.
export const DATAGOL_SERVICE_TOKEN = '<SERVICE_TOKEN from Step B>';
```

For non-Vite frameworks the token sourcing may differ (e.g. `process.env.DATAGOL_SERVICE_TOKEN` on a Node server), but the env/URL pattern is identical.

> **Don't invent a CORS workaround.** The DataGOL backend allows browser origins for the API endpoints used by generated apps — direct `fetch()` from `localhost:<sandbox-port>` works without a proxy. The pattern `import.meta.env.DEV ? '/datagol' : '<backend>'` is wrong: in dev it makes calls land on the Vite dev server (which has no proxy rule and 404s), and in prod it diverges from dev (which makes the dev iframe useless for testing). Always emit one absolute URL string.

**Mention the inherited env in the scaffold summary** so the user knows which backend the new app talks to:

> Built against the **prod** backend (`be.datagol.ai`) — the same env this codex is running in. To target test, restart the codex with `DATAGOL_ENV=test` and rebuild.

State the consequence: workbooks, service accounts, and connector data live in *that* env only. Switching envs later means re-provisioning everything.

**After writing `src/config.ts`, run this sanity check:**
```bash
grep 'DATAGOL_BASE_URL\|DATAGOL_ENV' src/config.ts
```
Verify the baked URL matches what `echo $DATAGOL_BASE_URL` returned. If they differ, correct the file before proceeding — do not move on to service account provisioning with a mismatched config.

## Provisioning the service account (one-time, codex-side, 3 calls)

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

## Storing the service token in the generated app

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

Every DataGOL API call from the generated app routes through one helper. Drop this into `src/api/datagol.ts` (or the framework-equivalent path):

```ts
// src/api/datagol.ts (generated)
import { DATAGOL_BASE_URL, DATAGOL_SERVICE_TOKEN } from '../config';

export async function dgFetch<T>(
  path: string,
  init?: RequestInit,
): Promise<T> {
  const res = await fetch(`${DATAGOL_BASE_URL}${path}`, {
    ...init,
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
  // Some endpoints return empty bodies (PATCH /rows, DELETE /rows). Guard for that.
  const text = await res.text();
  return (text ? JSON.parse(text) : undefined) as T;
}
```

Components, hooks, and feature modules all call into wrappers built on `dgFetch` (`src/api/workbooks.ts`, `src/api/connectors.ts`, etc.) — never raw `fetch` against `be.datagol.ai`.

## Hard rules

- **Never put the user's bearer JWT into a generated app.** It's short-lived, personal, and over-privileged. The bearer is used only inside the codex / Pi extension for one-time provisioning.
- **Never hard-code `be.datagol.ai` or `testing-be.datagol.ai` outside `src/config.ts`.** Every API wrapper imports `DATAGOL_BASE_URL`. A grep for the literal hostnames anywhere else in the generated tree should return nothing.
- **`DATAGOL_BASE_URL` must be a single absolute URL string, derived solely from `DATAGOL_ENV`.** Never gate it on `import.meta.env.DEV` / `process.env.NODE_ENV` / any "dev vs prod" flag. Never use a relative path like `/datagol`. Never scaffold a Vite proxy in `vite.config.ts` for `/datagol/*` — calls go to the DataGOL backend directly, in both dev and built modes.
- **Always provision the service account before scaffolding any DataGOL API call.** Steps A, B, and C run in order; if any fails, surface the failure to the user — don't silently proceed with a broken auth setup.
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

- **`datagol-connector`** — the parent connector skill. Builds on this for OAuth-based 3rd-party integrations (Gmail, Calendar, etc.).
- **`datagol-workbook-operations`** — workbook read/write API reference. The `dgFetch` helper here replaces the bearer-based examples in that skill for newly-scaffolded apps.
- **`datagol-ui-generation`** — generic CRUD UI patterns. Wire through `dgFetch`, not raw `fetch`.
- **`datagol-agent-chat-ui`** — chat UI scaffold. Same auth pattern when the chat calls DataGOL endpoints.
- **`datagol-create-dashboard`** — dashboard scaffold. Same.
- **`datagol-integrate`** — when grafting any of these into an existing user repo. Apply that skill's mounting/styling rules; this skill's auth rules still hold.
