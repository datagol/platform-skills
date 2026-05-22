---
name: datagol-index-document
description: Trigger and monitor enterprise-search indexing over the documents in a DataGOL workspace. Use this skill whenever the user wants to "index documents", "make my files searchable", "build the search index", "trigger a re-index", "check indexing status", "rebuild the knowledge base index", or otherwise process uploaded files for retrieval. Pairs with `datagol-upload-file` (puts files into the workspace) and `datagol-create-agent` (which can connect the indexed workspace as a `WORKSPACE_DOCUMENTS` connector for RAG).
---

# Index Workspace Documents for Search

Use this when the user has files in a DataGOL workspace and wants them searchable — either freshly uploaded ones (after running `datagol-upload-file`) or existing ones whose index needs a refresh. The skill triggers the enterprise-search re-index and reports status.

> **Status: temporary.** Hand-rolled wrapper around two `ai.datagol.ai` endpoints. Will be replaced when a dedicated Pi tool ships.

## Two flows

- **Flow A — agent-led.** Agent uses `bash`+`curl` to fire the re-index, takes one status snapshot, reports back. Default flow.
- **Flow B — generate a status UI.** User wants a page with a "Re-index" button and live status. Generates a React component using the postMessage token handshake.

## Workspace choice

Always determine the workspace before triggering anything. Either:
- The user invoked this right after `datagol-upload-file` — reuse the same `WORKSPACE_ID`.
- Otherwise call `datagol_list_workspaces`, present names, capture the user's pick.

---

## Flow A — agent-led (`bash` + `curl`)

The agent has `DATAGOL_TOKEN` in its env.

### Step 1 — trigger re-indexing

```bash
WORKSPACE_ID="<id>"
curl -s -X POST \
  "https://ai.datagol.ai/ai/api/v2/enterprise-search/connector/dg-s3-workspace/re-index?workspace_id=${WORKSPACE_ID}&full_reindex=false" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{}'
```

- **Empty body `{}` is required**, even though all parameters live in the query string.
- `full_reindex=false` (default) re-indexes only new/changed objects in the workspace's S3 prefix. Use `full_reindex=true` only when the user explicitly says "wipe and rebuild from scratch" — full reindex burns significantly more compute.

### Step 2 — read status (single snapshot)

```bash
curl -s \
  "https://ai.datagol.ai/ai/api/v2/enterprise-search/connector/dg-s3-workspace/${WORKSPACE_ID}/status" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}"
```

Relevant fields in the response:

```json
{
  "connector_status": {
    "cc_pair_status": "ACTIVE",
    "in_progress": true,
    "last_status": "in_progress" | "success" | "failed" | "canceled",
    "last_finished_status": "success" | "failed" | "canceled" | null,
    "latest_index_attempt": {
      "status": "in_progress" | "success" | "failed",
      "new_docs_indexed": 0,
      "total_docs_indexed": 0,
      "error_msg": null
    }
  }
}
```

Interpret:

| `in_progress` | `last_finished_status` | What to tell the user |
|---|---|---|
| `true` | (any) | "Indexing started — usually finishes in a few minutes for small docs. I'll check status if you ask." |
| `false` | `"success"` | "Indexing finished. **N** documents now searchable." (use `latest_index_attempt.total_docs_indexed`) |
| `false` | `"failed"` | "Indexing failed: `<error_msg>`." |
| `false` | `"canceled"` | "Indexing was canceled before completion." |

**Don't block the chat by polling.** Take one snapshot, report, return control. If the user asks "is it done yet?", run the status call again.

If the user explicitly says "wait until done" — poll every 6 seconds for up to 5 minutes, then surface whatever final state you observed. After timeout, tell them they can ask again later.

---

## Flow B — generate a status UI

User wants a sandbox page with a re-index button and live status. Reuse `datagol-agent-chat-ui`'s token handshake.

### Files to write

1. **`src/lib/token.ts`** — postMessage handshake. Same as `datagol-agent-chat-ui` / `datagol-upload-file` / `datagol-create-dashboard`.

2. **`src/lib/index-config.ts`** — workspace constant; hide UUIDs from the user-facing summary.

   ```ts
   // src/lib/index-config.ts
   export const WORKSPACE_ID = '<id from workspace choice>';
   export const DATAGOL_AI   = 'https://ai.datagol.ai/ai/api/v2';
   ```

3. **`src/components/IndexStatus.tsx`** — implementation contract:

   - Render a header showing `last_status` / `last_finished_status` / `latest_index_attempt.total_docs_indexed`.
   - "Re-index" button (calls `POST /re-index?workspace_id=…&full_reindex=false` with empty `{}`).
   - "Full re-index" button (same with `full_reindex=true`); show a `confirm()` first since it's expensive.
   - On mount AND every 6 seconds while `in_progress === true`, refresh status.
   - Stop polling once `in_progress === false`; allow manual "Refresh" to resume.
   - Error states (`last_finished_status === "failed"`) show `error_msg` and a retry button.
   - All requests use `Authorization: Bearer ${await awaitToken()}`.

### App.tsx wiring

Standalone: `<App>` returns `<IndexStatus />`. Existing multi-page app: add an `'index-status'` nav entry.

### After writing, tell the user

> Indexing status page added — open it from the nav. The page polls automatically while indexing is running.

Don't echo the workspace UUID or API URL.

---

## API reference (canonical)

All endpoints require `Authorization: Bearer <DATAGOL_TOKEN>`.

| Operation | Method · URL | Body |
|---|---|---|
| Trigger re-index | `POST https://ai.datagol.ai/ai/api/v2/enterprise-search/connector/dg-s3-workspace/re-index?workspace_id={ws}&full_reindex=false` | `{}` (empty JSON, required) |
| Status snapshot | `GET https://ai.datagol.ai/ai/api/v2/enterprise-search/connector/dg-s3-workspace/{ws}/status` | — |

## Hard Rules

- **`full_reindex=false` by default.** Full reindex is expensive. Only `true` on explicit request.
- **Don't block the chat on long polls.** Single snapshot → tell the user it's running → return. Poll only on explicit "wait until done", and bound the wait (≤ 5 minutes).
- **Never echo `DATAGOL_TOKEN`.** Pi has it via env; generated UI gets it via the postMessage handshake.
- **Don't echo the workspace UUID or connector internals.** Reference workspaces by name.
- **`{}` body is required on re-index.** A missing body returns a 400.
- **This skill doesn't upload files.** If the user is uploading new docs, run `datagol-upload-file` first, then this. They're separate building blocks.

## Cross-references

- **`datagol-upload-file`** — to add documents into a workspace before indexing them.
- **`datagol-create-agent`** — to wire a chat agent over the indexed workspace via a `WORKSPACE_DOCUMENTS` connector. The agent skill knows how.
- **`datagol-context`** — DataGOL data model.
