---
name: datagol-image-to-ui
description: When the user pastes or drops an image of a design (Figma export, mockup, screenshot of another site, hand-sketch, whiteboard photo, etc.) and asks the agent to build UI from it, treat the image as **authoritative design intent**. Triggered by "build this design", "match this mockup", "implement this screenshot", "make it look like this image", "build a page like this", "design like this", or *any* build / scaffold request where at least one image is attached. Reads the image, summarizes what it sees to the user, asks at most one clarifier if ambiguity is high, then scaffolds one page or component first, renders it for verification, and only then continues to the rest. Pairs with `datagol-app-development` (file structure + API wiring) and `datagol-frontend-design` (typography, motion, palette).
---

# DataGOL Image-to-UI

The user just attached an image and asked you to build something. That image is your **visual specification**. Treat it the way a developer treats a design spec from a designer: read it carefully, summarise what you see, confirm load-bearing details, then build a small piece first and verify before scaffolding everything.

This skill governs the *visual* contract. It does NOT replace `datagol-interview` or `datagol-detailed-plan` — those still gate workspace selection, auth model, and data model. The image gives you the answer to *"what should it look like?"* and nothing else.

## When to use

**Trigger:**
- An image is attached **and** the user's message is build-ish — "build", "create", "make", "scaffold", "implement", "design", "page like this", "match this", "based on this image", etc.
- Multiple images attached are fine — typically a multi-page flow (login, dashboard, detail) or a brand sheet. Handle one image at a time; ask which one to start with if intent isn't obvious.

**Do NOT use this skill when:**
- The image is *incidental* — a screenshot of an error message they want debugged, a chart they want analysed, a logo for a different reason. Differentiate by the user's text, not by image presence alone.
- The user is *iterating on the current preview* and the image is just a "make it like this" pointer for a tweak (e.g. *"change the header to look like this"*). In that case, smaller-scope edits via `edit` tool, no full scaffold protocol needed.

## How this fits with the other build skills

| Skill | What it supplies |
|---|---|
| `datagol-image-to-ui` (this) | The *visual* contract — what the UI must look like |
| `datagol-frontend-design` | The *aesthetic conventions* — typography stack, motion language, spacing grammar, palette discipline |
| `datagol-app-development` | The *file scaffold + API wiring* — Next.js + React layout, App Router structure, and `lib/datagol.ts` data layer |
| `datagol-interview` | Workspace target, auth model, data-model questions — still mandatory |
| `datagol-detailed-plan` | The user-approved plan for anything non-trivial — still mandatory |

The image is **not** a substitute for the interview. If the user pastes a CRM dashboard mockup and says *"build this"*, you still need to ask which workspace it goes into, whether it needs login, what entities are real-vs-placeholder. The image just removes the *"what should it look like?"* burden.

## The 4-step protocol

### 1. Read the image, echo what you see — *before any file write*

Write back to the user a short, structured description of what you see. This catches misreads before they cost a scaffold round-trip. Keep it 6 lines or fewer. Use this exact shape:

> **Layout:** *Top nav (60px), left rail (240px, dark surface), main content (3-col grid, 24px gutters). No footer.*
>
> **Primary components:** *KPI card row (top), data table (middle), trend chart (right column). Modal pattern for create/edit.*
>
> **Palette:** *background `#0F172A`, surface `#1E293B`, accent `#22D3EE`, text `#E2E8F0`, secondary text `#94A3B8`, border `#334155`.*
>
> **Typography:** *Sans-serif, geometric (Inter-class). ~14px body / 20px section heading / 28px page title. Weights 400 / 500 / 700.*
>
> **Spacing rhythm:** *Tight (8 / 12 / 16) for compact data; loose (24 / 32) for section gaps.*
>
> **Notable details:** *Rounded-md corners throughout. No drop shadows; subtle 1px borders. Status pills with coloured dot + light fill.*

Then say one sentence about what you'll do first (Step 3), and **pause**.

### 2. Ask one clarifier — only if a load-bearing detail is unclear

ONE question, not five. Skip this step entirely when the image is unambiguous. Examples worth asking:

- *"The numbers in the KPI cards (1,247 / 89% / $12.5K) — placeholders, or do you want them wired to real workbook data?"*
- *"The dark theme — your default, or should I also scaffold light?"*
- *"I see a sign-in screen — multi-user app? (Affects whether I scaffold `datagol-user-auth`.)"*
- *"The chart on the right — Recharts-style line chart with real data, or a static SVG that just looks like the screenshot?"*

Things that are NOT worth asking (decide them yourself):
- Exact pixel values that the image already shows.
- Whether to use Tailwind or inline styles (defer to host app conventions via `datagol-integrate`, or inline styles for a fresh sandbox per `datagol-app-development`).
- Whether to add hover states (yes, always — `datagol-frontend-design` covers this).

### 3. Build small, verify, expand

**Pick one page or one major component to scaffold first.** The most representative one — the landing / dashboard / list view, not an empty settings page. The point is to validate your design read against a working render before committing to scaffolding the rest.

Steps within step 3:

1. Write your prose summary of what you're about to build (one short paragraph, no headings).
2. Then call the file-write tools. For a *page in a generated app*, this is `App.tsx` + the one chosen page file (e.g. `pages/Dashboard.tsx`). For a *single component to render inline in chat*, this is a single `.html` file.
3. The inline HTML preview in chat renders any `.html` file you write directly under its tool row — use this for chart-style or single-component visualizations. For multi-page apps, the Preview tab in the workbench is the verification surface.
4. After the first render, **pause** and ask: *"Does this match what you had in mind? Once you confirm, I'll scaffold the rest."*

The temptation will be to scaffold 6 pages from one screenshot in one tool sequence. Don't. The whole point of this protocol is to fail fast on misreads.

### 4. Continue only after explicit confirmation

- On *"yes" / "looks good" / "ship it"* — scaffold the remaining pages, applying the same visual contract.
- On *"no" / corrections* — acknowledge what was off, adjust the design read (re-emit a short bullet of what changed in your interpretation), make a small targeted change, render it again, re-confirm.
- On the user adding *another* image after the first verification, treat it as a refinement or an additional view — read it the same way, apply the same protocol.

## Composition rules (inherits from `datagol-data-query`)

Same rendering discipline as elsewhere in the codex:

1. **Prose first, files last.** Write your design summary and what-you'll-do-next *before* calling any file-write tool. Don't interleave.
2. **No surprise side-effects.** A pasted design alone doesn't authorize you to call `datagol_create_workspace`, register data sources, or create agents. The interview gates those decisions; the image only fills the visual gap.
3. **One screenshot, one starting page.** Don't fan out across multiple file writes in step 3.

## Hard rules

- **Describe before building.** No file write happens before §1 has been written and the user has had a moment to react. Even on *"build it now"* — the 6-line description is 10 seconds for the user to read and could save 5 minutes of mis-scaffolded code.
- **One image, one page first.** Do not generate every page from a single screenshot in one shot. Step 3 produces *one* page; the rest waits for confirmation.
- **No pixel-perfect promises.** Image interpretation is good but not vector-tracing. Promise *directional match*, not *one-to-one reproduction*. Say so explicitly in your description: *"I'll match the layout, palette, and component shape; small details (exact corner radii, micro-spacing) will be approximate."*
- **Workspace and auth still need an interview.** If the image implies workbooks (a table of customers, a list of orders, etc.) and the user wants a real working app, trigger `datagol-interview` for the data-model questions before wiring `api.ts`. The image does not answer §2 (workspace target) or §3 (auth model) of the interview — those remain mandatory.
- **Placeholder data is placeholder.** Names like "Acme Corp", "John Doe", "$1,247.32", "user@example.com" visible in the image are *placeholders*, not real data. Always ask whether the user wants you to keep those values as static fixtures (for a demo) or wire to real workbook data (for a real app).
- **Don't OCR a wall of text.** If the image contains a long paragraph (e.g. a screenshot of an article), don't try to transcribe it into the component. Use Lorem-ipsum placeholders or summarise — and tell the user that's what you did.
- **Logos and brand assets.** Don't try to recreate a logo as code. If the image has a brand logo, suggest the user upload the actual asset (PNG/SVG) and reference it via an `<img>` tag. Don't auto-generate a fake logo.
- **Respect the host app's conventions** when grafting into an existing repo. Per `datagol-integrate`, if the host uses Tailwind, your design read needs to map palette + spacing back to Tailwind tokens, not inline styles. If they use CSS Modules, write CSS Modules. The image is the *visual* spec; the host is the *implementation* spec.
- **Transcript replay caveat.** Image content isn't persisted in `.datagol/transcript.jsonl` (only the user's text is). When the user reopens a project, your chat history is restored but the original screenshot is not in your context. If the user then says *"keep iterating on that design"*, ask them to re-paste the reference — don't pretend to remember it.

## Examples

### Good — clean dashboard mockup, fast path

> **User:** [pastes a screenshot of a sales dashboard] *Build me this.*
>
> **Agent:**
> Here's what I see:
> - **Layout:** Top nav (60px), left sidebar (220px) with section links, main area (3-col KPI row + full-width table).
> - **Primary components:** 3 KPI cards (Revenue / Customers / Conversion), data table with sortable headers, status pills (Active green, Trial blue, Churned grey).
> - **Palette:** background `#FFF`, surface `#F8FAFC`, primary `#3B82F6`, text `#0F172A`, muted `#64748B`, border `#E2E8F0`. Light theme.
> - **Typography:** Sans-serif, ~14px body / 28px page title. Weights 400 / 500 / 600.
> - **Spacing:** 16/24/32 rhythm. Plenty of whitespace.
> - **Notable details:** Rounded-lg corners, subtle 1px borders, no shadows.
>
> Quick question before I start: **the table data (Acme Corp, $124K, etc.) — placeholders, or do you want me to wire it to a real Customers workbook?**
>
> *(waits for answer)*
>
> **User:** *Placeholders for now, just want to see the design.*
>
> **Agent:** *(writes one prose paragraph + Dashboard.tsx + App.tsx, then pauses)* I've scaffolded the dashboard with hard-coded placeholder rows. Check the Preview tab — does this match what you had in mind? Once you confirm, I'll add the sidebar nav targets (Customers / Orders / Settings).

### Bad — what to avoid

❌ User pastes screenshot, agent immediately writes 5 files for a 5-page app, says *"all done!"* — skipped the description step, didn't pause for verification, scaffolded too much from one screenshot.

❌ User pastes screenshot of a sign-in form, agent silently scaffolds `datagol-user-auth` + a Users workbook + a sign-up page — skipped the interview's mandatory questions (workspace target, "is this multi-user?").

❌ Agent's description is 30 lines, describing every shadow and gradient — too long; the user skims past it and the protocol stops being useful.

❌ Agent's description includes guesses about *intent* ("I think this is for a B2B SaaS sales team") rather than visible facts. Stick to what's in the image.

## Cross-references

- **`datagol-app-development`** — file scaffold, API wiring, framework rules (Next.js + React for fresh apps; defer to host for grafted apps).
- **`datagol-frontend-design`** — typography stack, motion conventions, palette discipline, spatial composition. Apply these *within* the visual contract the image dictates.
- **`datagol-interview`** — workspace + auth + data model. The image doesn't answer these; the interview does.
- **`datagol-detailed-plan`** — required for non-trivial multi-page builds.
- **`datagol-user-auth`** — when the image shows a sign-in / sign-up screen and the user confirms multi-user.
- **`datagol-integrate`** — when the build target is an existing user repo; map the visual contract into the host's styling system.
- **`datagol-data-query`** — composition rules (prose first, files last) come from here; apply the same discipline.
