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
- Any request that would land you in `datagol-workbook-design`, `datagol-ui-generation`,
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

### 2. Workspace target — *MANDATORY*

Every build that creates workbooks, agents, dashboards, or data sources lands in *some* DataGOL workspace. **Never pick or create one silently** — the user owns this decision because workspaces carry permissions, billing, and discoverability implications. Ask exactly this:

> **Which DataGOL workspace should this go into?**
>
> 1. **Existing workspace** — name the one to reuse. (You can call `datagol_list_workspaces` first and show the user the names so they don't have to remember.)
> 2. **New workspace** — propose a sensible name from their intent (e.g. *"Sales CRM"* for a CRM build) and **wait for explicit confirmation** of that name before calling `datagol_create_workspace`.

Capture the chosen workspace `id` (or the agreed name for a new one) and carry it into the plan. Section 1 (Architecture) of `datagol-detailed-plan` must spell it out: *"Target workspace: `<name>` (existing)"* or *"Target workspace: `<name>` (will create)"* — so the user can veto or rename at plan-approval time before any modifying tool runs.

**Never call `datagol_create_workspace` until you have the user's explicit "yes" on a specific name.** Even if the user said "build me a CRM" and you'd normally derive a workspace name from intent, propose it and wait. *"I'll create a workspace called 'Sales CRM' — sound good?"* gets you to the same place without surprising the user.

### 3. User management / auth model — *MANDATORY*

Every app build needs an explicit answer here, because the architecture
forks at this fork. Ask exactly this:

> **Should the app have user accounts (login)?**
>
> 1. **Yes, multi-user with login** — each user signs up / signs in and sees
>    only their own data. Adds a sign-up + sign-in flow, per-row ownership,
>    and a session/auth layer.
> 2. **No, single-user demo** — no login. The whole app runs against one
>    shared DataGOL token; everyone who opens it sees the same data. Fastest
>    path; you can add auth later.
> 3. **Other** — describe (e.g. "shared login for the team, no per-user
>    isolation", "SSO via Google", "view-only public, edit only for me").

Hold the answer in the conversation; the `datagol-detailed-plan` skill will require
it in Section 1 (Architecture). When the answer is **#1** (multi-user with
login), the **`datagol-user-auth`** skill is the canonical implementation —
workbook-backed users with PBKDF2-hashed passwords. Read it to understand
the additional schema (Users workbook with `username`, `password_hash`,
`salt`, `role`) and the additional pages (sign-in / sign-up) that need to
appear in the plan.

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
> **Agent**: Got it. Before I start designing, where should this land —
> a new DataGOL workspace I create for you (I'd suggest *"Sales CRM"*),
> or an existing one I should reuse? I can list your existing workspaces
> if helpful.
>
> **User**: New one, that name's fine.
>
> **Agent**: Perfect. Three core entities for a sales CRM: **Contacts**
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

❌ **Silently picking a workspace.** Even if the user's intent obviously implies a workspace name (*"Sales CRM"* from a CRM build), **propose the name and wait for explicit confirmation** before calling `datagol_create_workspace`. Don't include workspace creation in a tool-call batch that the user hasn't approved. The Section-2 question above is non-negotiable on every build.

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

- **`datagol-human-in-the-loop`** — after the interview's summary, the platform's
  hard gates kick in for `datagol_create_*` and `bash`.
- **`datagol-workbook-design`** — once you know the entities, this skill tells you
  how to shape them.
- **`datagol-create-links`** — for translating "X has many Y" into LINK columns.
- **`datagol-ui-generation`** — for turning the data model into a CRUD UI.
- **`datagol-create-agent`** + **`datagol-agent-chat-ui`** — if the interview reveals the
  user wants a conversational interface over the data.
