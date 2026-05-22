---
name: datagol-data-publish
description: Publish a SQL query against a registered DataGOL data source as a **materialized-view workbook** that lives in the user's workspace. Triggered by "save this query as a workbook", "create a workbook from my Postgres data", "materialize <SQL> into a table", "publish this dataset", "I want a workbook of <X> from my database". Generates the SQL, posts to `/noCo/api/v2/workspaces/{workspaceId}/publishTable`, and returns the resulting workbook id. Pairs with `datagol-data-query` (use that for "show me right now" without persisting).
---

# DataGOL Data Publish — Generate SQL, Materialize as Workbook

This skill lets the agent take a SQL query against a DataGOL-registered data source and turn it into a **persistent workbook** in the user's workspace. The workbook is *materialized* from the SQL — DataGOL runs the query, captures the result, and creates a regular workbook the user can read/write/dashboard against using the rest of the platform (`datagol-workbook-operations`, `datagol-create-dashboard`, `datagol-create-agent` MCPs, etc.).

It's the durable companion to **`datagol-data-query`**:
- *Query* → "show me X right now" → ephemeral table in the chat.
- *Publish* → "save X as a workbook so I can build a dashboard / agent / app on it" → permanent workbook.

> **Read `datagol-data-connector` first.** Schema discovery (`getDataSourceSchema`) and the `Data Connectors` workbook are defined there. Most of the SQL-generation guidance lives in `datagol-data-query` — this skill cross-references rather than duplicates.

## When to use

- "Create a workbook of all customer orders from my Postgres."
- "Publish a table of top 100 films by rental count."
- "I want a dashboard of monthly revenue — first save the data as a workbook."
- "Materialize this query: `<SQL>`."
- "Make me a workbook from `datawarehouse_plsql.public.actor`."

When **not** to use:
- The user wants a one-off answer, not a saved table → use `datagol-data-query`.
- The user wants to *load* an existing workbook into the app → `datagol-workbook-operations`.
- The user wants to *modify* the source database → not supported; refuse.

## The 4-step flow

```
1. Identify the target data source        →  ask user by name (or use whatever was just registered)
2. Fetch schema                           →  GET /dataSources/api/v1/dataSources/{id}/schemas
3. Generate SQL grounded in the schema    →  same FROM convention as datagol-data-query
4. Pick the publish target workspace      →  ask user, or use the current workspace context
5. Publish                                →  POST /noCo/api/v2/workspaces/{ws}/publishTable
```

Steps 1–3 are identical to `datagol-data-query` — same "ask the user by name; don't presume the `Data Connectors` workbook exists" rule.

> **Workspace ≠ `Data Connectors` workbook.** The publish call's `{workspaceId}` URL parameter is *which DataGOL workspace the new workbook lands in* — required, but it has nothing to do with the `Data Connectors` workbook (which is a Flow-B-only artefact from the parent skill). If the user is in an existing workspace context, use it; if not, ask them to pick one or create one (`datagol_create_workspace`). Don't conflate the two.

### Step 4 — Publish via `/publishTable`

```bash
curl -X POST "${DATAGOL_BASE_URL}/noCo/api/v2/workspaces/${WORKSPACE_ID}/publishTable" \
  -H "x-auth-token: ${DATAGOL_SERVICE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "tableName": "plsql-test",
    "description": "",
    "isCopyRequired": false,
    "dataProvider": "JDBC",
    "queryEditorId": 46093,
    "materializedViewQuery": "select * from datawarehouse_plsql.public.actor"
  }'
```

Body fields:

| Field | Type | Notes |
|---|---|---|
| `tableName` | string | **Required.** The workbook display name. User-supplied; sanitize: trim, replace internal whitespace with `-` or `_`, drop weird characters. Reasonable max ~64 chars. |
| `description` | string | Optional human-readable description (defaults to `""`). Useful when the agent picks the SQL — write what it returns. |
| `isCopyRequired` | bool | **Always `false`** for materialized views. Setting it to `true` would create a one-time data copy instead of a live materialized view; we don't support that path in v1. |
| `dataProvider` | string | Always `"JDBC"` for the SQL data sources we support. |
| `queryEditorId` | number | **Required, but value is arbitrary.** The backend uses it for tracking; any positive integer works. Generate `Math.floor(Math.random() * 1_000_000) + 1` per call to keep them unique. Don't reuse a fixed value across calls — that's a footgun if the backend ever de-dupes. |
| `materializedViewQuery` | string | **Required.** The SQL that defines the workbook's contents. Must follow the same `<dataSourceName>.<schema>.<table>` FROM convention as `datagol-data-query`. |

```ts
// src/api/dataPublish.ts
import { dgFetch } from './datagol';

export interface PublishedTable {
  id: string;          // the new workbook id
  name: string;
  // Backend may include createdAt, columns, etc. — pass through.
}

export async function publishTable(
  workspaceId: string,
  tableName: string,
  materializedViewQuery: string,
  description = '',
): Promise<PublishedTable> {
  return dgFetch<PublishedTable>(
    `/noCo/api/v2/workspaces/${encodeURIComponent(workspaceId)}/publishTable`,
    {
      method: 'POST',
      body: JSON.stringify({
        tableName: sanitizeTableName(tableName),
        description,
        isCopyRequired: false,
        dataProvider: 'JDBC',
        queryEditorId: Math.floor(Math.random() * 1_000_000) + 1,
        materializedViewQuery,
      }),
    },
  );
}

function sanitizeTableName(name: string): string {
  return name.trim().replace(/\s+/g, '-').replace(/[^a-zA-Z0-9_-]/g, '').slice(0, 64) || 'untitled';
}
```

## Workflow & UX

1. **Confirm the user wants persistence.** If the request is ambiguous ("show me top customers") and the user hasn't said "save / publish / workbook / table", default to `datagol-data-query` (ephemeral). Only switch to publish on explicit intent.
2. **Show the generated SQL before publishing.** Just like `datagol-data-query`, echo the generated `materializedViewQuery` in a fenced code block so the user can verify. Wait for confirmation before calling `publishTable` — this is a workspace-modifying operation.
3. **Pick a sane `tableName`.** Either ask the user, or derive from intent:
   - User said "save the actor table as is" → `tableName = "actor"`.
   - User said "create a top-10-films workbook" → `tableName = "top-10-films"`.
   - When unsure, propose 2–3 names and let them pick. Avoid generic names like `query_result`.
4. **After success, surface the new workbook.** Return:
   - The workbook `id` (from response).
   - A short markdown link if the platform exposes one — `[Open <name>](https://app.datagol.ai/workspaces/${WORKSPACE_ID}/workbooks/${id})` (URL shape varies by env; cross-check against `datagol-context`).
   - A one-line "what now" suggestion: *"You can now build a dashboard on this with `datagol-create-dashboard`, expose it to an agent via `datagol-create-agent`'s MCP wiring, or query it inline with `datagol-workbook-operations`."*
5. **Don't auto-run a follow-up.** Publishing is a one-shot. If the user asks "and now show me the rows", that's a separate `datagol-workbook-operations` call against the new workbook.

## Materialized-view freshness

The published workbook is **a materialized view** — it captures the result of running the SQL at publish time. Whether (and how) it refreshes against the underlying data source is platform-managed; this skill doesn't touch refresh policy. If the user asks "how often does it refresh", say it's a DataGOL platform concern and they should check the workbook's settings UI; flag refresh-policy management as a follow-up workstream.

If the user wants **fresh, live** data (every read goes back to the source), the answer in v1 is: *"That requires live federation, which the publish flow doesn't enable. Use `datagol-data-query` for one-off live queries; for repeatable queries against fresh data, re-publish (or wait for the live-federation skill which doesn't exist yet)."*

## Hard rules

- **Always qualify FROM as `<dataSourceName>.<schema>.<table>`.** Same convention as `datagol-data-query`. Wrong qualifier = "relation does not exist" = workbook publish fails.
- **Confirm before publishing.** This creates a workbook in the user's workspace. Show the SQL + the proposed `tableName` and wait for explicit "yes / go / publish".
- **`isCopyRequired: false` always.** We materialize views, not copies, in v1.
- **`queryEditorId` is per-call random.** Don't hard-code a value; generate fresh on every publish.
- **Sanitize `tableName`.** Trim, replace whitespace with `-`, drop characters outside `[a-zA-Z0-9_-]`, cap at 64. The backend may reject special characters silently.
- **`materializedViewQuery` is `SELECT`-only.** Same SQL safety rules as `datagol-data-query` — no DML, no DDL.
- **Use `x-auth-token: <SERVICE_TOKEN>`** for the publish call.
- **Always echo the SQL** the agent intends to publish — in a fenced ```sql``` block in your prose, *every single time*. The chat's `Running` tool row is collapsed/truncated and is not a substitute. The user must be able to read, copy, and edit the `materializedViewQuery` before you call `publishTable`. Same trust-by-default rule as `datagol-data-query`.
- **Don't publish multi-statement scripts.** The endpoint takes one materialized view per call. If the user wants two related tables, publish twice.

## Cross-references

- **`datagol-data-connector`** — parent. Registration + schema discovery + `Data Connectors` workbook live there.
- **`datagol-data-query`** — sibling. Use when the user wants the result *now*, not saved.
- **`datagol-workbook-operations`** — what to use *after* publish to read rows from the materialized workbook.
- **`datagol-create-dashboard`** — what to use after publish if the user wants to chart the data.
- **`datagol-create-agent`** — what to use after publish if the user wants an MCP-style agent over the new workbook.
- **`datagol-app-auth`** — service token + env config. Required.
