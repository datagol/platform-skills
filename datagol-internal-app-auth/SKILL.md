---
name: datagol-internal-app-auth
description: How to add a DataGOL-credential login to a single-port DataGOL app (Vite client + Express server), for internal tools where every user is already a DataGOL user in the same company. **This is the Mode 3 auth path.** The client posts {email,password} to the app's /api/auth/login; the Express server calls DataGOL /idp/api/v1/user/login, captures the bearer from the Authorization RESPONSE HEADER, and returns it; the client then sends that bearer on every /api/* call and the server FORWARDS it to DataGOL. No service token, no signup button. Triggered when the interview's §3 answer is "Internal app — DataGOL users log in".
---

# DataGOL Internal-App Auth (Mode 3)

> **Runtime skill, single-port.** Read `datagol-app-development` (architecture)
> and `datagol-app-backend` (Express serving + `dgFetch`) first. Everything here
> is in the app's **`server/`** + **`client/`** — the **browser never calls
> `*.datagol.ai`**; the Express server is the only thing that talks to DataGOL.

Mode 3 has no service account, no Users workbook, no PBKDF2. Every user signs in
with their **existing DataGOL credentials** against DataGOL's IDP; the bearer
that comes back is what authenticates every subsequent call — but that call now
goes **through the app's own API**, which forwards the bearer to DataGOL.

## When to invoke

- Interview §3 answer **#3 — Internal app, DataGOL users log in**.
- Replaces `datagol-app-auth`'s service-token provisioning AND `datagol-user-auth`'s
  sign-up flow. No Steps A/B/C, no Users workbook, no sign-up page.
- The detailed-plan §1 must state "auth model: Mode 3 — internal app / DataGOL login".

## Architecture (server-proxied)

```text
┌──────── browser (client/) ────────┐   ┌──────── Express (server/) ────────┐   ┌─ DataGOL IDP ─┐
│ Login.tsx                          │   │                                    │   │               │
│  POST /api/auth/login {email,pass} ├──►│ POST /idp/api/v1/user/login ───────┼──►│ verify creds  │
│                                    │   │  reads bearer from RESPONSE HEADER │◄──┤ bearer in     │
│  ◄── { token, user } ──────────────┤   │  returns { token, user }           │   │ Authorization │
│                                    │   │                                    │   │ response hdr  │
│  localStorage: DATAGOL_USER_TOKEN  │   │                                    │   └───────────────┘
│  Authorization: Bearer <token>     │   │ forwardDataGol(req):               │   ┌─ DataGOL API ─┐
│  GET /api/<resource> ──────────────┼──►│  Authorization: Bearer <fwd>       ├──►│ workbooks /   │
│                                    │   │  x-appid: <DATAGOL_APP_ID>         │◄──┤ agents / etc. │
└────────────────────────────────────┘   └────────────────────────────────────┘   └───────────────┘
```

- **Browser ↔ app:** only `/api/*` (same origin). The bearer rides in the client's
  `Authorization` header to `/api/*`.
- **App ↔ DataGOL:** the server forwards the user's bearer + `x-appid`. No service
  token anywhere. DataGOL's IDP is the user store and source of truth.
- When the bearer expires (DataGOL 401 → `/api/*` 401), the client clears its
  session and routes to login.

## Server — `server/src/routes/auth.ts`

The login call to the DataGOL IDP runs **server-side**, and the critical detail
is unchanged: **the bearer is in the `Authorization` RESPONSE HEADER, not the
JSON body.** `await res.json()` returns the user object, not the token.

```ts
import { Router } from 'express'

const DATAGOL_BASE_URL = process.env.DATAGOL_BASE_URL!
const DATAGOL_APP_ID = process.env.DATAGOL_APP_ID!

export const auth = Router()

auth.post('/login', async (req, res) => {
  const { email, password } = req.body ?? {}
  if (!email || !password) return res.status(400).json({ error: 'Email and password required' })
  try {
    const r = await fetch(`${DATAGOL_BASE_URL}/idp/api/v1/user/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'x-appid': DATAGOL_APP_ID },
      body: JSON.stringify({ email, password }),
    })
    if (!r.ok) return res.status(401).json({ error: 'Invalid email or password' })  // don't echo backend msg
    // Token is in the Authorization RESPONSE header (check both casings; strip "Bearer ").
    const authHeader = r.headers.get('Authorization') ?? r.headers.get('authorization')
    if (!authHeader) return res.status(502).json({ error: 'Login response missing Authorization header' })
    const token = authHeader.replace(/^Bearer\s+/i, '')
    const user = await r.json().catch(() => null)  // body = user object (for display)
    res.json({ token, user })
  } catch { res.status(502).json({ error: 'Login failed' }) }
})
```

> **Never store the password.** It exists only for the duration of this call.
> Only the token is returned to the client.

## Server — forward the user bearer to DataGOL (`server/src/lib/datagol.ts`)

Mode 3's server `dgFetch` forwards the **per-request user bearer** (read from the
incoming `/api/*` request) instead of a service token. `x-appid` still rides on
every call.

```ts
const DATAGOL_BASE_URL = process.env.DATAGOL_BASE_URL!
const DATAGOL_APP_ID = process.env.DATAGOL_APP_ID!

// `bearer` is the user's token, taken from the incoming /api/* Authorization header.
export async function dgFetch<T>(bearer: string, path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${DATAGOL_BASE_URL}${path}`, {
    ...init,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${bearer}`,   // forwarded USER bearer — NOT a service token
      'x-appid': DATAGOL_APP_ID,
      ...(init?.headers ?? {}),
    },
  })
  if (!res.ok) { const e = new Error(`DataGOL ${path} → HTTP ${res.status}`); (e as any).status = res.status; throw e }
  const text = await res.text()
  return (text ? JSON.parse(text) : undefined) as T
}
```

`server/src/lib/require-user.ts` — pull the bearer off `/api/*` and 401 if missing:

```ts
import type { Request, Response, NextFunction } from 'express'
export function requireUser(req: Request & { bearer?: string }, res: Response, next: NextFunction) {
  const h = req.headers.authorization
  if (!h?.startsWith('Bearer ')) return res.status(401).json({ error: 'Not authenticated' })
  req.bearer = h.slice(7); next()
}
```

Data routes: `app.use('/api/tasks', requireUser, tasks)`, and inside, call
`dgFetch(req.bearer!, …)`. Map a DataGOL 401 (`err.status === 401`) to a `401`
on `/api/*` so the client knows to re-login.

## Env (server only)

`server/.env` (+ `.env.example`): `DATAGOL_BASE_URL`, `DATAGOL_APP_ID`,
`DATAGOL_AI_URL` (if agents). **No `DATAGOL_SERVICE_TOKEN`.** A grep for
`DATAGOL_SERVICE_TOKEN` or `x-auth-token` anywhere in a Mode 3 app must return
nothing — that's a Mode 1/2 pattern leaking in.

## Client — talks only to `/api/*`

### `client/src/lib/api.ts` — send the bearer, clear on 401

```ts
const TOKEN_KEY = 'DATAGOL_USER_TOKEN'
export function getToken() { return localStorage.getItem(TOKEN_KEY) }
export function setToken(t: string | null) { t ? localStorage.setItem(TOKEN_KEY, t) : localStorage.removeItem(TOKEN_KEY) }

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const token = getToken()
  const res = await fetch(import.meta.env.BASE_URL + 'api' + path, {
    ...init,
    headers: { 'Content-Type': 'application/json', ...(token ? { Authorization: `Bearer ${token}` } : {}), ...(init?.headers ?? {}) },
  })
  if (res.status === 401) {           // expired/revoked → clear + bounce to login
    setToken(null); localStorage.removeItem('DATAGOL_USER')
    try { sessionStorage.setItem('DATAGOL_SESSION_EXPIRED', '1') } catch {}
    window.location.assign(import.meta.env.BASE_URL ?? '/')
    throw new Error('Session expired')
  }
  if (!res.ok) throw new Error(`${path} → HTTP ${res.status}: ${await res.text().catch(() => '')}`)
  return res.json() as Promise<T>
}
```

### `client/src/lib/use-auth.ts` — session hook

One `useAuth()` call in `App.tsx`; pass `setSession`/`signOut` down. (Calling it
in a child creates an independent state instance, so the gate never flips.)

```ts
import { useState } from 'react'
import { api, setToken } from './api'
export interface DatagolUser { email: string; [k: string]: unknown }

export function useAuth() {
  const [session, setS] = useState<{ token: string; user: DatagolUser | null } | null>(() => {
    const token = localStorage.getItem('DATAGOL_USER_TOKEN')
    if (!token) return null
    try { return { token, user: JSON.parse(localStorage.getItem('DATAGOL_USER') || 'null') } } catch { return { token, user: null } }
  })
  const setSession = (s: { token: string; user: DatagolUser | null } | null) => {
    setToken(s?.token ?? null)
    if (s?.user) localStorage.setItem('DATAGOL_USER', JSON.stringify(s.user)); else localStorage.removeItem('DATAGOL_USER')
    setS(s)
  }
  const login = async (email: string, password: string) => {
    const r = await api<{ token: string; user: DatagolUser | null }>('/auth/login', { method: 'POST', body: JSON.stringify({ email, password }) })
    setSession(r); return r
  }
  return { session, login, setSession, signOut: () => setSession(null) }
}
```

### Login form + gate

- `client/src/pages/Auth/Login.tsx`: email + password, calls `login()`. **No
  sign-up link, no "Create account", no "Forgot password"** — internal users are
  provisioned in DataGOL out-of-band. Show a one-time "Your session expired"
  notice if `sessionStorage['DATAGOL_SESSION_EXPIRED']` is set, then clear it.
- `App.tsx`: one `useAuth()`; `if (!session?.token) return <Login …/>` else the app.

```tsx
export default function App() {
  const { session, login, signOut } = useAuth()
  if (!session?.token) return <Login onLogin={login} />
  return <AppShell user={session.user} signOut={signOut} />
}
```

## Hard rules

1. **Never store the password** — only the token survives the login call.
2. **Capture the token from the `Authorization` RESPONSE header server-side**, not
   the JSON body. Check both casings; strip a leading `Bearer `.
3. **No service token in Mode 3.** A grep for `DATAGOL_SERVICE_TOKEN` /
   `x-auth-token` must return nothing.
4. **Browser never calls `*.datagol.ai`.** Login and all data go through `/api/*`;
   the server forwards the bearer. A grep for `datagol.ai` in `client/` must be empty.
5. **No sign-up / create-account / forgot-password** paths in the app.
6. **No refresh-token / silent renewal** — let the token expire, redirect to login.
7. **`x-appid` on every server→DataGOL call** (the login call and `dgFetch`).
8. **On 401**, clear the token client-side and hard-reload to `import.meta.env.BASE_URL`;
   show the one-time session-expired notice on the login page.
9. **Login host = `DATAGOL_BASE_URL`** (server env, injected by the cli-server). Never
   hard-code `be.datagol.ai` / `testing-gcp-be.datagol.ai`.

## Anti-patterns

- ❌ Reading the token from `await res.json()` (body = user object; token is the header).
- ❌ Storing `"Bearer eyJ…"` with the prefix — strip it.
- ❌ Calling DataGOL (login or data) from the browser — it goes through `/api/*`.
- ❌ A service token / Users workbook in a Mode 3 app.
- ❌ Sign-up / forgot-password links; refresh-token flows.
- ❌ Token in a URL query string, UI, debug panel, or analytics event.
- ❌ `useAuth()` in multiple components (independent state → gate never flips).

## Cross-references

- **`datagol-app-development` / `datagol-app-backend`** — single-port architecture +
  the server `dgFetch` / `/api/<resource>` route pattern this builds on.
- **`datagol-app-frontend`** — the `client/src/lib/api.ts` wrapper this extends.
- **`datagol-app-auth`** — its §Environment / §`x-appid` / base-URL rules apply to
  Mode 3 too; its service-token provisioning does NOT.
- **`datagol-interview`** — §3 #3 triggers this skill.
- **`datagol-user-auth`** — the Mode 2 alternative (app-owned users + app bearer).
  An app is Mode 2 or Mode 3, never both.
- **`datagol-detailed-plan`** — §1 must say "Mode 3"; the login page goes first in §6.
