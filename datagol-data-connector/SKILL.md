---
name: datagol-data-connector
description: Register an external **data source** with DataGOL — SQL (Postgres, MSSQL, MySQL, Snowflake, Redshift, BigQuery, …) or REST (HubSpot, etc.). Triggered by "connect a database", "connect my Postgres", "add a data source", "wire up Postgres", "register a database", "connect Postgres", "wire up PostgreSQL", "register Postgres data source", postgres-flavoured hosts (`*.rds.amazonaws.com`, `*.postgres.database.azure.com`, `*.supabase.co`, Neon, Railway-Postgres), "connect HubSpot", "register HubSpot", "add my HubSpot data source", "sync HubSpot", or when the user provides a HubSpot private app access token. The default flow is two API calls (register + syncSchema) — no workspace, no workbook, no scaffolding. Provider-specific defaults (driver name, default port, form-field set, REST endpoint shape) live in reference files alongside this skill: `reference/postgres.md`, `reference/hubspot.md`. Day-1 scope is **registration only** — query / ingest / list / delete are separate workstreams (see `datagol-data-query`, `datagol-data-publish`).
---

# DataGOL Data Connector — Register a Data Source

This skill is the *external-data-source* counterpart of `datagol-connector`. When the user says *"connect my Postgres"*, *"register my MSSQL warehouse"*, *"connect HubSpot"*, etc., the agent does **two HTTP POSTs** — register the source, then sync its schema so the streams become queryable. Nothing else.

It pairs with a **provider reference file** alongside this skill (e.g. `reference/postgres.md`, `reference/hubspot.md`) that supplies provider-specific defaults — driver name / endpoint URL, default port, the field set to ask for, type=SQL vs type=REST. The parent (this skill) owns the API call shape and the credential-handling rules.

## Provider references

When the user names a provider, immediately read the matching reference
**before doing anything else** — it carries the driver name, default
port, form-field set, and any provider-specific quirks (e.g. HubSpot
uses `type=REST` with `driverName=""`).

| User mentions | Read |
| --- | --- |
| Postgres, PostgreSQL, postgres URL, `*.rds.amazonaws.com` running Postgres, `*.postgres.database.azure.com`, `*.supabase.co`, Neon, Railway-Postgres | `reference/postgres.md` |
| HubSpot, "register HubSpot", "add my HubSpot data source", HubSpot private app access token | `reference/hubspot.md` |

If the user names a provider not in this table (MSSQL, MySQL, Snowflake,
Redshift, BigQuery, etc.), confirm with them — we may not have a
reference yet and should fall back to the parent's generic flow plus
writing a new reference file at the end.

## How this differs from `datagol-connector`

| | `datagol-connector` | `datagol-data-connector` |
|---|---|---|
| Use case | OAuth-based SaaS APIs (Gmail, Slack, …) | Direct SQL databases (Postgres, MSSQL, …) |
| Credential model | OAuth → backend token broker | User types DB username + password into chat; agent POSTs once |
| Setup endpoint | `/idp/api/v1/oauth2/{service}/authorize` | `POST /dataSources/api/v1/dataSources` |
| Scaffolding by default | None (creates a connector row in a workbook only when wiring an app) | None |

If the user asks to "connect a database" / "import data from SQL" / "wire up Postgres" — that's this skill. If they ask "connect Slack" / "sync my inbox" — that's `datagol-connector`.

## When to use (and when not to)

- Use when the user wants DataGOL to know about their SQL database: Postgres, MSSQL, MySQL/MariaDB, Snowflake, Redshift, BigQuery, etc.
- Use alongside the matching provider reference file (`reference/postgres.md`, `reference/hubspot.md`, future `mssql.md`, …) for provider-specific defaults.
- **Do not** use this skill for OAuth-based APIs — that's `datagol-connector`.
- **Do not** use this skill for static data the user pastes in — that's `datagol-create-links` / `datagol-workbook-design`.

## The default flow — register a data source

This is what runs for *"connect my Postgres"*, *"register a database"*, *"add a data source"*, or any phrasing without an explicit app/UI/dashboard intent. Four steps:

1. **Ask the user for the connection params** (one chat exchange). The matching provider reference defines exactly which fields and their defaults — for Postgres: `name`, `host`, `port` (default `5432`), `dbName`, `schema` (default `public`), `username`, `password`.
2. **POST to `/dataSources/api/v1/dataSources`** with the user's bearer token (`process.env.DATAGOL_TOKEN`). Body shape below.
3. **POST to `/dataSources/api/v1/dataSources/<id>/syncSchema`** to tell DataGOL to introspect the source and materialize its streams. **Without this step the schemas endpoint below returns nothing** — every freshly-registered source is in a "registered but not synced" state and the agent can't query it. Same bearer token, no body.
4. **Report back** the returned `id` and `name`. One sentence. Done.

That's it. **No workspace creation. No workbook creation. No scaffolding.** The agent's bearer token (`DATAGOL_TOKEN`) is the same identity used by every other DataGOL Pi tool today — no service-account provisioning is needed because there's no generated app for a service account to authenticate as.

### The registration call

```bash
curl -X POST "${DATAGOL_BASE_URL}/dataSources/api/v1/dataSources" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "<user-given name>",
    "username": "<DB username>",
    "password": "<DB password>",
    "properties": {
      "dataSourceProvider": "<postgresql | mssql | mysql | …>",
      "dbName": "<database name>",
      "host": "<hostname only — no protocol prefix>",
      "port": "<port as STRING, not number>",
      "schema": "<schema if applicable>"
    },
    "driverName": "<org.postgresql.Driver | …>",
    "type": "SQL"
  }'
```

Verified Postgres example (matches the curl shared by the user):

```bash
curl -X POST "${DATAGOL_BASE_URL}/dataSources/api/v1/dataSources" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "DataWareHouse_plsql",
    "username": "lobdev",
    "password": "<password>",
    "properties": {
      "dataSourceProvider": "postgresql",
      "dbName": "dvd_rental",
      "host": "datawarehousedemo.postgres.database.azure.com",
      "port": "5432",
      "schema": "public"
    },
    "driverName": "org.postgresql.Driver",
    "type": "SQL"
  }'
```

Response: `{ "id": "<dataSourceId>", "name": ..., "createdAt": ..., ... }` — the `id` is the durable handle; capture and report it.

### Response handling

- **2xx** — surface the `id` and `name` to the user, in one sentence: *"Registered `DataWareHouse_plsql` (id `<dataSourceId>`). Try `'show me top 10 actors'` to query it, or `'save the actor table as a workbook'` to publish."*
- **4xx with `dataSourceProvider` / `driverName` / property error** — the body shape was wrong. Most common: `port` sent as number instead of string, or `host` includes `postgresql://`. Re-check against the provider reference, fix, retry once.
- **4xx with auth error** — the bearer is missing or expired. Don't try to "fix" it client-side; surface the message to the user and stop.
- **5xx / connection refused / timeout** — the DataGOL backend tried to validate the connection and the database refused it. Most common: wrong host/port, the database isn't reachable from DataGOL's network, the user/password is wrong, or the DB requires SSL config not exposed by this skill. Quote the upstream message verbatim and ask the user to verify the params.

## Sync schema (mandatory after registration)

A freshly registered data source has **no streams** until DataGOL introspects it. The agent must call `syncSchema` immediately after a successful POST to `/dataSources` — otherwise the schemas endpoint returns an empty / not-ready response and `datagol-data-query` has nothing to work with.

```bash
curl -X POST "${DATAGOL_BASE_URL}/dataSources/api/v1/dataSources/<dataSourceId>/syncSchema" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}"
```

- Body: none. The endpoint is the trigger — DataGOL reads the connection it already has on file.
- Bearer: same `DATAGOL_TOKEN` used for registration.
- Response: the introspection result (streams / tables / columns) or an ack that the sync completed. **Treat any 2xx as success.**
- Failure handling: a non-2xx here usually means the connection that *registered* fine isn't actually reachable from DataGOL's introspector (firewall, IP allowlist, expired creds). Quote the upstream message and ask the user to re-check the connection params; offer to re-register if needed.

This is provider-agnostic — the same call shape works for Postgres, MSSQL, HubSpot, etc. Each provider's stream/table set is whatever the source exposes to the registered identity.

## Schema discovery

After `syncSchema` succeeds, fetching the resolved schema is a single GET. The response describes every database / schema / table / column the registered DB user can see — agents downstream use it to generate SQL.

```bash
curl "${DATAGOL_BASE_URL}/dataSources/api/v1/dataSources/<dataSourceId>/schemas" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}"
```

Same bearer token, no body. Cache the response in memory for the rest of the conversation (or as `.datagol/schemas/<dataSourceId>.json` for long sessions). Refetch if the user mentions tables/columns the cached schema doesn't show — DBs evolve.

The response includes a **logical data-source name** that downstream SQL must qualify FROM clauses with — typically the lowercased + sanitized version of the registered `name`. For *"DataWareHouse_plsql"* that's `datawarehouse_plsql`. SQL written by `datagol-data-query` and `datagol-data-publish` looks like:

```sql
SELECT * FROM datawarehouse_plsql.public.actor
```

Not `dvd_rental.public.actor`, not `actor`. The DataGOL execution layer (Spark/Trino) routes via the data-source name, not the database name. See `datagol-data-query` for the full SQL-generation contract.

## Flow B — when you're *also* building an app

If — and only if — the user explicitly asks to build a generated app/UI/dashboard that uses this data source, *additional* setup applies. Triggers:

- *"Build an app on top of my Postgres."*
- *"Let users pick a data source from a dropdown in this app."*
- *"Add a Connect Postgres button to the Connections page."*
- *"Generate a CRUD UI for the `actor` table."*

In that case, do the registration flow above, **and additionally** route through the existing app-building skills:

- **`datagol-app-auth`** — provisions a service token + `.env.local` so the generated app authenticates as a service account, not the user.
- **`datagol-ui-generation`** — scaffolds the actual app UI. If the user wants a "data sources" picker, this is where the workbook-listing UI is generated.
- **`datagol-create-dashboard`** — for dashboards over query results.

The `Data Connectors` workbook concept — a per-workspace workbook that stores a row per registered data source so the generated app can list them — only earns its weight when there's a generated app that needs to enumerate sources. **Don't create it during the default flow.** When you do create it (in Flow B), you're just calling `datagol_create_workbook` with whatever columns the app actually needs — it isn't part of this skill's contract.

## Hard rules

- **Don't scaffold anything by default.** No workspace creation. No workbook creation. No `Data Connectors` workbook. No Connections-page UI. The default flow for "connect my Postgres" is *exactly* the register + syncSchema pair documented above — nothing more. Scaffolding only happens when the user explicitly asks for an app or a UI on top of the data source.
- **Use the user's bearer token (`Authorization: Bearer ${DATAGOL_TOKEN}`)** for the registration call. No service-token plumbing for the default flow — there's nothing for a service account to authenticate as.
- **Never persist the DB password.** It lives in three places only: the user's chat input while typing, the request body in flight, DataGOL's encrypted credential store. Don't echo it back, don't write it to a workbook, don't include it in error messages, don't keep it in conversation memory after the call resolves.
- **`port` is always a string** in the `properties` map (`"5432"`, not `5432`). The DataGOL API rejects numbers there.
- **`host` is hostname-only.** No `postgresql://` / `jdbc:postgresql://` prefix, no port suffix, no path. Validate before sending.
- **Recommend a read-only DB user.** When you ask for `username`, mention this — *"DataGOL only needs `SELECT` on the tables you plan to import; consider creating a read-only role."* Most users grant write access by default and don't realize it isn't needed.
- **Don't try to query the database after registration unless the user asks.** Schema discovery is fine when the user wants to query (it's a separate call); a probe-on-register would be eager and confusing.
- **Don't re-emit a status table after every intermediate step.** When the flow involves multiple actions (register source → syncSchema → create warehouse → create pipeline → …) each emitting a partial confirmation, write **one short prose line per step** (`Registered source codex-hubspot (id 4751326).`, `Created warehouse codex_warehouse.`, `Pipeline codex-hubspot → codex_warehouse ready.`) and reserve any `Parameter | Value` summary table for the **final** message of the flow — emitted exactly once. Without this rule the chat panel stacks one table per intermediate emission because each text-block sits in its own message item, and the user sees the same table four times in a row with one extra row each. See the example chat output below for the right shape.

  **Good (one table at the end):**

  > Registered source `codex-hubspot` (id `4751326`). Triggered schema sync.
  > Created warehouse `codex_warehouse`.
  > Pipeline `codex-hubspot → codex_warehouse` ready.
  >
  > | Parameter | Value |
  > |---|---|
  > | Source | `codex-hubspot` |
  > | Source ID | `4751326` |
  > | Warehouse | `codex_warehouse` |
  > | Pipeline | `codex-hubspot → codex_warehouse` |

  **Bad (table re-emitted at every step — what the bug looks like):**

  > | Parameter | Value |
  > |---|---|
  > | Source | `codex-hubspot` |
  >
  > [tool call]
  >
  > | Parameter | Value |
  > |---|---|
  > | Source | `codex-hubspot` |
  > | Source ID | `4751326` |
  >
  > [tool call]
  >
  > | Parameter | Value |
  > |---|---|
  > | Source | `codex-hubspot` |
  > | Source ID | `4751326` |
  > | Warehouse | `codex_warehouse` |
- **Don't fake unbuilt endpoints.** This skill covers registration, `syncSchema`, and schema discovery only. List, delete, test-connection, ingest — all flagged as separate workstreams. If the user asks for them, say they're not wired up, don't invent the endpoint shape.

## Cross-references

- **`reference/postgres.md`** (alongside this skill) — Postgres-specific defaults (driver, port, schema), the field set, the example body. Read on demand.
- **`reference/hubspot.md`** (alongside this skill) — HubSpot REST data source defaults + private app token handling. Read on demand.
- **`datagol-data-query`** — operation child: schema → SQL → execute → render result inline. Use when the user wants to run an ad-hoc SQL query against a registered data source.
- **`datagol-data-publish`** — operation child: SQL → materialized-view workbook via `/publishTable`. Use when the user wants to *save* the query result as a workbook for ongoing use.
- **`datagol-connector`** — the OAuth-based connector parent. Sister skill family for SaaS APIs (not databases).
- **`datagol-ui-generation`**, **`datagol-app-auth`**, **`datagol-create-dashboard`** — only relevant in Flow B (when the user is also building an app on top of the data source).
- **`datagol-workbook-operations`** — once `datagol-data-publish` lands a workbook, this is how the generated app reads rows from it.
