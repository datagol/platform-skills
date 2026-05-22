---
name: datagol-workbook-operations
description: Full reference for DataGOL workbook operations — listing, querying, and mutating rows, columns, and AI columns via REST. Use this whenever you need to read or write actual workbook data (not just schema).
---

# Workbook Operations

This skill covers everything you can do **inside a workbook** via the DataGOL REST API: query and paginate rows, fetch a row by id, add/update/delete rows (single and bulk), and create/update/delete columns (including AI columns).

For **creating workbooks** and **shaping the schema** (relationships, design choices), use:
- `datagol_create_workbook` tool + `datagol-workbook-design` skill — to scaffold a new workbook.
- `datagol_add_column` tool + `datagol-create-links` skill — to add scalar columns or LINK columns.
- `datagol_get_workspace_schema` tool — to discover IDs.

This skill is for **everything else**: rows, AI columns, and operations that don't have a dedicated Pi tool.

## Authentication

All endpoints require a Bearer JWT in the `Authorization` header. The token comes from one of:

- **Login** — `POST https://be.datagol.ai/idp/api/v1/user/login` (email + password); the token comes back in the `Authorization` response header. This is what our platform uses.
- **Generate token** — `POST https://be.datagol.ai/idp/api/v1/user/token` with `{ email, password }`; returns `{ token: "ey..." }` in the JSON body.

```http
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json
```

Inside the Pi extension, reuse `process.env.DATAGOL_TOKEN`. Inside generated chat / CRUD apps, the user pastes their token (`localStorage.DATAGOL_TOKEN`) — see the `datagol-ui-generation` and `datagol-agent-chat-ui` skills.

## Base URL

```
https://be.datagol.ai/noCo/api/v2
```

All paths below are relative to that base. Path placeholders:
- `{workspaceId}` — UUID from `datagol_list_workspaces`.
- `{workbookId}` — UUID from `datagol_get_workspace_schema` (`workbooks[].id`).
- `{rowId}` — integer or string row id from a query response.
- `{columnId}` — UUID from the schema response.

---

## Reading Data

### Get table schema (columns, types, link metadata)

`GET /workspaces/{workspaceId}/tables/{workbookId}`

Returns the full table definition: every column's `id`, `name` (internal key used in `cellValues`), display `title`, `uiDataType`, `colOptions` (link/lookup metadata), and `isDataEditable` flag.

> ⚠️ **MANDATORY before any write.** Always call this (or `datagol_get_workspace_schema`) immediately before `POST /rows`, `POST /rows/bulk`, or `PATCH /rows/:id` — even if you wrote to the same table earlier in the session. The schema tells you:
>
> - The exact internal `name` to use as each `cellValues` key (display titles do NOT work).
> - Which columns are `DATE` (so you can run `normalizeDate()` on them).
> - Which columns are `CHECKBOX` / `NUMBER` / `LINK` (so you coerce to the right JSON type — booleans, numbers, arrays of linked-row ids).
> - Which columns are read-only (`isDataEditable: false`, link reciprocals, formula columns) — drop these from the body.
> - Which keys are NOT real columns at all — any `cellValues` key that isn't in this response will 400 the **entire** batch.
>
> Skipping this step is the single most common cause of `400 Bad Request` on bulk inserts. Cache the response per `(workspaceId, workbookId)` for the duration of the operation; refetch if you get a 400 mentioning an unknown field.

Minimal validation pass before sending a bulk write:

```ts
const { columns } = await getTableSchema(workspaceId, workbookId);
const byName = new Map(columns.map(c => [c.name, c]));
const writable = columns.filter(c => c.isDataEditable !== false);
const dateCols = writable.filter(c => c.uiDataType === 'DATE' || c.uiDataType === 'DATETIME');

for (const row of rows) {
  for (const key of Object.keys(row.cellValues)) {
    if (!byName.has(key)) throw new Error(`Unknown column "${key}" — not in table schema`);
    if (byName.get(key)!.isDataEditable === false) delete row.cellValues[key];
  }
  for (const col of dateCols) {
    if (col.name in row.cellValues) row.cellValues[col.name] = normalizeDate(row.cellValues[col.name]);
  }
}
```

### Resolve the underlying SQL table name (for use with `datagol-data-query`)

Every DataGOL workbook that is backed by a materialized view or a published SQL query has an underlying database table name stored in its metadata. You **must** retrieve this name from the API — never guess it or hard-code it. The table name is NOT derivable from the workbook UUID alone.

**Step 1 — fetch workbook metadata:**

```bash
curl -s "${DATAGOL_BASE_URL}/noCo/api/v2/workspaces/{workspaceId}/tables/{workbookId}" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}"
```

**Step 2 — read the `tableName` field from the response:**

```json
{
  "id": "6806cf69-e8b9-4e25-bfc5-3dea63e038b6",
  "title": "Hubspot - All Deals Info",
  "tableName": "table_ab8ef6fa",
  "sourceType": "...",
  ...
}
```

The value of `tableName` (e.g. `table_ab8ef6fa`) is the actual Postgres table name in the `app_connection_read` schema.

**Step 3 — use it in SQL queries:**

```sql
SELECT * FROM app_connection_read.table_ab8ef6fa LIMIT 10
```

> ⚠️ **Always call the metadata endpoint — never guess the table name.** The `tableName` field follows a pattern like `table_<8-char-hex>` but the hex suffix is **not** derived from the workbook UUID in any predictable way. The only reliable source is `GET /tables/{workbookId}` → `tableName`.

> ✅ **Cross-reference with `datagol-data-query`.** Once you have `tableName`, use it with the `app_connection_read` data source for SQL analytics via `POST /dataSources/api/v1/internal/queryResults`. This is more powerful than the `/cursor` endpoint for aggregations, JOINs, and complex filters:

```bash
curl -X POST "${DATAGOL_BASE_URL}/dataSources/api/v1/internal/queryResults" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "sqlQuery": "SELECT * FROM app_connection_read.table_ab8ef6fa ORDER BY revenue::numeric DESC LIMIT 10",
    "dataProvider": "JDBC"
  }'
```

> ⚠️ **`generate_series` and DDL are not supported** via the JDBC query endpoint. Use subqueries to handle type casting (cast inside a subquery, then filter/sort in the outer query) — direct `column::type` casts in `WHERE` clauses sometimes fail when the data source returns 0 rows silently.

---

### Query rows (paginated, filtered, sorted)

`POST /workspaces/{workspaceId}/tables/{workbookId}/cursor`

```json
{
  "requestPageDetails": { "pageNumber": 1, "pageSize": 100 },
  "sortOptions":  [{ "columnName": "created_at", "direction": "DESC" }],
  "whereClause":  "`status` = 'open' AND `priority` > 3"
}
```

- `requestPageDetails` (required) — `pageNumber` (1-based) and `pageSize`.
- `sortOptions` (optional) — array of `{ columnName, direction: 'ASC' | 'DESC' }`.
- `whereClause` (optional) — SQL-like predicate. Wrap column names in backticks. Supports `AND`/`OR`, `=`/`!=`/`<`/`>`/`<=`/`>=`, `LIKE '%foo%'`, `BETWEEN '...' AND '...'`, `IN (...)`.

Use the **internal column name** (the lowercase `name` in `datagol_get_workspace_schema`), not the display title.

> ⚠️ **`whereClause` format gotcha:** The filter syntax is raw SQL — NOT the `(column,eq,value)` tuple format used by some other APIs. Using `(username,eq,john)` causes `ERROR: column "eq" does not exist`. Always use backtick-quoted SQL:
> ```
> "`username` = 'john'"
> "`status` = 'open' AND `priority` > 3"
> ```
> In TypeScript: `` whereClause: `\`username\` = '${value}'` ``

**Response shape:**

```json
{
  "rows": [
    {
      "id": 1,
      "cellValues": { "name": "Acme", "status": "open", "datagol_created_at_xxx": "...", "noco_xxx": 1 },
      "cellValuesByColumnId": {}
    }
  ],
  "totalNumberOfRecords": 42,
  "pageNumber": 1
}
```

> ⚠️ **Data is nested inside `cellValues`, not flat on the row.** Also contains system columns (`datagol_created_at_xxx`, `noco_xxx`, position keys) — ignore these when mapping to app types.

Flatten before use:

```ts
const rows = data.rows ?? [];
return rows.map((r: any) => ({ id: r.id, ...r.cellValues }));
```

### Get a single row by id

`GET /workspaces/{workspaceId}/tables/{workbookId}/rows/{rowId}`

Response:

```json
{
  "id": 1,
  "values": { "name": "dega", "status": "Active", "ai_summary": "…" },
  "createdAt": "2026-02-20T10:00:00Z",
  "updatedAt": "2026-02-26T12:45:00Z"
}
```

---

## Writing Rows

> ⚠️ **Text columns are `varchar(255)` under the hood — truncate before writing.** Both `SINGLE_LINE_TEXT` and `MULTI_LINE_TEXT` map to Postgres `varchar(255)` on at least `testing-be` (and likely prod). The `MULTI_LINE_TEXT` name is misleading — it does NOT mean unlimited length, it just means the UI renders a multi-line input. Sending anything longer 400s with:
>
> ```
> ERROR: value too long for type character varying(255)
> ```
>
> And because bulk insert is atomic, **one over-long value kills the entire batch.** This bites hard when stuffing JSON blobs (Slack `blocks_json` / `files_json`, Gmail headers, raw API payloads) into `MULTI_LINE_TEXT` columns — they routinely exceed 255 chars.
>
> **Fix:** in the generated app's API layer, fetch the table schema once (cached per `tableId`) and run every write through a sanitizer that truncates `SINGLE_LINE_TEXT` and `MULTI_LINE_TEXT` values to ~250 chars (leave a margin under 255) and coerces DATE values via `toDataGolDate()`. Wire it into `insertRow`, `bulkInsertRows`, and `updateRowById` so callers can't forget.
>
> ```ts
> // src/api/workbooks.ts
> const TEXT_MAX = 250;
> const schemaCache = new Map<string, Promise<TableColumn[]>>();
>
> async function getTableColumns(tableId: string) {
>   let p = schemaCache.get(tableId);
>   if (!p) {
>     p = dgFetch(`/noCo/api/v2/workspaces/${WORKSPACE_ID}/tables/${tableId}`)
>       .then((d: any) => d?.columns ?? []);
>     schemaCache.set(tableId, p);
>   }
>   return p;
> }
>
> export async function sanitizeCellValues(tableId: string, cv: Record<string, unknown>) {
>   const cols = await getTableColumns(tableId);
>   const byName = new Map(cols.map((c: any) => [c.name, c]));
>   const out: Record<string, unknown> = {};
>   for (const [k, v] of Object.entries(cv)) {
>     const col = byName.get(k);
>     if (!col) continue;                         // unknown column → drop (would 400)
>     if (v == null) { out[k] = v; continue; }
>     if (col.type === 'DATE' || col.type === 'DATETIME') { out[k] = toDataGolDate(v); continue; }
>     if (col.type === 'SINGLE_LINE_TEXT' || col.type === 'MULTI_LINE_TEXT') {
>       const s = typeof v === 'string' ? v : String(v);
>       out[k] = s.length > TEXT_MAX ? s.slice(0, TEXT_MAX) : s;
>       continue;
>     }
>     out[k] = v;
>   }
>   return out;
> }
> ```
>
> If users actually need long text (chat messages, doc contents, raw JSON), the workbook design is wrong — those fields belong on a workbook with a real `TEXT` / large-string column type, not `MULTI_LINE_TEXT`. Flag this back to the user instead of silently truncating data they care about.


### Add one row

`POST /workspaces/{workspaceId}/tables/{workbookId}/rows`

```json
{
  "cellValues": {
    "first_name": "John",
    "years_of_experience": 5,
    "is_active": true,
    "start_date": "2024-02-18T05:00:00+00:00"
  }
}
```

- Keys are **internal column names** (not display titles).
- Dates use ISO 8601 with **time + timezone** (see below).
- Booleans / numbers / strings as native JSON types.

> ⚠️ **Body must be wrapped in `cellValues`.** A flat body (`{ "name": "Acme" }`) returns `"Validation failed: cellValues must not be null or empty"`. Always wrap: `{ "cellValues": { "name": "Acme" } }`.

> ⚠️ **DATE columns are picky about format.** The API accepts exactly one shape: `YYYY-MM-DDTHH:MM:SS±HH:MM` (ISO 8601, seconds precision, explicit offset — no milliseconds, no `Z`). Every other common form needs to be normalized before sending. The four ways this bites in practice:
>
> 1. **`<input type="date">`** → `2026-04-30` → must become `2026-04-30T00:00:00+00:00`.
> 2. **`<input type="datetime-local">`** → `2026-04-30T15:23` → must become `2026-04-30T15:23:00+00:00`.
> 3. **`new Date().toISOString()`** → `2026-04-30T15:23:51.482Z` → must become `2026-04-30T15:23:51+00:00`. **This is the most common cause of mysterious 400s on bulk inserts** — `.toISOString()` looks correct, but the trailing `.482Z` (millis + `Z`) is rejected, and because bulk is atomic the *entire batch* fails. We have hit this multiple times — never call `.toISOString()` directly when writing to a DATE column.
> 4. **Unix epoch seconds (e.g. Slack `ts`, Stripe timestamps)** → multiply by 1000, build a `Date`, then run through the normalizer below.
>
> Use **one** helper for every DATE-bound value, regardless of input type. Read the schema via `datagol_get_workspace_schema` (or `GET /tables/{id}`) to know which columns are `DATE` and apply this to each of them:
>
> ```ts
> /**
>  * Coerce any reasonable input into the exact shape DataGOL DATE columns accept:
>  * `YYYY-MM-DDTHH:MM:SS±HH:MM` (no milliseconds, no `Z`).
>  * Accepts: Date, epoch ms (number), or string in any of the common partial forms.
>  */
> function toDataGolDate(value: unknown): unknown {
>   if (value == null || value === '') return value;
>   if (value instanceof Date) return value.toISOString().replace(/\.\d{3}Z$/, '+00:00');
>   if (typeof value === 'number') return new Date(value).toISOString().replace(/\.\d{3}Z$/, '+00:00');
>   if (typeof value !== 'string') return value;
>   // 2026-04-30 → 2026-04-30T00:00:00+00:00
>   if (/^\d{4}-\d{2}-\d{2}$/.test(value)) return `${value}T00:00:00+00:00`;
>   // 2026-04-30T15:23 → 2026-04-30T15:23:00+00:00
>   if (/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}$/.test(value)) return `${value}:00+00:00`;
>   // 2026-04-30T15:23:00 (no tz) → append +00:00
>   if (/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}$/.test(value)) return `${value}+00:00`;
>   // 2026-04-30T15:23:51.482Z → 2026-04-30T15:23:51+00:00  (the .toISOString() trap)
>   if (/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$/.test(value)) return value.replace(/\.\d{3}Z$/, '+00:00');
>   // Already a full ISO 8601 with explicit offset — leave alone.
>   return value;
> }
>
> // Apply per DATE column when building the cellValues body:
> for (const col of dateColumns) {
>   if (col.name in cellValues) cellValues[col.name] = toDataGolDate(cellValues[col.name]);
> }
> ```
>
> Export this helper from your generated app's API layer (e.g. `src/api/workbooks.ts`) and use it at every write site — not just form submits. Background sync code (Slack, Gmail, Calendar, etc.) frequently constructs dates via `new Date(epoch * 1000).toISOString()` and falls into the `.482Z` trap; route those through `toDataGolDate()` too.
>
> Failure modes you'll see if you skip this:
> - `Validation failed: <field> must match the date format` (single insert).
> - Generic `400 Bad Request` with no field name (bulk insert) — because one bad date kills the whole batch.

**Response shape:**

```json
{ "id": 1, "cellValues": { "name": "Acme", ... }, "cellValuesByColumnId": {} }
```

Flatten with: `{ id: data.id, ...data.cellValues }`

### Add many rows (bulk, atomic)

`POST /workspaces/{workspaceId}/tables/{workbookId}/rows/bulk`

Body is a JSON **array** of `{ cellValues }` objects:

```json
[
  { "cellValues": { "first_name": "John",  "is_active": true  } },
  { "cellValues": { "first_name": "Jane",  "is_active": false } }
]
```

All rows insert atomically — if one fails validation, none are inserted.

### Update a row

`PUT /workspaces/{workspaceId}/tables/{workbookId}/rows`

```json
{ "id": 42, "cellValues": { "name": "dega", "status": "Closed" } }
```

Only the fields in `cellValues` change. Triggers AI-column regeneration if any AI column depends on a changed field; recalculates formulas.

> ⚠️ **The correct verb is `PUT /rows` (no `:id` in the path), not `PATCH /rows/:id`.** The row id goes in the **request body** as `"id": <integer>`, not in the URL. `PATCH /rows/:id` returns HTTP 405 Method Not Allowed. `PUT /rows` returns HTTP 200 with the updated row on success.
>
> ```ts
> // ✅ correct
> await fetch(`${BASE}/workspaces/${WS}/tables/${tableId}/rows`, {
>   method: 'PUT',
>   headers: { 'Content-Type': 'application/json', 'x-auth-token': token },
>   body: JSON.stringify({ id: Number(rowId), cellValues: { status: 'Closed' } }),
> });
>
> // ❌ wrong — always returns 405
> await fetch(`${BASE}/workspaces/${WS}/tables/${tableId}/rows/${rowId}`, {
>   method: 'PATCH', ...
> });
> ```

> ⚠️ **Strip system fields before echoing rows back on update.** If your UI reads a row, lets the user edit it, and then PATCHes the result, you MUST drop fields the server added on read — they make the API reject the request. The set to strip:
>
> - Anything starting with `datagol_` — `datagol_created_at_*`, `datagol_created_by_*`, `datagol_updated_at_*`, `datagol_updated_by_*` (audit columns).
> - Anything starting with `noco_` — internal row identifier (e.g. `noco_56379c52`).
> - Anything starting with `_position` — UI ordering state.
> - **Link reciprocal counts** — fields whose value is a number and whose name matches a workbook display name (e.g. `Leads`, `Opportunities`). These come from `LINK` columns on other tables and aren't editable here.
> - The top-level `id` field on the row (the row id goes in the URL, not the body).
>
> Concrete helper for the generated app:
>
> ```ts
> // sanitizeForWrite — drop server-added fields so PATCH bodies stay valid.
> function sanitizeForWrite<T extends Record<string, unknown>>(row: T): Partial<T> {
>   const out: Record<string, unknown> = {};
>   for (const [k, v] of Object.entries(row)) {
>     if (k === 'id') continue;
>     if (k.startsWith('datagol_')) continue;
>     if (k.startsWith('noco_')) continue;
>     if (k.startsWith('_position')) continue;
>     out[k] = v;
>   }
>   return out as Partial<T>;
> }
>
> // Usage in the update flow:
> await fetch(`${BASE}/rows/${rowId}`, {
>   method: 'PATCH',
>   headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
>   body: JSON.stringify({ cellValues: sanitizeForWrite(editedRow) }),
> });
> ```
>
> If you want to be defensive about link reciprocals, also drop any field whose key isn't in the workbook's editable column list (look it up via `datagol_get_workspace_schema` and filter by `isDataEditable`).

### Delete a single row

`DELETE /workspaces/{workspaceId}/tables/{workbookId}/rows/{rowId}`

No body. Returns HTTP 200 with an **empty body** on success.

### Delete multiple rows (bulk)

`DELETE /workspaces/{workspaceId}/tables/{workbookId}/rows`

```json
{ "rowIds": [40, 41, 42] }
```

User must have write/delete permission on the workbook.

---

## Schema Mutations

For the common cases — creating a workbook with columns, adding scalar columns, adding LINK columns — **prefer the Pi tools** (`datagol_create_workbook`, `datagol_add_column`, `datagol_create_link`). They wrap the right boilerplate.

For everything else, hit these endpoints directly:

### Update a column

`PUT /workspaces/{workspaceId}/tables/{workbookId}/columns/{columnId}`

```json
{
  "id":          "<columnId>",
  "name":        "internal_name",
  "uiDataType":  "SINGLE_LINE_TEXT",
  "uiMetadata":  { "title": "New Display Title", "precision": null },
  "colOptions":  {}
}
```

Use to rename a column (display title), change its data type, or tweak `colOptions` for LINK / LOOKUP columns.

### Delete a column

`DELETE /workspaces/{workspaceId}/tables/{workbookId}/columns/{columnId}`

No body. Removes the column and all cell values in it. Irreversible — confirm with the user before calling.

### Create an AI column

`POST /workspaces/{workspaceId}/tables/{tableId}/column`

```json
{
  "name":       "first_name",
  "uiDataType": "SINGLE_LINE_TEXT",
  "settings": {
    "aiSettings": {
      "enabled":   true,
      "prompt":    "Extract first name from full_name",
      "model":     "gpt-4o-mini",
      "webAccess": false
    }
  }
}
```

The AI runs the prompt on every row, with other column values available by name. Pick a small / fast model (`gpt-4o-mini`) unless the user asks for higher quality. Set `webAccess: true` only when the prompt actually needs the live web.

---

## Workflow Patterns

### "Show me the last 10 deals"

1. `datagol_get_workspace_schema` → find the `Deal` workbook id.
2. Query: `POST /cursor` with `requestPageDetails: { pageNumber: 1, pageSize: 10 }` and `sortOptions: [{ columnName: 'created_at', direction: 'DESC' }]`.
3. Render the rows.

### "Add a contact named …"

1. `datagol_get_workspace_schema` to confirm the `Contact` workbook id and column names.
2. Map the user-provided values to `cellValues` keyed by internal column names.
3. `POST /rows` (single).
4. Confirm to the user with the returned row id.

### "Edit row N — change status from Open to Closed"

This is the read-edit-write round-trip the CRUD UIs hit most often. The trap: the row you read back includes system fields the API will reject if echoed.

1. Read the row (cursor or single-row endpoint) → flatten with `{ id, ...cellValues }`.
2. User edits a few fields in the UI.
3. **`sanitizeForWrite(editedRow)`** to drop `datagol_*`, `noco_*`, `_position*`, `id`, and link reciprocals.
4. `PUT /rows` with `{ id: Number(rowId), cellValues: <sanitized> }`. **Not PATCH — `PATCH /rows/:id` returns 405.**

Skipping step 3 is the single most common cause of `Validation failed` on update.

### "Backfill these 200 rows from the spreadsheet I pasted"

1. Parse the user input into `[{cellValues: {...}}, ...]`.
2. `POST /rows/bulk` (single call, atomic).
3. If validation fails, the error message will name the offending field — fix and retry.

### "Add a column that summarises X using AI"

1. Confirm the workbook + the source columns the prompt will reference.
2. `POST /column` with `settings.aiSettings` populated. Default to `gpt-4o-mini`.
3. New rows / row updates trigger regeneration automatically.

---

## Hard Rules

- **Always fetch the table schema before writing.** Call `GET /workspaces/{wsId}/tables/{tableId}` (or `datagol_get_workspace_schema`) immediately before any `POST /rows`, `POST /rows/bulk`, or `PATCH /rows/:id`. Validate every `cellValues` key against the returned `columns[].name`, drop non-editable columns, and coerce values to the column's `uiDataType`. Skipping this is the #1 cause of 400s on bulk inserts.
- **Always discover IDs first.** Call `datagol_get_workspace_schema` to get workbook + column ids. Do not fabricate.
- **Use internal column names** (lowercase `name`) as `cellValues` keys, not display titles.
- **ISO 8601 dates with seconds + explicit offset** for any DATE column — exactly `YYYY-MM-DDTHH:MM:SS±HH:MM`. Run every DATE-bound value through `toDataGolDate()` (helper above), including:
  - `<input type="date">` / `<input type="datetime-local">` values.
  - Anything from `new Date().toISOString()` — the trailing `.482Z` (millis + `Z`) is rejected and silently 400s entire bulk batches. **Never write `.toISOString()` directly into a DATE cell value.** This has bitten us repeatedly.
  - Epoch timestamps from third-party APIs (Slack `ts`, Gmail `internalDate`, etc.).
  Bare `YYYY-MM-DD` and the `.482Z` form both surface as `must match the date format` (single insert) or a generic `400 Bad Request` (bulk).
- **Confirm before destructive ops.** `DELETE rows` and `DELETE column` are irreversible — present a plan and wait for explicit "yes" (or rely on the platform's hard gate; see `datagol-human-in-the-loop` skill).
- **Bulk over loop.** When inserting > 1 row, use `POST /rows/bulk` instead of N calls to `POST /rows`.
- **Truncate text to 250 chars before writing.** Both `SINGLE_LINE_TEXT` and `MULTI_LINE_TEXT` are `varchar(255)` on the backend — *yes, even MULTI_LINE_TEXT*. Run every write through a schema-aware `sanitizeCellValues()` (helper above) that truncates strings and drops unknown columns. One over-long value 400s the entire bulk batch with `value too long for type character varying(255)`. If callers need to store long content (full message bodies, JSON blobs, doc text), the column type is wrong — don't paper over it with truncation, raise it with the user.
- **Never assume flat row shape.** Both the cursor and single-row endpoints return `{ id, cellValues: { ... } }`. Always flatten with `{ id: r.id, ...r.cellValues }` before mapping to TypeScript types. Never access fields directly on the raw row object.
- **Always wrap writes in `cellValues`.** `POST /rows` requires `{ cellValues: { ...fields } }`. `PUT /rows` (update) requires `{ id: <integer>, cellValues: { ...fields } }`. A flat body returns `"Validation failed: cellValues must not be null or empty"`.
- **Strip system fields before write.** When the UI does a read-edit-write round-trip, drop `datagol_*`, `noco_*`, `_position*`, the `id` field, and link reciprocal counts before sending. The API rejects updates that try to set these. Use the `sanitizeForWrite()` helper above.
- **Token reuse.** The DataGOL token from the user's session is the same token used for every endpoint here. Don't ask the user to re-authenticate.

---

## Cross-References

- **`datagol-data-query`** — when running SQL analytics against a workbook's underlying table. Always resolve the `tableName` from workbook metadata first (see above).
- **`datagol-workbook-design`** — when designing the workbook itself (column choice, types, naming).
- **`datagol-create-links`** — when adding LINK columns / building relationships.
- **`datagol-context`** — DataGOL data model (Workspace → Workbook → Column / Row).
- **`datagol-ui-generation`** — when building a CRUD UI on top of these endpoints.
- **`datagol-human-in-the-loop`** — what to gate before mutating rows or schema.
