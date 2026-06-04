---
name: datagol-create-agent
description: How to create a custom AI agent on DataGOL — including RAG agents that retrieve from workspace documents, agents that read/write workbooks via MCP, and capability-based agents (web search, browser, deep research). Triggered by "create an agent", "build a chatbot", "create a RAG agent", "create an assistant that knows my docs", "agent over my files", "agent over my workbooks", or any request to package a conversational AI on DataGOL.
---

# Creating a Custom Agent on DataGOL

Use this when the user asks to "create an agent", "build a chatbot", "make an assistant", or anything that describes a packaged conversational AI on DataGOL.

> **Prerequisite: run `datagol-interview` first.** Even for "just an agent over my
> data", you need the mandatory user-management answer (does each end-user
> sign in? does the agent see only their rows? or is it a shared agent over
> shared data?), the workbook IDs to wire as MCP servers, and the system
> prompt scope. After the interview, present a `datagol-detailed-plan` (Section 5
> covers the agent specifically) before calling `datagol_create_agent`.

## When to Use

- The user wants a published agent inside DataGOL (not a generated React app — that's the `datagol-app-development` skill).
- They want the agent to read/write specific workbooks (Contacts, Deals, etc.) via MCP.
- They want **RAG over documents** — the agent should answer from PDFs/DOCX/etc. uploaded into a workspace. Use a `WORKSPACE_DOCUMENTS` connector (see "RAG: workspace document connectors" below).
- They want web search, browser, or deep-research capabilities baked in.

## "Create a RAG agent" — guided setup flow

When the user says "create a RAG agent" (or "agent over my docs", "chatbot for my PDFs", etc.), this is **one conversation**, not three separate skill invocations. Walk them through the prereqs yourself, calling the API endpoints from `datagol-upload-file` and `datagol-index-document` inline. Don't tell them "go run the `datagol-upload-file` skill first" — just do it.

Ask the questions **one at a time**, in this order. Wait for each answer before asking the next.

### Step 1 — Are the documents already in DataGOL?

> Do you already have the knowledge-base documents in a DataGOL workspace, or do we need to upload them now?

- **Yes, already there** → ask which workspace (call `datagol_list_workspaces`, present names). Skip to Step 5 (verify indexed).
- **No, need to upload** → continue to Step 2.

### Step 2 — New workspace or existing?

> Which workspace should hold the documents — should I create a new one, or use an existing workspace?

- **New** → ask for a name; call `datagol_create_workspace({ name })`; capture `id`.
- **Existing** → present `datagol_list_workspaces` results; capture the picked `id` and `name`.

### Step 3 — Folder?

> Want to keep the documents organized in a folder inside the workspace, or drop them at the workspace root?

- **In a folder** → ask for a folder name (suggest a sensible default based on context, e.g. *"Knowledge Base"*); create with `POST /workspaces/{ws}/folder` (per `datagol-upload-file` skill); capture `folderId`.
- **Workspace root** → set `parentId: null` for uploads; no folder created.

### Step 4 — Upload

> Please upload the files you want to index — drag them into the chat or use the paperclip button.

When the user attaches files (they land at `.uploads/<safe-name>` per the chat-attachment pipeline), upload each one via the multipart endpoint in the `datagol-upload-file` skill. One `curl` per file. Report back with how many files were uploaded.

### Step 5 — Trigger indexing

Call the re-index endpoint in the `datagol-index-document` skill (`POST .../re-index?workspace_id=...&full_reindex=false` with body `{}`). Take ONE status snapshot, tell the user it's running. Don't block the conversation polling — proceed straight to Step 6 while indexing runs in the background.

> Indexing started. It usually finishes in a couple of minutes for small docs. I'll create the agent now; retrieval will be live once indexing completes.

### Step 6 — Gather agent details

Now collect the actual agent inputs:

- **Name** — short label, e.g. *"Sales Knowledge Bot"*.
- **System prompt** — what should it do? Use the user's intent + the standard formatting clause (see Workflow §1 below).
- **Capabilities** — by default, web search **off** for RAG (so the agent answers strictly from the docs). Confirm with the user.
- **Workbook MCPs?** — usually no for a pure RAG agent, but ask if they want to mix.

### Step 7 — Create the agent (raw API)

Use `POST https://be.datagol.ai/customAgents/api/v1` (the Pi tool `datagol_create_agent` does NOT support `WORKSPACE_DOCUMENTS` connectors). Body must include the workspace-document connector:

```json
"connectors": [
  { "id": "<workspaceId>", "type": "WORKSPACE_DOCUMENTS", "sourceType": "BACKEND", "data": { "name": "<workspace name>" } }
]
```

Full body shape is in the "RAG: workspace document connectors" section below.

### Step 8 — Confirm

Use the standard post-creation message template, including the **Knowledge base** section so the user sees the workspace they connected.

### What to do if the user is impatient

If they say "just create the agent now, I'll upload later" — fine, create the agent with the workspace connector pointing at an empty/un-indexed workspace. Tell them retrieval will be empty until they upload + index. Don't block.

## Tool

> ⚠️ **Agent ownership pattern:** agents are created by the platform user (user JWT). The service account does *not* inherit access from the workspace grant — it needs an explicit per-element grant via `POST /noCo/api/v2/elementPermissions/bulk` right after creation. Without that grant, the generated app's `x-auth-token: SERVICE_TOKEN` call to `POST /ai/api/v2/conversations` returns `400 "User does not have access to the custom agent"`. **Always run the grant step.** See "Grant the service account access to the agent" below.

`datagol_create_agent` accepts these parameters:

| Field | Type | Notes |
|---|---|---|
| `name` | string | **Required.** Display name shown to users. |
| `prompt` | string | **Required.** System prompt — defines behavior and persona. |
| `description` | string | Short summary. |
| `instructions` | string | Long-form usage notes shown to end users. |
| `examples` | string | Example interactions shown to end users. |
| `workbooks` | array | Workbooks to expose as MCP servers — see below. |
| `webSearchEnabled` | bool | Default `true`. |
| `browserEnabled` | bool | Default `false`. |
| `deepResearchEnabled` | bool | Default `false`. |
| `isPublished` | bool | Default `false` (saves as draft). |
| `conversationStarters` | string[] | Suggested opener prompts. |

## Workflow

1. **Ask the user** what the agent should *do*. Draft a tight system `prompt` that captures the persona, the data it has access to, and any rules. **Always include the rich formatting clause below** so responses render cleanly in any DataGOL chat UI:

   > Output and rendering rules:
   > - Format every response in polished, clean Markdown.
   > - Use `##` and `###` headings for structure.
   > - Use short paragraphs and bullets for scanability.
   > - Use numbered lists for step-by-step plans.
   > - Use Markdown tables for summaries, schedules, comparisons, prioritization matrices, status reports, and other structured data. Keep tables practical and no wider than 5 columns when possible.
   > - Use `**bold**` for key terms, decisions, statuses, and action labels.
   > - Use `inline code` for identifiers, IDs, field names, values, commands, and short literals.
   > - Use fenced code blocks only for exact snippets, templates, API payloads, SQL, scripts, or multiline examples.
   > - Use Markdown links as `[label](https://...)` only for real external URLs. Do not turn internal workbook IDs or UUID-style references into clickable links.
   > - Use Markdown images as `![clear alt text](https://...)` when an image is useful and a valid image URL is available; always include useful alt text and a brief caption in prose.
   > - If a chart, timeline, kanban summary, diagram, or other visualization would be clearer than prose and the runtime supports HTML output, provide compact HTML/SVG that fits inside a chat bubble.
   > - Never output raw, unformatted JSON unless the user explicitly asks for raw JSON.
   > - Keep paragraphs short and make the final answer easy to scan.

   This clause is generic — adapt domain-specific table columns if useful, but keep the rendering rules in every prompt. The chat UI scaffolded by the `datagol-agent-chat-ui` skill renders these elements; without the clause the agent often replies in flat prose, which looks worse and is harder to scan.
2. **Identify which workbooks the agent needs.** If the user names entities ("Contacts", "Deals"), call `datagol_get_workspace_schema` to look up workbook IDs in the current workspace. Pick only the workbooks the agent actually needs — every additional MCP server widens the agent's surface area and slows it down.
3. **Confirm the plan with the user** — show them the proposed name, prompt, and workbooks before creating the agent (this is a modifying call).
4. **Create the agent** with the user bearer (`datagol_create_agent`, or `POST /customAgents/api/v1` with `Authorization: Bearer ${DATAGOL_TOKEN}` for RAG / mixed-mode agents).
5. **Grant the service account `CREATOR` on the new agent.** This is non-negotiable for any agent the generated app will invoke. See "Grant the service account access to the agent" below.
6. **Report the result with the template below.** Don't paraphrase — emit the exact markdown so the chat UI renders the link, headers, and lists consistently.

## Post-creation message format

After `datagol_create_agent` succeeds, output exactly this markdown (filling
in the values from the tool response). The chat UI renders `##` headers,
`**bold**`, bullet lists, inline code, and `[text](https://…)` links — keep
to those. Don't include backticks around the appUrl link itself, and **never**
echo any URL from `mcpServers` (those carry tokens).

```markdown
## ✓ Agent created: **<name>**

**[Open in DataGOL →](<appUrl>)**

**Status:** <Draft | Published>

**Connected workbooks**
- <workbook name 1>
- <workbook name 2>

**Knowledge base**
- <workspace name>  *(only if a `WORKSPACE_DOCUMENTS` connector is attached)*

**Capabilities**
- Web search: <on | off>
- Browser: <on | off>
- Deep research: <on | off>

**Next steps**
- Click the link above to try the agent.
- To publish, edit it from DataGOL → Agents.
- Want a website that talks to it? Ask: "build a chat UI for this agent".
```

Notes:
- "Draft" maps to `isPublished: false`, "Published" to `true`.
- Omit any **Capabilities** line whose flag is `false` if the agent kept defaults — only show what's actually enabled. (Web search defaults on; browser/deep research default off.)
- **Omit the `Knowledge base` section entirely** unless a `WORKSPACE_DOCUMENTS` connector was attached.
- If `workbooks` AND knowledge base are both empty (rare — pure web-search agent), replace **Connected workbooks** with `_None — answers from web search only._`.

## Grant the service account access to the agent

Right after the agent is created (whether via `datagol_create_agent` or `POST /customAgents/api/v1`), grant the service account `CREATOR` permission on the new agent. **This is required** — without it, `x-auth-token: SERVICE_TOKEN` calls to `POST /ai/api/v2/conversations` will return `400 "User does not have access to the custom agent"` even though the service token is valid and the service account has workspace-level access.

The grant is per-element, not per-workspace. Workbook reads from MCP work fine without it (those inherit from the workspace `bulkUsers` grant in `datagol-app-auth` Step C), but agents and other element-scoped resources need this extra step.

```bash
# Required values:
#   AGENT_ID                 = the new agent's id (from the create response)
#   SERVICE_ACCOUNT_USER_ID  = the service account's numeric userId
#                              (= SVC_ID captured in datagol-app-auth Step A;
#                               same value as response.id from POST /idp/api/v1/company/serviceAccount)
#   DATAGOL_TOKEN            = the platform user's bearer (NOT the service token —
#                              the grant is performed by the agent's owner)

curl -X POST "${DATAGOL_BASE_URL}/noCo/api/v2/elementPermissions/bulk" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"permissions\": [
      {
        \"elementId\":   \"${AGENT_ID}\",
        \"elementType\": \"CUSTOM_AGENT\",
        \"userId\":      ${SERVICE_ACCOUNT_USER_ID},
        \"permission\":  \"CREATOR\"
      }
    ]
  }"
```

> ⚠️ **The grant can return HTTP 200 but still fail — and `userId` is the usual culprit.** In some environments `elementPermissions/bulk` does not resolve the service account from its numeric `userId`: the call returns `200` but the response **body** reports the operation failed (no permission is actually written), and the app keeps getting `400 "User does not have access to the custom agent"`. **Don't trust the status code — inspect the body.** When the `userId` form fails this way, re-grant by service-account **email** instead, which resolves the service-account user correctly:
>
> ```bash
> #   SERVICE_ACCOUNT_EMAIL = the service account's email
> #                           (from POST /idp/api/v1/company/serviceAccount response, or
> #                            datagol-app-auth Step A — same place SVC_ID came from)
> curl -X POST "${DATAGOL_BASE_URL}/noCo/api/v2/elementPermissions/bulk" \
>   -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
>   -H "Content-Type: application/json" \
>   -d "{
>     \"permissions\": [
>       {
>         \"elementId\":   \"${AGENT_ID}\",
>         \"elementType\": \"CUSTOM_AGENT\",
>         \"email\":       \"${SERVICE_ACCOUNT_EMAIL}\",
>         \"permission\":  \"CREATOR\"
>       }
>     ]
>   }"
> ```

After this returns 200, the generated app can use `x-auth-token: ${DATAGOL_SERVICE_TOKEN}` (the same token it uses for workbook calls) for **all** agent endpoints — conversations, streaming, fetching agent metadata. No user bearer needs to be baked into the app.

**Verify the grant actually took — don't assume.** Because the grant call can lie (200 with a failure body, above), confirm by exercising the exact path the app uses: create a conversation with the **service token**, and check the agent reports the service account as `CREATOR`.

```bash
# Smoke-test with the SAME token the app uses (service token, x-auth-token):
curl -s -o /dev/null -w "conversation_probe_status=%{http_code}\n" \
  -X POST "${DATAGOL_BASE_URL}/ai/api/v2/conversations" \
  -H "x-auth-token: ${DATAGOL_SERVICE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"customAgentId\": \"${AGENT_ID}\", \"userId\": ${SERVICE_ACCOUNT_USER_ID}, \"name\": \"perm probe\"}"
# Expect: conversation_probe_status=200   (403/400 → the grant didn't take; re-grant by email)
```

A healthy grant shows `conversation_probe_status=200` and the agent's `permissionOfUser` reads `CREATOR`. Only then is the agent safe to invoke from the generated app.

> **Why `Authorization: Bearer ${DATAGOL_TOKEN}` and not the service token?** Element permissions are granted *by* the element's owner. The agent owner is the platform user (since `datagol_create_agent` runs under the user JWT), so the grant call must also be made under the user JWT. The service account can't grant itself permissions.

> **Doing this from a Pi `bash` tool call:** both `DATAGOL_TOKEN` and `DATAGOL_BASE_URL` are already in the codex env. `SERVICE_ACCOUNT_USER_ID` and `AGENT_ID` are values you captured from prior steps — pass them as shell variables, don't re-fetch.

## Workbook → MCP Mapping

Each workbook in `workbooks` becomes one MCP server entry of the form:

```json
{
  "serverType": "WORKBOOK",
  "name": "<workbook display name>",
  "transport": "streamable_http",
  "url": "",
  "description": "",
  "data": { "id": "<workbookId>", "workspaceId": "<workspaceId>" }
}
```

The tool builds this for you — just pass the IDs.

## RAG: workspace document connectors

When the agent should retrieve from documents (PDFs, DOCX, etc. uploaded into a workspace), connect the **workspace itself** as a `WORKSPACE_DOCUMENTS` connector. The Pi tool `datagol_create_agent` does **not** currently expose this — call the underlying API directly via `bash`+`curl` for any agent that needs RAG.

### Connector entry shape

```json
{
  "id":         "<workspaceId>",
  "type":       "WORKSPACE_DOCUMENTS",
  "sourceType": "BACKEND",
  "data":       { "name": "<workspace display name>" }
}
```

`id` is the workspace UUID; `data.name` is its display name (look both up via `datagol_list_workspaces`).

### Create call (full body) — `POST https://be.datagol.ai/customAgents/api/v1`

```bash
WORKSPACE_ID="<id>"
WORKSPACE_NAME="<display name>"
AGENT_BODY=$(jq -n \
  --arg name "RAG Agent" \
  --arg prompt "$AGENT_PROMPT" \
  --arg wsId "$WORKSPACE_ID" \
  --arg wsName "$WORKSPACE_NAME" \
  '{
     name: $name,
     description: "",
     instructions: "",
     examples: "",
     prompt: $prompt,
     skills: [],
     configs: {
       connectors: [
         { id: $wsId, type: "WORKSPACE_DOCUMENTS", sourceType: "BACKEND", data: { name: $wsName } }
       ],
       mcpConfigs: [],
       webSearchEnabled: false,
       browserEnabled: false,
       showImages: false,
       voiceConfig: { voiceEnabled: false, textToSpeech: { provider: "", apiKey: "" }, speechToText: { provider: "", apiKey: "" } },
       deepResearchEnabled: false,
       subAgents: [],
       skillIds: []
     },
     isPublished: false,
     uiMetadata: {
       bestFor: [], conversationStarters: [], logo: null,
       style: { logo: { bgColor: "", textColor: "" } },
       input: { placeholder: "" },
       capabilities: {
         // Pin the agent to claude-sonnet-4-6 — this is what's actually
         // wired in test/prod and matches the model the streaming endpoint
         // is configured for. Leaving models disabled or empty lets the
         // backend fall through to a default that may not be available.
         models: { enabled: true, defaultValue: "claude-sonnet-4-6", supportedModels: ["claude-sonnet-4-6"] },
         allowDownload: false,
         conversationFiles: { enabled: false }
       }
     }
   }')

# Always use the user bearer — the platform user owns the agent.
# Right after this call returns 200, run the elementPermissions/bulk grant
# (see "Grant the service account access to the agent" above) before the
# generated app tries to invoke the agent with x-auth-token.
curl -s -X POST "${DATAGOL_BASE_URL}/customAgents/api/v1" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "$AGENT_BODY"
```

The response includes the new agent's `id` and `appUrl` (same shape as `datagol_create_agent`'s response).

### Mixed RAG + workbook MCP

Document connectors and workbook MCPs coexist in the same body — populate both `configs.connectors[]` and `configs.mcpConfigs[]`. `mcpConfigs` entries follow the same shape that `datagol_create_agent` would have built (see "Workbook → MCP Mapping" above), or you can let `datagol_create_agent` create the agent first (with workbooks only) and then `PUT /customAgents/api/v1/{id}` to add the `WORKSPACE_DOCUMENTS` connector — see "Update existing agent" below.

### Update existing agent — `PUT https://be.datagol.ai/customAgents/api/v1/{agentId}`

`PUT` replaces the agent record wholesale (not a partial PATCH). To add a connector to an existing agent, you must:

1. Read the current record first (or carry forward the values you used at create time).
2. Append the new entry to `configs.connectors[]`.
3. PUT the full body, including `id` in both the URL path **and** the body.

Use this when the user says "add docs to my agent" or asks to extend an already-created agent with a knowledge base. The shape mirrors the create body plus the `id` field.

### Decide when to use which path

- **MCP-only agent (workbooks only, no documents)** → `datagol_create_agent` is fine. Simpler, no curl needed.
- **RAG agent (documents required) or mixed RAG + MCP** → raw API: `POST /customAgents/api/v1` with `Authorization: Bearer ${DATAGOL_TOKEN}`. The Pi tool can't add `WORKSPACE_DOCUMENTS` connectors.
- **In both cases, if the agent is invoked from a generated app, run the elementPermissions/bulk grant** (see above) right after creation. Skipping it always causes the 400 in the app.

## Example Call

User: *"Create a sales QA agent that can answer questions about my Contacts and Company workbooks."*

> This agent will be invoked from a generated app. Create with user bearer, then grant the service account `CREATOR` via `elementPermissions/bulk`.

```bash
# 1. Look up workbook IDs
#    datagol_get_workspace_schema → Contacts: '11f5...', Company: '5b31...', workspaceId: '24afe968-...'

# 2. Create with user bearer (datagol_create_agent works here for MCP-only;
#    raw curl shown for completeness / RAG cases)
curl -s -X POST "${DATAGOL_BASE_URL}/customAgents/api/v1" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sales QA Agent",
    "description": "Answer questions about Contacts and Company data.",
    "prompt": "You are a sales QA agent. Answer questions strictly using data in the connected Contacts and Company workbooks. If a question requires data outside these, say so.\n\nFormat every response in clean Markdown. Use ## headers for sections, bullet lists for enumerations, **bold** for key terms, `inline code` for IDs and field names, and Markdown tables for tabular data (≤5 columns). Keep paragraphs short.",
    "skills": [],
    "configs": {
      "connectors": [],
      "mcpConfigs": [
        { "serverType": "WORKBOOK", "name": "Contacts", "transport": "streamable_http", "url": "", "description": "", "data": { "id": "11f5...", "workspaceId": "24afe968-..." } },
        { "serverType": "WORKBOOK", "name": "Company",  "transport": "streamable_http", "url": "", "description": "", "data": { "id": "5b31...", "workspaceId": "24afe968-..." } }
      ],
      "webSearchEnabled": false, "browserEnabled": false, "showImages": false,
      "voiceConfig": { "voiceEnabled": false, "textToSpeech": {"provider":"","apiKey":""}, "speechToText": {"provider":"","apiKey":""} },
      "deepResearchEnabled": false, "subAgents": [], "skillIds": []
    },
    "isPublished": true,
    "uiMetadata": {
      "bestFor": [], "conversationStarters": [], "logo": null,
      "style": {"logo":{"bgColor":"","textColor":""}}, "input": {"placeholder":""},
      "capabilities": {"models":{"enabled":true,"defaultValue":"claude-sonnet-4-6","supportedModels":["claude-sonnet-4-6"]},"allowDownload":false,"conversationFiles":{"enabled":false}}
    }
  }'
# Response: { "id": "024c4a46-...", "name": "Sales QA Agent", "isPublished": true, ... }
# Capture: AGENT_ID = response.id

# 3. Grant the service account CREATOR on the new agent
#    SERVICE_ACCOUNT_USER_ID was captured in datagol-app-auth Step A (SVC_ID)
curl -s -X POST "${DATAGOL_BASE_URL}/noCo/api/v2/elementPermissions/bulk" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"permissions\":[{\"elementId\":\"${AGENT_ID}\",\"elementType\":\"CUSTOM_AGENT\",\"userId\":${SERVICE_ACCOUNT_USER_ID},\"permission\":\"CREATOR\"}]}"
# 200 OK → the generated app can now use x-auth-token: ${DATAGOL_SERVICE_TOKEN} for all agent calls
```

## Response Shape

Both `datagol_create_agent` and the raw `POST /customAgents/api/v1` return the same shape:

```json
{
  "id": "024c4a46-...",
  "name": "Test Agent",
  "isPublished": false,
  "appUrl": "https://app.datagol.ai/agents/024c4a46-...",
  "mcpServers": [
    { "name": "Contacts", "workbookId": "11f5...", "workspaceId": "24af..." }
  ]
}
```

The DataGOL backend enriches each MCP entry with a tokenised URL like
`https://mcp.datagol.ai/<workbook>?workspace_id=...&workbook_id=...&token=...`.
**Do not echo those URLs to the user** — they contain credentials. The user
accesses the agent through `appUrl`.

## Invocation (after creation)

Invoking the agent is a **two-step flow**: first create a conversation, then
stream messages into it. The conversation persists thread state on the server.
Never mint conversationId client-side.

### Step 1 — Create a conversation (once per thread)

```
# Use x-auth-token with the service token.
# This works because the service account was granted CREATOR on the agent
# via elementPermissions/bulk right after agent creation (see above).
POST ${DATAGOL_BASE_URL}/ai/api/v2/conversations
x-auth-token: <DATAGOL_SERVICE_TOKEN>
Content-Type: application/json

{
  "name":      "<short label, e.g. the first user message truncated>",
  "active":    true,
  "scope":     "PRIVATE",
  "agentType": "CustomAgent",
  "uiMetadata": {
    "type": "CustomAgent",
    "parameters": {
      "customAgentId": "<agent id from creation response>",
      "parameters":    { "customAgentId": "<agent id from creation response>" }
    },
    "configuration": { "llmModel": "claude-sonnet-4-6" }
  }
}
```

The response includes an `id` field — that is the **conversationId** to use
in step 2. Save it for the lifetime of the chat thread.

Note: do NOT pass `userId` and do NOT decode the JWT to find one — the server
derives the user from the bearer token.

### Step 2 — Send a message (per turn)

```
# ⚠️ Streaming lives on a SEPARATE host from the main DataGOL API.
# Do NOT use DATAGOL_BASE_URL here.
#   prod: https://ai.datagol.ai/ai/api/v2/messages/streaming
#   test: https://testing-core-ai.datagol.ai/ai/api/v2/messages/streaming
# Note: prod and test hosts don't share a naming pattern — hard-code
# the branch on DATAGOL_ENV in src/config.ts:
#   export const DATAGOL_AI_URL = DATAGOL_ENV === 'test'
#     ? 'https://testing-core-ai.datagol.ai'
#     : 'https://ai.datagol.ai';
POST ${DATAGOL_AI_URL}/ai/api/v2/messages/streaming
x-auth-token: <DATAGOL_SERVICE_TOKEN>
Content-Type: application/json

{
  "agentType":        "CustomAgent",
  "type":             "CustomAgent",
  "conversationId":   "<id from step 1>",
  "message":          "<user input>",
  "parameters":       { "customAgentId": "<agent id from creation response>" },
  "uiMetadata":       { "parameters": {} },
  "selectedLlmModel": "claude-sonnet-4-6"
}
```

Response is **Server-Sent Events**; consume as a stream.

The wire format is non-trivial — payload lines may be bare JSON (no `data:`
prefix), `:heartbeat`/`:keep-alive` comments are interleaved, and the stream
emits four `response_type` values you must handle: `message_id`, `text`,
`tool_call`, and `subagent_task_end` (whose `tool_response[0].text` is itself
a JSON-encoded string requiring a second parse). When generating a chat UI,
**read the `datagol-agent-chat-ui` skill** — it has the verified parser and message
shapes. Do not improvise the parser; missing the tool events is the most
common first-try failure.

### Rules for invocation

- **Never mint conversationId client-side** — it must come from the POST
  `/ai/api/v2/conversations` response. A client-generated UUID will produce a
  thread the server has no record of.
- **One conversation per thread.** Reuse the same `conversationId` across all
  messages in a thread. Start a new conversation (call step 1 again) when the
  user clicks "New chat".
- **Always use `x-auth-token: <DATAGOL_SERVICE_TOKEN>`** for both conversation creation
  and streaming. `Authorization: Bearer` also works but only when the caller is the
  agent owner. Since we always create agents with the service account, `x-auth-token`
  with the service token is the consistent, correct pattern — same as every other
  DataGOL API call in the generated app. No JWT, no postMessage bridge needed.
- **`selectedLlmModel: 'claude-sonnet-4-6'`** — match the agent's `uiMetadata.capabilities.models.defaultValue`. The streaming endpoint validates this against the agent's allowlist; an unsupported model returns a 400.
- **Streaming uses `DATAGOL_AI_URL`, not `DATAGOL_BASE_URL`.** Prod streaming lives at `https://ai.datagol.ai`, test at `https://testing-core-ai.datagol.ai`. Using `be.datagol.ai` for `/ai/api/v2/messages/streaming` returns 404. Define `DATAGOL_AI_URL` in `src/config.ts` alongside `DATAGOL_BASE_URL` (see `datagol-app-auth`). Conversation creation (`POST /ai/api/v2/conversations`) still goes to `DATAGOL_BASE_URL`.

When the user wants a website / frontend that talks to the agent, **read the
`datagol-agent-chat-ui` skill** — it has the full scaffold including conversation
creation, SSE parser, and thread management. Pass the `id` from the creation response as the `AGENT_ID` constant in that scaffold.

## Rules

- **Always create agents with `Authorization: Bearer ${DATAGOL_TOKEN}`** (user JWT). `datagol_create_agent` does this automatically; the raw `POST /customAgents/api/v1` needs the header set explicitly. `x-auth-token: SERVICE_TOKEN` on agent creation returns `400 "Either userId, email, or anonymousUserId is required for element permission"` — that path doesn't work.
- **Always grant the service account `CREATOR` on the new agent** via `POST /noCo/api/v2/elementPermissions/bulk` immediately after creation. Without this, the generated app gets `400 "User does not have access to the custom agent"` on its first conversation call. Use `userId: SERVICE_ACCOUNT_USER_ID` (= `SVC_ID` captured in `datagol-app-auth` Step A), `elementType: "CUSTOM_AGENT"`, `permission: "CREATOR"`. The grant call itself runs under the user bearer (the agent's owner authorises the grant).
- **Generated app uses `x-auth-token: ${DATAGOL_SERVICE_TOKEN}` for everything — workbooks and agents.** No user bearer in the generated app. The elementPermissions grant is what unlocks the service token for agent calls.
- **Plan before acting.** Always confirm the agent's name, prompt, and connected workbooks/workspaces with the user before creating (this is a modifying call).
- **For RAG agents, check prereqs first.** The user needs a workspace with **indexed** documents. If they don't have one, run `datagol-upload-file` then `datagol-index-document` (in that order) before creating the agent. Don't attempt to add a `WORKSPACE_DOCUMENTS` connector pointing at a workspace with no indexed files — retrieval will return nothing and the user will think the agent is broken.
- **Pi tool vs raw API:** `datagol_create_agent` is fine for MCP-only agents. Use raw `POST /customAgents/api/v1` (with the user bearer) for RAG or mixed agents that need `WORKSPACE_DOCUMENTS` connectors. Either way, the elementPermissions grant happens immediately after.
- **Don't enable `isPublished: true` unless the user explicitly asks** — drafts are safer.
- **Don't connect every workbook or workspace by default.** Only attach what the agent's prompt actually references.
- **Don't fabricate workbook or workspace IDs.** Look them up via `datagol_get_workspace_schema` / `datagol_list_workspaces`.
- **Never echo MCP server URLs** from the API response — they contain a token. Same for the workspace UUID in any user-facing summary; reference workspaces by name.
- After creation, give the user the `appUrl` and tell them they can edit/publish from DataGOL → Agents.

## Troubleshooting

### `400 "User does not have access to the custom agent"`

The app calls `POST /ai/api/v2/conversations` with `x-auth-token: SERVICE_TOKEN` and gets this error.

**Root cause:** The elementPermissions grant was skipped or failed. Agents are owned by the platform user (since creation runs under the user bearer); the service account doesn't inherit access from the workspace `bulkUsers` grant — it needs a per-element grant for each agent.

**Fix:** run the `elementPermissions/bulk` call against the affected agent. Re-run it any time the user creates a new agent the app needs to invoke.

```bash
# Look up the new agent's id and re-grant.
curl -X POST "${DATAGOL_BASE_URL}/noCo/api/v2/elementPermissions/bulk" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"permissions\":[{\"elementId\":\"${AGENT_ID}\",\"elementType\":\"CUSTOM_AGENT\",\"userId\":${SERVICE_ACCOUNT_USER_ID},\"permission\":\"CREATOR\"}]}"
# 200 → the existing service token now works for /ai/api/v2/conversations calls against this agent
```

> **`POST /customAgents/api/v1` with `x-auth-token: SERVICE_TOKEN` does NOT work for agent creation** — the endpoint returns `400 "Either userId, email, or anonymousUserId is required for element permission"`. Agents must be created under the user JWT. The elementPermissions grant is the mechanism that lets the service token access user-created agents afterwards.

**Generated app config — single token only:**

```ts
// src/config.ts
export const DATAGOL_SERVICE_TOKEN = '<service token>';  // used everywhere — workbooks and agents
export const AGENT_ID = '<agent id>';

// src/lib/agent.ts
import { DATAGOL_SERVICE_TOKEN, AGENT_ID } from '../config';

function headers() {
  return { 'x-auth-token': DATAGOL_SERVICE_TOKEN, 'Content-Type': 'application/json' };
}

export async function createConversation(name: string, userId: number): Promise<string> {
  const res = await fetch(`${DATAGOL_BASE_URL}/ai/api/v2/conversations`, {
    method: 'POST',
    headers: headers(),
    body: JSON.stringify({
      name, active: true, scope: 'PRIVATE', userId, agentType: 'CustomAgent',
      uiMetadata: { type: 'CustomAgent',
        parameters: { customAgentId: AGENT_ID, parameters: { customAgentId: AGENT_ID } },
        configuration: { llmModel: 'claude-sonnet-4-6' } },
    }),
  });
  // ...
}
```

`userId` in the conversation body is the **end-user's id** (the person using the app), not the service account's. For single-user demo apps with no login, hard-code the developer's userId at scaffold time. For multi-user apps (`datagol-user-auth`), pass the logged-in user's id at runtime.

> **No `DATAGOL_USER_TOKEN` needed.** The earlier "bake user bearer into the app" workaround is obsolete — the elementPermissions grant lets the service token cover both workbooks and agents.

### `403` on Slack Messages / Slack Channels workbooks (via MCP)

If the agent's MCP tool calls fail with 403 on workbook access, check in order:
1. **Does the agent exist?** `GET /customAgents/api/v1/agents/{id}` with service token → 404 means the agent was deleted. Recreate it (see above).
2. **Service account workspace access.** `POST /noCo/api/v2/workspaces/{wsId}/bulkUsers` with user JWT to re-grant `CREATOR`. A 400 response with `"already has access"` means access is fine — not the cause.
3. **Direct workbook test.** `POST /noCo/api/v2/workspaces/{wsId}/tables/{tableId}/cursor` with `x-auth-token: SERVICE_TOKEN` → HTTP 200 means the workbook itself is accessible and the 403 is coming from the (deleted) agent's MCP context, not workbook-level ACL.

## Cross-references

- **`datagol-upload-file`** — pre-flight for RAG agents: puts documents into a workspace folder.
- **`datagol-index-document`** — pre-flight for RAG agents: triggers and monitors the search index over uploaded documents. Run after `datagol-upload-file`.
- **`datagol-agent-chat-ui`** — when the user wants a website / standalone app that talks to the created agent.
- **`datagol-context`** — DataGOL data model.
- **`datagol-workbook-operations`** — when the agent's job is to read/write structured rows rather than retrieve from documents.
