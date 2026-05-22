---
name: datagol-data-query
description: Generate and execute ad-hoc SQL against a DataGOL-registered data source, then render the result inline in chat. Triggered by "show me X from my Postgres", "query my database for Y", "how many Z are in my data source", "what's the top N of W", "run a SQL query against <data source>", or any analytical question that targets a registered data source. Loads the source's schema, generates the SQL, posts it to `/dataSources/api/v1/internal/queryResults`, and shows the rows. Pairs with `datagol-data-publish` when the user wants the result saved as a workbook instead of (or in addition to) being shown right now.
---

# DataGOL Data Query — Generate SQL, Execute, Render

This skill lets the agent answer data questions by writing and running SQL against any data source the user has registered via `datagol-data-connector`. It's an **agent-runtime** skill — the agent itself fetches the schema, generates the SQL, calls the query API, and includes the result in its chat response. It does *not* scaffold a UI in the generated app; for that, use `datagol-data-publish` (creates a queryable workbook) or build a CRUD UI on top of the published workbook with `datagol-ui-generation`.

> **Read `datagol-data-connector` first.** This skill assumes the user has already registered a data source and you have its `data_source_id`. Schema discovery (`getDataSourceSchema`) lives in the parent.

---

## ⛔ STOP — read this template before generating anything

**Every single answer this skill produces follows the same shape.** No exceptions, no creative reordering.

```
1. ```sql
   <the SQL the agent ran>
   ```

2. <prose summary OR Markdown table OR single number — the actual answer>

3. <follow-up question, ONE line, only when relevant>:
   "Want me to chart this?"  ← only if the data could benefit from a chart AND the user didn't already ask for one
```

That's the entire response shape. Three blocks, in this order, every time. **Do not** write a chart file (`*.html`, `*.svg`) in this same turn unless the user's *current* message contains an explicit visual ask (chart / plot / graph / visualize / draw / dashboard / "make it visual"). Saying *"All data in. Now let me build the chart."* and creating an HTML file is **wrong** — it skips the answer (block 2) and assumes a request the user didn't make. Stop after block 3 and wait.

### Anti-patterns (don't do these)

❌ **Skip the data, jump to the chart.** *"Got the rows. Building the chart now."* + write_file `chart.html`. The user can't read raw rows; you owe them block 2 first. And you owe them silence after the question, not a chart they didn't ask for.

❌ **Bury the SQL in the curl tool row.** The chat's `Running` tool row truncates after ~200 chars and is collapsed by default. The user can't review or copy the SQL from there. The fenced ```sql block in your prose is the contract.

❌ **"They'll probably want a chart."** No. If they wanted one, they would have said so. Suggest, don't pre-build.

✅ **Good.** SQL block → 5-row Markdown table or one-line summary → *"Want me to chart this?"* → STOP.

---

## When to use

- "How many films are in my Postgres database?"
- "Show me the top 10 customers by revenue from my warehouse."
- "What's the distribution of payment amounts last month?"
- "Run this query against my data source: `…`"
- "Sample 50 rows from the `actor` table."

When **not** to use:
- The data isn't in a DataGOL-registered SQL source — it's in a workbook. Use `datagol-workbook-operations` instead.
- The user wants to **save** the result, not just see it. Use `datagol-data-publish`.
- The user wants to repeatedly query the same shape from a generated app's UI. Publish first, then build a workbook UI with `datagol-ui-generation`.

## Response composition (read this before generating anything)

Three rules that override anything below if they conflict. They apply on every single answer this skill produces.

### 1. Always show the executed SQL

Echo the SQL the agent executed in a fenced ```sql``` block in your prose answer — *every time*, with no exceptions. The chat panel may also show a `Running` tool row that *contains* the SQL inside a curl, but that row is collapsed/truncated and isn't a substitute. Users need to be able to read, copy, and edit the query without expanding tool args. Place the SQL block in the prose, before the result.

```sql
SELECT product_name, units_sold, revenue
FROM datawarehouse_plsql.public.sales_summary
ORDER BY units_sold DESC
LIMIT 5
```

### 2. Prose / table first, visuals last

When the answer has both a textual component (prose summary, Markdown table, single number) **and** a visual (chart, HTML, SVG), render the textual answer **first** and the visual at the end. Order in the rendered chat must be:

1. The SQL block (rule 1)
2. Prose summary or Markdown table — the answer the user can read without rendering anything
3. *Then* the chart / HTML file (which the chat panel renders inline as an iframe under the `Writing chart.html` tool row)

Practically: write your prose response *before* calling the file-write tool. The agent's natural pattern of "build first, then announce" produces the wrong order.

### 3. Don't build a visual unless asked

If the user's question can be answered with a number, a sentence, or a small table — answer that way. **Do not auto-create a chart / HTML file.** Charts are heavyweight (file write, sandbox HMR, iframe render); they're worth it only when the user explicitly asked for one or when the data really demands one (5+ dimensions, time series, etc.).

When you *think* a visual would help but the user didn't ask, end your prose answer with one short follow-up — *"Want me to chart this?"* — and **stop**. Don't build the chart in the same turn. Wait for confirmation.

Triggers that count as the user asking for a visual:
- *"Chart / plot / graph / visualize / draw …"*
- *"Show me … as a bar / line / pie chart"*
- *"Make a dashboard of …"*
- *"I want a visual of …"*

Anything else is *not* a visual request, even if a chart would be neat.

## The 4-step flow

```
1. Identify the target data source        →  read `Data Connectors` workbook, pick by name or ask user
2. Fetch schema                           →  GET /dataSources/api/v1/dataSources/{id}/schemas
3. Generate SQL grounded in the schema    →  use the FROM convention, ALWAYS scope to known tables
4. Execute and render                     →  POST /dataSources/api/v1/internal/queryResults
```

### Step 1 — Identify the data source

Ask the user which registered data source to query — by `name`. If they've used this conversation to register a source already (via `datagol-data-connector`), you already have the `data_source_id`; reuse it without asking again.

If the user is in a generated app (Flow B in the parent) and the app has a `Data Connectors` workbook, you *can* list candidates from there — but the workbook is a Flow-B-only artefact and **doesn't exist for the default chat-only flow**. Don't presume it. The conversation context (or just asking the user) is the source of truth.

If a list endpoint becomes available on DataGOL (`GET /dataSources/api/v1/dataSources` or similar), prefer it; until then, work from what you know in conversation.

### Step 2 — Fetch schema

Per the parent skill, `getDataSourceSchema(dataSourceId)`. Cache the response in memory for the rest of the conversation. If the user mentions a table the schema doesn't have, refetch — DBs evolve.

### Step 3 — Generate the SQL

**Always qualify FROM clauses as `<dataSourceName>.<schema>.<table>`** — that's the DataGOL convention (the execution layer routes via data-source name, not the actual DB name). The `dataSourceName` comes from the registered name of the data source.

**If the data source name contains spaces, wrap it in double quotes:**

```sql
-- Name with spaces → double-quote the data source name
SELECT * FROM "datawarehouse demo".public.actor LIMIT 100

-- Name without spaces → no quoting needed
SELECT * FROM datawarehouse_plsql.public.actor LIMIT 100
```

Not `dvd_rental.public.actor`, not `public.actor`, not `actor`. Get this wrong and the query fails with a "table not found" or "cross-database reference" error.

**Quoting + identifier rules:** stick to ANSI SQL where possible. Postgres quoting is `"identifier"` (double quotes); MSSQL accepts both `[brackets]` and `"quotes"`. The `datagol-data-query` skill should match the dialect of the source (read `provider` from the connector row). For the v1 Postgres-only flow, double-quote anything case-sensitive:

```sql
SELECT "first_name", "last_name" FROM "datawarehouse demo".public.actor
```

**Defensive limits.** Always include `LIMIT` (Postgres / MySQL), `TOP` (MSSQL), or `FETCH FIRST n ROWS ONLY` (Snowflake / standard SQL) for any query that could return a lot of rows. Default to `LIMIT 1000` for ad-hoc exploration. If the user asks for "all", confirm first — Postgres tables with millions of rows will time out.

**Read-only.** Only generate `SELECT` and `WITH … SELECT`. Never generate `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `ALTER`, `TRUNCATE`, `GRANT`, `REVOKE`. The data-source user *might* have write permission, but this skill exists to read. If the user asks to modify, refuse and point them at the right path (workbook writes via `datagol-workbook-operations`).

### Step 4 — Execute via `/queryResults`

```bash
curl -X POST "${DATAGOL_BASE_URL}/dataSources/api/v1/internal/queryResults" \
  -H "x-auth-token: ${DATAGOL_SERVICE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "sqlQuery": "select * from datawarehouse_plsql.public.actor limit 50",
    "dataProvider": "JDBC"
  }'
```

Body fields:
- `sqlQuery` (required) — the SQL string. The endpoint takes one statement at a time.
- `dataProvider` (required) — always `"JDBC"` for v1 (covers Postgres, MSSQL, MySQL).

**Optional header:** the test backend's web UI sends `spark_job_group_id: <uuid>` for cancel-tracking. The endpoint accepts the call without it; include only if you need to cancel a long-running query mid-flight (out of scope for v1).

Response shape (TBD with the backend; the helper below decodes the common variants):

```ts
// src/api/dataQuery.ts
import { dgFetch } from './datagol';

export interface QueryResult {
  columns: Array<{ name: string; type?: string }>;
  rows: Array<Record<string, unknown>>;
  // Some variants return `data` instead of `rows`, or a flat 2D array.
  rowCount?: number;
}

export async function runSql(sqlQuery: string): Promise<QueryResult> {
  const raw = await dgFetch<any>('/dataSources/api/v1/internal/queryResults', {
    method: 'POST',
    body: JSON.stringify({ sqlQuery, dataProvider: 'JDBC' }),
  });
  // Defensive normalization until the contract is locked.
  const rows = raw.rows ?? raw.data ?? [];
  const columns = raw.columns
    ?? (Array.isArray(raw.fields) ? raw.fields : [])
    ?? (rows.length > 0 ? Object.keys(rows[0]).map((name: string) => ({ name })) : []);
  return { columns, rows, rowCount: raw.rowCount ?? rows.length };
}
```

## Rendering the result

When the agent has the rows, format them for the chat UI. **The SQL block (Response composition rule 1) goes above this content; any chart goes below it.**

- **≤ 20 rows × ≤ 5 columns** → render as a Markdown table inline. Easiest to scan.
- **> 5 columns** or many rows → render the **first 10 rows** as a table, then say *"showing 10 of N rows — say 'show all' to dump the full result, or 'publish' to save this as a workbook"*. Don't dump 1000 rows into chat.
- **Single value (count, sum, etc.)** → state it in prose: *"There are 200 rows in the `actor` table."*
- **Errors** → quote the upstream `body.message` verbatim if present. The most common errors:
  - `relation "X" does not exist` — wrong FROM qualifier (missing `<dataSourceName>` prefix).
  - `column "X" does not exist` — schema is stale; refetch schema and retry once.
  - `permission denied for table X` — the registered DB user lacks SELECT; tell the user to grant it.
  - 401 / 403 — service token missing or wrong; check `datagol-app-auth`.

**Always echo the SQL the agent generated** above the result, in a fenced code block, so the user can review/correct it. Treat the agent as a SQL writing partner — opaque magic erodes trust. The response format must always be:

1. The SQL query in a fenced `sql` code block
2. Then the result table (or prose for single values)

Never show the result without the query.

## Querying workbook warehouse tables (`app_connection_read.<table>`)

When the selection context is a **workbook** (not a registered SQL data source), queries route through a **Postgres-compatible** JDBC layer over the warehouse mirror. This is true even when the workbook is a materialized view whose `materializedViewQuery` is written in Spark SQL — that Spark code only runs at MV refresh time. Ad-hoc queries are Postgres.

**Don't be misled by `materializedViewQuery`.** It tells you how the view was *built* (Spark: `GET_JSON_OBJECT`, `LATERAL VIEW EXPLODE`, `FROM_JSON`, etc.). It is **not** a guide to the dialect available to you at query time.

### Rules specific to this path

1. **Every statement must have a real table in `FROM`.** Bare `SELECT CURRENT_DATE` and `SELECT generate_series(...)` both fail with HTTP 400. Set-returning functions like `generate_series` must be **cross-joined** with an existing workbook table:
   ```sql
   FROM app_connection_read.<table>, generate_series(...) AS g
   ```
2. **Postgres syntax only.** No Spark (`LATERAL VIEW EXPLODE`, `sequence(...)` with bare `INTERVAL 7 DAY`), no Trino (`UNNEST(sequence(...))`, `DATE_ADD('day', n, d)`).
3. **No `::` casts.** Use `CAST(x AS DATE)`.
4. **Date arithmetic:** `d + INTERVAL '7 day'`, `d - INTERVAL '371 days'`. Wrap in `CAST(... AS DATE)` if you need a DATE back instead of a TIMESTAMP.
5. **Series generation:** `generate_series(start_date, stop_date, INTERVAL '7 day')` — once cross-joined as in rule 1.
6. **The schema is `app_connection_read.<tableName>`** where `<tableName>` is the `tableName` field from `datagol_get_workbook_schema` (e.g. `table_c84e4c5c`), **not** the human title.

### Error decoding

- **HTTP 400 `"Unable to parse the query"`** → parser rejection. Check (a) missing FROM clause, (b) `::` casts, (c) function names from the wrong dialect.
- **HTTP 500 `BadSqlGrammarException`** → driver rejection after parsing. Almost always dialect: an `EXPLODE` / `sequence` / `UNNEST` / `DATE_ADD('day', ...)` slipped in. Switch to the Postgres equivalent.

If you hit either error, **change one thing at a time** (replace `::` with `CAST`, or swap the function) rather than rewriting the whole query — the FROM and the rest of the structure are usually fine.

## Hard rules

- **Always qualify FROM with `<dataSourceName>.<schema>.<table>`.** This is the single most common mistake and the one that produces "relation does not exist" errors. **Double-quote the data source name if it contains spaces** (e.g. `"datawarehouse demo".public.actor`).
- **Generate `SELECT` only.** No DML, no DDL, no permissions changes. Refuse if the user asks for them.
- **Always cap row counts.** Default `LIMIT 1000`; confirm with the user before lifting the limit.
- **Always echo the generated SQL** to the user before/with the result — in a fenced `sql` code block, every single time. Never show results without the query above them. The chat's `Running` tool row is collapsed/truncated and is *not* a substitute. (See Response-composition rule 1.)
- **Render prose / table first, charts last.** When an answer mixes a textual result and a visual, the prose / Markdown table comes first; the chart / HTML file is written last so it renders below the answer in chat. Don't call file-write tools mid-prose. (See Response-composition rule 2.)
- **Don't build a visual unless the user asked.** If a chart would help but they didn't request one, end your prose with a single follow-up — *"Want me to chart this?"* — and stop. Wait for confirmation before writing any chart file. (See Response-composition rule 3.)
- **Use `x-auth-token: <SERVICE_TOKEN>`** for the query call. Never the user's bearer.
- **Refetch the schema on `column does not exist`** and retry the query once. Stop and ask if it still fails.
- **Refuse to generate SQL for sources you haven't fetched the schema for.** Hallucinated table/column names are common across LLMs; grounding is non-negotiable.
- **Don't include the SQL `password` or any credential in error messages or logs** — even on failure, the user's DB password should never reappear in the agent's output.
- **Don't run multi-statement scripts.** The endpoint accepts one statement; semicolons inside the string would either be ignored or cause a parse error depending on the driver. Keep it to one statement per call.

## Cross-references

- **`datagol-data-connector`** — parent. Schema discovery + the `Data Connectors` workbook live there. If the schemas endpoint comes back empty or "not ready", the data source has been registered but `syncSchema` hasn't been called — see the parent's *Sync schema* section and trigger `POST /dataSources/api/v1/dataSources/<id>/syncSchema` before re-querying.
- **`datagol-data-publish`** — sibling. Use when the user wants the query result saved as a workbook instead of just shown.
- **`datagol-data-connector/reference/postgres.md`** — provider reference file under the parent. If the schema response is missing `dataSourceName`, fall back to the lowercased + sanitized form of the registered `name`.
- **`datagol-app-auth`** — service token + env config. Required.
- **`datagol-workbook-operations`** — for reading rows from workbooks (not data sources). Different API surface, different use case.
