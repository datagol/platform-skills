---
name: datagol-create-links
description: How to create LINK relationships between DataGOL workbooks using the datagol_create_link tool. Use this skill whenever the user asks to link tables, create relationships, or build a relational data model.
---

# Create Links Between Workbooks

DataGOL uses **LINK columns** to create relationships between workbooks. This is different from a regular column — links are bidirectional and must be created via the `datagol_create_link` tool.

## When to Use This Skill

Use `datagol_create_link` when:
- The user wants to connect two workbooks (e.g. "link Contacts to Companies")
- A workbook schema requires foreign-key style relationships
- Building a CRM, project management, e-commerce, or any relational domain

**Never** use `datagol_add_column` with `uiDataType: LINK` or `RELATION` — that will fail. The only correct way to create a link is `datagol_create_link`.

## API Details

**Endpoint:** `POST /noCo/api/v2/workspaces/{workspaceId}/tables/{sourceTableId}/column`

**Request body:**
```json
{
  "uiDataType": "LINK",
  "name": "contacts",
  "uiMetadata": { "title": "Contacts" },
  "colOptions": {
    "relationType": "HAS_MANY",
    "associateTableId": "<target-workbook-id>",
    "associateWorkspaceId": "<workspace-id>"
  },
  "settings": {}
}
```

This is handled automatically by `datagol_create_link`. You only need to provide:
- `sourceWorkbookId` — the workbook that "has" the relation (parent)
- `targetWorkbookId` — the workbook being linked to (child)
- `relationType` — `HAS_MANY`, `ONE_TO_ONE`, or `MANY_TO_MANY`
- `title` — display name for the link column

## Relation Types

| Type | Direction | Example |
|------|-----------|---------|
| `HAS_MANY` | One source → many targets | Company has many Contacts |
| `ONE_TO_ONE` | One source ↔ one target | User ↔ Profile |
| `MANY_TO_MANY` | Many ↔ many | Posts ↔ Tags, Students ↔ Courses |

**HAS_MANY** creates two columns automatically:
- A "Contacts" link column in the **Company** workbook (source)
- A "Company" link column in the **Contact** workbook (target, reciprocal)

## Step-by-Step Workflow

**Always follow this order — IDs are required before creating links:**

### Step 1: Create all workbooks without links
Use `datagol_create_workbook` for each table with only scalar columns (text, number, date, etc.).
Do not try to add link columns during workbook creation.

### Step 2: Get the schema to retrieve workbook IDs
Call `datagol_get_workspace_schema` to get the actual UUIDs for each workbook.
You need the `id` field from each workbook in the schema.

### Step 3: Create each link
Call `datagol_create_link` once per relationship:
```
datagol_create_link(
  sourceWorkbookId = "<id-of-Company>",
  targetWorkbookId = "<id-of-Contact>",
  relationType = "HAS_MANY",
  title = "Contacts"
)
```

### Step 4: Verify
Call `datagol_get_workspace_schema` again — the LINK columns should now appear in both workbooks.

## Complete CRM Example

```
1. datagol_create_workbook("Company", [name, website, industry])
   → returns id: "company-uuid"

2. datagol_create_workbook("Contact", [name, email, phone])
   → returns id: "contact-uuid"

3. datagol_create_workbook("Deal", [name, value, stage, closeDate])
   → returns id: "deal-uuid"

4. datagol_get_workspace_schema()
   → confirms all IDs

5. datagol_create_link(source="company-uuid", target="contact-uuid", relationType="HAS_MANY", title="Contacts")
   → Company gets "Contacts" column; Contact gets "Company" column

6. datagol_create_link(source="company-uuid", target="deal-uuid", relationType="HAS_MANY", title="Deals")
   → Company gets "Deals" column; Deal gets "Company" column

7. datagol_create_link(source="contact-uuid", target="deal-uuid", relationType="MANY_TO_MANY", title="Deals")
   → Contact gets "Deals" column; Deal gets "Contacts" column
```

## Error Handling

- `400 Bad Request` — likely a duplicate column name or invalid relation type
- `404 Not Found` — workbook ID is wrong; call `datagol_get_workspace_schema` to verify IDs
- `401 Unauthorized` — session expired; user needs to sign in again

If a link creation fails, check the workbook IDs are from the **same workspace** as `workspaceId`.
