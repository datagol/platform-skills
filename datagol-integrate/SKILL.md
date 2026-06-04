---
name: datagol-integrate
description: Orientation skill for adding DataGOL agents (chat UI, dashboard, RAG) into an existing user codebase that was opened via "Open repo from GitHub". Use this whenever the sandbox already contains a real `package.json` (i.e., not the blank Vite template) — covers framework detection, picking the right route/mount point, respecting existing styling, and not breaking the user's build. Pairs with `datagol-agent-chat-ui`, `datagol-create-dashboard`, `datagol-create-agent`, and `datagol-app-development` — those skills tell you *what* to scaffold; this one tells you *how* to graft it onto an existing app.
---

# Integrating DataGOL into an existing app

You're operating inside a sandbox that was checked out from a user's real GitHub repo (look for a `.git/` directory, a `.datagol-project.json` whose `source === "github"`, or a non-template `package.json`). **Do not assume a blank canvas.** The user wants DataGOL features grafted onto their existing app, with their existing build, framework, routing, and styling. Your job is to integrate, not regenerate.

This is an *orientation* skill. The actual scaffolds for chat UIs, dashboards, RAG, and CRUD apps live in their own skills (`datagol-agent-chat-ui`, `datagol-create-dashboard`, `datagol-create-agent`, `datagol-app-development`). When any of those skills come up in an existing-codebase context, follow this skill's rules for *how* to apply them.

## First: read the codebase before writing anything

Before drafting any new file:

1. **Read `package.json`.** Identify:
   - Framework: React, Next.js (pages-router or app-router?), Vite, CRA, SvelteKit, Astro, Remix, etc. Look at `dependencies` for `next`, `react-router-dom`, `@remix-run/*`, `astro`, `svelte`, etc.
   - Package manager: presence of `package-lock.json` (npm), `pnpm-lock.yaml` (pnpm), `yarn.lock` (yarn), `bun.lockb` (bun). Use whatever's there; do not switch.
   - Dev script: `scripts.dev` or `scripts.start`. The Codex sandbox is already running it on whichever port the framework chose — don't change the script.
   - Existing DataGOL deps: maybe none, maybe a stale embed. Don't reinstall what's there.

2. **Walk the directory tree** (`ls`, `find`). Note where source files live:
   - `src/`, `app/`, `pages/`, `routes/`, `components/`, `lib/`, etc.
   - Where the **routing entry point** is. Examples:
     - Next.js app router → `app/layout.tsx`, `app/page.tsx`; new pages live at `app/<route>/page.tsx`.
     - Next.js pages router → `pages/_app.tsx`, `pages/index.tsx`; new pages at `pages/<route>.tsx`.
     - Vite + react-router → look for `<Routes>` / `createBrowserRouter` and add a `<Route>`.
     - Astro → `src/pages/*.astro`.
     - SvelteKit → `src/routes/<route>/+page.svelte`.

3. **Identify the styling convention.** Check in this order:
   - **Tailwind CSS:** look for `tailwind.config.{ts,js}`, `postcss.config.*`, or `@tailwind` in any CSS file. If present, use Tailwind classes on every new component.
   - **CSS Modules:** files named `*.module.css` or `*.module.scss`. New components use the same pattern.
   - **styled-components / emotion / CSS-in-JS:** check `package.json` deps. Match.
   - **Plain CSS / SCSS:** look at how existing components import styles.
   - **No convention apparent:** fall back to inline styles (the Codex blank-template default).

4. **Note the TypeScript posture.** `tsconfig.json` strictness, `jsx` setting (`preserve` for Next, `react-jsx` for Vite), import aliases (`@/`), etc. Match.

## Where to mount DataGOL components

The skills that scaffold UI (chat, dashboard, RAG upload, full CRUD) generally produce a top-level component you need to render *somewhere*. The right place depends on framework and intent:

### Next.js (app router)

- Whole new page → `app/<route>/page.tsx`. Add a link to it from the existing nav (find the layout/sidebar component and edit it, don't replace).
- Floating chat button on every page → put the component in `app/layout.tsx`, **wrap it in `"use client";`** at the top of the file (the chat UI uses `useState`, `useEffect`, `postMessage` — none work in Server Components).
- App-router file edits: ALWAYS check whether `"use client"` is needed; if you add browser-only React hooks, prepend it AND mention it in your summary.

### Next.js (pages router)

- New page → `pages/<route>.tsx`. Link from the nav as in app-router.
- Global → put the component in `pages/_app.tsx`'s `<Component>` wrapper.

### Vite + React Router

- Find the router config (`<BrowserRouter>` + `<Routes>` or `createBrowserRouter`). Add a `<Route path="/<route>" element={<NewComponent />} />`. Add a `<Link to="/...">` from the nav.
- **Check the router's `basename`.** The app runs inside the Codex preview iframe at `/preview/<sessionId>/`. If `BrowserRouter` has no `basename` (or `createBrowserRouter` no `basename` option), routes won't match — fix while you're touching the file:

  ```tsx
  <BrowserRouter basename={import.meta.env.BASE_URL}>
  ```

  Vite sets `import.meta.env.BASE_URL` to the `--base` prefix automatically, so the same line works in every session. Always use `<Link to="/foo">` (not raw `<a href="/foo">`) so React Router handles the navigation without a full page reload.

### Astro / SvelteKit / Remix / others

- Add a new file at the framework's idiomatic page path. Cross-link from the nav.
- Astro components that need React must use the `client:load` directive on the React island.

### When in doubt

Add the component as a standalone page (route), not as a global overlay. Easier to reason about, fewer cross-cutting concerns, simpler to remove if the user changes their mind.

## What NOT to change

The user opened *their* codebase to add DataGOL. They did not consent to:

- **Bundler / build switches.** Don't migrate Vite → Next, don't add Webpack configs, don't touch `vite.config.*` / `next.config.*` / `astro.config.*` unless absolutely required (and if so, surface the change loudly).
- **Package manager.** Don't add a different lockfile. Use whichever is in the repo.
- **Global state libs.** Don't introduce Zustand / Redux / Jotai. The DataGOL components are self-contained — keep them so. Use local React state.
- **Styling system.** Don't add Tailwind to a non-Tailwind project, don't add styled-components if they're using CSS Modules, etc.
- **Linting / formatting / TypeScript rules.** Don't relax `strict`, don't disable rules, don't add `// @ts-ignore` to make your code fit. Make the code fit.
- **Existing components.** Don't refactor the user's nav, layout, theme, etc. while adding DataGOL — even if it'd be nicer. Add what's needed; flag any improvements as suggestions.

## Token handshake — same as in the from-scratch skills

The platform's token postMessage handshake is host-framework-agnostic. The Codex preview iframe still posts `{ type: 'datagol_token', token }` to the loaded app's window. The `awaitToken()` snippet from `datagol-agent-chat-ui` / `datagol-create-dashboard` works as-is — drop it into `<src or app or lib>/datagol-token.ts` (whichever path matches the project's convention) and import from your new components.

## Surface mismatches loudly

If something about the user's app forces a deviation from the standard scaffolds:

- **Server Components vs client hooks** — say so, and add `"use client"`.
- **No `package.json scripts.dev`** — the sandbox is in code-only mode (no preview iframe). Tell the user; offer to add a dev script if their framework needs one.
- **TypeScript strict null checks** rejecting your code — fix the code, don't suppress the rule.
- **A naming/path collision** with an existing file — don't overwrite. Pick a different name and tell the user.
- **Missing peer dep** for a needed snippet (e.g., `react-router-dom` if you wanted to add a `<Route>`) — install it via the project's package manager and mention it in your summary, since it'll change `package.json`.

When you finish, summarize:
- What files were added or edited.
- Any package additions.
- Any deviations from the standard scaffold (and why).
- The route / URL where the new feature lives.

## Cross-references

- **`datagol-agent-chat-ui`** — the chat UI scaffold. Apply this skill's rules when grafting it onto an existing app.
- **`datagol-create-dashboard`** — the dashboard scaffold. Same.
- **`datagol-create-agent`** — the agent creation flow (including RAG). The agent itself is server-side; the chat UI for end users is client-side and uses `datagol-agent-chat-ui`.
- **`datagol-app-development`** — generic CRUD UI patterns. Apply this skill's rules when adding to an existing app.
- **`datagol-upload-file`** / **`datagol-index-document`** — RAG building blocks. The UIs they scaffold should also follow this skill's rules in an existing-app context.
- **`datagol-workbook-operations`** — workbook read/write API reference; framework-agnostic.
