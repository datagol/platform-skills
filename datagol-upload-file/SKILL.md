---
name: datagol-upload-file
description: Upload one or more files into a DataGOL workspace folder. Use this skill whenever the user wants to "upload a file", "upload documents", "add files to a workspace", "drop PDFs into the workspace", "add docs for indexing", "create a folder for files", or otherwise put binary content into a workspace's S3-backed file store. Pairs with `datagol-index-document` (which kicks off the search index over uploaded files) and `datagol-create-agent` (which can connect the workspace as a `WORKSPACE_DOCUMENTS` connector for RAG).
---

# Upload Files to a Workspace Folder

Use this when the user wants to put one or more files into a DataGOL workspace. The skill creates a folder if needed, then uploads each file via multipart/form-data. It does **not** trigger indexing — that's `datagol-index-document`'s job, which the agent can call right after if the user wants RAG.

> **Status: temporary.** Hand-rolled wrapper around two DataGOL HTTP endpoints. Will be replaced when a dedicated Pi tool ships.

## Two flows

- **Flow A — agent-led upload.** User attaches files in chat (they land at `.uploads/<safe-name>` per the existing attachments pipeline). Agent uses `bash`+`curl` to create the folder and POST each file to DataGOL.
- **Flow B — generate an upload UI.** User asks for a page where end users can drop files. Generates a React component using the postMessage token handshake.

Pick by intent; if unclear, ask one question and proceed.

## Workspace + folder choice

Always determine these before uploading:

1. **Workspace.** Either pick existing (call `datagol_list_workspaces`, present names, capture id) or create new (`datagol_create_workspace({ name })`). Capture as `WORKSPACE_ID`.
2. **Folder.** Default to a sensible name based on the user's intent (`Documents`, `Knowledge-Base`, `Reports`, etc.). The user can override. The folder is created under the workspace root unless the user names a parent.

---

## Flow A — agent-led upload (`bash` + `curl`)

The agent has `DATAGOL_TOKEN` in its env. User-attached files are at `.uploads/<safe-name>`.

### Step 1 — create the folder

```bash
WORKSPACE_ID="<id>"
FOLDER_NAME="<name>"

FOLDER_RESPONSE=$(curl -s -X POST \
  "https://be.datagol.ai/noCo/api/v2/workspaces/${WORKSPACE_ID}/folder" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"${FOLDER_NAME}\",\"parentId\":null}")
FOLDER_ID=$(echo "$FOLDER_RESPONSE" | jq -r '.id')
```

`parentId: null` puts the folder at the workspace root. Pass an existing folder id to nest. Response includes `id`, `name`, `workspaceId`, etc.

### Step 2 — upload each file (one request per file)

```bash
for f in .uploads/*; do
  curl -s -X POST \
    "https://be.datagol.ai/noCo/api/v2/workspaces/${WORKSPACE_ID}/folder/upload" \
    -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
    -F "files=@${f}" \
    -F "parentId=${FOLDER_ID}"
done
```

- **Use `-F`, not `-d`.** This is `multipart/form-data`.
- `files=@<path>` reads the file from disk and sends as the `files` form field.
- `parentId` is the folder id from step 1.
- One file per request — combining is unsupported.

### Step 3 — confirm + suggest next step

End with a short summary referencing files by name (not path):

> Uploaded **N** file(s) into folder *Documents* in workspace *Acme Knowledge*. Run `datagol-index-document` skill next if you want them searchable for a RAG agent.

Don't echo the workspace UUID, folder UUID, or full S3 paths.

---

## Flow B — generate an upload UI

User wants a sandbox page for non-technical end users to upload files. Reuse `datagol-agent-chat-ui`'s token handshake pattern.

### Files to write (verbatim)

1. **`src/lib/token.ts`** — copy the postMessage handshake from `datagol-agent-chat-ui` / `datagol-create-dashboard`. Same shape, no changes.

2. **`src/lib/upload-config.ts`** — workspace + folder constants. Hide UUIDs from the user-facing summary per the convention in `datagol-create-dashboard`.

   ```ts
   // src/lib/upload-config.ts
   export const WORKSPACE_ID  = '<id captured from workspace choice>';
   export const FOLDER_NAME   = '<sensible default>';
   export const DATAGOL_BE    = 'https://be.datagol.ai/noCo/api/v2';
   ```

3. **`src/components/FileUpload.tsx`** — drag-and-drop zone, file list with per-file progress, optional "+ New folder" if the user wants nested foldering. Implementation contract:

   - On first upload, lazily create the folder via `POST /workspaces/{ws}/folder` and stash `id` in `localStorage.UPLOAD_FOLDER_ID`. Subsequent uploads reuse it.
   - **Upload one file per request** with `FormData`: `files` (the `File`) + `parentId` (the folder id).
   - Per-file row shows: filename, size, status (`queued`, `uploading`, `done`, `failed`), and an `XHR.upload.onprogress` percentage during upload.
   - All requests use `Authorization: Bearer ${await awaitToken()}`.
   - Don't trigger indexing here — leave that to a separate "Index documents" button (or the `datagol-index-document` skill's UI).

### App.tsx wiring

Standalone (fresh project): `<App>` returns `<FileUpload />`.

Existing multi-page app: add `'upload'` to the nav state and route to `<FileUpload />`.

### After writing, tell the user

> Upload page added — open it from the nav and drop files in.

Don't surface the workspace UUID, folder UUID, or API URL.

---

## API reference (canonical)

All endpoints require `Authorization: Bearer <DATAGOL_TOKEN>`.

| Operation | Method · URL | Body / form fields |
|---|---|---|
| Create folder | `POST https://be.datagol.ai/noCo/api/v2/workspaces/{workspaceId}/folder` | JSON `{ name: string, parentId: string \| null }`. Returns `{ id, name, workspaceId, … }`. |
| Upload file | `POST https://be.datagol.ai/noCo/api/v2/workspaces/{workspaceId}/folder/upload` | multipart with `files=@<path>` (one per request) and `parentId=<folderId>`. Returns the file record. |

## Hard Rules

- **Don't hardcode or echo `DATAGOL_TOKEN`.** Pi has it via env; generated UI gets it via the postMessage handshake. Never print it.
- **Always create the folder before the first upload** — `parentId: null` is allowed but lands files at workspace root, which is messy.
- **One file per upload request.** The `files` field is single-valued; multiple in one request is undefined.
- **Don't trigger indexing here.** That's `datagol-index-document`. Keep this skill focused.
- **Don't echo the workspace UUID or folder UUID** in the user-facing summary. Reference them by name.
- **Don't ask the user for the API URL or token.** Both are baked in / injected.

## Cross-references

- **`datagol-index-document`** — trigger and check the search index over uploaded files. Run after this skill when the user wants RAG.
- **`datagol-create-agent`** — when the user is ready to wire a chat agent over the indexed workspace; that skill knows how to add a `WORKSPACE_DOCUMENTS` connector.
- **`datagol-workbook-operations`** — for structured (row/column) data instead of unstructured documents.
- **`datagol-context`** — DataGOL data model.
