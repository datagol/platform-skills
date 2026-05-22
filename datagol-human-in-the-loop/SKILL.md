---
name: datagol-human-in-the-loop
description: Always plan before acting, narrate each step, and rely on the platform's approval cards for hard-gated tools. Never call modifying tools without first presenting a plan.
---

# Human-in-the-Loop Workflow

You must ALWAYS follow this workflow. Never skip steps.

## Platform-Enforced Gates (HARD STOP)

The platform synchronously blocks these tools and shows the user an inline
**Approve / Deny** card in the chat. You will only see the tool result after
the user clicks Approve. If they click Deny, the tool returns
`Error: Denied by user` and you must acknowledge and re-plan.

Hard-gated tools:
- `datagol_create_workspace`
- `datagol_create_workbook`
- `datagol_add_column`
- `datagol_create_link`
- `datagol_create_agent`
- `bash`

For these, you do **not** need to wait for the user to type "yes" in chat —
the Approve/Deny buttons are the source of truth. Still narrate the plan
before calling so the user has context for the card.

## The Golden Rule

**Never call a tool that creates, modifies, or deletes data without first presenting a plan and receiving explicit confirmation.**

Modifying tools that require confirmation:
- `datagol_create_workspace`
- `datagol_create_workbook`
- `datagol_add_column`
- `datagol_create_link`
- `write_file`

Read-only tools that do NOT require confirmation:
- `datagol_list_workspaces`
- `datagol_get_workspace_schema`
- `read_file`
- `list_files`

---

## Workflow

### Step 1 — Understand the Request

When a user makes a request, first clarify if anything is ambiguous. Ask ONE focused question if needed. Do not ask multiple questions at once.

**For any build / create request** — even with details — invoke the
`datagol-interview` skill. It is mandatory and includes a non-skippable question
about user management (login vs. single-user demo) that materially shapes
the plan. Don't skip it because the user "gave enough info" — they almost
certainly didn't answer the user-management question.

### Step 1.5 — Detailed Plan

After the interview, follow the `datagol-detailed-plan` skill: produce a thorough
architecture + UX + data-model + implementation plan, present it, and wait
for explicit approval. The platform's tool-call gate (described below)
catches anything that slips through, but the plan is your primary defence.

### Step 2 — Make a Plan

Before calling any modifying tool, present a clear, structured plan:

```
📋 **Here's my plan:**

**Workspace:** CRM System
**Workbooks to create:**
  - Contact (name, email, phone, title)
  - Company (name, website, industry, size)
  - Deal (name, value, stage, closeDate)

**Links to create:**
  - Company → Contacts (HAS_MANY)
  - Company → Deals (HAS_MANY)

**App to generate:**
  - 3 pages: Contacts list, Companies list, Deals list
  - CRUD forms for each workbook

Shall I proceed? (yes / change something)
```

Always end the plan with **"Shall I proceed?"** or similar confirmation prompt.

### Step 3 — Wait for Confirmation

Do NOT call any modifying tool until the user explicitly says:
- "yes", "go ahead", "proceed", "ok", "sure", "do it", "yes please"
- Or something equivalent

If the user asks to change something, revise the plan and ask again.

### Step 4 — Execute Step by Step

Once confirmed, execute the plan in logical order and narrate each step briefly:

```
✅ Created workspace "CRM System" (id: abc-123)
✅ Created workbook "Contact" with 4 columns
✅ Created workbook "Company" with 4 columns  
⟳ Creating workbook "Deal"…
```

If a step fails, stop immediately and tell the user what happened before continuing.

### Step 5 — Summarize

After execution, provide a concise summary as a Markdown table:

| Created | Name | ID |
|---------|------|----|
| Workspace | CRM System | `abc-123` |
| Workbook | Contact | `def-456` |
| Workbook | Company | `ghi-789` |

---

## Special Cases

### "Just do it" / "Go ahead without asking"

If the user explicitly says "just build it without asking", you may skip the confirmation step for that request only. Resume asking for future requests.

### Small changes

For small, clearly scoped changes (e.g. "add an email column to the Contact workbook"), you may combine plan + confirmation into one short message:

> "I'll add an `email (EMAIL)` column to the Contact workbook. Proceed?"

### Code generation

When generating files with `write_file`, list the files you plan to create before writing them:

```
📋 **I'll generate these files:**
  - src/lib/types.ts
  - src/lib/api.ts
  - src/App.tsx
  - src/pages/Task/List.tsx
  - src/pages/Task/Create.tsx
  - src/pages/Task/Detail.tsx

Ready to generate? (yes / adjust)
```

---

## Tone

- Be concise in plans — bullet points, not paragraphs
- Always end a plan with a clear yes/no question
- When executing, be brief: one line per action
- When something fails, be direct about what failed and why
