
# DataGOL Postgres Connector

This skill is a child of **`datagol-data-connector`**. It supplies the Postgres-specific bits — provider config and the field set — and nothing else. The shared mechanics (the registration call, response handling, hard rules around password handling) all live in the parent.

> **Read `datagol-data-connector` first.** This skill assumes the parent's default flow: ask the user for connection params in chat, POST to `/dataSources/api/v1/dataSources` with the user's bearer token, report the `id`. **No workspace, no workbook, no scaffolding** unless the user is explicitly asking to build an app on top of the connection (Flow B in the parent).

## When to use

- The user says *"connect a Postgres database"*, *"wire up PostgreSQL"*, *"register Postgres"*, *"add my Postgres source"*, or names a Postgres-flavoured host: `*.rds.amazonaws.com` (RDS-Postgres), `*.postgres.database.azure.com`, `*.supabase.co`, `*.neon.tech`, `*.render.com` running Postgres, etc. Cloud variants — Aurora-Postgres, Azure Database for PostgreSQL, Google Cloud SQL Postgres, Supabase, Neon, Railway-Postgres — are all standard Postgres from the JDBC driver's perspective; this skill covers them.
- The user pastes a `postgresql://…` connection string. **Parse it** into the field set below — the API takes them as separate properties, not as a URL.
- **Do not** use this skill for Redshift (different SQL dialect; its own future child skill) or for Postgres-compatible databases like CockroachDB without testing — provider name and driver may differ.

## Provider config (these go into the registration body verbatim)

| Field | Value | Why |
|---|---|---|
| `properties.dataSourceProvider` | `"postgresql"` | What DataGOL keys the data source on |
| `driverName` | `"org.postgresql.Driver"` | The JDBC driver class — Postgres standard |
| `type` | `"SQL"` | Always `"SQL"` for relational databases |

The other `properties.*` fields come from chat (host, port, dbName, schema). `username` and `password` are top-level on the body.

## Fields to ask the user for

The agent asks for these in chat — one short prompt, all fields together. Don't render a UI; this is a chat exchange.

| Field | Submitted as | Required | Default | Notes |
|---|---|---|---|---|
| Display name | `name` | Yes | — | A user-given label, e.g. *"Production OLTP"* — appears in the DataGOL UI and downstream FROM clauses (sanitized) |
| Host | `properties.host` | Yes | — | Hostname only — no `postgresql://` prefix, no port suffix, no path |
| Port | `properties.port` | Yes | `"5432"` | **String**, not number; trim whitespace |
| Database | `properties.dbName` | Yes | — | The database name, e.g. `dvd_rental` |
| Schema | `properties.schema` | No | `"public"` | Postgres default schema |
| Username | `username` | Yes | — | DB user — **strongly recommend a read-only role** |
| Password | `password` | Yes | — | Treat as ephemeral — never echo, never log, never store after the request resolves |

When you ask the user for these, include the read-only-role nudge: *"For `username`, we recommend creating a read-only Postgres role — DataGOL only needs `SELECT` on the tables you plan to import."*

If the user pastes a `postgresql://` URL instead of typing the fields one by one, parse it (see "Connection-string parsing" below) and confirm the parsed values back to them before submitting.

## Example registration body (Postgres)

```json
{
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
}
```

Verified shape against the test backend.

## Connection-string parsing

Users frequently paste `postgresql://user:pass@host:port/dbname?schema=public` from their cloud console. Parse it into the field set so they can verify before submission — never pass the URL as-is to the API.

```ts
function parsePostgresUrl(url: string) {
  try {
    const u = new URL(url);
    if (u.protocol !== 'postgresql:' && u.protocol !== 'postgres:') return null;
    return {
      host: u.hostname,
      port: u.port || '5432',
      username: decodeURIComponent(u.username),
      password: decodeURIComponent(u.password),
      dbName: u.pathname.replace(/^\//, ''),
      schema: u.searchParams.get('schema') || 'public',
    };
  } catch { return null; }
}
```

If parsing succeeds, echo the parsed values back to the user (with the password redacted) and ask them to confirm — or to provide a name. If parsing fails (malformed URL, unsupported scheme like `jdbc:postgresql://`), tell them and ask for the fields one by one.

## Hard rules (Postgres-specific)

- **`port` is always a string.** `"5432"`, never `5432`. The DataGOL API rejects numbers in the `properties` map.
- **`host` is hostname-only.** No `postgresql://` prefix, no `/dbname` suffix, no `?param=...` query. Validation regex: `/^[a-zA-Z0-9.-]+$/`.
- **`schema` defaults to `"public"`** — if the user leaves it blank, send `"public"`, not the empty string. The API expects the property present.
- **No SSL options in the body.** The API doesn't expose `sslmode` / `sslrootcert` as top-level properties. DataGOL handles SSL automatically for managed cloud Postgres (RDS / Azure / Cloud SQL); custom certs require a backend-side change — not part of this skill.
- **Recommend a read-only DB user** when asking for `username`.
- **Don't try to `SELECT` from the database after registration.** Schema discovery is the parent's job and only runs when the user asks to query. A probe-on-register would be eager and confusing.
- **Reject pasted URLs that don't start with `postgresql:` or `postgres:`.** Other shapes (`jdbc:postgresql:`, `psql ...`, plain hostnames) need manual entry — silently ignoring them is worse than a clear error.
- **Never persist the password** anywhere — not in conversation memory after the call resolves, not in any workbook, not in any echo of the registration body.

## Cross-references

- **`datagol-data-connector`** — parent. The shared mechanics (the API call, response handling, the *"Don't scaffold anything by default"* hard rule) live there.
- **`datagol-data-query`** — sibling. Use when the user wants to run a SQL query against this Postgres source after it's registered.
- **`datagol-data-publish`** — sibling. Use when the user wants the result of a SQL query saved as a workbook.
- **`datagol-connector`** — the *other* connector parent (OAuth-based SaaS APIs). Different family, sister skill.
