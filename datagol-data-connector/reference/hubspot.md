
# DataGOL HubSpot Connector

This skill registers a **HubSpot REST data source** in DataGOL. It is based on the DataGOL `dataSources` API, not the OAuth SaaS connector flow.

Default flow: ask for HubSpot connection params, POST once to `/dataSources/api/v1/dataSources`, report the returned data source id. Do **not** create a workspace, workbook, dashboard, app, or pipeline unless the user explicitly asks for one after registration.

## When to use

Use this skill when the user says things like:

- "Connect HubSpot"
- "Register HubSpot as a data source"
- "Add my HubSpot connector"
- "Use this HubSpot access token"
- "Sync HubSpot data into DataGOL"
- Provides a curl/body with `dataSourceProvider: "hubspot"`

Do **not** use this skill for:

- Generic SQL databases such as Postgres/MySQL/Snowflake — use the SQL data connector skills.
- OAuth UI flows through `/idp/api/v1/oauth2/.../authorize` — this HubSpot flow uses a private app access token in the `dataSources` API.
- Publishing/querying data after registration — use data query / publish skills if the user asks for that next.

## API endpoint

```bash
POST ${DATAGOL_BASE_URL}/dataSources/api/v1/dataSources
Authorization: Bearer ${DATAGOL_TOKEN}
Content-Type: application/json
```

The user's bearer token (`DATAGOL_TOKEN`) is used for registration. No generated app service account is needed for the default registration-only flow.

## Required fields to ask the user for

Ask for these in one short chat prompt:

| Field | Submitted as | Required | Default | Notes |
|---|---|---:|---|---|
| Display name | `name` | Yes | — | Human-readable label in DataGOL, e.g. `HubSpot Production` |
| Start date | `properties.start_date` | Yes | today's date or user-provided | Format `YYYY-MM-DD`; used as the initial sync lower bound |
| HubSpot access token | `properties.access_token` | Yes | — | HubSpot private app token / access token; never echo or persist |

Recommended prompt:

```text
Please send:

Display name:
Start date: YYYY-MM-DD
HubSpot access token:

For HubSpot, use a private app token with read-only scopes for the objects you want DataGOL to access. I will not echo or store the token; it is only sent to DataGOL's encrypted credential store.
```

## Registration body

Body shape verified from the DataGOL backend:

```json
{
  "name": "HubSpot Production",
  "properties": {
    "dataSourceProvider": "hubspot",
    "start_date": "2025-05-21",
    "access_token": "<hubspot-access-token>"
  },
  "driverName": "",
  "type": "REST"
}
```

Important constants:

| Field | Value |
|---|---|
| `properties.dataSourceProvider` | `"hubspot"` |
| `driverName` | `""` |
| `type` | `"REST"` |

## Example curl — sanitized

Never include the user's actual bearer token or HubSpot token in examples or logs.

```bash
curl -X POST "${DATAGOL_BASE_URL}/dataSources/api/v1/dataSources" \
  -H "Authorization: Bearer ${DATAGOL_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "HubSpot Production",
    "properties": {
      "dataSourceProvider": "hubspot",
      "start_date": "2025-05-21",
      "access_token": "<hubspot-access-token>"
    },
    "driverName": "",
    "type": "REST"
  }'
```

## Implementation pattern

Use a single POST. Example Python runtime helper:

```python
import json, os, urllib.request

base = os.environ["DATAGOL_BASE_URL"].rstrip("/")
token = os.environ["DATAGOL_TOKEN"]

body = {
    "name": display_name,
    "properties": {
        "dataSourceProvider": "hubspot",
        "start_date": start_date,          # YYYY-MM-DD
        "access_token": hubspot_token,     # do not log
    },
    "driverName": "",
    "type": "REST",
}

req = urllib.request.Request(
    f"{base}/dataSources/api/v1/dataSources",
    data=json.dumps(body).encode("utf-8"),
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    },
    method="POST",
)

with urllib.request.urlopen(req, timeout=90) as resp:
    result = json.loads(resp.read().decode("utf-8"))
```

## Response handling

- **2xx** — report the returned `id` and `name` only:
  - `Registered HubSpot data source <name> with id <id>.`
- **400 / validation error** — check:
  - `properties.dataSourceProvider` is exactly `hubspot`
  - `type` is exactly `REST`
  - `driverName` is exactly an empty string `""`
  - `start_date` is present and formatted as `YYYY-MM-DD`
  - `access_token` is present under `properties`
- **401 / 403 from DataGOL** — DataGOL bearer is missing/expired/unauthorized. Surface the error and stop.
- **HubSpot auth/permission error** — token may be invalid or missing required read scopes. Ask the user to verify the private app token and scopes.
- **5xx / timeout** — quote the upstream error message if present, then ask the user to retry or verify the token/scopes.

## Credential handling rules

- Treat the HubSpot access token as a secret.
- Never echo the token back to the user.
- Never write the token to a workbook, file, generated app, transcript summary, logs, or examples.
- Do not include the token in final responses or error messages.
- The token should live only in:
  1. The user's chat input while they provide it.
  2. The outbound POST request body.
  3. DataGOL's credential store after registration.
- If the user pasted a full curl with an `Authorization: Bearer ...` value, do not copy that bearer token into the skill file or response. Use environment variable `DATAGOL_TOKEN` instead.

## HubSpot token guidance

Recommend a HubSpot **private app token** with read-only scopes for the objects the user wants DataGOL to access, for example contacts, companies, deals, tickets, owners, engagements, and related CRM objects as needed.

Do not request write scopes unless the downstream use case explicitly requires writing back to HubSpot. Registration and read-only sync should use read-only scopes.

## Hard rules

- Default behavior is **registration only** — no workspace, workbook, pipeline, dashboard, UI, or generated app.
- Use `Authorization: Bearer ${DATAGOL_TOKEN}` for the registration call.
- `properties.dataSourceProvider` must be `"hubspot"`.
- `type` must be `"REST"`.
- `driverName` must be `""`.
- `start_date` must be `YYYY-MM-DD`.
- Do not query, sync, publish, or create a workbook after registration unless the user explicitly asks.
- Do not invent list/delete/test endpoints if the user asks for operations not covered by this skill.

## Example success response

```text
Registered HubSpot data source `HubSpot Production` with id `12345`.
```

Optionally add one short next step:

```text
You can now ask me to inspect its schema, query it, or publish a dataset to a workbook.
```

## Cross-references

- `datagol-data-connector` — shared registration mindset: one POST, no scaffolding by default.
- `datagol-data-query` — use only after registration when the user asks to query/show data.
- `datagol-data-publish` — use only when the user asks to save HubSpot data as a workbook.
- `datagol-app-auth` and `datagol-app-development` — only relevant if the user explicitly asks to build an app/UI on top of HubSpot.
