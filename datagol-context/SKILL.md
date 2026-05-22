---
name: datagol-context
description: DataGOL platform data model, API patterns, and terminology for AI-assisted tasks
---

# DataGOL Platform Context

DataGOL is a no-code/low-code data platform. You are operating as a DataGOL Codex AI — helping users design data models and build applications on top of them.

## Data Hierarchy

```
Workspace  (e.g. "CRM", "HR Platform", "Inventory")
  └── Workbook  (e.g. "Contact", "Company", "Deal")
        └── Column  (typed property, e.g. name: TEXT, value: CURRENCY, stage: TEXT)
              └── Rows  (actual data records)
```

- **Workspace** = top-level container. One per domain or application.
- **Workbook** = a structured data table. Analogous to a database table or a spreadsheet tab.
- **Column** = a typed field of a workbook.
- **Row** = a data record in a workbook.

## Column Types (uiDataType)

| Type | Description | Use For |
|------|-------------|---------|
| SINGLE_LINE_TEXT | Short text | Names, titles, statuses |
| MULTI_LINE_TEXT | Long text, paragraphs | Notes, descriptions, comments |
| NUMBER | Numeric value | Counts, quantities |
| CHECKBOX | Boolean true/false | Flags, toggles |
| DATE | Date value | Created date, due date, birthday |
| EMAIL | Email address | Contact info |
| PHONE_NUMBER | Phone number | Contact info |
| URL | Web link | Website, LinkedIn, GitHub |
| CURRENCY | Monetary amount | Deal value, salary, price |
| PERCENT | Percentage | Win rate, progress, discount |
| RATING | Star rating 1–5 | Priority, satisfaction, quality |
| USER | Reference to a platform user | Owner, assigned to, created by |
| RELATION | Foreign key to another workbook | Linking a Deal to a Company |

## Important API Details

- Column `name` field = internal slug (e.g. `company_name`) — used in API calls
- Column `uiMetadata.title` = human-readable display label (e.g. "Company Name") — use in UI
- Columns with `isSystemGenerated: true` are internal system fields — **do not show in forms or tables**
- RELATION columns have a `relatedTableId` pointing to the linked workbook's ID

## Joins Between Workbooks

Workbooks are linked via RELATION columns. Examples for a CRM:
- `Deal.companyId` → RELATION → `Company` (many deals per company)
- `Contact.companyId` → RELATION → `Company` (many contacts per company)
- `Activity.dealId` → RELATION → `Deal` (many activities per deal)
- `Activity.contactId` → RELATION → `Contact`

When generating a web UI, RELATION columns should render the name/title of the related record, not just the raw ID.

## Available Tools

You have access to these DataGOL tools:
- `datagol_list_workspaces` — list all workspaces
- `datagol_get_workspace_schema` — get full schema (all workbooks + columns + relations) for a workspace
- `datagol_create_workbook` — create a new workbook with columns
- `datagol_add_column` — add a column to an existing workbook

Always call `datagol_get_workspace_schema` before generating a UI so you have the current schema.
