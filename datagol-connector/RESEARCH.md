# Research notes — DataGOL connector skills

Background notes captured while designing `datagol-app-auth`, `datagol-connector`, and `datagol-google-connector`. Keep this around as a reference; the SKILL.md files are the authoritative spec.

## Existing skills inventory (`server/skills/`)

`datagol-agent-chat-ui`, `datagol-user-auth`, `datagol-create-agent`, `datagol-create-dashboard`, `datagol-create-links`, `datagol-context`, `datagol-deploy-to-vercel`, `datagol-detailed-plan`, `datagol-download-code`, `datagol-frontend-design`, `datagol-human-in-the-loop`, `datagol-index-document`, `datagol-integrate`, `datagol-interview`, `datagol-push-to-github`, `datagol-ui-generation`, `datagol-upload-file`, `datagol-workbook-design`, `datagol-workbook-operations`. None currently use a `datagol-` prefix; renaming all of them is a flagged follow-up workstream.

Closest analogues for shape and length:

- `datagol-integrate` (~107 lines) — orientation skill for grafting features onto an existing repo. Useful as a *style template*.
- `datagol-create-agent` (~411 lines) — multi-step guided flow ("Step 1 — Are the documents already in DataGOL?"). Good model for the OAuth + workbook + cursor walk-through.
- `datagol-workbook-operations` (~364 lines) — API reference with hard rules at the bottom. The new connector skills lean on this for row writes (and inherit its `whereClause`, `cellValues`, `normalizeDate`, `sanitizeForWrite` conventions).
- `datagol-push-to-github` / `datagol-deploy-to-vercel` — closest external-auth precedents in this repo; both rely on cli-server routes that we're explicitly not adding for Google.

## SKILL.md conventions

- Front-matter with `name` + `description` only. Description is 1–2 sentences and lists trigger phrases inline.
- Body opens with one paragraph stating what the skill is.
- Repeated section names: *When to Use* / *Workflow* / *Hard Rules* / *Cross-References*.
- Code samples in fenced blocks (`bash`, `http`, `json`, `ts`).
- Cross-references at the bottom: `**\`other-skill\`** — short reason`.
- Hard rules are bolded imperatives with a one-line rationale.

## Workbook & workspace API surface

Base URL: `https://be.datagol.ai/noCo/api/v2` (or `https://testing-be.datagol.ai/noCo/api/v2` in test). New auth header for generated apps: **`x-auth-token: <SERVICE_TOKEN>`** — replaces the legacy `Authorization: Bearer ${DATAGOL_TOKEN}` from `localStorage`.

- `POST /workspaces/{wsId}/tables/{wbId}/rows` — single insert; body wrapped in `{ "cellValues": {...} }`.
- `POST /workspaces/{wsId}/tables/{wbId}/rows/bulk` — atomic bulk insert; body is a JSON array of `{ cellValues }` objects.
- `POST /workspaces/{wsId}/tables/{wbId}/cursor` — paginated/filtered query; SQL-like `whereClause` with backtick-quoted column names.
- `PATCH /workspaces/{wsId}/tables/{wbId}/rows/{rowId}` — partial update; also wrapped in `cellValues`.
- DATE columns require ISO 8601 *with* time + tz (e.g. `2026-04-30T00:00:00+00:00`); use `normalizeDate()` from `datagol-workbook-operations`.
- Strip `datagol_*`, `noco_*`, `_position*`, `id`, and link reciprocals before PATCH (`sanitizeForWrite()`).
- `datagol_create_workspace({name})`, `datagol_get_workspace_schema` — Pi tools used during scaffold.

## Service-account provisioning (3-call flow, all bearer-authed)

1. `POST /idp/api/v1/company/serviceAccount` body `{ name, description }` → response `{ id, email: "datagol-xxx@inncretech.iam.serviceaccount.com", ... }`. **No token in this response.**
2. `POST /idp/api/v1/company/serviceAccount/{id}/token/reset?neverExpires=true` → `{ token: "14a09...", neverExpires: true }`. The actual service token.
3. `POST /noCo/api/v2/workspaces/{wsId}/bulkUsers` body `{ "workSpaceUsers": [{ "email": "<svc-email>", "role": "CREATOR" }], "message": "" }` — grants the service-account email workspace access.

After Step 3, the generated app uses `x-auth-token: <SERVICE_TOKEN>` for every DataGOL call. The user's bearer is never touched again from generated code.

## Environment toggle

| Env  | Backend                           | Frontend                          |
|------|-----------------------------------|-----------------------------------|
| prod | `https://be.datagol.ai`           | `https://app.datagol.ai`          |
| test | `https://testing-be.datagol.ai`   | `https://testing.datagol.ai`      |

API paths are identical between envs. **Default `prod`.** Codex switches to `test` only on explicit user phrases ("build in test", "use the test environment", "target testing"). Choice baked into `src/config.ts` as `DATAGOL_ENV` + `DATAGOL_BASE_URL`. Codex announces the picked env at the end of scaffolding so the user can redirect with one phrase.

## OAuth precedent — GitHub (cli-server, *not* used here)

Files: `server/src/lib/github-oauth.ts`, `server/src/lib/github-store.ts`, `server/src/routes/github.ts`, `client/src/pages/App/WorkbenchPanel/PublishMenu.tsx`.

Flow: server-side OAuth with `client_secret` on the cli-server, AES-256-GCM-encrypted credential store at `~/.pi/agent/github-tokens.json`, `setInterval` session sweep alongside.

We are explicitly **not** mirroring this for Google. The DataGOL backend (out-of-repo) hosts the OAuth redirect endpoint and the token broker, so the cli-server isn't extended for Google.

## Background work / scheduling

`server/src/pi/session-registry.ts` uses `setInterval()` (60s default) for session cleanup. That's the only scheduling primitive in this repo — no cron, no Bull, no queue. The connector skill uses the same primitive *in the generated page* (browser-side `setInterval`), with the explicit caveat that polling only runs while the page is open.

## Sandbox & generated-code execution model

Per-session Vite sandbox under `/tmp/datagol-pi-sessions/<sessionId>`, on a per-session port. `.datagol-project.json` records `{ sessionId, name, createdAt, updatedAt, source, github }`. Generated code runs in the **browser iframe**, not a per-user server process.

The legacy auth flow: platform piped `DATAGOL_TOKEN` (user bearer) via `postMessage` into the iframe's `localStorage`. The new flow: `DATAGOL_SERVICE_TOKEN` lives in `.env.local`, surfaced via `import.meta.env.VITE_DATAGOL_SERVICE_TOKEN`.

## Google OAuth client credentials — ownership

User shared an OAuth client from the `datagol-codex-testing` Google Cloud project with `client_id` and `client_secret` (the secret was pasted in chat — must be rotated). Architecturally:

- `client_secret` lives **only** in the DataGOL backend env. Never in this repo, never in generated apps.
- `client_id` lives on the DataGOL backend too — the backend assembles Google's auth URL when the generated app calls `/idp/api/v1/oauth2/{service}/authorize`. The skill does not bake the `client_id`.
- `javascript_origins` (currently `["http://localhost:3000"]`) is irrelevant to the server-flow design — only matters for browser-OAuth flows like GIS, which we explicitly rejected.
- **Authorized redirect URI** registered in Google Cloud must match the DataGOL backend's OAuth callback URL. Confirm before first run.

Hard rule (in `datagol-google-connector`): "Never put any Google OAuth client credentials (id or secret) into a generated app or any file in this repo."

## Connector backend contract — confirmed endpoints

User confirmed: backend is a **token broker**, not a Google-API proxy. The generated app calls Google APIs (Gmail, Calendar) directly with the access token.

### Flow

1. **OAuth start**:

   ```
   GET ${BASE}/idp/api/v1/oauth2/{service}/authorize
       ?sourceType=connector
       &clientRedirectUri=<encodeURIComponent(window.location.href)>
       [&jwtToken=<TBD>]
   ```

   Header: `x-auth-token: <SERVICE_TOKEN>`. `{service}` ∈ `gmail`, `gcalendar`. Backend redirects browser to Google consent.

2. Google → DataGOL backend (the fixed redirect URI registered in Google Cloud — already built). Backend exchanges code with server-held `client_secret`, persists tokens against a new `connectorId`.

3. Backend 302s to `<clientRedirectUri>?connectorId=<id>&status=success` (or `&status=error&error=<msg>`).

4. **Token broker**:

   ```
   GET ${BASE}/connector/api/v1/instance/byId/{connectorId}
   ```

   Header: `x-auth-token: <SERVICE_TOKEN>`. Returns access token + refresh token + expiry + connector metadata (account email, service type). Exact field names TBD — wrap in stable `{ accessToken, refreshToken, expiresAt, accountEmail, serviceType }` shape.

5. Generated app calls Google APIs directly with `Authorization: Bearer <accessToken>`.

6. On expiry: re-call step 4. Backend handles refresh against Google internally.

DataGOL calls: `x-auth-token: <SERVICE_TOKEN>`. Google calls: `Authorization: Bearer <accessToken>`.

### Open contract questions

These are flagged as placeholders inside the skill text and should be confirmed once the backend team can share the spec:

1. **`jwtToken` query param semantics.** The example curls carried the user's JWT in *both* `Authorization: Bearer` and `?jwtToken=`. With `x-auth-token` auth, is `jwtToken` still required? If yes, what does it carry — the service token? A short-lived user JWT? Or is it omitted entirely? The skill scaffolds without it initially; if the backend rejects, add `&jwtToken=<SERVICE_TOKEN>` and try again.
2. **`/connector/api/v1/instance/byId` response shape.** Need the exact field names for access token, refresh token, expires_at, account email, service type. The `getTokens()` wrapper accepts both camel-case and snake-case to be defensive; tighten once confirmed.
3. **Disconnect / revoke endpoint.** Does `DELETE /connector/api/v1/instance/byId/{id}` work? Or is there a separate revocation route? The skill scaffolds against DELETE for now.

## Follow-up workstreams (not in the initial three skills)

- Rename existing skills to `datagol-*` (e.g. `datagol-agent-chat-ui` → `datagol-agent-chat-ui`). Touches every skill's cross-references and any code that references skill names by string.
- Update existing skills (`datagol-workbook-operations`, `datagol-ui-generation`, `datagol-agent-chat-ui`, `datagol-create-dashboard`, `datagol-user-auth`) to reference `datagol-app-auth` and stop documenting bearer-from-localStorage as the default for generated apps.
- Future child skills: `datagol-microsoft-connector` (Outlook + Microsoft Calendar + OneDrive), `datagol-slack-connector`, `datagol-salesforce-connector`. Each is a child of `datagol-connector`.
- A way to surface the generated app's port for the OAuth client's `javascript_origins` if we ever add direct browser OAuth (we deliberately didn't — this is hypothetical).
- A backend-side scheduler so polling can run while the generated app's page is closed. Out of scope for the initial three skills; flagged loudly to the user at scaffold time.
