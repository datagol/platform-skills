---
name: datagol-create-pipeline
description: Create DataGOL data pipelines from existing data sources into warehouse destinations. Use this when the user asks to create a pipeline, sync a connector/source, make REST connector streams queryable, materialize source streams/tables into a warehouse, run a connector sync, or schedule a standard source-to-warehouse pipeline. Most standard pipelines share the same shape regardless of connector/provider.
---

# DataGOL Pipeline Creation

Create a **standard DataGOL data pipeline** from an existing data source into a warehouse destination.

The common path is:

```text
existing data source → warehouse destination
```

This applies broadly across REST connectors and other source types where the source already exists and exposes streams/tables through DataGOL schema discovery. Examples include HubSpot, Jira, Gmail, Mailchimp, Attio, Appsflyer, and similar connectors.

## Core principle

Do **not** silently default all pipeline parameters. Ask the user for the required choices, recommend sensible defaults, then show a final plan before modifying anything.

## When to use

Use this skill when the user says things like:

- "Create a pipeline"
- "Create a pipeline for this connector"
- "Sync this REST connector"
- "Make tickets queryable"
- "Sync HubSpot/Jira/Gmail/Mailchimp streams"
- "Create a warehouse from this data source"
- "Materialize these streams into a warehouse"
- "Run the connector sync"
- "Schedule this data source to sync"

Do **not** use this skill for:

- Registering the connector/data source itself — use the appropriate data connector skill first.
- Querying a SQL source ad hoc — use `datagol-data-query`.
- Publishing a SQL query as a workbook — use `datagol-data-publish`.
- Creating workspaces, workbooks, dashboards, agents, or apps unless the user separately asks.

## Required parameter interview

Before creating a pipeline, ask for or confirm:

| Parameter | Required? | Example |
|---|---:|---|
| Pipeline name | Yes | `codex-hubspot → codex_warehouse` |
| Source data source | Yes | `codex-hubspot` / `4751326` |
| Destination warehouse | Yes | existing warehouse ID/name, or new warehouse name |
| Streams/tables | Yes | `tickets`, `contacts`, `companies` |
| Sync method per active stream | Yes | `FULL_REFRESH`, `INCREMENTAL_APPEND_V2` |
| Cursor column | If incremental | `updatedAt` |
| Stream status | Yes | requested streams `ACTIVE`; others usually `PAUSED` |
| Schedule type | Yes | `MANUAL`, `SCHEDULED`, `CRON` |
| Schedule details | If scheduled | every 24h / cron expression |
| Run immediately? | Yes | yes/no; maps to `syncNow` |

Recommended prompt:

```text
I can create the pipeline. Please confirm:

1. Pipeline name:
2. Destination warehouse: create new or use existing? If new, warehouse name:
3. Streams to activate:
4. Sync method for each active stream: FULL_REFRESH or incremental?
5. If incremental, cursor column:
6. Schedule: MANUAL / SCHEDULED / CRON
7. Run immediately after creation? yes/no
```

If the user has already provided enough detail, summarize the inferred config and ask for approval.

## Warehouse choice

Every standard pipeline needs a warehouse destination. Ask whether to:

1. **Use an existing warehouse** — ask for the warehouse data source ID or name.
2. **Create a new warehouse** — ask for the warehouse name.

### Create warehouse endpoint

```http
POST /dataSources/api/v1/dataSources
```

Body:

```json
{
  "name": "codex_warehouse",
  "format": "PARQUET",
  "properties": {
    "tenant_path": "codex_warehouse"
  },
  "type": "WAREHOUSE"
}
```

Notes:

- `name` is the displayed warehouse data source name.
- `properties.tenant_path` typically matches the warehouse name.
- `format` should be `PARQUET` for the standard warehouse path.
- The create response returns the warehouse data source `id`, which must be used in the pipeline destination.

## Default pipeline model

Most standard pipelines use these defaults only after user confirmation:

| Setting | Default |
|---|---|
| Pipeline type | `STANDARD` |
| Schedule type | `MANUAL` |
| Query engine | `SPARK` |
| Destination type | `WAREHOUSE` |
| `syncNow` | ask user; default recommendation `false` unless they ask to run now |
| `isMvPipeline` | `false` |
| Stream sync mode | ask user; default recommendation `FULL_REFRESH` |
| Stream status | requested streams `ACTIVE`, others `PAUSED` when sending full stream list |

## Core API endpoints

### Source metadata

```http
GET /dataSources/api/v1/dataSources/{dataSourceId}
```

### Source schema / streams

```http
GET /dataSources/api/v1/dataSources/{dataSourceId}/schemas
```

### ETL stream sync capabilities

Use this to discover supported sync modes and cursor columns:

```http
GET /etl/api/v1/existingSourceSchema/{dataSourceId}
```

Response shape:

```json
[
  {
    "streamName": "tickets",
    "supportedSyncModes": ["FULL_REFRESH", "INCREMENTAL_APPEND", "INCREMENTAL_APPEND_V2"],
    "sourceDefinedCursorColumn": "updatedAt"
  }
]
```

### Generate stream query

Use this to avoid guessing connector-specific SQL/query syntax:

```http
POST /dataPipeline/api/v2/generateQuery
```

Body:

```json
{
  "dsId": 4751326,
  "stream": "tickets"
}
```

The response is a map from stream name to query:

```json
{
  "tickets": "select ... from codex-hubspot.tickets"
}
```

### Existing pipelines

```http
GET /dataPipeline/api/v2?includeJobs=false
```

Use this read-only endpoint to inspect successful examples if uncertain about request shape.

### Create pipeline

```http
POST /dataPipeline/api/v2
```

### Fetch created pipeline / check status

```http
GET /dataPipeline/api/v2/{pipelineId}
```

This is the most reliable status endpoint. The response can include recent/running jobs embedded under `jobs[]`, including job status and per-stream status.

Example fields to inspect:

```json
{
  "id": 4751565,
  "name": "codex-hubspot → codex_warehouse",
  "jobs": [
    {
      "id": 4778125,
      "jobGroupId": "3951e55b-9d6b-4949-ac6d-3ff2cc629e81",
      "status": "STARTED",
      "streams": [
        {
          "name": "deals",
          "status": "IN_PROGRESS",
          "dataPipelineStreamId": 4752167
        }
      ]
    }
  ]
}
```

Interpretation:

- Pipeline job `status: STARTED` means the pipeline is still running.
- Stream `status: IN_PROGRESS` means that stream is still syncing.
- Completed/failed states may appear on the job and/or stream objects.

### Run pipeline / jobs

If `syncNow` is false and the user later asks to run:

```http
POST /dataPipeline/api/v1/{pipelineId}/run
```

Job-specific endpoints exist, but may be less reliable than fetching the pipeline by ID in some environments:

```http
GET /dataPipeline/api/v2/{pipelineId}/jobs
GET /dataPipeline/api/v2/{pipelineId}/job/{jobId}
POST /dataPipeline/api/v1/job/status
```

If those endpoints fail or require pagination parameters, fall back to:

```http
GET /dataPipeline/api/v2/{pipelineId}
```

## Authentication

Use the current DataGOL API token:

```http
Authorization: Bearer ${DATAGOL_TOKEN}
x-auth-token: ${DATAGOL_TOKEN}
Content-Type: application/json
```

Never print user-provided bearer tokens or connector credentials in examples, logs, skill files, or responses. Use environment variables in examples.

## Final confirmation template

Before creating a warehouse or pipeline, show:

```text
📋 Pipeline plan

- Pipeline name: codex-hubspot → codex_warehouse
- Source: codex-hubspot (`4751326`)
- Source type: REST
- Destination warehouse: create new `codex_warehouse`
- Pipeline type: STANDARD
- Query engine: SPARK
- Schedule: MANUAL
- Run now: yes
- Streams:
  - tickets: ACTIVE, INCREMENTAL_APPEND_V2, cursor `updatedAt`
  - all other discovered streams: PAUSED, FULL_REFRESH

Proceed?
```

## Pipeline creation body shape

For standard connector/source → warehouse pipelines, use destination/source IDs, not only names.

```json
{
  "queryEngine": "SPARK",
  "scheduleType": "MANUAL",
  "syncNow": true,
  "name": "codex-hubspot → codex_warehouse",
  "streams": [
    {
      "name": "tickets",
      "syncMode": "INCREMENTAL_APPEND_V2",
      "query": "<generated query>",
      "outputTableName": "tickets",
      "status": "ACTIVE",
      "columns": [],
      "customQueryDefined": false,
      "cursor": {
        "cursorName": "updatedAt",
        "cursorDataType": null
      }
    }
  ],
  "type": "STANDARD",
  "sources": [
    {
      "id": "4751326"
    }
  ],
  "destination": {
    "id": "4751327"
  }
}
```

Important:

- Include `streams` as a full stream list when the UI/API expects it.
- Requested streams should be `ACTIVE`.
- Unrequested streams should be `PAUSED`.
- Include `columns: []` and `customQueryDefined: false` to match UI-created payloads.
- For incremental sync, include `cursor.cursorName` and `cursor.cursorDataType`.
- Use `/dataPipeline/api/v2/generateQuery` for each stream query when possible.

## Stream configuration rules

### Active full refresh stream

```json
{
  "name": "contacts",
  "syncMode": "FULL_REFRESH",
  "query": "<generated query>",
  "outputTableName": "contacts",
  "status": "ACTIVE",
  "columns": [],
  "customQueryDefined": false
}
```

### Active incremental stream

```json
{
  "name": "tickets",
  "syncMode": "INCREMENTAL_APPEND_V2",
  "query": "<generated query>",
  "outputTableName": "tickets",
  "status": "ACTIVE",
  "columns": [],
  "customQueryDefined": false,
  "cursor": {
    "cursorName": "updatedAt",
    "cursorDataType": null
  }
}
```

### Paused stream

```json
{
  "name": "campaigns",
  "syncMode": "FULL_REFRESH",
  "query": "<generated query>",
  "outputTableName": "campaigns",
  "status": "PAUSED",
  "columns": [],
  "customQueryDefined": false
}
```

## Schedule examples

Manual:

```json
{
  "scheduleType": "MANUAL"
}
```

Scheduled daily:

```json
{
  "scheduleType": "SCHEDULED",
  "schedule": {
    "basicSchedule": {
      "timeUnit": "HOURS",
      "units": "24"
    },
    "cronExpression": "0 0 0 1/1 * ? *",
    "cronExpressionValue": "0 0 0 1/1 * ? *"
  }
}
```

Cron:

```json
{
  "scheduleType": "CRON",
  "schedule": {
    "basicSchedule": {
      "timeUnit": "HOURS"
    },
    "cronExpression": "0 00 09 * * ? *",
    "cronExpressionValue": "0 00 09 * * ? *"
  }
}
```

## Error handling

### 400 validation error

Check stream names, source ID, destination ID, sync modes, cursor fields, and schedule fields.

### 500 internal server error

Common causes:

- destination warehouse was not created first
- using destination name instead of `destination.id`
- missing generated `query` fields on streams
- missing `columns: []` or `customQueryDefined: false` fields expected by UI-created payloads
- invalid incremental cursor
- source/destination IDs passed with wrong shape

Recover by:

1. Fetching source metadata and ETL schema.
2. Creating/selecting a warehouse and capturing its ID.
3. Generating queries via `/dataPipeline/api/v2/generateQuery`.
4. Retrying with `sources: [{"id":"..."}]` and `destination: {"id":"..."}`.

### REST source query error

Do not use SQL query endpoint against REST data sources. REST streams must first be synced/materialized through a pipeline into a warehouse destination.

## Success response

```text
Created pipeline `codex-hubspot → codex_warehouse`.

| Created | Name | ID |
|---|---|---:|
| Warehouse | codex_warehouse | 4751327 |
| Pipeline | codex-hubspot → codex_warehouse | 4752000 |

Active streams: `tickets`
Sync mode: `INCREMENTAL_APPEND_V2` on cursor `updatedAt`
Run now: yes
```

## Hard rules

- Ask whether to create a new warehouse or use an existing one.
- If creating a warehouse, create it first and use its returned ID.
- Always inspect source metadata before creation.
- Always fetch source schema / ETL stream capabilities before creation.
- Always confirm stream list, sync mode, schedule, and run-now choice with the user.
- Do not activate every stream unless explicitly requested.
- Do not expose or log credentials.
- Do not query REST sources directly via SQL.
- Use generated stream queries when possible.
- Prefer `FULL_REFRESH` unless incremental cursor support is verified.
- Treat custom/MV pipelines as an advanced separate path.

## Advanced path: custom / MV pipelines

The common skill path is `STANDARD` pipelines. DataGOL also supports custom/materialized-view-style pipelines with:

```json
{
  "type": "CUSTOM",
  "isMvPipeline": true
}
```

Only use this advanced path when the user explicitly asks for a custom SQL/materialized-view pipeline, not for ordinary connector stream sync.

## Cross-references

- connector-specific skills — register source data sources before pipeline creation.
- `datagol-data-query` — query SQL/warehouse sources after data is materialized.
- `datagol-data-publish` — publish SQL query results to workbooks.
- `datagol-workbook-operations` — read/write workbook rows after publishing to a workbook.
