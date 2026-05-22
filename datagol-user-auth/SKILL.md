---
name: datagol-user-auth
description: How to add username/password authentication to a generated app, backed by a DataGOL Users workbook. Triggered when the user picks "multi-user with login" in the interview's mandatory user-management question, or otherwise asks for sign-up + sign-in. Passwords are hashed in the browser via PBKDF2 (Web Crypto API). Demo-grade auth — secure enough for prototypes/internal tools, NOT for anything sensitive.
---

# App-Level Authentication (workbook-backed users)

When the user wants their generated app to have logins, this skill describes
the full pattern: how to model the user store as a DataGOL workbook, hash
passwords client-side, scaffold sign-in/sign-up screens, manage the session,
and scope per-user data. There's no app server — everything runs in the
browser, against the DataGOL REST API.

## When to Trigger

- The interview's mandatory **user management** question got answer
  *"Yes, multi-user with login"*.
- The user explicitly asks for "logins", "sign-up", "user accounts",
  "roles / permissions", "per-user data".

## Honest security assessment (read this first)

This pattern uses a DataGOL workbook as the user database. Because there's
no app server, **all comparison logic runs in the browser**. That means:

- **The DataGOL bearer token is shared** by the entire app deployment. Every
  app user's browser holds the same token and can read all rows in the
  Users workbook (including `password_hash` and `salt`). Make the
  workbook private; assume it's leaked anyway.
- **A malicious user can forge a logged-in state** by editing localStorage,
  since there's no server validating sessions. Don't ship anything sensitive
  this way.
- **Brute-force / credential stuffing isn't rate-limited** at the app level
  (only at DataGOL's API level).
- **No password reset / email verification / 2FA** out of the box. Add as
  separate skills or recommend a real auth provider.

This is appropriate for: internal tools, prototypes, demos, "we want
multiple people to keep their own data separate without a real auth
system". It is **not** appropriate for: anything with PII, financial data,
externally-facing apps, regulated industries.

When the user wants real auth, point them at Supabase / Clerk / Auth0 — out
of scope for this skill.

## Architecture

```
┌──────────── browser ────────────┐
│                                 │
│  Sign-in form                   │
│      │                          │
│      ▼                          │
│  PBKDF2 hash (client-side)      │
│      │                          │
│      ▼                          │
│  DataGOL REST  ─── shared bearer ──► Users workbook
│  (look up user, compare hash)         (username, password_hash, salt, role)
│      │                          │
│      ▼                          │
│  localStorage.APP_USER          │
│      │                          │
│      ▼                          │
│  Other app pages                │
│  (filter rows by Owner = app username)
└─────────────────────────────────┘
```

- **DataGOL bearer:** one shared token, pasted by the deployer once on first
  load (the existing chat-ui-style paste UI). All app users use this token
  to talk to DataGOL.
- **App identity:** stored in the **Users workbook**. Username + hashed
  password + role. Has nothing to do with the underlying DataGOL identity.
- **Data isolation:** every user-scoped workbook gets an `Owner` column
  (SINGLE_LINE_TEXT, holds the app username). Every API call filters
  `whereClause: \`\`Owner\`\` = '<currentUser>'`.

## Step 1 — Users workbook

Create alongside the rest of the app's workbooks. Recommend these columns:

| Column          | Type             | Required | Notes |
|-----------------|------------------|----------|-------|
| username        | SINGLE_LINE_TEXT | ✓        | Unique by convention. Lowercase recommended. |
| password_hash   | SINGLE_LINE_TEXT | ✓        | hex-encoded PBKDF2-SHA256 output, 256 bits |
| salt            | SINGLE_LINE_TEXT | ✓        | hex-encoded random 16 bytes |
| role            | SINGLE_LINE_TEXT | ✓        | one of: `admin`, `editor`, `viewer`, `user` (document the list in the app's README) |
| email           | EMAIL            |          | optional; used for password-reset later if added |
| created_at      | DATE             |          | auto-fill on sign-up |
| last_login_at   | DATE             |          | auto-update on sign-in |

The plan's §3 (Data Model) must include this table. It must come **first** in
the §6 Implementation Order so the auth scaffold has somewhere to write to.

## Step 2 — Hashing helper (`src/lib/auth-crypto.ts`)

Use the browser's built-in Web Crypto API — no dependencies. PBKDF2-SHA256
with 100k iterations and a per-user 16-byte salt is the sweet spot.

```ts
// src/lib/auth-crypto.ts
const ITERATIONS = 100_000;
const KEY_LEN_BITS = 256;

export function generateSalt(): string {
  const bytes = crypto.getRandomValues(new Uint8Array(16));
  return bytesToHex(bytes);
}

export async function hashPassword(password: string, saltHex: string): Promise<string> {
  const salt = hexToBytes(saltHex);
  const enc = new TextEncoder();
  const baseKey = await crypto.subtle.importKey(
    'raw', enc.encode(password), 'PBKDF2', false, ['deriveBits'],
  );
  const bits = await crypto.subtle.deriveBits(
    { name: 'PBKDF2', salt, iterations: ITERATIONS, hash: 'SHA-256' },
    baseKey,
    KEY_LEN_BITS,
  );
  return bytesToHex(new Uint8Array(bits));
}

/** Constant-time compare — equal-length strings only. */
export function constantTimeEqual(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  let diff = 0;
  for (let i = 0; i < a.length; i++) diff |= a.charCodeAt(i) ^ b.charCodeAt(i);
  return diff === 0;
}

function bytesToHex(b: Uint8Array): string {
  return Array.from(b, x => x.toString(16).padStart(2, '0')).join('');
}
function hexToBytes(h: string): Uint8Array {
  const out = new Uint8Array(h.length / 2);
  for (let i = 0; i < h.length; i += 2) out[i/2] = parseInt(h.slice(i, i+2), 16);
  return out;
}
```

## Step 3 — Auth API helpers (`src/lib/auth.ts`)

Wraps DataGOL row queries / inserts to the Users workbook.

```ts
// src/lib/auth.ts
import { listItems, createItem, updateItem } from './api';
import { generateSalt, hashPassword, constantTimeEqual } from './auth-crypto';

const USERS_TABLE_ID = '<paste-users-workbook-id>';

export interface AppUser {
  id: string;          // workbook row id
  username: string;
  role: 'admin' | 'editor' | 'viewer' | 'user';
}

export async function signUp(username: string, password: string, role: AppUser['role'] = 'user'): Promise<AppUser> {
  username = username.trim().toLowerCase();
  if (!username || !password) throw new Error('Username and password required');
  if (password.length < 8)   throw new Error('Password must be at least 8 characters');

  // Reject duplicate username.
  const existing = await findUserByUsername(username);
  if (existing) throw new Error('Username already taken');

  const salt = generateSalt();
  const password_hash = await hashPassword(password, salt);

  const row = await createItem(USERS_TABLE_ID, {
    username,
    password_hash,
    salt,
    role,
    created_at: new Date().toISOString(),
  });
  return { id: row.id, username, role };
}

export async function signIn(username: string, password: string): Promise<AppUser> {
  username = username.trim().toLowerCase();
  // Guard blank inputs before touching the API.
  if (!username || !password) throw new Error('Please enter your username and password');
  const row = await findUserByUsername(username);
  if (!row) throw new Error('Invalid username or password');

  const hash = await hashPassword(password, row.salt);
  if (!constantTimeEqual(hash, row.password_hash)) {
    throw new Error('Invalid username or password');
  }
  await updateItem(USERS_TABLE_ID, row.id, { last_login_at: new Date().toISOString() })
    .catch(() => { /* best-effort */ });
  return { id: row.id, username: row.username, role: row.role };
}

interface UserRow {
  id: string;
  username: string;
  password_hash: string;
  salt: string;
  role: AppUser['role'];
}

async function findUserByUsername(username: string): Promise<UserRow | null> {
  // ⚠️  MUST use single-quoted SQL string literals.
  // JSON.stringify produces double-quoted strings ("alice") which the DataGOL
  // SQL engine treats as column identifiers, causing HTTP 400
  // "column \"alice\" does not exist".  Single quotes ('alice') are the correct
  // SQL string literal syntax.
  //
  // ⚠️  MUST wrap in try/catch.
  // Any API error (malformed query, network hiccup, etc.) must be caught here
  // and returned as null so the caller shows a friendly "Invalid username or
  // password" message instead of leaking a raw DataGOL error to the UI.
  try {
    const safe = username.replace(/'/g, "''"); // escape single quotes
    const rows = await listItems(USERS_TABLE_ID, {
      whereClause: `\`username\` = '${safe}'`,
      pageSize: 1,
    });
    return (rows[0] ?? null) as UserRow | null;
  } catch {
    return null;
  }
}
```

## Step 4 — Session storage + auth hook (`src/lib/use-auth.ts`)

Tiny hook over `localStorage`. Don't bring in Zustand for one piece of
state.

```ts
// src/lib/use-auth.ts
import { useState, useEffect, useCallback } from 'react';
import type { AppUser } from './auth';

const KEY = 'APP_USER';

function read(): AppUser | null {
  try {
    const raw = localStorage.getItem(KEY);
    return raw ? JSON.parse(raw) as AppUser : null;
  } catch { return null; }
}

export function useAuth() {
  const [user, setUser] = useState<AppUser | null>(read);

  useEffect(() => {
    function onStorage(e: StorageEvent) {
      if (e.key === KEY) setUser(read());
    }
    window.addEventListener('storage', onStorage);
    return () => window.removeEventListener('storage', onStorage);
  }, []);

  const setSession = useCallback((u: AppUser | null) => {
    if (u) localStorage.setItem(KEY, JSON.stringify(u));
    else   localStorage.removeItem(KEY);
    setUser(u);
  }, []);

  return { user, setSession, signOut: () => setSession(null) };
}
```

## Step 5 — Sign-in / sign-up screens

Two pages: `src/pages/Auth/SignIn.tsx` and `src/pages/Auth/SignUp.tsx`.

> ⚠️ **MANDATORY: `setSession` must come from `App.tsx` and be passed down as a prop.**
> Do NOT call `useAuth()` inside `SignIn` or `SignUp`. Each `useAuth()` call
> creates an independent `useState` instance. If `SignIn`/`SignUp` call their
> own `useAuth()` and invoke `setSession`, they update *their own* local state —
> `App.tsx`'s `user` state never changes, the auth gate never flips, and the
> user appears to be silently stuck on the form. See Step 6 for the correct
> `App.tsx` wiring.

```tsx
// src/pages/Auth/SignIn.tsx
import { useState } from 'react';
import { signIn } from '../../lib/auth';
import type { AppUser } from '../../lib/auth';

interface Props {
  setSession: (u: AppUser | null) => void; // passed from App.tsx
  switchToSignUp: () => void;
}

export default function SignIn({ setSession, switchToSignUp }: Props) {
  const [username, setU] = useState('');
  const [password, setP] = useState('');
  const [busy, setBusy]  = useState(false);
  const [err, setErr]    = useState<string | null>(null);

  async function submit(e: React.FormEvent) {
    e.preventDefault();
    setBusy(true); setErr(null);
    try {
      const user = await signIn(username, password);
      setSession(user); // updates App.tsx state → auth gate flips
    } catch (e) {
      setErr(e instanceof Error ? e.message : String(e));
    } finally { setBusy(false); }
  }

  return (
    <form onSubmit={submit} style={card}>
      <h2>Sign in</h2>
      <input value={username} onChange={e => setU(e.target.value)}
             placeholder="Username" autoComplete="username" required />
      <input type="password" value={password} onChange={e => setP(e.target.value)}
             placeholder="Password" autoComplete="current-password" required />
      {err && <div style={errBox}>{err}</div>}
      <button disabled={busy} type="submit">{busy ? 'Signing in…' : 'Sign in'}</button>
    </form>
  );
}

const card: React.CSSProperties = { /* … */ };
const errBox: React.CSSProperties = { /* … */ };
```

`SignUp.tsx` mirrors the prop signature (`setSession`, `switchToSignIn`).
After `signUp()` succeeds, **show a brief success screen** before logging
the user in — never silently call `setSession` with no feedback:

```tsx
// src/pages/Auth/SignUp.tsx (success handling)
const [success, setSuccess] = useState<AppUser | null>(null);

async function submit(e: React.FormEvent) {
  // … validate …
  const user = await signUp(username, password);
  setSuccess(user);
  // Auto-login after 1.8 s so the user sees the confirmation.
  setTimeout(() => setSession(user), 1800);
}

// Render success screen before the form:
if (success) {
  return (
    <div style={page}>
      <div style={card}>
        <div style={successMark}>✓</div>
        <h2>Account created!</h2>
        <p>Welcome, <strong>{success.username}</strong>. Signing you in…</p>
        <button onClick={() => setSession(success)}>Skip waiting</button>
      </div>
    </div>
  );
}
```

Add a "confirm password" input on sign-up and validate that both fields
match before calling `signUp`.

## Step 6 — App.tsx auth gate

`App.tsx` is the **single place** where `useAuth()` is called. It owns the
`user` state and passes `setSession` as a prop to `SignIn`/`SignUp`.
This is the only pattern that makes the auth gate flip correctly on login.

```tsx
import { useState } from 'react';
import { useAuth } from './lib/use-auth';
import SignIn from './pages/Auth/SignIn';
import SignUp from './pages/Auth/SignUp';
import Home from './pages/Home';

export default function App() {
  // Call useAuth ONCE here. setSession is passed down as a prop so that
  // SignIn/SignUp update THIS instance's state and trigger a re-render.
  const { user, setSession, signOut } = useAuth();
  const [view, setView] = useState<'signin' | 'signup'>('signin');

  if (!user) {
    return view === 'signin'
      ? <SignIn setSession={setSession} switchToSignUp={() => setView('signup')} />
      : <SignUp setSession={setSession} switchToSignIn={() => setView('signin')} />;
  }

  return <Home user={user} onSignOut={signOut} />;
}
```

## Step 7 — Per-user data scoping

Every user-scoped workbook (Tasks, Notes, etc.) gets an `Owner` column
(SINGLE_LINE_TEXT, not LINK). Two changes to the rest of the app:

1. **On insert:** set `Owner: currentUser.username` in `cellValues`.
2. **On query:** add `whereClause: \`\`Owner\`\` = '<currentUser>'` (or
   skip the filter for `admin`-role users).

Show how this looks for one entity in the plan's §3, and note the rule
applies uniformly.

## Step 8 — Role-gated UI

Read `user.role` from the hook and gate destructive UI:

```tsx
const { user } = useAuth();
const canDelete = user?.role === 'admin' || user?.role === 'editor';
{canDelete && <button onClick={onDelete}>Delete</button>}
```

For the role values, document the list in the app's README so users know
who can do what. A simple matrix:

| Role    | Read | Create | Update own | Update others | Delete |
|---------|------|--------|------------|---------------|--------|
| admin   | ✓    | ✓      | ✓          | ✓             | ✓      |
| editor  | ✓    | ✓      | ✓          |               | ✓ (own)|
| user    | ✓    | ✓      | ✓ (own)    |               |        |
| viewer  | ✓    |        |            |               |        |

Enforce client-side; a hostile client can skip these checks (see security
caveats above).

## Hard rules

1. **Never store plaintext passwords.** Anywhere — not in the workbook, not
   in localStorage, not in error messages, not in logs.
2. **Always hash with a per-user random salt.** Don't reuse a global salt.
3. **PBKDF2-SHA256, 100k+ iterations.** Don't drop iterations to "make
   sign-in faster"; the cost is the security.
4. **Use constant-time compare** when checking the hash. The
   `constantTimeEqual` helper is in `auth-crypto.ts`.
5. **Don't echo the password in any UI.** No "your password is …" — show
   "your password meets the requirements" if you need feedback.
6. **Keep the Users workbook private.** It contains hashes; keep the
   blast radius small.
7. **Always include the auth scaffold first** in the plan's
   Implementation Order — every other page assumes `useAuth` returns a
   valid user.
8. **Tell the user up-front** that this is demo-grade auth and not suitable
   for sensitive data. Recommend a real auth provider when warranted.

## Anti-patterns

❌ **MD5 / SHA-1 / unsalted SHA-256.** All trivially crackable for common
passwords.

❌ **Storing the password "encrypted" with a constant key.** That's
reversible — same problem.

❌ **Sending the password to a chat-bot endpoint.** Don't. The hashing
must happen in the browser before the request.

❌ **Skipping the duplicate-username check on sign-up.** You'll end up with
multiple rows for the same username and the sign-in lookup gets ambiguous.

❌ **Using `email` as username in the workbook lookup column.** Allowed
but the `username` column should be the canonical identifier; stick to one.

❌ **Using `JSON.stringify(username)` in the `whereClause`.** `JSON.stringify`
produces a double-quoted string (`"alice"`). The DataGOL SQL engine treats
double-quoted identifiers as **column names**, not values — resulting in
`HTTP 400: column "alice" does not exist`. Always use single-quoted SQL string
literals and escape any embedded single quotes by doubling them:
```ts
const safe = username.replace(/'/g, "''");
whereClause: `\`username\` = '${safe}'`
```

❌ **Letting raw API errors surface in the UI.** `findUserByUsername` must
always wrap its `listItems` call in `try/catch` and return `null` on any
error. Never let a raw `HTTP 400: column … does not exist` (or any other
DataGOL error) reach the sign-in form. The UI should only ever show
`'Invalid username or password'`.

❌ **Calling `useAuth()` inside `SignIn` or `SignUp`.** Each `useAuth()` call
creates its own independent `useState`. Calling `setSession` on a child
component's hook instance updates that component's local state only —
`App.tsx`'s `user` never changes and the auth gate never flips. Always call
`useAuth()` once in `App.tsx` and pass `setSession` down as a prop.

❌ **Calling `setSession` silently after sign-up with no feedback.** The user
has no idea the account was created. Always show a success screen ("Account
created! Signing you in…") with a short delay (e.g. `setTimeout 1800ms`)
before calling `setSession`. Include a "Skip waiting" button for impatient users.

❌ **Calling the Users workbook from chat-app code.** The chat app uses
the streaming agent endpoint, not workbook CRUD; it has its own auth model
(see `datagol-agent-chat-ui`).

## Cross-references

- **`datagol-interview`** — the user-management answer here (multi-user with login)
  triggers this skill.
- **`datagol-detailed-plan`** — Section 1 must restate "auth model: workbook-backed
  app accounts"; Section 3 must include the Users table; Section 6 must put
  the Users workbook + auth files first in the implementation order.
- **`datagol-workbook-design`** — for the rest of the app's data model. Remember to
  add an `Owner` column on every user-scoped workbook.
- **`datagol-workbook-operations`** — `signIn` / `signUp` use `POST /rows` and the
  `cursor` query. Read its API spec.
- **`datagol-ui-generation`** — `auth-crypto.ts`, `auth.ts`, `use-auth.ts`,
  `pages/Auth/{SignIn,SignUp}.tsx`, and the App.tsx gate are the new files
  to add to that skill's standard scaffold.
