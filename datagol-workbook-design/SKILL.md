---
name: datagol-workbook-design
description: Best practices for designing DataGOL workbooks, schemas, and relationships
---

# Workbook Design Guide

Use this skill when a user asks you to help design their data model, create workbooks, or think through their schema.

> **Prerequisite: run `datagol-interview` first.** This skill is for shaping
> well-understood requirements into workbooks; the user-management answer
> from the interview decides whether each user-scoped workbook needs an
> `Owner` column, which changes the schema you build here. Don't design
> workbooks before the interview's user-management question is answered.
>
> **If multi-user login is in the plan, also read `datagol-user-auth`** — it
> defines the Users workbook (username, password_hash, salt, role) that
> sits alongside the entities you design here.

## Common Domain Schemas

### CRM (Customer Relationship Management)

```
Workspace: CRM

Contact
  - name (SINGLE_LINE_TEXT, required)
  - email (EMAIL)
  - phone (PHONE_NUMBER)
  - title (SINGLE_LINE_TEXT)         # Job title
  - company (RELATION → Company)

Company
  - name (SINGLE_LINE_TEXT, required)
  - website (URL)
  - industry (SINGLE_LINE_TEXT)
  - size (SINGLE_LINE_TEXT)          # e.g. "1-10", "11-50", "50+"
  - notes (MULTI_LINE_TEXT)

Deal
  - name (SINGLE_LINE_TEXT, required)
  - value (CURRENCY)
  - stage (SINGLE_LINE_TEXT)          # e.g. "Lead", "Qualified", "Proposal", "Won", "Lost"
  - closeDate (DATE)
  - company (RELATION → Company)
  - contact (RELATION → Contact)
  - probability (PERCENT)

Activity
  - type (SINGLE_LINE_TEXT)           # e.g. "Call", "Email", "Meeting"
  - notes (MULTI_LINE_TEXT)
  - date (DATE)
  - deal (RELATION → Deal)
  - contact (RELATION → Contact)
```

### Project Management

```
Workspace: Projects

Project
  - name (SINGLE_LINE_TEXT, required)
  - description (MULTI_LINE_TEXT)
  - status (SINGLE_LINE_TEXT)         # "Planning", "Active", "On Hold", "Complete"
  - startDate (DATE)
  - dueDate (DATE)
  - owner (USER)

Task
  - title (SINGLE_LINE_TEXT, required)
  - description (MULTI_LINE_TEXT)
  - status (SINGLE_LINE_TEXT)         # "Todo", "In Progress", "Done"
  - priority (RATING)
  - dueDate (DATE)
  - assignee (USER)
  - project (RELATION → Project)

Milestone
  - name (SINGLE_LINE_TEXT, required)
  - dueDate (DATE, required)
  - achieved (CHECKBOX)
  - project (RELATION → Project)
```

### E-Commerce / Inventory

```
Workspace: Store

Product
  - name (SINGLE_LINE_TEXT, required)
  - description (MULTI_LINE_TEXT)
  - price (CURRENCY, required)
  - sku (SINGLE_LINE_TEXT)
  - stock (NUMBER)
  - category (RELATION → Category)

Category
  - name (SINGLE_LINE_TEXT, required)
  - slug (SINGLE_LINE_TEXT)

Order
  - orderNumber (SINGLE_LINE_TEXT)
  - status (SINGLE_LINE_TEXT)         # "Pending", "Processing", "Shipped", "Delivered"
  - total (CURRENCY)
  - orderDate (DATE)
  - customer (RELATION → Customer)

Customer
  - name (SINGLE_LINE_TEXT, required)
  - email (EMAIL, required)
  - phone (PHONE_NUMBER)
```

## Design Principles

### Naming Conventions
- Workbook names: PascalCase singular noun ("Contact", not "contacts" or "contact_list")
- Column titles: Title Case with spaces ("First Name", "Company Name", "Created Date")
- Keep names short and descriptive

### When to Use RELATION vs USER
- Use **USER** when referencing a DataGOL platform user (owner, assignee, created_by)
- Use **RELATION** when linking to records in another workbook (a contact's company, a deal's contact)

### Many-to-Many Relationships
DataGOL uses a junction workbook pattern for many-to-many:
```
Tag (id, name)
PostTag (post: RELATION→Post, tag: RELATION→Tag)
Post (id, title, ...)
```

### Required Fields
Mark a column as `required: true` only for fields that truly cannot be empty:
- Primary identifier (name, title, email for contacts)
- Foreign keys where the relation is mandatory
- Avoid marking many fields required — it makes the form hard to use

### Column Order
Put the most important/identifying columns first:
1. Primary identifier (name/title)
2. Key status or category field
3. Related records (RELATION columns)
4. Contact info (email, phone)
5. Dates
6. Notes/description last

## When Helping Users Design

1. Ask what domain/use case the workspace is for
2. Suggest a complete schema based on the domain
3. Explain what each workbook represents and how they relate
4. Ask if they need any custom fields beyond the standard ones
5. Use `datagol_create_workbook` to actually create the workbooks once confirmed
6. Always create RELATION columns with the `relatedTableId` pointing to the correct workbook
