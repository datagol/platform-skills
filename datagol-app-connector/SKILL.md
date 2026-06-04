---
name: datagol-app-connector
description: Add a Composio-backed provider integration (Gmail, Slack, Notion, GitHub, HubSpot, Google Calendar, Drive, Sheets, Fireflies, LinkedIn, Amplitude, and ~250 more) to an app by calling a running compose-io backend. Invoke when the user asks to "connect to X", "integrate X into my app", "let users sign in to X and pull/post data", "build an agent that can read/write X", or any similar provider-integration request.
---

# DataGOL App Connector — provider integration via compose-io

This skill is the contract for using the **compose-io backend** to add
provider integrations to an app you're building. The backend is a thin
HTTP service in front of [Composio](https://composio.dev) that handles
auth-config discovery, user OAuth, MCP server provisioning, and direct
tool execution. You — the coding agent — call it; you do not need to
import the `composio` SDK.

Source of truth for endpoint details: `backend/README.md` in this repo
(or the running backend's `/docs` Swagger UI). This skill is the playbook;
the README is the reference.

## When to invoke this skill

Trigger phrases from the user:
- "Add Slack integration"
- "Let users connect their Gmail"
- "Pull HubSpot deals into our DB"
- "Build an agent that can post to Notion"
- "Read my calendar events"
- Anything that requires a user-OAuth'd connection to a third-party SaaS.

If the request doesn't involve a third-party SaaS auth, this skill is the
wrong tool.

## Prerequisites

The user must already have a running compose-io backend:

```
BACKEND_URL=http://localhost:8000   # default; ask the user if different
```

Confirm it's reachable before doing anything else:

```bash
curl -s "$BACKEND_URL/api/health"
# → {"status":"ok"}
```

If it's not running, stop and tell the user to start it (`cd backend &&
uvicorn app.main:app --reload --port 8000`).

The backend itself needs `COMPOSIO_API_KEY` configured — you don't see
that key, the backend uses it server-side. The only exception is the
MCP/agent path (section 5), where the agent's runtime ALSO needs the
key as a transport header.

## Decision tree

Before doing anything, decide which path to take:

```
1. Need an LLM to choose which tool to call (chat UI, autonomous agent)?
   ├─ YES → MCP path (sections 1, 3, 5)
   └─ NO  → Direct execute path (sections 1, 3, 4)

2. Multiple end-users?
   └─ Pass each user's stable id as `user_id` (their email, UUID, etc.).
      Never share user_ids across humans.

3. Need a service-account / "team-shared" data view?
   └─ Use a single synthetic user_id ("acme-corp-shared") for every
      invocation, regardless of the human invoker. See section 6.

4. Does the app need to show, search, filter, refresh, or act on connector data?
   └─ YES → Persist the connector state and synced provider data into
      DataGOL workbooks in the active workspace. This is mandatory for
      generated DataGOL apps. See section 0.
```

## 0. Mandatory DataGOL workbook persistence

For any DataGOL-generated app, connector data must live in **workbooks in
the user's workspace**. Do not leave connector state or fetched provider
data only in React state, localStorage, JSON files, an in-memory cache, or
the compose-io backend. The compose-io backend is only the OAuth/tool-call
bridge; DataGOL workbooks are the app's system of record.

Before scaffolding connector UI or sync code:

1. Read `datagol-app-auth` for the service-token / `x-auth-token` API
   pattern used by generated apps.
2. Call `datagol_get_workspace_schema` for the active workspace.
3. Reuse existing connector-related workbooks when they match the need;
   otherwise create the required workbooks before wiring the UI.
4. Store all durable connector metadata and all provider records the app
   depends on in those workbooks.

Minimum workbook pattern:

### Connected Accounts workbook

Create or reuse a workbook named something like `Connected Accounts` with
columns sufficient to track OAuth state:

- `Provider` — text, e.g. `gmail`, `googlecalendar`, `slack`
- `Provider Label` — text, e.g. `Gmail`
- `Auth Config ID` — text
- `Connection ID` — text, Composio connected-account / connection id
- `User ID` — text, the stable Composio `user_id` used for tenancy
- `Status` — text, e.g. `INITIATED`, `ACTIVE`, `EXPIRED`, `FAILED`, `DISCONNECTED`
- `Connection Request ID` — text, when returned by `/connect`
- `Connected Email` / `Account Label` — text, if available from provider data
- `Last Checked At` — date
- `Last Sync At` — date
- `Last Error` — long text

Rules:

- After starting OAuth, upsert a `Connected Accounts` row with
  `INITIATED`, `auth_config_id`, `connection_request_id`, `provider`, and
  `user_id`.
- After popup/redirect completion and after every `/api/connections` poll,
  upsert the row to the latest backend status. The UI should render
  connection status from the workbook whenever possible, not from a
  transient local variable alone.
- Never store OAuth access tokens, refresh tokens, API keys, or secrets in
  workbooks. Store IDs, status, labels, timestamps, and non-sensitive sync
  metadata only.

### Provider data workbooks

For every provider dataset the app surfaces or uses, create a dedicated
workbook (or reuse an existing one) and sync records into it. Examples:

- Gmail inbox UI → `Gmail Messages`
- Google Calendar agenda UI → `Calendar Events`
- Slack channel browser → `Slack Messages` / `Slack Channels`
- HubSpot CRM views → `HubSpot Contacts`, `HubSpot Companies`, `HubSpot Deals`

Each provider-data workbook should include:

- `External ID` — provider record id; use this for idempotent upserts
- `Provider` — text
- `User ID` — text, matching the Composio tenancy id
- `Connected Account` — LINK to `Connected Accounts` when available
- Source-specific display fields, e.g. subject/from/snippet/date for Gmail,
  title/start/end/calendar for Calendar, channel/text/timestamp for Slack
- `Raw Payload` — long text JSON for fields not modeled yet
- `Synced At` — date

Rules:

- Any direct tool execution that fetches provider data must write the
  returned records into the relevant workbook before or while displaying
  them. Rendering fetched connector data without persisting it is a bug.
- Use `External ID + User ID + Provider` as the logical uniqueness key and
  implement idempotent upsert behavior in the generated app/API helper.
- Keep `Connected Accounts.Last Sync At` and `Last Error` updated after
  sync attempts.
- If creating multiple provider-data workbooks, create LINK relationships
  back to `Connected Accounts` using `datagol_create_link`.
- If the user asks for a read-only connector status page, at minimum persist
  the `Connected Accounts` workbook. If the user asks to pull/sync/show
  provider records, persist those records too.

Generated-app implementation requirements:

- Server/API routes should call the compose-io backend, normalize the
  result, then write rows to DataGOL via the service-token pattern.
- Frontend components should read durable connector status and synced data
  from DataGOL workbooks; they may also optimistically update local state
  during popup/polling, but must refresh from the workbook after completion.
- Do not invent a separate database unless the user explicitly asks for one;
  DataGOL workbooks are the default database for connector integrations.

## 1. Discover available providers

```bash
curl -s "$BACKEND_URL/api/connectors" | jq
```

```python
import httpx
connectors = httpx.get(f"{BACKEND_URL}/api/connectors").json()
for c in connectors:
    print(c["toolkit"], c["auth_config_id"], c["label"])
```

Each entry:

```json
{
  "auth_config_id": "ac_l8luIBcI40SP",
  "toolkit": "gmail",
  "label": "Gmail",
  "logo": "https://logos.composio.dev/api/gmail",
  "description": "Gmail is Google's email service…",
  "auth_scheme": "OAUTH2",
  "is_composio_managed": true,
  "is_disabled": null
}
```

You'll use `auth_config_id` for everything that follows.

**If the provider the user asked for isn't in this list** — go to section 2.

## 2. Adding a missing provider

The standalone backend does NOT expose an endpoint to *create* auth configs
(by design — it's a deployment-time admin task). If the user wants to
integrate a provider that's missing:

1. Stop and tell the user to open https://platform.composio.dev → **Auth
   Configs** → **New auth config**.
2. They pick the toolkit (search e.g. "slack", "notion") and either:
   - Use **Composio-managed** credentials (fastest, fine for dev/testing)
   - Or **BYO** their own OAuth client id/secret from the provider (required
     for production / branded consent screens)
3. Save. Refetch `GET /api/connectors` — the new entry should appear.

This is the one step you cannot automate from the agent side. Hand it off
clearly and resume once they confirm.

## 3. Connect a user

Each end-user must complete OAuth once per provider. Two-stage flow:

### 3a. Get the redirect URL

```bash
curl -s -X POST "$BACKEND_URL/api/connectors/$AUTH_CONFIG_ID/connect" \
  -H 'Content-Type: application/json' \
  -d '{"user_id": "alice@acme.com"}'
# → {"redirect_url": "https://connect.composio.dev/link/lk_...", "connection_request_id": "ca_..."}
```

```python
r = httpx.post(
    f"{BACKEND_URL}/api/connectors/{auth_config_id}/connect",
    json={"user_id": user_id},
).json()
redirect_url = r["redirect_url"]
```

Request body:

```jsonc
{
  "user_id": "alice@acme.com",  // REQUIRED — your stable per-user id
  "mode": "redirect",           // or "popup" — default "redirect"
  "allow_multiple": false       // default false; reuse existing connection
}
```

### 3b. The user visits the redirect URL

They get Composio's hosted consent page, then Google/Slack/etc.'s OAuth
screen. On success Composio redirects to your backend's
`GET /api/connections/callback`, which:
- In `mode=redirect`: HTTP-redirects to `${FRONTEND_URL}/?status=success&…`
- In `mode=popup`: returns a tiny HTML page that postMessages the opener
  window and self-closes. The opener should listen for
  `{ source: "composio-oauth", status, auth_config_id }` on `message` events.

### 3c. Confirm the connection is active

Poll until ACTIVE:

```bash
curl -s "$BACKEND_URL/api/connections?user_id=alice@acme.com" | jq
```

```json
[
  { "id": "ca_…", "toolkit": "gmail", "status": "ACTIVE", "auth_config_id": "ac_…", ... }
]
```

Status meanings:
- `INITIATED` — flow started, OAuth not yet completed
- `ACTIVE` — usable; the only status the next sections work with
- `EXPIRED` — token refresh failed; user must re-connect
- `FAILED` — Composio couldn't complete; user must retry

### Edge cases

- **`allow_multiple=true`**: lets one `user_id` accumulate multiple
  connected accounts under the same `auth_config_id` (e.g. personal +
  work Gmail). But `mcp.generate` and `tools.execute` will silently pick
  one — you can't choose. **Strongly prefer distinct `user_id` strings**
  for multi-account scenarios.
- **Disconnect**: `DELETE /api/connections/{connection_id}` revokes.

## 4. Direct execute (no LLM)

The path for any "I know which call to make": cron jobs, webhook
handlers, dashboard backfills, server-side data fetches.

### 4a. Find the tool slug

```bash
curl -s "$BACKEND_URL/api/toolkits/gmail/tools" | jq -r '.[].slug'
```

Returns up to 500 tools (the backend overrides Composio's default
20-result page cap). Look for the action you need:
`GMAIL_FETCH_EMAILS`, `GMAIL_SEND_EMAIL`, `GOOGLECALENDAR_EVENTS_LIST`,
`SLACK_SENDS_A_MESSAGE_TO_A_SLACK_CHANNEL`, etc.

The same endpoint also returns each tool's `name` (human readable) and
`description` — use those to disambiguate when the user's intent is
vague.

### 4b. Execute

```bash
curl -X POST "$BACKEND_URL/api/tools/GMAIL_FETCH_EMAILS/execute" \
  -H 'Content-Type: application/json' \
  -d '{
    "user_id": "alice@acme.com",
    "arguments": { "max_results": 5, "query": "in:inbox" }
  }'
```

```python
def execute(tool_slug: str, user_id: str, arguments: dict) -> dict:
    r = httpx.post(
        f"{BACKEND_URL}/api/tools/{tool_slug}/execute",
        json={"user_id": user_id, "arguments": arguments},
        timeout=60,
    )
    r.raise_for_status()
    body = r.json()
    if not body.get("successful"):
        raise RuntimeError(body.get("data") or "tool failed")
    return body["data"]

messages = execute(
    "GMAIL_FETCH_EMAILS",
    user_id="alice@acme.com",
    arguments={"max_results": 5, "query": "in:inbox"},
)
```

Response:

```json
{ "successful": true, "data": { "messages": [ ... ] }, "logId": "..." }
```

`arguments` is whatever shape the tool itself accepts — Composio passes
it through to the provider call. Look at the tool's `inputSchema` from
section 4a if you're unsure. Most Google tools take their native REST
shapes (`calendarId`, `maxResults`, `q`, `singleEvents`, etc.).

Toolkit versioning is handled server-side (the backend passes
`dangerously_skip_version_check=True` so the latest version is always
used). You don't need to pin versions.

A canonical example lives at
`samples/direct-execute/list_calendar_events.py`.

## 5. MCP / agent path

The path when an LLM picks tools at runtime — chat assistants,
autonomous task agents, anything where the user's request is in natural
language.

### 5a. Create an MCP server

An MCP server is a *named bundle of (auth_config, allowed_tools)* that
agents connect to. Shared across users, fixed allow-list. Create one
per agent definition / per use-case.

```bash
curl -X POST "$BACKEND_URL/api/mcp-servers?validate_for_user_id=alice@acme.com" \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "team-mail-and-cal",
    "tools": [
      {
        "toolkit_slug": "gmail",
        "auth_config_id": "ac_l8luIBcI40SP",
        "tool_slugs": ["GMAIL_FETCH_EMAILS", "GMAIL_SEND_EMAIL"]
      },
      {
        "toolkit_slug": "googlecalendar",
        "auth_config_id": "ac_pwQZTqsml_XW",
        "tool_slugs": ["GOOGLECALENDAR_EVENTS_LIST"]
      }
    ]
  }'
# → { "id": "f2fbc3b9-…", "name": "team-mail-and-cal", "allowed_tools": [...], ... }
```

**Two valid request shapes — never mix in one server:**

**Shape 1** — every pick has `tool_slugs` → exact allow-list:

```json
"tools": [
  { "toolkit_slug": "gmail", "auth_config_id": "ac_x", "tool_slugs": ["GMAIL_FETCH_EMAILS"] }
]
```

**Shape 2** — every pick omits `tool_slugs` → all tools in those toolkits:

```json
"tools": [
  { "toolkit_slug": "gmail", "auth_config_id": "ac_x" },
  { "toolkit_slug": "googlecalendar", "auth_config_id": "ac_y" }
]
```

Mixing the two (one pick with `tool_slugs`, another without) returns 400
with detail `"Mixed mode not supported…"`. If the user wants
"all of A + specific from B", **create two separate MCP servers**.

`?validate_for_user_id=X` (optional) fail-fasts with 400 if user X is
missing any of the required ACTIVE connections. Use it when the agent
that will invoke the server has a fixed acting user; skip it when the
server will be used by many users.

### 5b. Mint a per-user URL

The same `server_id` serves any number of users — you mint a distinct URL
per user at call time.

```bash
curl -s "$BACKEND_URL/api/mcp-servers/$SERVER_ID/url?user_id=alice@acme.com"
```

```json
{
  "url": "https://backend.composio.dev/v3/mcp/<server_id>?user_id=alice%40acme.com&include_composio_helper_actions=true",
  "user_id": "alice@acme.com",
  "allowed_tools": ["GMAIL_FETCH_EMAILS", ...],
  "auth_config_ids": ["ac_l8luIBcI40SP", "ac_pwQZTqsml_XW"]
}
```

Treat the URL like a session credential — never embed in a browser
bundle. Mint fresh per session (the URL pattern can change).

### 5c. Connect from an agent

Use `langchain-mcp-adapters` (Python) or any MCP-compliant client. The
canonical recipe (Python + LangChain + Claude):

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent
from langchain_anthropic import ChatAnthropic

mcp_url = httpx.get(
    f"{BACKEND_URL}/api/mcp-servers/{server_id}/url",
    params={"user_id": user_id},
).json()["url"]

client = MultiServerMCPClient({
    "composio": {
        "transport": "streamable_http",
        "url": mcp_url,
        "headers": {"x-api-key": COMPOSIO_API_KEY},  # REQUIRED on transport
    }
})
tools = await client.get_tools()
agent = create_agent(ChatAnthropic(model="claude-sonnet-4-5-20250929"), tools)
result = await agent.ainvoke({"messages": [{"role": "user", "content": "..."}]})
```

**MUST KNOW:**
- The `x-api-key: COMPOSIO_API_KEY` header on the streamable_http
  transport is **required** — Composio gates the MCP endpoint with it.
  The URL alone is not sufficient. The agent's runtime needs the key.
  This is separate from how the backend uses its own copy of the key
  server-side.
- The MCP server is **shared** across users; per-user behavior is via
  the URL's `user_id` query param.
- Composio's MCP backend has its own IP allow-list separate from the
  REST API. If you get persistent 5xx from the MCP transport while
  `/api/connectors` works, that's a Composio dashboard config issue.

Full reference implementation:
`samples/langchain-agent/agent.py` in this repo (~150 lines, including a
`current_datetime` tool and a one-shot retry on transient MCP 500s).

## 6. Tenancy patterns

`user_id` is the tenancy unit; it determines whose connected_account
Composio uses at tool-call time. Pick the right pattern up front:

| Pattern | When | `user_id` |
|---|---|---|
| **Per-end-user** | SaaS where each customer connects their own accounts | Their stable id (email, UUID, account_id) |
| **Shared service account** | Admin creates one shared connection; every invoker sees the same data | A synthetic id like `"acme-corp-shared"` |
| **Hybrid** | App authenticates the human, then maps to the right tenancy id | Picked at call time from auth context |

For the **shared service account** pattern: the admin connects toolkits
*as* the synthetic `user_id` (i.e., the human running the OAuth flow
passes that id when starting `/api/connectors/.../connect`). After that,
any caller passing the same `user_id` sees the admin's connected
accounts.

Composio doesn't enforce who's allowed to use which `user_id` — your app's
authz layer is what gates that.

## 7. Errors and what to do

| status | detail snippet | likely cause | next action |
|---|---|---|---|
| `400` | "no ACTIVE connection for …" | User hasn't OAuth'd yet | Section 3 — connect first |
| `400` | "Mixed mode not supported…" | One MCP create request mixed Shape 1 and Shape 2 picks | Split into two servers |
| `400` | "Every tool pick must include toolkit_slug and auth_config_id." | Frontend bug, missing fields | Check your request body |
| `401` | "Invalid API key" or "not authorized from current IP address" | Backend's `COMPOSIO_API_KEY` wrong, revoked, or IP-blocked | Tell the user to fix in Composio dashboard → Settings → API keys → IP Allowlist |
| `404` | "Agent not found" / "MCP server not found" | Bad id | Re-list and retry |
| `502` | "Composio MCP create failed: … allowed_tools contains invalid slug …" | Mis-typed tool slug | Re-fetch `/api/toolkits/{slug}/tools` for the canonical spelling |
| `502` | "Composio MCP create failed: …" (generic) | Composio outage / wrong auth_config_id for the toolkit | Retry once; check the `auth_config_id` matches the `toolkit_slug` |

**Runtime errors inside the agent loop** (during `client.get_tools()` /
tool invocation, not from our backend):
- `500 Internal Server Error` from `backend.composio.dev/v3/mcp/...` — Composio's MCP backend hiccups occasionally. Retry once with a ~1.5s backoff. The reference agent in `samples/langchain-agent/agent.py` does this.
- `Tool 'X' not found` — the slug isn't in the server's `allowed_tools`. Either add it (create a new server — allow-lists are immutable) or pick a slug that is.

## 8. Quick reference

```
GET    /api/connectors                              list providers
POST   /api/connectors/{auth_config_id}/connect     start user OAuth   body: { user_id, mode?, allow_multiple? }
GET    /api/connections                             list user's accounts  ?user_id=… [&auth_config_id=…] [&toolkit=…]
DELETE /api/connections/{connection_id}             revoke a connection
GET    /api/connections/callback                    Composio hits this — don't call directly

GET    /api/toolkits/{slug}/tools                   list tool slugs for a toolkit
POST   /api/tools/{tool_slug}/execute               run a tool directly   body: { user_id, arguments }

GET    /api/mcp-servers                             list MCP servers   [?auth_config_id=…]
POST   /api/mcp-servers                             create server   body: { name, tools: [{ toolkit_slug, auth_config_id, tool_slugs? }] } [?validate_for_user_id=…]
GET    /api/mcp-servers/{server_id}                 get one server
DELETE /api/mcp-servers/{server_id}                 delete server
GET    /api/mcp-servers/{server_id}/url             mint per-user MCP URL   ?user_id=…
```

## 9. Pointers

- Full endpoint reference: `backend/README.md` in this repo, or
  `$BACKEND_URL/docs` on a running backend.
- LangChain agent reference: `samples/langchain-agent/agent.py` +
  `samples/langchain-agent/README.md`.
- Direct REST reference: `samples/direct-execute/list_calendar_events.py`
  + `samples/direct-execute/README.md`.
- Composio docs: <https://docs.composio.dev>.
- Composio dashboard (where humans add auth configs): <https://platform.composio.dev>.

## Portability

This skill lives at `<compose-io repo>/.claude/skills/connector-integration/SKILL.md`,
so it's auto-discovered when working inside the repo. To use it from
another project:

- **Copy**: `cp -r <compose-io>/.claude/skills/connector-integration ~/.claude/skills/`
  to make it available across all your Claude sessions.
- **Symlink** (stays in sync with the repo):
  `ln -s <compose-io>/.claude/skills/connector-integration ~/.claude/skills/connector-integration`

The skill is self-contained — once installed it doesn't read other files
from the compose-io repo at runtime; it only points at them as
human-readable references.
