---
name: datagol-detailed-plan
description: Mandatory step right after the `datagol-interview` skill — translate the interview answers into a thorough architecture + UX + data-model + implementation plan, present it for explicit approval, then execute. Detail level should be high enough that the user spots gaps before any data is created.
---

# Detailed Plan & Confirm

The bridge between **discovery** (`datagol-interview`) and **execution**. Once the
user has answered the questions you needed, the next deliverable is a
**comprehensive plan**, not a one-paragraph summary. The plan must cover
**architecture, UX, data model, and implementation order** — in that order —
with enough detail that the user can spot anything you got wrong before any
modifying tool fires.

## When to Trigger

- Immediately after `datagol-interview` produces a complete picture.
- Before any multi-step build (workspace + workbooks + links + UI, or any
  combo of two or more modifying calls).
- Any time the user requests something with > ~3 modifying steps, even if
  no formal interview happened.

Skip when:

- The user said "just do it" / "go ahead without asking".
- It's a single-step change (e.g. add one column) — a one-line confirm is fine.
- You're inside a flow the user already approved (don't re-confirm step by step).

## Plan Format

Use this Markdown structure. Be **thorough** in each section — vague plans
get rubber-stamped and produce wrong builds.

```markdown
## 📋 Implementation Plan

**Goal:** <one-line restatement of what you're building>
**Outcome the user will see:** <one-line description of the working app/agent at the end>

---

### 1. Architecture

A short paragraph + a small diagram of the system shape. **Mandatory
sub-sections:**

- **User management / auth model.** Restate the answer the user gave in the
  interview (mandatory question §2). Pick one:
  - **Multi-user with login** — implement via the **`datagol-user-auth`** skill:
    Users workbook (username, password_hash, salt, role), PBKDF2 hashing
    in the browser, sign-up + sign-in pages, role-gated UI, per-user
    `Owner` column on every other workbook. Spell out the role list (e.g.
    `admin / editor / user / viewer`). Call out the security caveat
    (demo-grade — `datagol-user-auth` skill explains why) so the user knows.
  - **Single-user demo** — no login. Whole app runs against a single shared
    DataGOL bearer (paste-token UI on first load + localStorage). All
    viewers see the same rows. Note in plan: "auth can be added later via
    the `datagol-user-auth` skill".
  - **Other** — describe the variant (shared team login, SSO, public read /
    auth'd write, etc.) and how the app enforces it.
- **Where data lives** — DataGOL workspace `<name>` (**state explicitly whether this is *existing* and reused, or *will be created***; restate the answer from `datagol-interview` §2). N workbooks, M LINK relationships. If multi-user, name the column(s) used to scope rows to the owning user (typically a `USER` column called `Owner` on every user-scoped workbook). The user must be able to read the workspace decision from this section and veto it before approving the plan — never bury workspace creation in the implementation-order step.
- **Where logic lives** — generated React app (sandbox preview), or DataGOL
  custom agent + MCP, or both.
- **External calls** — DataGOL REST (`/noCo/api/v2/...`), agent streaming
  (`/ai/api/v2/messages/streaming`), GitHub OAuth (if "Connect GitHub"),
  etc.

ASCII diagram (required for non-trivial cases — show the auth path):

Single-user demo:
```
Browser ──► React app ──► DataGOL REST  (one shared bearer)
```

Multi-user with login:
```
Browser ──► React app ──► /auth/login  ── DataGOL ──► { token }
            │
            └──► DataGOL REST  with per-user bearer
                  └─ rows filtered by Owner = currentUserId
```

The auth-model choice cascades into the data model (do we need an Owner
column on every workbook?), the UX (sign-up / sign-in screens?), and the
implementation order (auth pages first if multi-user). Reflect those
cascades in §2 (UX), §3 (Data Model), and §6 (Implementation Order).

### 2. UX (page by page)

Start with an **aesthetic direction** (one paragraph) drawn from the
`datagol-frontend-design` skill — pick a real point of view (refined/editorial,
brutalist, retro-futuristic, organic, etc.) and name the design tokens
that will follow from it: display font + body font, primary palette
(hex), accent, motion approach, density. The user gets to push back on
this *before* any code is written.

> **Aesthetic direction:** Editorial, calm, type-led. Display font: Söhne
> Headline (or Fraunces); body: Inter Tight. Palette: ivory `#f8f5ee`
> ground, near-black `#15110a` text, oxblood `#7a1f1f` accent. Motion:
> single page-load reveal with 60ms staggered delays per section; no
> hover micro-anims. Density: generous, ~32px gutters, ~1.6 line-height.

Then, for each page / screen the user will land on, describe:
- **Route / entry point** — how the user gets there
- **Purpose** — what the user does on this screen, in one sentence
- **Layout** — header / sidebar / main / footer; primary content
- **Components** — list (e.g. SearchBar, ContactsTable, Pagination)
- **Empty state** — what shows when there's no data
- **Loading state** — skeleton / spinner placement
- **Error state** — inline banner vs. modal vs. toast
- **Key actions** — buttons / links that lead elsewhere

Example:

> **`/contacts` — Contacts List**
> - Purpose: browse and find contacts.
> - Layout: header with search + "+ New", main = table, no sidebar.
> - Components: `ContactsTable`, `SearchInput`, `Pagination`, `EmptyState`.
> - Empty: "No contacts yet — Add your first" + CTA.
> - Loading: 5 skeleton rows.
> - Error: red banner above table; retry button.
> - Actions: row click → `/contacts/:id`, "+ New" → `/contacts/new`.

Cover the navigation between pages explicitly (e.g. sidebar links, breadcrumbs).

### 3. Data Model

#### 3.1 Workspace
- Create workspace **"<name>"** (color: `#xxxxxx`)
  *or:* Reuse existing workspace **"<name>"** (`<id>`)

#### 3.2 Workbooks (per-workbook column tables)

For each workbook, a full column table — not a one-line summary:

**Contact**
| Column     | Type             | Required | Notes |
|------------|------------------|----------|-------|
| Name       | SINGLE_LINE_TEXT | ✓        | Display name |
| Email      | EMAIL            | ✓        | Unique by convention |
| Phone      | PHONE_NUMBER     |          | International format |
| Title      | SINGLE_LINE_TEXT |          | Job title |
| LastSeen   | DATE             |          | Last interaction |

**Company**
| Column      | Type             | Required | Notes |
|-------------|------------------|----------|-------|
| Name        | SINGLE_LINE_TEXT | ✓        | |
| Website     | URL              |          | |
| Industry    | SINGLE_LINE_TEXT |          | Free text — promote to dropdown if list stabilises |
| Size        | NUMBER           |          | Headcount |

(continue for every workbook…)

#### 3.3 Relationships (LINK columns)

| Source → Target | Type | Direction | Why |
|---|---|---|---|
| Company → Contacts | HAS_MANY | One company has many contacts | Bidirectional reciprocal in Contact |
| Company → Deals    | HAS_MANY | | |
| Contact → Deals    | HAS_MANY | A contact participates in many deals | |
| Deal → Owner (User) | BELONGS_TO | Each deal has one owner | |

#### 3.4 Enums / value lists

Where a column will hold a constrained set of values (e.g. Deal Stage),
list them so the user can correct before generation:
- **Deal Stage:** Lead, Qualified, Proposal, Negotiation, Closed-Won, Closed-Lost

### 4. App / UI tech plan (if applicable)

Apps are **single-port full-stack** — see `datagol-app-development` (parent) and
its children `datagol-app-frontend` / `datagol-app-backend`. State this shape:

- Architecture: one Express (TS) process serves the built React client **and**
  `/api/*` on one port / one container. Dev = Vite (5173) + Express (3001) with
  Vite proxying `/api`. **Data layer is DataGOL workbooks — no SQL DB.**
- **Data flow:** client → same-origin `/api/*` → Express → DataGOL REST. The
  **client never calls DataGOL and never holds the service token**; the token +
  `x-appid` live in **server env only**.
- Stack: Vite + React 19 + TypeScript (`client/`); Express + TypeScript
  (`server/`); npm workspaces monorepo.
- Routing (client): React Router, **one route per entity** (`/contacts`,
  `/contacts/:id`, …); wrap with `basename={import.meta.env.BASE_URL}` for the
  preview. Routerless only for a single-purpose app.
- Server data layer: `server/src/lib/datagol.ts` (`dgFetch` + token + `x-appid`
  from `process.env`) wrapped behind typed `server/src/routes/<resource>.ts`
  endpoints (row CRUD, schema, date coercion, write sanitization, where-clause
  safety).
- Client data layer: `client/src/lib/api.ts` — a thin `fetch('/api/...')`
  wrapper, the only data entry point. No DataGOL URL/token in the client.
- Theming: **light + dark mandatory** (CSS tokens + persisted toggle).
- Tables: native for simple CRUD; `@tanstack/react-table` only when
  workbook-grade sorting/filtering/column control is needed.
- Monorepo layout:
  ```
  package.json            # workspaces: [client, server]
  Dockerfile              # multi-stage, single-port (EXPOSE 3001)
  .env / .env.example     # server: DATAGOL_BASE_URL, DATAGOL_SERVICE_TOKEN, DATAGOL_APP_ID, table ids
  client/  src/{main.tsx,App.tsx,pages/,components/,lib/api.ts}
  server/  src/{index.ts (serving trick), config.ts, lib/datagol.ts, routes/<resource>.ts}
  ```
- Token strategy: service token + `x-appid` in **`server/.env`** (runtime-
  injected, never baked, never in the client bundle) — per `datagol-app-auth`.
- Loading/error UX: standard skeleton/banner described in section 2.

### 5. Custom Agent (if applicable)

- **Agent name:** "<name>"
- **System prompt** (verbatim, ~3-6 lines, restricting scope):
  > "You are a sales-support agent. You answer questions strictly about the
  > Contacts, Companies, and Deals workbooks via the connected MCP servers.
  > If a question requires data outside these, say so. Be concise."
- **MCP-connected workbooks:** Contact, Company, Deal
- **isPublished:** false (draft — user publishes after testing)
- **Conversation starters:** 2–3 example prompts for the agent's empty state.

### 6. Implementation Order

Ordered list of every modifying call you'll make. Number them so the user
can sanity-check dependencies and call out anything they want re-ordered.

1. `datagol_create_workspace("Sales CRM")` → capture `workspaceId`
2. `datagol_create_workbook("Contact", [Name, Email, Phone, Title, LastSeen])` → `contactId`
3. `datagol_create_workbook("Company", [Name, Website, Industry, Size])` → `companyId`
4. `datagol_create_workbook("Deal", [Title, Value, Stage, CloseDate])` → `dealId`
5. `datagol_create_link(Company, Contacts, HAS_MANY, "Contacts")`
6. `datagol_create_link(Company, Deals,    HAS_MANY, "Deals")`
7. `datagol_create_link(Contact, Deals,    HAS_MANY, "Deals")`
8. *(if UI)* `write package.json`, `tsconfig.json`, `next.config.ts`
9. `write app/globals.css`, `app/layout.tsx`, `app/page.tsx`
10. `write lib/config.ts`, `lib/schema.ts`, `lib/datagol.ts`
11. `write components/Icons.tsx` and other shared components
12. … (one entry per generated route/component file under `app/` and `components/`)
13. *(if agent)* `datagol_create_agent("Sales Assistant", workbooks=[contactId, companyId, dealId])`

### 7. Out of Scope (for clarity)

Explicitly list things that aren't being built so the user can flag missing
expectations *before* you start:

- Importing existing CSV / spreadsheet data
- Custom roles or per-record permissions
- Email / Slack notifications
- Mobile-specific layouts
- A Vercel / GitHub publish (mention these are separate one-click flows
  available after the build)

### 8. Test & Verification (mandatory — runs after the build)

Before declaring the build done, run a **verification pass**. Surface a
report to the user; only call the build complete once everything green-
lights or the user explicitly accepts known issues.

#### 9.1 Button-wiring audit

For every page in §2 (UX), trace every interactive element:

| Page | Element | Handler | Calls | Status |
|------|---------|---------|-------|--------|
| `/contacts` | "+ New" button | `() => navigate('/contacts/new')` | (no API) | ✓ wired |
| `/contacts/new` | Submit button | `onSubmit → createContact()` | `POST /tables/{contact}/rows` | ✓ wired |
| `/contacts/:id` | Save button | `onSave → updateContact()` | `PUT /tables/{contact}/rows` | ✓ wired |
| `/contacts/:id` | Delete button | `onDelete → deleteContact()` | `DELETE /tables/{contact}/rows` | ✓ wired |
| `/dashboard` | "Refresh" button | — | — | ⚠ no handler attached |

Method: read each page file, grep for `onClick`, `onSubmit`,
`onChange={`, `onKeyDown=`. Trace the handler to its definition; check
that any API-related handler eventually hits a function in `lib/datagol.ts`
(or the app's established API/data client). Flag any button without a handler,
any handler that throws / no-ops, any API call whose endpoint is wrong.

#### 9.2 Static type-check

Run `npx tsc --noEmit` in the sandbox (gated by `bash` HITL approval —
remind the user this is a read-only check, no install). Catches missing
imports, wrong signatures, typos in column names. Surface all errors;
fix in source and re-check until clean.

#### 9.3 API endpoint smoke-test (READ-only)

For each API path the app references, hit it once with `curl` using the
agent's bearer token (env: `DATAGOL_TOKEN`) — but only **read endpoints**.
Never POST/PUT/DELETE during verification, you'll dirty the workspace.

> **This check is the *agent* verifying DataGOL reachability at build time — it
> correctly goes straight to `*.datagol.ai`.** That's different from the
> *running app*, whose data flows client → `/api/*` → Express → DataGOL (the
> browser never calls DataGOL). Build-time/info-gathering calls (schema
> discovery, provisioning, this smoke-test) are direct; only the shipped app's
> data access goes through the API. Don't reroute this smoke-test through
> `/api/*`. Optionally, once the app's server is up, also `curl` its
> `/api/<resource>` to confirm the proxy layer returns the same data.

| Endpoint | Method | Expected | Status |
|----------|--------|----------|--------|
| `/noCo/api/v2/workspaces` | GET | 200 | ✓ |
| `/noCo/api/v2/workspaces/{ws}/tables` | GET | 200 | ✓ |
| `/noCo/api/v2/workspaces/{ws}/tables/{contact}/cursor` | POST (read query) | 200 | ✓ |
| `/noCo/api/v2/workspaces/{ws}/tables/{deal}/cursor` | POST (read query) | 200 | ✓ |

The `cursor` endpoint *is* a POST, but it's a read-only query. Send a
minimal body (`requestPageDetails: {pageNumber: 1, pageSize: 1}`) and
verify it returns 200 with the expected shape. See `datagol-workbook-operations`
skill for the body schema.

#### 9.4 Sandbox preview reachability

The Vite dev server is already running on the sandbox port. Confirm:
- The page renders without a Vite/TS error overlay (curl the index, look
  for `<script type="module" src="/src/main.tsx">` and the absence of
  `[plugin:vite:react-babel]` or `Failed to resolve` strings).
- No dependency-resolution errors (would indicate a broken
  `node_modules` symlink — see `datagol-app-development`'s "never run npm install"
  rule; if you see this, alert the user, don't try to fix with install).

#### 9.5 Verification report

Surface the result back to the user as a single bordered block:

```
✅ Build verified
   • 14 buttons across 5 pages — all wired
   • tsc clean
   • 4 endpoints reachable (200)
   • Preview rendering, no Vite errors
   • Open the Preview tab to try it.
```

Or, when something failed:

```
⚠ Build complete with issues
   • 14 buttons — 13 wired, 1 missing handler:
     /dashboard "Refresh" button has no onClick
   • tsc: 2 errors (missing import in pages/Deals/Form.tsx line 14)
   • Endpoints OK
   Fix now? (yes / proceed anyway)
```

If the user picks "fix now", iterate until the report is green.

### 9. Estimated counts

- Modifying tool calls: **~13** (workspace + 3 workbooks + 3 links + 6 files)
- File writes: **~6** (types, api, App, 3 page directories with 9 sub-files)
- Verification calls: **~4–8** bash invocations (tsc + read-only curls)
- Approximate time: **3–5 minutes** including verification

---

**Shall I proceed?** (yes / change something / skip a section / ask Q)
```

## Rules

1. **Detail over brevity.** A one-paragraph plan gets misread. Use the eight
   sections above; only collapse sections that genuinely don't apply (e.g. no
   Custom Agent).
2. **Tables for column lists.** Required-flag, type, and a one-line "why" per
   column makes it scannable.
3. **Page-by-page UX.** "It'll have CRUD pages" is too vague — list every page
   with purpose, layout, components, states.
4. **Numbered Implementation Order.** Show every modifying call. The user
   should be able to count them and spot missing dependencies.
5. **Compact type names** in the *summary* (Section 6 etc.), but **full
   `UI_DATA_TYPE` names** in the data-model column tables (Section 3).
6. **System prompt verbatim** for any custom agent — the user reviews the
   wording, not a summary of it.
7. **Out of Scope is mandatory.** Explicitly listing what's not being built
   prevents post-hoc complaints.
8. **End with one clear ask:** "Shall I proceed?" with explicit alternatives.

## Handling User Edits

If the user wants changes to the plan, **revise the plan and re-present**.
Re-render the affected sections with diffs called out:

> User: "Drop the Tasks workbook and add a Stage field to Deal."
> Agent: *(updates plan, says "Updated: removed §3.2 Tasks, added Stage to
> Deal table; sections 5 & 6 also adjusted." Then re-renders the full plan.
> Asks "Shall I proceed?" again.)*

After 2–3 revision rounds, if the user keeps tweaking, ask: "Want me to
proceed with what we have and iterate after?" — that's a signal to stop
re-planning and start building.

## After Approval

Once the user says "yes" / "proceed" / "go ahead":

1. Execute step-by-step in the exact order from Section 6.
2. Narrate each step briefly:
   ```
   ✅ Created workspace "Sales CRM" (`abc-123`)
   ✅ Created workbook "Contact" (5 columns)
   ⟳ Creating workbook "Company"…
   ```
3. If a step fails, **stop**, show the error, ask what to do (retry / skip /
   abort). Don't barrel forward.
4. Note: the platform's hard gate (`datagol-human-in-the-loop` skill) still fires
   on each `datagol_create_*` and `bash` call — that's an extra safety check,
   not a replacement for this plan-confirm flow.
5. **Run §8 Test & Verification.** This is mandatory — don't declare the
   build complete until the report passes (or the user accepts the issues).
   Concretely:
   - Walk every page from §2 (UX) and audit button → handler → API.
   - `bash`-run `npx tsc --noEmit` in the sandbox; surface any errors.
   - For each unique READ endpoint the app uses, `curl` it once with the
     agent's bearer to confirm it responds 200.
   - Curl the sandbox preview's `/` and grep for Vite error overlays.
   - Surface the verification report as `✅ Build verified` or
     `⚠ Build complete with issues` (see §8.5 for the format).
   - If issues: ask "Fix now? (yes / proceed anyway)". If yes, iterate
     until clean.
6. End with a summary table:
   ```
   | Created   | Name        | ID         |
   |-----------|-------------|------------|
   | Workspace | Sales CRM   | `abc-123`  |
   | Workbook  | Contact     | `def-456`  |
   | …         | …           | …          |
   ```
   …and link to the verification report (or repeat the green-light line).

## Anti-Patterns

❌ **Skipping the plan and inferring approval.** "Yes" to a small
clarification is not "yes" to a full multi-step build. Show the plan
separately.

❌ **One-paragraph plan.** Even short builds get the eight-section template;
sections that don't apply collapse to one line, but they appear.

❌ **Showing tool-level mechanics in the plan.** Users don't need to see
`POST /noCo/api/v2/...` — they need to see "Create workbook Contact with 5
columns".

❌ **Vague column lists.** "Contact will have a few fields" — name them.
The user often spots a missing field at this stage.

❌ **Skipping Architecture or UX sections** for an app build. The architecture
section forces you to be explicit about *where* logic and data live; the UX
section forces you to think through every screen before code-gen starts.

❌ **Re-confirming during execution.** Once approved, run the whole sequence;
only stop on errors. The platform-level gate handles per-tool confirmation
where needed.

❌ **Skipping the §8 verification step.** Declaring a build "done" without
walking the buttons + running tsc + smoke-testing the read endpoints means
shipping broken UIs the user will discover. The verification is short
(seconds–to-a-minute) and catches the obvious failures (un-wired buttons,
wrong endpoint paths, type errors). Always run it.

❌ **POST/PUT/DELETE during verification.** §8.3 says READ-only smoke-test
for a reason — verification calls run with the agent's real bearer, so a
POST would dirty the user's workspace with test rows. If you really need
to confirm a write endpoint works, do it once *with the user's explicit
permission* and clean up after.

## Cross-References

- **`datagol-interview`** — runs first; gathers the inputs this plan summarizes.
  **Don't write a plan without an interview having run.**
- **`datagol-human-in-the-loop`** — defines the platform-enforced gates that fire
  during execution (after this plan is approved).
- **`datagol-workbook-design`** — for choosing types, naming, and shape decisions
  *while building Section 3*.
- **`datagol-create-links`** — for translating "X has many Y" into Section 3.3.
- **`datagol-app-development`** — for fleshing out Section 4 (file layout, page code).
- **`datagol-agent-chat-ui`** — for chat-app variants of Section 4.
- **`datagol-create-agent`** — for Section 5 details (prompt, MCP wiring).
- **`datagol-workbook-operations`** — for any non-schema operation (rows, AI columns)
  the plan will execute.
