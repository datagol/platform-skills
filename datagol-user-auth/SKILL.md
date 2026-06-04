---
name: datagol-user-auth
description: How to add username/password authentication to a single-port DataGOL app (Vite client + Express server), backed by DataGOL Users + Roles workbooks. **This is the Mode 2 (B2B/B2C) auth path** — the app owns its own user identity store, separate from DataGOL's user identities. Triggered when the user picks "B2B / B2C app — the app owns its own user table" in the interview's §3. The Express server verifies credentials and issues an APP bearer token (JWT); the client sends that bearer on every /api/* call; the server still uses the DataGOL SERVICE token to reach DataGOL. For internal apps where users sign in with their own DataGOL credentials, see datagol-internal-app-auth (Mode 3) instead.
---

# App-Level Authentication (workbook-backed users)

> **Runtime skill — DataGOL access is server-side.** Read the parent
> `datagol-app-development` (single-port architecture) and `datagol-app-backend`
> (Express serving + the server `dgFetch`) first. Everything here lives in the
> app's **`server/`** and **`client/`** — the browser never calls `*.datagol.ai`
> and never holds the service token.

## Two layers of auth — keep them separate

Mode 2 has **two independent credentials**. Conflating them is the #1 mistake.

| Layer | Credential | Who issues / holds it | Sent on |
|---|---|---|---|
| **A. App session** (who the end user is) | **app bearer token** (a JWT the app signs) | issued by `POST /api/auth/login`; held by the browser | every `client → /api/*` request (`Authorization: Bearer <appToken>`) |
| **B. DataGOL access** (how the app reads/writes data) | **DataGOL service token** | server env only (`DATAGOL_SERVICE_TOKEN`) | every `server → DataGOL` request (`x-auth-token`), never leaves the server |

So: the user logs in → gets an **app** token → sends it on each request → the
Express server validates it → and then uses the **service** token to talk to
DataGOL on that user's behalf. The two tokens never mix and the service token
never reaches the browser.

```
browser                         Express server (server/)              DataGOL
  POST /api/auth/login  ───────► verify pw vs Users workbook ──x-auth-token──► Users wb
  {username,password}            (service token, server-side)
        ◄── { token: <appJWT> } ─ sign app JWT (APP_JWT_SECRET)
  Authorization: Bearer <appJWT>
  GET /api/tasks  ─────────────► requireAppAuth → req.user ──x-auth-token──► Tasks wb
                                  scope whereClause to req.user.username      (Owner = user)
```

## When to Trigger

- Interview §3 answer **#2 — B2B / B2C app, the app owns its own user table**.
- The user asks for "sign-up", "user accounts", "their own user database",
  "roles / permissions", "per-user data".
- **Do NOT use for Mode 3** (internal / DataGOL login) — that has no sign-up, no
  Users/Roles workbooks; use `datagol-internal-app-auth`. An app is Mode 2 **or**
  Mode 3, never both.

## Security posture

Because verification, session validation, and row-scoping all run **server-side**
now, this is materially stronger than a browser-only scheme:

- Password hashes live in the Users workbook, readable **only** by the server
  (service token never ships to the browser).
- Sessions are signed JWTs the server validates — a user can't forge login state
  by editing `localStorage`.
- Row-scoping is enforced in the API from the validated token — the client
  can't widen it.

It is still **app-grade auth**, not a managed IdP: no email verification, no
password reset, no MFA, no built-in rate-limiting beyond what you add. For PII /
financial / regulated data, recommend Auth0 / Clerk / Supabase. State this to the
user up front.

## Data model — Users + Roles workbooks

Create both alongside the app's other workbooks (plan §3 Data Model; they come
**first** in §6 Implementation Order). Use `datagol_create_workbook` /
`datagol-workbook-design`.

**`Users`**

| Column | Type | Notes |
|---|---|---|
| username | SINGLE_LINE_TEXT | unique by convention; lowercase |
| password_hash | LONG_TEXT | hex scrypt/PBKDF2 output (server-computed) |
| salt | LONG_TEXT | hex random 16 bytes |
| role | SINGLE_LINE_TEXT | role name; must match a row in `Roles` |
| email | EMAIL | optional |
| created_at | DATE | set on signup |
| last_login_at | DATE | set on login |

**`Roles`** (the role table)

| Column | Type | Notes |
|---|---|---|
| name | SINGLE_LINE_TEXT | `admin` / `editor` / `user` / `viewer` (extend as needed) |
| permissions | LONG_TEXT | optional CSV/JSON of capabilities the server enforces |
| description | LONG_TEXT | optional |

Seed `Roles` with the role set at scaffold time. The server validates a signup's
requested role against `Roles`; `role` on `Users` references a `Roles.name`.

## Server — auth lives in `server/`

### `server/src/lib/auth-crypto.ts` — hashing (Node crypto, server-only)

```ts
import { randomBytes, scryptSync, timingSafeEqual } from 'crypto'

export function generateSalt(): string { return randomBytes(16).toString('hex') }

export function hashPassword(password: string, saltHex: string): string {
  return scryptSync(password, Buffer.from(saltHex, 'hex'), 32).toString('hex')
}

export function verifyPassword(password: string, saltHex: string, expectedHex: string): boolean {
  const got = Buffer.from(hashPassword(password, saltHex), 'hex')
  const exp = Buffer.from(expectedHex, 'hex')
  return got.length === exp.length && timingSafeEqual(got, exp)
}
```

> The browser never hashes and never sees a hash. Hashing is server-side now that
> a server exists.

### `server/src/routes/auth.ts` — signup / login / app JWT

Uses the server `dgFetch` (service token + `x-appid`, from `datagol-app-backend`)
to read/write the Users workbook, and signs an app JWT with `APP_JWT_SECRET`.

```ts
import { Router } from 'express'
import jwt from 'jsonwebtoken'
import { dgFetch, getWorkspaceId } from '../lib/datagol.js'  // dgFetch + workspace resolved from the service token
import { generateSalt, hashPassword, verifyPassword } from '../lib/auth-crypto.js'

const USERS_TABLE = process.env.USERS_TABLE_ID!
const ROLES_TABLE = process.env.ROLES_TABLE_ID!
const APP_JWT_SECRET = process.env.APP_JWT_SECRET!     // server env; NEVER in the client
// Workspace id is resolved from the service token (scoped to one workspace),
// not read from env — use `await getWorkspaceId()` at each call site.

export const auth = Router()

async function findUser(username: string) {
  const safe = username.replace(/'/g, "''")           // single-quoted SQL literal; escape quotes
  const data = await dgFetch<{ rows?: any[] }>(
    `/noCo/api/v2/workspaces/${await getWorkspaceId()}/tables/${USERS_TABLE}/cursor`,
    { method: 'POST', body: JSON.stringify({ whereClause: `\`username\` = '${safe}'`, requestPageDetails: { pageNumber: 1, pageSize: 1 } }) },
  )
  const raw = (data.rows ?? [])[0]
  return raw ? { id: raw.id, ...raw.cellValues } : null
}

auth.post('/signup', async (req, res) => {
  try {
    const username = String(req.body.username ?? '').trim().toLowerCase()
    const password = String(req.body.password ?? '')
    const role = String(req.body.role ?? 'user')
    if (!username || password.length < 8) return res.status(400).json({ error: 'Username and 8+ char password required' })
    if (await findUser(username)) return res.status(409).json({ error: 'Username already taken' })
    // (optional) validate role against the Roles table here.
    const salt = generateSalt()
    const password_hash = hashPassword(password, salt)
    const row = await dgFetch<any>(
      `/noCo/api/v2/workspaces/${await getWorkspaceId()}/tables/${USERS_TABLE}/rows`,
      { method: 'POST', body: JSON.stringify({ cellValues: { username, password_hash, salt, role, created_at: new Date().toISOString() } }) },
    )
    const user = { id: row.id, username, role }
    res.status(201).json({ token: signApp(user), user })
  } catch { res.status(500).json({ error: 'Signup failed' }) }
})

auth.post('/login', async (req, res) => {
  try {
    const username = String(req.body.username ?? '').trim().toLowerCase()
    const password = String(req.body.password ?? '')
    const u = await findUser(username)
    // Same generic error whether the user is missing or the password is wrong.
    if (!u || !verifyPassword(password, u.salt, u.password_hash)) {
      return res.status(401).json({ error: 'Invalid username or password' })
    }
    const user = { id: u.id, username: u.username, role: u.role }
    res.json({ token: signApp(user), user })
  } catch { res.status(500).json({ error: 'Login failed' }) }
})

function signApp(user: { id: string; username: string; role: string }): string {
  return jwt.sign(user, APP_JWT_SECRET, { expiresIn: '7d' })
}
```

### `server/src/lib/require-app-auth.ts` — validate the app bearer

```ts
import jwt from 'jsonwebtoken'
import type { Request, Response, NextFunction } from 'express'
const APP_JWT_SECRET = process.env.APP_JWT_SECRET!

export interface AppUser { id: string; username: string; role: string }

export function requireAppAuth(req: Request & { user?: AppUser }, res: Response, next: NextFunction) {
  const h = req.headers.authorization
  if (!h?.startsWith('Bearer ')) return res.status(401).json({ error: 'Not authenticated' })
  try { req.user = jwt.verify(h.slice(7), APP_JWT_SECRET) as AppUser; next() }
  catch { res.status(401).json({ error: 'Session expired' }) }
}
```

Mount in `server/src/index.ts`: `app.use('/api/auth', auth)` (public), and put
`requireAppAuth` on every data router (`app.use('/api/tasks', requireAppAuth, tasks)`).

### Per-user scoping + roles — enforced in the API

Data routes derive the owner from the **validated token**, not the request body:

```ts
// inside a data route, after requireAppAuth:
const owner = (req.user!.username).replace(/'/g, "''")
const where = req.user!.role === 'admin' ? undefined : `\`Owner\` = '${owner}'`
// list → pass `where`; create → force cellValues.Owner = req.user!.username
```

Role checks gate destructive endpoints **server-side** (the source of truth);
the client may mirror them for UX but the server enforces.

### Env

Add to `server/.env` (+ `.env.example`), never the client bundle:
`APP_JWT_SECRET`, `USERS_TABLE_ID`, `ROLES_TABLE_ID` (plus the `DATAGOL_*` from
`datagol-app-auth`/`datagol-app-backend`).

## Client — talks only to `/api/*`

### `client/src/lib/api.ts` — attach the app bearer

Extend the base `api()` wrapper (from `datagol-app-frontend`) to send the stored
app token on every call and to clear + redirect on 401:

```ts
const TOKEN_KEY = 'APP_TOKEN'
export function getToken() { return localStorage.getItem(TOKEN_KEY) }
export function setToken(t: string | null) { t ? localStorage.setItem(TOKEN_KEY, t) : localStorage.removeItem(TOKEN_KEY) }

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const token = getToken()
  const res = await fetch(import.meta.env.BASE_URL + 'api' + path, {
    ...init,
    headers: { 'Content-Type': 'application/json', ...(token ? { Authorization: `Bearer ${token}` } : {}), ...(init?.headers ?? {}) },
  })
  if (res.status === 401) { setToken(null); throw new Error('Unauthorized') }
  if (!res.ok) throw new Error(`${path} → HTTP ${res.status}: ${await res.text().catch(() => '')}`)
  return res.json() as Promise<T>
}
```

### `client/src/lib/use-auth.ts` — session hook (token + user)

```ts
import { useState, useCallback } from 'react'
import { api, setToken } from './api'
export interface AppUser { id: string; username: string; role: string }

export function useAuth() {
  const [user, setUser] = useState<AppUser | null>(() => {
    try { return JSON.parse(localStorage.getItem('APP_USER') || 'null') } catch { return null }
  })
  const setSession = useCallback((token: string | null, u: AppUser | null) => {
    setToken(token)
    if (u) localStorage.setItem('APP_USER', JSON.stringify(u)); else localStorage.removeItem('APP_USER')
    setUser(u)
  }, [])
  const login = useCallback(async (username: string, password: string) => {
    const { token, user } = await api<{ token: string; user: AppUser }>('/auth/login', { method: 'POST', body: JSON.stringify({ username, password }) })
    setSession(token, user); return user
  }, [setSession])
  const signup = useCallback(async (username: string, password: string) => {
    const { token, user } = await api<{ token: string; user: AppUser }>('/auth/signup', { method: 'POST', body: JSON.stringify({ username, password }) })
    setSession(token, user); return user
  }, [setSession])
  return { user, login, signup, signOut: () => setSession(null, null) }
}
```

### Screens + gate

- `client/src/pages/Auth/SignIn.tsx` + `SignUp.tsx` POST via `login()`/`signup()`.
- **Call `useAuth()` ONCE in `App.tsx`** and pass `login`/`signup`/`setSession`
  down as props. Calling `useAuth()` inside `SignIn`/`SignUp` creates an
  independent state instance, so the gate never flips. (Same trap as before;
  still applies.)
- After **signup**, show a brief success screen ("Account created! Signing you
  in…", ~1.8s) before routing in — never flip silently.
- `App.tsx` gate: `if (!user) return <SignIn …/SignUp …/>` else the app.

```tsx
// App.tsx — single useAuth, props down
export default function App() {
  const { user, login, signup, signOut } = useAuth()
  const [view, setView] = useState<'signin'|'signup'>('signin')
  if (!user) return view === 'signin'
    ? <SignIn onLogin={login} switchToSignUp={() => setView('signup')} />
    : <SignUp onSignup={signup} switchToSignIn={() => setView('signin')} />
  return <Home user={user} onSignOut={signOut} />
}
```

Role-gated UI mirrors the server check for UX only:
```tsx
const canDelete = user?.role === 'admin' || user?.role === 'editor'  // server still enforces
```

| Role | Read | Create | Update own | Update others | Delete |
|---|---|---|---|---|---|
| admin | ✓ | ✓ | ✓ | ✓ | ✓ |
| editor | ✓ | ✓ | ✓ | | ✓ (own) |
| user | ✓ | ✓ | ✓ (own) | | |
| viewer | ✓ | | | | |

## Hard rules

1. **Never store or log plaintext passwords.** Hash server-side with a per-user
   random salt; never send a hash to the browser.
2. **App token ≠ service token.** The app JWT authenticates the *user to the app*;
   the service token authenticates the *app to DataGOL*. The service token stays
   in `server/` env. A grep for `DATAGOL_SERVICE_TOKEN` in `client/` must be empty.
3. **Validate the app bearer on every data route** (`requireAppAuth`) and derive
   identity from the token, never from request body fields.
4. **Enforce row-scoping + role checks server-side.** Client checks are UX only.
5. **`whereClause` uses single-quoted SQL literals**, escape embedded quotes
   (`username.replace(/'/g, "''")`). Never `JSON.stringify` a value into it.
6. **Return a generic `Invalid username or password`** for both missing-user and
   wrong-password; never leak a raw DataGOL error to the client.
7. **`APP_JWT_SECRET` is required server env**; rotate per deployment. No default.
8. **Auth scaffold first** in the implementation order — every data route assumes
   `requireAppAuth`.
9. **Tell the user** this is app-grade auth; recommend a managed IdP for
   sensitive data.

## Anti-patterns

❌ Hashing in the browser / shipping the service token to the client (old model).
❌ MD5 / SHA-1 / unsalted hashes.
❌ Trusting `req.body.username`/`role` for identity instead of the validated token.
❌ Enforcing roles/scoping only in the client.
❌ `JSON.stringify(value)` in a `whereClause` (double-quotes = column identifier → HTTP 400).
❌ Calling `useAuth()` inside `SignIn`/`SignUp` (independent state → gate never flips).
❌ Silent post-signup redirect with no confirmation screen.

## Cross-references

- **`datagol-interview`** — §3 #2 (B2B/B2C) triggers this skill.
- **`datagol-app-development`** / **`datagol-app-backend`** — single-port
  architecture + the server `dgFetch` and `/api/<resource>` route pattern the
  auth routes build on.
- **`datagol-app-frontend`** — the `client/src/lib/api.ts` wrapper this extends.
- **`datagol-app-auth`** — provisions the **service token** + workspace grants the
  server uses to read/write the Users/Roles workbooks. Runs alongside Mode 2.
- **`datagol-internal-app-auth`** — the Mode 3 alternative (DataGOL login). An app
  is Mode 2 or Mode 3, never both.
- **`datagol-detailed-plan`** — §1 must state "Mode 2"; §3 must include the Users
  **and** Roles tables; §6 must put the auth scaffold (server routes + client
  screens) first.
- **`datagol-workbook-design` / `datagol-workbook-operations`** — Users/Roles
  schema + the cursor/rows API the server calls. Add an `Owner` column on every
  user-scoped workbook.
