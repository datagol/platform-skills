---
name: datagol-self-test
description: After the agent finishes building (or substantially modifying) a generated app, run a two-tier verification pass — **Tier 1 sanity checks run automatically and silently** (TypeScript compile, preview health, per-workbook read probe); **Tier 2 UI tests run only if the user opts in** (Playwright drives each entity's CRUD cycle, captures network + console errors, agent auto-patches up to 3 attempts). Triggered automatically on `agent_done` after a build turn that wrote files OR created workbooks OR registered data sources, and on-demand when the user says "test the app", "verify it works", "check everything", "run the tests", etc.
---

# DataGOL Self-Test

The agent finishes a build, then verifies its own work. Two tiers — cheap stuff runs without asking; expensive stuff asks first.

> **Why this skill exists.** *"I built it"* is not the same as *"it works"*. Without verification, the agent has been known to: emit code that doesn't compile, hard-code workbook IDs that don't exist, send `api.ts` request bodies in the wrong shape (missing `cellValues` wrapper, `port` as number not string), call `PATCH /rows/:id` instead of `PUT /rows`. Each is invisible until the user opens the preview and gets a blank page or a network 4xx. This skill closes that loop.

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

## Tier 1 — Sanity checks (auto, ~5–10s, silent on pass)

Run these three in sequence. Stop and self-heal on the first failure; once it passes, continue.

### Check 1: TypeScript compiles

```bash
cd <sandbox-cwd> && npx tsc --noEmit 2>&1
```

Pass: exit code 0, no output.
Fail: any `error TS####` lines. Each line is `path/file.ts(LINE,COL): error TSxxxx: <message>`.

**Auto-fix protocol:**
- Read the file at the named line.
- Apply a targeted fix — never refactor the whole file. The TS message tells you *exactly* what's wrong (*"Property 'X' does not exist on type 'Y'"*, *"Cannot find module 'Z'"*, *"Type 'string' is not assignable to type 'number'"*).
- Re-run `tsc --noEmit`. If still failing, attempt 2; up to 3 attempts total.

### Check 2: Preview is reachable

Get the preview URL from the most recent `sandbox_ready` event in the transcript, or read `${API_URL}/preview/${sessionId}/` from the cli-server.

```bash
curl -s -o /dev/null -w "%{http_code}" "<previewUrl>"
```

Pass: `200`.
Fail: `502` (Vite crashed), `503` (Vite still starting — wait 2s and retry once), any other status.

**Auto-fix protocol:**
- If `502` and the Vite dev server log mentions a syntax error or missing import, locate the file from the error and patch it.
- If `503` after retry, give up — Vite needs more time; surface to user.
- If anything else, surface verbatim.

### Check 3: Per-workbook read probe

For each workbook the generated app references (look in `src/lib/api.ts` or `src/config.ts` for hard-coded `WORKBOOK_ID` / `TABLE_ID` constants — the build skills require these to be hard-coded), run a cursor read against DataGOL. **Use the same auth header the generated app uses** — `x-auth-token: $DATAGOL_SERVICE_TOKEN` if a service token was provisioned, else `Authorization: Bearer $DATAGOL_TOKEN`:

```bash
curl -s -X POST "${DATAGOL_BASE_URL}/noCo/api/v2/workspaces/${WS_ID}/tables/${WB_ID}/cursor" \
  -H "x-auth-token: ${DATAGOL_SERVICE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"requestPageDetails":{"pageNumber":1,"pageSize":1}}' \
  -w "\n---STATUS:%{http_code}"
```

Pass: status `200` and JSON body parseable.
Fail: any 4xx/5xx — quote the response body, it's almost always descriptive (*"workbook not found"*, *"permission denied"*, *"cellValues must not be null"*).

**Auto-fix protocol:**
- `404 workbook not found` → the hard-coded ID in api.ts is wrong; call `datagol_get_workspace_schema` to find the correct one, patch the constant.
- `403 permission denied` → the service account wasn't granted access to the workspace. Re-run the `datagol-app-auth` Step C (grant `CREATOR` via `bulkUsers`).
- `401` → service token is missing/wrong; check `.env.local`.
- Anything else, surface to user.

### Tier 1 output format

Print **one** tight block. Don't narrate each individual check unless it failed:

```
🧪 Sanity checks
✓ TypeScript compiles
✓ Preview is live
✓ All 4 workbook reads succeed

App is ready.
```

On a failure that was auto-patched:

```
🧪 Sanity checks
✓ TypeScript compiles
✗ Preview was failing — `Cannot find module './pages/Customers'` in src/App.tsx:8
  → Patched src/App.tsx:8 (import path was missing /index)
✓ Preview is live (retry)
✓ All 4 workbook reads succeed

App is ready.
```

On a failure that exhausted 3 attempts:

```
🧪 Sanity checks
✓ TypeScript compiles
✗ Workbook reads — Contacts (id `xyz...`) returns 404 "workbook not found"
  Attempted fixes (3):
    1. Re-read schema, replaced ID — still 404
    2. Looked for typo in WORKSPACE_ID — none found
    3. Re-fetched workspace schema fresh — workbook genuinely missing

The Contacts workbook doesn't seem to exist in the workspace. Did the create
call fail silently, or was it deleted? Want me to recreate it?
```

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

- **Tier 1 is silent on pass.** Don't celebrate ✓✓✓ when nothing was broken — one tight summary line is enough.
- **Tier 1 is autonomous on fail.** Don't ask the user permission to fix a missing import or wrong workbook ID — just fix it. Surface the trail (one line per attempt), not a confirmation dialog.
- **Tier 2 is never autonomous.** Always ask before running. Always show every step's network log when running — the whole point of opting in is to see the trace.
- **Don't run tests on read-only turns.** If the agent didn't modify anything in the most recent turn, skip. Re-running checks after every clarification question is annoying.
- **Don't test placeholder data as bugs.** If api.ts uses fixture row IDs (`'00000000-0000-0000-0000-000000000000'` or the agent's own scaffold defaults) and the cursor read returns 404, that's a misconfiguration to fix, not a real bug — patch the constants to the real IDs from `datagol_get_workspace_schema`.
- **Service token is required for the read probe.** If the build didn't go through `datagol-app-auth` provisioning (Flow A only, no service account), use the user's bearer (`$DATAGOL_TOKEN`) instead. Both work for read-only cursor calls.
- **Skip if the user said skip.** *"don't run tests"*, *"skip tests this turn"* — respect it for the current cycle. Re-asks on the next turn are fine.

## Cross-references

- **`datagol-detailed-plan`** — §8 (Test & Verification) is the source-of-truth for which read-only endpoints to probe. This skill operationalises that section: instead of telling the user *"you can curl this"*, the agent runs it.
- **`datagol-ui-generation`** — defines the "hard-code real workspace + workbook IDs" rule. This skill's read probe verifies those IDs actually exist.
- **`datagol-app-auth`** — the service-token provisioning the read probe authenticates with. If the probe fails with 401, this is the path to re-check.
- **`datagol-data-query`** — composition rules (prose first, network errors verbatim) come from here.
- **`datagol-create-agent`** — Phase C will add an "agent round-trip" Tier 2 check that drives a created agent's chat with a canned prompt.
