---
name: datagol-deploy-to-vercel
description: Deploy the generated DataGOL web app from the sandbox folder to Vercel. Triggered when the user asks to "deploy", "publish to vercel", "make it live", or similar. The Codex platform also has a "Publish" UI button that does the same thing.
---

# Deploy Generated App to Vercel

The generated app is normally a standard Next.js + React project that Vercel auto-detects
(build = `next build`). If the current repo is a legacy/static app, Vercel will
use the framework detected from `package.json`. The Codex platform exposes two
equivalent paths:

1. **UI button** — Workbench → **Publish** dropdown → **Vercel**. Opens a
   modal asking for project name, streams `vercel` CLI output, returns the
   `*.vercel.app` URL on success.
2. **Direct API** — `POST /api/session/{sessionId}/publish/vercel` with
   `{ projectName, prod? }`. Streams Server-Sent Events. Use from `bash` when
   the user asks the agent to handle it.

**Tell the user about the UI button first** — it's faster.

## Prerequisites (verify before triggering)

The user must have the Vercel CLI installed and authenticated. Check with:

```bash
vercel whoami
```

If `vercel` isn't installed: tell them to run `npm i -g vercel`.
If they're not authenticated: tell them to run `vercel login` (interactive
browser flow) or set `VERCEL_TOKEN=<token-from-vercel.com/account/tokens>`
in the server's env.

## Pre-deploy checklist

Before deploying, **always confirm with the user**:

1. **App actually builds.** If you've made recent edits, the dev preview is
   one signal; for confidence, run `npm run build` in the sandbox first
   (`bash` gated). Vercel will reject a broken build.
2. **`DATAGOL_TOKEN_PLACEHOLDER` is still in place** in any API client file.
   The Vercel deploy is publicly accessible — bundling a real token would
   leak it. The placeholder is fine; users replace it after deploy via
   Vercel env vars.
3. **Project name** — sensible default: `datagol-app-<8-char-prefix>` of the
   sessionId. The user usually wants something more descriptive (it becomes
   `<projectName>.vercel.app`).
4. **Production vs preview.** Default is `prod: true` (production). For dry
   runs, suggest `prod: false`.

## Workflow

1. Confirm the user wants to deploy. Walk through the checklist.
2. Point them at the **Publish → Vercel** UI button, **or** call the API
   from `bash`:

   ```bash
   curl -N -X POST -H 'Content-Type: application/json' \
     -H "Authorization: Bearer $DATAGOL_TOKEN" \
     -d '{"projectName":"my-app","prod":true}' \
     http://localhost:3001/api/session/<sessionId>/publish/vercel
   ```

3. Stream the output. The final stdout line is typically the deployment URL
   (`https://<project>-<hash>.vercel.app`). The server extracts and returns
   it as `{type:'done', url}`.

4. Suggest next steps:
   - Open the URL in a new tab.
   - Set environment variables on Vercel for the real DataGOL token (so the
     deployed app can call `be.datagol.ai` without exposing the token in
     source).
   - Connect the GitHub repo (if they did `datagol-push-to-github` earlier) so future
     pushes auto-deploy.

## Configuration tips (give to the user as needed)

- **Custom domain** — Vercel dashboard → Settings → Domains.
- **Env vars** — Vercel dashboard → Settings → Environment Variables. Add
  `DATAGOL_TOKEN` and update the app config/data layer to read it through the
  framework-appropriate environment mechanism (for Next.js client-side values,
  use a `NEXT_PUBLIC_` variable only when the value is safe to expose; otherwise
  route sensitive calls through server-side code).
- **Re-deploy** — `vercel deploy --prod` from the sandbox dir, or push to
  GitHub if the project is connected.

## Errors & how to handle them

| Error | What to tell the user |
|---|---|
| `Vercel CLI not found` | `npm i -g vercel` then retry. |
| `Error: No existing credentials found` | `vercel login` interactively or set `VERCEL_TOKEN`. |
| `Build failed` | Check the streamed log for the actual `next build` / `tsc` error. Usually a missing import or a type mismatch in generated code. Fix in source, retry. |
| `Project name already exists in your scope` | Pick a different name or accept the auto-generated suffix. |

## Anti-patterns

❌ **Deploying with a leaked real token in source** — the deploy is public.
Always verify the placeholder is intact.

❌ **Skipping the build check** when the user has made manual edits to the
generated code. A failed deploy is frustrating; a failed local build is
quick to diagnose.

❌ **Hardcoding secrets into client-side code** expecting Vercel to hide them. Next.js only exposes variables prefixed with `NEXT_PUBLIC_` to the browser, and those are public by design. Keep real secrets server-side or in the platform's approved auth/config pattern.

## Cross-references

- **`datagol-push-to-github`** — usually run first; pushes the source. Vercel can
  also deploy directly from a GitHub repo (no CLI needed) — mention this if
  the user prefers a connected workflow.
- **`datagol-ui-generation`** — controls what gets generated (and thus deployed).
- **`datagol-human-in-the-loop`** — `bash` is hard-gated; the user sees an approval
  card before the curl fires.
