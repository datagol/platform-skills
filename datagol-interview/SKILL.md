---
name: datagol-interview
description: Mandatory discovery step before building anything new. Trigger any time the user asks to build / create / scaffold an app, workspace, agent, workbook, schema, or feature — even if they think they've given you enough detail. Ask focused questions one at a time, then hand off to `datagol-detailed-plan`. Skip ONLY for trivial single-step changes or when the user has explicitly opted out.
---

# Interview Workflow

A short structured conversation to fill in the gaps **before** you start
calling modifying tools. **This is mandatory** for any build/create request —
even when the user already wrote a paragraph of detail, there are *always*
gaps (field choices, edge cases, scope) you need to surface explicitly.
Ten quick questions beats ten failed builds.

## Trigger — MANDATORY

Run this skill **always** when the user's request includes any of:

- "Build / create / make / scaffold / set up / design / generate" + an app,
  workspace, agent, workbook, schema, table, dashboard, page, form, or feature
- "I want a / I need a / Help me make a …" anything that requires more than
  one modifying tool call
- Any request that would land you in `datagol-workbook-design`, `datagol-app-development`,
  `datagol-create-agent`, `datagol-agent-chat-ui`, or `datagol-create-links` — those skills assume
  the interview has already run

**Don't skip the interview because the user gave details.** Detailed requests
still have gaps — the user describes intent in free-form prose; you need to
confirm specifics like field types, required-ness, relationship cardinality,
default values, and out-of-scope items. Treat their detail as *initial input
to the interview*, then ask the questions that detail didn't answer.

### Calibrating depth

The interview's *length* scales with how much the user already specified:

| User input shape | Min. questions before plan |
|---|---|
| One-liner ("build a CRM") | 4–6 |
| Paragraph with entities listed | 2–3 (skip "what entities") |
| Detailed spec with fields | 2 (confirm scope + skipped categories) |
| Iterating on an existing workspace, < 3 tool calls | 0 (small change — go straight to plan) |
| Truly trivial (add one column) | 0 |

The minimum-2-questions rule for any non-trivial build holds even with a
detailed spec. Use those questions for things the user almost certainly
forgot: out-of-scope confirmation, output shape (just data vs. data + UI vs.
data + UI + agent), or the single most important user flow.

## Skip ONLY when

- The change is **truly small**: ≤ 2 modifying tool calls, well-scoped (e.g.
  "add a `phone` column to Contact").
- The user **explicitly opts out**: "just do it", "use your judgment", "skip
  the questions". Honor it for that request only — resume on the next.
- You're **continuing an approved plan**: the user already approved a
  `datagol-detailed-plan` and you're executing it. Don't re-interview during execution.

## Core Principles

1. **One question per message.** Multi-question messages go unread.
2. **Open, then narrow.** Start broad ("what should this do?"), then tighten
   to specifics ("what fields does a Contact need?").
3. **Offer 2–4 options** when the choice space is small. Free-text only when
   it's truly open.
4. **Echo what you heard** before the next question. Confirms understanding.
5. **Stop early when you have enough.** Don't drain the user's patience for
   nice-to-haves; sensible defaults are fine.
6. **Summarize at the end.** Present a tight bullet list and get explicit
   "yes, build it" before any modifying call.

## What to Ask

Cover the categories below in roughly this order. **Categories marked
*MANDATORY* must always be asked** for any non-trivial build — they
materially change the architecture, so skipping them produces wrong code.
Other categories are picked situationally.

### 1. Purpose & users
- "What problem does this app solve?"
- "Who will use it day-to-day?"

### 2. Workspace target — *MANDATORY when data is needed*

**Never ask the user to paste a workspace UUID.** That's the worst possible question — UUIDs aren't memorable, and the user doesn't have one handy.

Two paths:

1. **No data backing needed** (pure-UI app, local state, calculator, todo with localStorage, kanban with React state, etc.) — skip workspace entirely. Don't call `datagol_get_workspace_schema`, don't create one, don't ask about it.

2. **The build needs DataGOL data backing** — **ALWAYS** ask the user this *category* question **BEFORE invoking any workspace tool** (`datagol_create_workspace`, `datagol_get_workspace_schema`, `datagol_list_workspaces`). No exceptions. Even if the user said "build me a CRM with workbooks" and it's obvious data is needed, ask first; the user owns this decision because workspaces carry permissions, billing, and discoverability implications.

   > **Should I create a new DataGOL workspace for this, or put it in an existing one?**
   >
   > - **New** — I'll create one called *"\<derived-name\>"* (e.g. *"Sales CRM"* for a CRM build, *"Task Management"* for a project tracker). Reply with a different name if you'd prefer.
   > - **Existing** — I'll list your workspaces so you can pick by name.

   **Wait for the user's reply** — do not pre-emptively call `datagol_create_workspace` in the same turn as asking. Only after they reply:
   - **New** (with the proposed name, or any name they wrote) → call `datagol_create_workspace` with that name. Capture the returned `id` and use it for every subsequent tool call.
   - **Existing** → call `datagol_list_workspaces`, show the names (not UUIDs), let the user pick by name, then look up the `id` from the list yourself.

   Either way, the user never types a UUID — they pick by name or accept your proposal.

### 3. Authentication model — *MANDATORY*

**Knowing what kind of authentication the app uses is non-negotiable.** The architecture, the scaffold files, the API headers, and the provisioning tool calls all fork on this answer. Ask the question before any scaffolding tool fires; never infer, never default. The detailed-plan must restate the chosen mode in §1 Architecture.

Ask exactly this:

> **Which authentication model should the app use?**
>
> 1. **No authentication — no login screen, public-facing apps or quick prototypes.** Everyone who opens the app uses a shared DataGOL service token under the hood; all visitors see the same data. Pick this for public-facing sites where there's no concept of a "user", or for quick prototypes where login is out of scope.
> 2. **B2B / B2C app — the app owns its own user table.** Adds a sign-up + sign-in flow where the app's Express server verifies credentials (server-side hashing) against DataGOL Users + Roles workbooks and issues an **app bearer token** the client sends on every request; per-row ownership + roles are enforced server-side. DataGOL is reached with the service token (server-side only); app users are a separate identity store from DataGOL. Pick this for shipping a product to external users who don't have DataGOL accounts.
> 3. **Internal app — DataGOL users log in.** A login form that takes a DataGOL email + password; the app's server authenticates against DataGOL's IDP and the signed-in user's **bearer token** is forwarded by the server on subsequent calls (no service account). **No signup button.** Everyone who uses the app must already be a DataGOL user in this company. Pick this for internal tools where the team already has DataGOL accounts.

**Routing for the build phase** — wire the answer into the plan and into which skills the scaffold reads:

- **Mode 1 (No auth)** → only `datagol-app-auth` runs. The service-token 3-call provisioning is mandatory; the token lives in the app's **server** env (single-port app).
- **Mode 2 (B2B/B2C)** → `datagol-app-auth` (service token + workspace grants) PLUS `datagol-user-auth` (Users + Roles workbooks; server-side `/api/auth/login` → app bearer JWT; client sends the bearer; server uses the service token for DataGOL).
- **Mode 3 (Internal / DataGOL login)** → `datagol-internal-app-auth` only. **No service account.** The app's **server** calls `/idp/api/v1/user/login`, captures the bearer from the `Authorization` response header, and returns it; the client sends that bearer to `/api/*` and the server **forwards** it to DataGOL.

Hold the answer in the conversation; the `datagol-detailed-plan` skill will require it in Section 1 (Architecture). The mode must be stated as "Mode 1 / Mode 2 / Mode 3" (or by the friendly name above) so downstream skills can route deterministically.

### 4. Entities (data model)
- "What kinds of records does it track?" (Contacts, Deals, Tasks, Orders, etc.)
- For each unfamiliar entity: "What does a {Entity} actually represent?"

### 5. Relationships
- "How do these connect?" (e.g. *a Company has many Contacts; a Deal links
  one Contact to one Company*)

### 6. Fields (lightweight)
- For the most important entity: "What are the must-have fields?" Examples
  are fine: "name, email, status, due date".
- Don't drill into every entity's fields — start small. The user will add more.

### 7. Key flows
- "What's the single most important thing a user does in this app?"
- "What do they need to see first when they open it?" (dashboard / list /
  detail / form)

### 8. Output shape
- "Should I generate just the data model (workbooks), a UI on top, or both?"
- "Do you also want a chat agent over this data?"

### 9. Constraints (only if relevant)
- Specific terminology / brand language?
- Privacy / sharing scope (private vs. company-wide)?

## Example Transcript (good)

> **User**: I want to build a CRM.
>
> **Agent**: Got it. Quick scope check — is this for tracking sales
> opportunities (deals/pipeline) or more for managing customer support
> tickets, or both?
>
> **User**: Sales pipeline.
>
> **Agent**: Got it. Three core entities for a sales CRM: **Contacts**
> (people), **Companies** (orgs), **Deals** (opportunities). Want me to keep
> just those three, or add anything else like Tasks/Activities?
>
> **User**: Add Tasks.
>
> **Agent**: Done — Contacts, Companies, Deals, Tasks. For Deals, what stages
> do you want? A few options:
>   1. **Standard 5-stage:** Lead → Qualified → Proposal → Negotiation → Closed
>   2. **Simple 3-stage:** Open → Won → Lost
>   3. **Custom** — tell me yours
>
> **User**: 1.
>
> **Agent**: Got it. Last question — should I also generate a CRUD UI on top
> (List/Create/Edit pages per entity) or just the workbooks for now?
>
> **User**: Both.
>
> **Agent**: Here's what I'll build:
>   - **Workspace:** "Sales CRM"
>   - **Workbooks:** Contact, Company, Deal, Task
>   - **Links:** Company → Contacts (HAS_MANY), Company → Deals (HAS_MANY),
>     Contact → Deals (HAS_MANY), Task → (Contact|Deal) (BELONGS_TO)
>   - **Deal stages:** Lead, Qualified, Proposal, Negotiation, Closed
>   - **UI:** Vite+React app with List/Create/Detail pages per entity
>
>   Shall I proceed?
>
> **User**: Yes.

## Anti-Patterns (don't do this)

❌ **Multi-question dump:**
> "What's the app for? Who uses it? What entities? What fields? What stages?
> Do you want a UI? What model?"

❌ **Silent assumptions:** Building without asking when the user said "build
me a CRM" with no other context. Ask first.

❌ **Asking the user to paste a workspace UUID.** Don't. Section 2: skip workspace for pure-UI builds, or ask the new-vs-existing *category* question and resolve the ID yourself (create new, or pick from `datagol_list_workspaces` by name).

❌ **Picking the auth mode silently** (or defaulting to Mode 1) without the user's confirmed answer to §3. Knowing the auth mode is non-negotiable — every downstream skill (`datagol-app-auth`, `datagol-user-auth`, `datagol-internal-app-auth`, `datagol-app-development`) forks on it. If the answer isn't in the conversation, ask the question; do not infer from "internal-sounding" or "demo-sounding" project descriptions.

❌ **Skipping the interview because the user gave detail.** Detailed prose
still has gaps. Acknowledge what they gave you, then ask the *unanswered*
questions (out-of-scope, output shape, key flow). Minimum 2 questions for
any non-trivial build, no exceptions.

❌ **Over-interviewing:** Asking 12 questions to scaffold a 3-table todo
app. After 5–6 questions, summarize and proceed.

❌ **Yes/no questions when there's a meaningful choice:** "Do you want
it to be good?" — useless. Offer real options.

❌ **Forgetting earlier answers:** If the user said "Sales pipeline" in
question 1, don't re-ask it as question 4.

## Cross-References

- **`datagol-app-auth`** — runs when §3 = Mode 1 or Mode 2. Provisions the service token; establishes the `x-auth-token` header pattern.
- **`datagol-user-auth`** — runs when §3 = Mode 2. Adds the Users workbook + PBKDF2 sign-up/sign-in scaffold on top of `datagol-app-auth`.
- **`datagol-internal-app-auth`** — runs when §3 = Mode 3. Replaces `datagol-app-auth`'s provisioning entirely; login form captures bearer from DataGOL IDP, no service token.
- **`datagol-human-in-the-loop`** — after the interview's summary, the platform's
  hard gates kick in for `datagol_create_*` and `bash`.
- **`datagol-workbook-design`** — once you know the entities, this skill tells you
  how to shape them.
- **`datagol-create-links`** — for translating "X has many Y" into LINK columns.
- **`datagol-app-development`** — for turning the data model into a CRUD UI. Its prerequisite block forks on the §3 auth mode.
- **`datagol-create-agent`** + **`datagol-agent-chat-ui`** — if the interview reveals the
  user wants a conversational interface over the data.
