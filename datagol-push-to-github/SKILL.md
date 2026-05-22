---
name: datagol-push-to-github
description: Push the generated app in the sandbox folder to a new (or existing) GitHub repository. Triggered when the user asks to "push to github", "create a repo", "publish my code", or similar. The Codex platform also has a "Publish" UI button that does the same thing — point the user there if they prefer clicking.
---

# Push Generated App to GitHub

The generated app lives in the sandbox cwd (where `write` / `edit` have been
landing files). There are two paths the user can take:

### Preferred: OAuth ("Connect GitHub")

The Codex platform has a one-click GitHub integration:

1. **UI button** — Workbench → **Publish** dropdown → **GitHub**. If the user
   isn't connected yet, this triggers the OAuth flow (redirects to GitHub,
   asks for repo permission, stores the token server-side keyed by the
   user's email). Once connected, the same dropdown shows their GitHub
   avatar and **@username**, and clicking it opens the publish modal.
2. **Status check API** — `GET /api/github/status` → `{connected, login?, avatarUrl?}`.
   Use this to know whether the user has connected before calling publish.
3. **Publish API** — `POST /api/github/publish` with
   `{ sessionId, repoName, isPrivate?, description? }`. Streams Server-Sent
   Events (`{type:'log',stream,line}` then `{type:'done',url}` or
   `{type:'error',message}`). Uses the OAuth token under the hood — no CLI
   needed on the user's machine.

**Always tell the user to use the UI button.** It's the fastest path, and
the Connect-GitHub flow is a single click.

### Fallback: local `gh` CLI

For users who don't want OAuth (or whose deployment doesn't have OAuth
configured), there's an older endpoint that shells out to the locally-installed
`gh` CLI:

`POST /api/session/{sessionId}/publish/github` with `{ repoName, isPrivate?, description? }`.

This requires `gh` installed (`brew install gh`) and authed (`gh auth login`
or `GH_TOKEN` env). Prefer the OAuth path unless the user explicitly asks
for the CLI fallback.

## Prerequisites

**For OAuth (preferred):**
- The server must have `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET` set in
  its env (one-time operator setup).
- The user clicks **Connect GitHub** once. After that, the token is stored
  server-side and reused.
- If `GET /api/github/status` says `connected: false`, tell the user to
  click the GitHub option in the Publish dropdown to start the OAuth flow.
  Don't try to do the OAuth flow from chat — it's a browser redirect.

**For CLI fallback:**
- `gh auth status` shows logged in.
- If `gh` isn't installed: `brew install gh`.
- If not authenticated: `gh auth login`.

## Pre-push checklist

Before pushing, **always confirm with the user**:

1. **Token placeholder still in place.** Generated apps include
   `DATAGOL_TOKEN_PLACEHOLDER` in `src/lib/api.ts` (or wherever the API client
   lives). If a real token has leaked into the code, **stop and tell the
   user** — pushing it to a public repo is a credential leak.
2. **`.gitignore` is sane.** The publish API writes one if missing
   (excluding `node_modules`, `dist`, `.vite`, `.env*`). If you've added any
   custom secret files, mention them.
3. **Repo name** — sensible default: `datagol-app-<8-char-prefix>` of the
   sessionId. The user usually wants something more descriptive.
4. **Private vs public** — default to private. Only push public if the user
   explicitly asks.

## Workflow

1. Confirm the user wants to push (walk through the pre-push checklist).
2. Tell them to click **Publish → GitHub** in the workbench:
   - If they aren't connected yet, the dropdown's "GitHub" item kicks off
     the OAuth flow. After they approve on github.com, the browser comes
     back and the dropdown shows their avatar + username.
   - Click again, fill in the repo name + private/public toggle, click
     **Push**. The modal streams progress and shows the final repo URL.
3. If the user explicitly wants the agent to drive it from chat (rare),
   verify connection first then POST:

   ```bash
   # Check connection
   curl -s -H "Authorization: Bearer $DATAGOL_TOKEN" \
     http://localhost:3001/api/github/status

   # Push (OAuth-based, no gh CLI needed)
   curl -N -X POST -H 'Content-Type: application/json' \
     -H "Authorization: Bearer $DATAGOL_TOKEN" \
     -d '{"sessionId":"<id>","repoName":"my-app","isPrivate":true}' \
     http://localhost:3001/api/github/publish
   ```

4. Surface the final repo URL to the user as a clickable link.
5. Suggest next steps: opening the repo, adding collaborators, or running
   **Vercel** (see `datagol-deploy-to-vercel` skill) for a live URL.

## Errors & how to handle them

| Error | What to tell the user |
|---|---|
| `GitHub not connected. Click "Connect GitHub" first.` | Open the Publish dropdown → click **GitHub** → approve on github.com. |
| `GitHub OAuth not configured` | Operator-side issue — the server needs `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET` in its env. Tell the user to ask whoever runs the server. |
| `GitHub repo create failed (422): name already exists` | Pick a different name. We don't auto-pick a suffix because pushing into someone else's repo would be confusing. |
| `git push exited with code 128` | Usually means the OAuth token doesn't have `repo` scope. Disconnect + reconnect — the scope is requested on every authorize. |
| `git ... exited with code N` (other) | Look at the streamed log lines — the actual error is one or two lines above the exit. Common: trying to push to a non-empty repo (we create empty ones). |
| (CLI fallback) `gh CLI not found` | `brew install gh`, or use the OAuth path instead. |

## Anti-patterns

❌ **Pushing without confirming the token placeholder** — biggest risk. Always
verify `DATAGOL_TOKEN_PLACEHOLDER` (or equivalent) is still in source before
push.

❌ **Defaulting to public** — assume the user wants their work private unless
told otherwise.

❌ **Re-running the push tool to "fix" a non-deterministic error** — read the
log, name the actual problem, then retry deliberately.

❌ **Showing the gh access token** in chat. If you somehow surface it from
the env, redact it.

## Cross-references

- **`datagol-deploy-to-vercel`** — natural follow-up; once code is on GitHub, deploy
  it.
- **`datagol-download-code`** — for users who don't want a remote repo, just a ZIP.
- **`datagol-human-in-the-loop`** — `bash` is hard-gated; the user will see an
  approval card before the curl runs.
