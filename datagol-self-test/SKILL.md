---
name: datagol-self-test
description: After the agent finishes building (or substantially modifying) a generated app, run a two-tier verification pass — **Tier 1 sanity checks run automatically and silently** (TypeScript compile, preview health, per-workbook read probe); **Tier 2 UI tests run only if the user opts in** (Playwright drives each entity's CRUD cycle, captures network + console errors, agent auto-patches up to 3 attempts). Triggered automatically on `agent_done` after a build turn that wrote files OR created workbooks OR registered data sources, and on-demand when the user says "test the app", "verify it works", "check everything", "run the tests", etc.
---

# DataGOL Self-Test

The agent finishes a build, then verifies its own work. Two tiers — cheap stuff runs without asking; expensive stuff asks first.

> **Why this skill exists.** *"I built it"* is not the same as *"it works"*. Without verification, the agent has been known to: emit code that doesn't compile, hard-code workbook IDs that don't exist, send request bodies in the wrong shape (missing `cellValues` wrapper), call `PATCH /rows/:id` instead of `PUT /rows`. Each is invisible until the user opens the preview and gets a blank page or a network 4xx. This skill closes that loop.

## ⛔ COMPLETION GATE — read first, non-negotiable

**You may NOT tell the user the app is ready until the Tier 1 gate below has run
THIS turn and every check PASSED, and you have pasted the evidence block.** This
is a hard rule, not a suggestion.

- **Forbidden phrases unless the evidence block is present and all-green:**
  "it's ready", "the app is ready", "done", "all set", "you can use it now",
  "it works", "go ahead and try it", "I've built/finished the app", "complete",
  or any paraphrase that implies the app is usable. Saying any of these without a
  passed, shown Tier 1 block is a **failure of this skill.**
- **Evidence is mandatory.** Claims are not evidence. You must run the checks and
  **paste the actual results** (HTTP status codes / exit codes / the block
  below). "I verified it" without the block does not count.
- **Any check fails or cannot run → you must NOT claim ready.** Either auto-fix
  and re-run (per each check's protocol), or, if you can't, tell the user plainly
  **what failed** and that the app is **not** ready — never paper over it.
- **No skipping "to save time."** The gate is cheap (seconds). If the user
  explicitly said "skip tests", you may skip — but then you must say *"not
  verified this turn"*, NOT "ready".
- **It ran this turn.** A pass from an earlier turn doesn't count after you've
  written more files — re-run.

The required evidence block (Tier 1 output format) and the exact checks are
defined below. Produce that block before any completion statement.

## When to run

**Automatically** (Tier 1 only — silent unless something fails):

- After every `agent_done` on a turn that:
  - Wrote one or more files into `src/`, OR
  - Called `datagol_create_workbook` / `datagol_add_column` / `datagol_create_workspace`, OR
  - Registered a data source via `datagol-data-connector`.
- Run **once per turn**, not once per tool call. Wait for the agent's own work to settle before invoking.

**On-demand** (Tier 1 + offer Tier 2):

- User asks: *"test the app"*, *"verify it works"*, *"check everything"*, *"run the tests"*, *"is it working?"*.
- User reports a problem after a build (*"the create form doesn't work"*) — start Tier 1 to triage; offer Tier 2 if Tier 1 passes but the user's symptom isn't visible.

**Skip** when:

- The turn was pure exploration (no writes, no creates).
- The turn was a one-line content tweak the user explicitly confirmed (*"change the title to X"* — no need to ping DataGOL afterwards).
- The user said *"skip tests this turn"* / *"don't run tests"*. The flag persists for that build cycle; ask again on the next turn.

---

## ⚙️ Process model — the platform owns the server. NEVER hand-manage it.

The single-port app is **rebuilt and restarted for you by the platform** at the
end of every turn. You do not — and must not — start, stop, or build the server
yourself. Read this before you touch a terminal:

- **On `agent_done` (end of each turn), the cli-server automatically runs
  `npm install` + `npm run build` (client vite build + server tsc) and restarts
  the Node process on the SAME, stable preview port.** There is **no HMR** for
  single-port apps — the restart IS the reload.
- **The running preview reflects the PREVIOUS turn's build, not your
  uncommitted edits this turn.** Source edits you make now go live only after
  the turn ends and the auto-rebuild runs. This is expected — do not
  "work around" it.
- **⛔ NEVER run `node server/dist/index.js`, `npm run dev`, `npm start`, or
  `npm run build` followed by a manual launch, and NEVER `kill` the server
  process.** Hand-spawning creates an **orphan process (parent PID 1)** that
  races the platform's managed process, binds a stray port, and serves **stale
  code** — the #1 cause of phantom "404 / hangs / works-when-I-curl-it-directly
  but-not-in-preview" symptoms. If you see duplicate `server/dist/index.js`
  processes for one app dir, an earlier turn hand-spawned one; that's a bug to
  avoid, not to reproduce.
- **Do NOT diagnose transport from a stale/orphan process.** If a POST appears
  to "hang" or 404 through the preview, the cause is almost always (a) you're
  hitting the pre-rebuild process mid-turn, or (b) an orphan you spawned — NOT
  the proxy mishandling JSON bodies. **Never invent header-based / no-body
  login workarounds** to dodge a phantom "JSON POST hangs" problem; standard
  `POST /api/auth/login` with a JSON body is correct. Let the turn end, then
  re-probe the managed preview.
- **To verify live behavior, just probe the preview URL** (`$P/...` below). If
  it's `502`/`503`, the managed rebuild is still running or just kicked off —
  wait ~2s and retry. Do not "fix" a 502 by launching your own server.
- **The only thing you may run for verification is the read-only `npm run
  build` typecheck (Check 1) and `curl`/`node .datagol/ui-test.mjs` probes.**
  Never a long-lived `listen`.

## Tier 1 — the gate (single-port apps)

Run **every** check in sequence through the app's preview URL (the same path a
real user hits). Stop and self-heal on the first failure; re-run after a fix.
**Probe the running app via `/preview/<id>/…`, NOT DataGOL directly** — the whole
point is to exercise client → `/api/*` → Express → DataGOL end-to-end. Set
`P=${API_URL}/preview/<sessionId>` for the snippets below.

### Check 1: Build passes (client + server)

```bash
cd <sandbox-cwd> && npm run build 2>&1     # vite build (client) + tsc (server)
```

Pass: exit 0. Fail: any `error TS####` or vite build error. **Auto-fix:** read the
named file/line, apply a targeted fix (the message says exactly what's wrong),
rebuild; up to 3 attempts. A red build = NOT ready, full stop.

### Check 2: App server is up + UI loads

```bash
curl -s -o /dev/null -w "ui=%{http_code}\n" "$P/"            # built React index.html
curl -s -o /dev/null -w "health=%{http_code}\n" "$P/api/health"
```

Pass: both `200`. Fail: `502`/`503` = the Express server crashed or is still
building (wait 2s, retry once; if still down, read the `[app:…]` stderr and patch
the **source** cause, then let the auto-rebuild restart it — do NOT launch the
server yourself; see "Process model" above). **`/api/health` is NOT proof the app works** — it never touches
DataGOL. It only proves the server booted. Continue to Check 3.

### Check 3: A REAL data endpoint returns 200 (the check that actually matters)

For **each** `/api/<resource>` the app exposes (read `server/src/routes/`), call
the **read** endpoint through the preview:

```bash
curl -s -o /dev/null -w "%{http_code}" "$P/api/<resource>"   # e.g. /api/products
```

Pass: `200` with a JSON body. **Fail: `401`/`500`/`undefined`-in-URL → the app is
NOT ready**, even though `/api/health` was green. This is the single most
important check — it proves the server's DataGOL wiring (service token, workspace
id, table ids) is actually populated and valid.

**Auto-fix protocol — diagnose by status:**
- **`401` on `/api/<resource>` but `/api/health` is 200** → the server's
  **`DATAGOL_SERVICE_TOKEN` is missing/empty/wrong** in `server/.env`, OR a
  **user bearer (`DATAGOL_TOKEN`) was used where a service token is required**.
  The codex injects `DATAGOL_TOKEN` (a *user* bearer) — it is **not** a service
  token. You must **provision the service token and populate `server/.env`** per
  `datagol-app-auth` (Steps A/B/C + the "populate server/.env" step). Then rebuild
  (the after-turn rebuild picks up `.env`) and re-probe.
- **`undefined` in the request URL / `500` "workspace/table not found"** → a
  `*_TABLE_ID` is missing from `server/.env`. Get the real ids from
  `datagol_get_workspace_schema` and write them into `server/.env`. **Do NOT add
  `DATAGOL_WORKSPACE_ID`** — the workspace is resolved at runtime from the service
  token via `getWorkspaceId()` (see `datagol-app-backend`); a baked id is a bug,
  not a fix. If the resolver throws "resolves to no workspace", the service
  account grant is missing → that's the `403` case below (Step C).
- **`403`** → the service account lacks workspace access; re-run `datagol-app-auth`
  Step C (`bulkUsers` grant `CREATOR`).
- After any env change, confirm `server/.env` actually contains non-empty
  `DATAGOL_SERVICE_TOKEN`, `DATAGOL_APP_ID`, and every `*_TABLE_ID` — an empty
  `.env` (just `PORT`) is the #1 cause of this failure. (`DATAGOL_WORKSPACE_ID`
  is intentionally absent — resolved from the token, not baked.)

### Check 4: Auth round-trip (Mode 2 / Mode 3 apps only)

If the app has login (interview §3 = Mode 2 or 3), prove the gate works:

```bash
curl -s -o /dev/null -w "protected_no_auth=%{http_code}\n" "$P/api/<protected-resource>"   # expect 401
# happy-path login returns a token / sets a session:
curl -s -o /dev/null -w "login=%{http_code}\n" -X POST "$P/api/auth/login" \
  -H 'Content-Type: application/json' --data '{"username":"<seed-or-test-user>","password":"<pw>"}'
```

Pass: protected route **401 without auth**, login **200** (and an authed call to a
protected route returns 200). Fail → auth isn't wired; NOT ready. (If there's no
seeded test user yet, state that the login path is unverified and say so in the
block — don't claim auth works.)

### Tier 1 output format — the MANDATORY evidence block

Unlike before, this is **not silent on pass** — the block is the *proof* the
completion gate requires. Paste it with the **actual** status codes you observed
(not from memory — from the commands you just ran). Only after an all-green block
may you tell the user the app is ready.

All-green (you MAY now say "ready"):

```
🧪 Verification
✓ build           npm run build → exit 0
✓ server + ui     ui=200  health=200
✓ data endpoints  /api/products=200  /api/orders=200  /api/categories=200
✓ auth            protected_no_auth=401  login=200        (Mode 2/3 only)

App is ready. Open the preview.
```

After an auto-fix (show what broke + the fix, then the green re-run):

```
🧪 Verification
✓ build
✗ data endpoints  /api/products=401 (health was 200) → server .env had no
  DATAGOL_SERVICE_TOKEN. Provisioned a service account + token, granted CREATOR,
  wrote SERVICE_TOKEN + *_TABLE_ID into server/.env (workspace resolved from the
  token, not baked), rebuilt.
✓ data endpoints  /api/products=200  /api/orders=200  (retry)

App is ready. Open the preview.
```

Failure that couldn't be fixed (you MUST NOT say "ready"):

```
🧪 Verification
✓ build
✗ data endpoints  /api/products=401 after 3 attempts
    1. Populated server/.env with provisioned service token — still 401
    2. Re-granted CREATOR on the workspace — still 401
    3. Verified token against /noCo/api/v2/workspaces directly — 401 there too

The service token isn't authenticating against this backend. The app is **not
ready** — data calls fail. <next step / question for the user>
```

Note the second example is the **exact class of bug** that previously shipped as
"ready": `/api/health` green but `/api/products` 401 because `server/.env` had no
service token. Check 3 + this block make that impossible to miss.

---

## Tier 2 — UI tests (opt-in, ~30–60s)

After Tier 1 passes, **ask once**:

> Sanity checks pass. Want me to run the full UI tests too?
> They drive each CRUD cycle in a headless browser and capture the network log —
> catches issues sanity checks miss (form submission, table refresh, modal flows).
> ~30 seconds.
>
> Reply *"yes"* to run them, anything else to skip.

If the user replied affirmatively (*"yes"*, *"go"*, *"run them"*, *"sure"*, *"please"*), run Tier 2. Anything else (no reply, *"skip"*, *"no"*, a follow-up that changes the subject) means skip.

**Remember the answer for the session.** If they opted in once, future build turns auto-run Tier 2 without re-asking. If they opted out, future turns skip Tier 2 without asking. Only re-ask if they explicitly say *"ask me about tests again"* or *"actually run the UI tests"*.

Phase B is a **page-load probe with full network + console capture**. The driver
loads the preview URL in headless Chromium, watches every `fetch()` response
and every console / pageerror event for ~3 seconds, and exits non-zero if
anything failed. Generic — no schema introspection, works against any
generated app shape (CRUD, dashboard, agent chat UI). CRUD-flow driving
(clicking Create, filling forms, asserting refresh) is deferred to Phase C.

### Pre-flight

- **Tier 1 must have passed first.** If `tsc` is broken or the preview is 502,
  Tier 2 is wasted effort — fix Tier 1 failures before running. The skill is
  invoked *after* the Tier 1 block above; respect that ordering.
- The driver lives at `<sandbox>/.datagol/ui-test.mjs` and is written
  per-session by the cli-server. The Pi subprocess has `DATAGOL_PREVIEW_URL`
  injected into its env, so the driver picks the preview up automatically.

### Invocation

```bash
cd <sandbox-cwd> && node .datagol/ui-test.mjs
```

The driver prints a single JSON blob to stdout and exits `0` on success, `1`
when failures were captured, `2` for setup errors (missing env, browser
launch failure).

### Output schema

```jsonc
{
  "url": "http://localhost:5173/preview/<sessionId>/",
  "network": [
    {
      "url": "https://<datagol>/noCo/api/v2/workspaces/X/tables/Y/cursor",
      "method": "POST",
      "status": 200,
      "requestBody": "{\"requestPageDetails\":...}",
      "responseBody": null
    }
    // ...one entry per response observed during load
  ],
  "console": [
    { "level": "error", "text": "TypeError: ...", "location": { "url": "...", "lineNumber": 42 }, "stack": "..." }
  ],
  "failures": [
    // pre-filtered slice of the above where status >= 400 or level === 'error'
    { "type": "network", "url": "...", "method": "POST", "status": 404, "requestBody": "...", "responseBody": "workbook not found" },
    { "type": "console", "level": "error", "text": "Cannot read property 'name' of undefined", "stack": "..." }
  ]
}
```

The agent only needs to read `failures` — `network` and `console` are kept
for context when reporting back to the user.

### Failure interpretation

Each entry in `failures` has a *type* and a recognisable shape. The patch
target is almost always `src/lib/api.ts`, the component that owns the broken
render, or a hard-coded ID constant. Map failures like so:

| Failure | Diagnosis | Fix path |
|---|---|---|
| `network` 404 against `.../tables/<id>/cursor` | Hard-coded workbook ID is wrong | Call `datagol_get_workspace_schema`, patch the constant in `src/lib/api.ts` or `src/config.ts` |
| `network` 400 with body mentioning `cellValues` | Write shape is wrong (missing wrapper, or `null` value) | Patch the request builder in `src/lib/api.ts` |
| `network` 400 with body mentioning a typed field | Cell type mismatch (e.g. `port` as number, must be string) | Patch the field serialisation in `src/lib/api.ts` |
| `network` 401 / 403 | Auth header missing/expired/lacks workspace access | Re-check `datagol-app-auth` step C; verify `x-auth-token` vs `Authorization: Bearer` use |
| `network` 405 / `PATCH /rows/:id` | Used wrong HTTP verb — `PUT /rows` is the correct shape | Patch `src/lib/api.ts` |
| `network` 5xx | Backend error — not the app's fault | Retry once; if still 5xx, **surface to user verbatim, don't try to "fix" it in app code** |
| `console` `TypeError: Cannot read property 'X' of undefined` | Render path expects nested cell shape but rows were flattened (or vice-versa) | Read the stack, find the component, patch the access path |
| `console` `Failed to fetch dynamically imported module` | Vite chunk boundary issue, usually a wrong import path | Patch the import in the file the stack points to |
| `console` pageerror with no stack | Hard runtime error during render — read `.text` and infer | Read the file, locate the failing call, patch |

### Self-heal loop (3 attempts per failure, max)

1. Run `node .datagol/ui-test.mjs`.
2. Parse stdout as JSON. If `failures.length === 0`, report green and stop.
3. Pick the **first** failure. Diagnose using the table above.
4. Apply one targeted patch (one file, ideally one or two lines).
5. Re-run the driver. If the *same* failure persists, attempt 2. If a
   *different* failure appears, treat it as a new failure (its own 3-attempt
   counter).
6. After 3 attempts on the same failure, **stop**. Quote the captured
   network response body / console stack verbatim and ask the user how to
   proceed.

### Output format

Tier 2 is user-opted-in, so **show the trace**. One line per step:

```
🧪 UI tests
✓ Page loaded (1.2s)
✗ POST /workspaces/.../tables/contacts/cursor → 404 "workbook not found"
  → Hypothesis: hard-coded WORKBOOK_CONTACTS in src/lib/api.ts is stale
  → Refetched schema, patched WORKBOOK_CONTACTS to `7f3e...`
✓ POST /workspaces/.../tables/contacts/cursor → 200 (retry)
✓ No console errors

UI tests pass.
```

On exhausted failure:

```
🧪 UI tests
✓ Page loaded
✗ TypeError: Cannot read property 'name' of undefined
  at ContactList (src/pages/Contacts/List.tsx:24:18)

  Attempted fixes (3):
    1. Switched `r.cellValues.name` → `r.name` (assumed rows flattened) — still throws
    2. Switched back, added `?.` chain — error moved to line 28
    3. Reverted, re-checked api.ts cursor mapping — mapping looks correct;
       the data actually lacks `name` for some rows

The data has rows without a `name` cell. Want me to handle the missing
case (render "—" placeholder), or is the data itself wrong?
```

---

## Self-heal discipline (applies to both tiers)

- **Targeted patches only.** Each fix touches the specific file:line the failure points at. Never refactor a whole file to fix one error.
- **Cap at 3 attempts per failure.** If three targeted patches don't resolve it, **stop**. Report the failure + the three attempted hypotheses + the resulting test output, and ask the user how to proceed.
- **Log each hypothesis in chat** (Tier 1 keeps it terse; Tier 2 shows the full trace because the user opted in). The user must be able to see what was tried.
- **Don't loop silently.** Every fix is one line of chat output minimum, even on success.
- **Don't double-fix.** Once a check passes, don't re-touch the file that just got patched even if a later check fails — the later failure is a *different* bug; fix the right file.

## Composition rules

Inherits from `datagol-data-query`:

- **Prose first, then tool calls.** When announcing what tests will run, say so as a one-line message *before* the first bash call. Don't dump tool rows with no context.
- **Network errors verbatim.** Quote DataGOL's response body in fenced code, don't paraphrase it. The body is usually self-explanatory; the agent's paraphrase often loses the key clue.

## Hard rules

- **NEVER hand-manage the server process.** No `node server/dist/index.js`, no
  `npm run dev/start`, no manual `kill`. The platform auto-rebuilds + restarts
  on `agent_done`. Hand-spawning creates stale orphan processes — the root
  cause of phantom 404/hang symptoms. See "Process model" above. Corollary:
  never add header-based / no-body login hacks to dodge a fake "JSON POST
  hangs through the proxy" problem — it doesn't; you were hitting a stale
  process.
- **Tier 1 is a GATE, not a silent check.** Always paste the evidence block (above) before any completion statement — the block IS the permission to say "ready". A pass is not "silent" anymore; the proof must be shown. (Keep it to the one tight block — don't narrate each curl beyond that.)
- **Tier 1 is autonomous on fail.** Don't ask the user permission to fix a missing import or wrong workbook ID — just fix it. Surface the trail (one line per attempt), not a confirmation dialog.
- **Tier 2 is never autonomous.** Always ask before running. Always show every step's network log when running — the whole point of opting in is to see the trace.
- **Don't run tests on read-only turns.** If the agent didn't modify anything in the most recent turn, skip. Re-running checks after every clarification question is annoying.
- **Don't test placeholder data as bugs.** If api.ts uses fixture row IDs (`'00000000-0000-0000-0000-000000000000'` or the agent's own scaffold defaults) and the cursor read returns 404, that's a misconfiguration to fix, not a real bug — patch the constants to the real IDs from `datagol_get_workspace_schema`.
- **Service token is required for the read probe.** If the build didn't go through `datagol-app-auth` provisioning (Flow A only, no service account), use the user's bearer (`$DATAGOL_TOKEN`) instead. Both work for read-only cursor calls.
- **Skip if the user said skip.** *"don't run tests"*, *"skip tests this turn"* — respect it for the current cycle. Re-asks on the next turn are fine.

## Cross-references

- **`datagol-detailed-plan`** — §8 (Test & Verification) is the source-of-truth for which read-only endpoints to probe. This skill operationalises that section: instead of telling the user *"you can curl this"*, the agent runs it.
- **`datagol-app-development`** — defines the "hard-code real workspace + workbook IDs" rule. This skill's read probe verifies those IDs actually exist.
- **`datagol-app-auth`** — the service-token provisioning the read probe authenticates with. If the probe fails with 401, this is the path to re-check.
- **`datagol-data-query`** — composition rules (prose first, network errors verbatim) come from here.
- **`datagol-create-agent`** — Phase C will add an "agent round-trip" Tier 2 check that drives a created agent's chat with a canned prompt.
