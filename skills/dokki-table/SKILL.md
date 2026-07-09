---
name: dokki-table
description: Create tables with typed columns, and edit rows, columns, and cells. Use for structured/tabular data.
argument-hint: <action> [table-name-or-id] [details]
allowed-tools: mcp__dokki__create mcp__dokki__read mcp__dokki__edit
---

# Dokki Table — Create & Manage

Tables live behind `create {action:"table"}`, `read {action:"table"}`, and
`edit {action:"table.edit"}`. Each is `<facade> {action, <ids>, args:{…}}`.

## Is a table the right resource?

Use a **table** for: task trackers, inventories, directories, CRM lists, anything with repeated records sharing the same shape.

Use a **document** instead for: narrative content, tutorials, PRDs, meeting notes — even if they contain Markdown tables. Markdown tables in docs don't give you sort/filter/cell-level editing.

If data is for **visualization**, still create the table first (source of truth), then `dokki-artifact` reads from it for the chart.

---

## CREATE Mode

### Action

```
create:
  action: "table"
  workspace_id: string
  parent_id?: string
  args:
    name: string
    description?: string
    columns?: [{ headerName, type }]
    rows?: [{ <headerName>: <value>, … }]   # keyed by headerName
```

### Column Type Selection

| Data | Type | Notes |
|------|------|-------|
| Free text, names | `text` | Default |
| Numeric values | `number` | For sorting, math |
| Yes/no flags | `boolean` | Checkbox UI |
| Dates | `date` | ISO 8601 (`2026-04-15`) |
| Fixed set of options | `select` | Single choice |
| Multiple tags/options | `multiSelect` | Multi-choice |
| Links | `url` | Click-to-open |
| Email addresses | `email` | Click-to-mail |

### Template: Project Tracker

```json
{
  "name": "Sprint Tracker",
  "columns": [
    { "headerName": "Task",     "type": "text" },
    { "headerName": "Assignee", "type": "text" },
    { "headerName": "Status",   "type": "select" },
    { "headerName": "Priority", "type": "select" },
    { "headerName": "Due",      "type": "date" },
    { "headerName": "Done",     "type": "boolean" }
  ]
}
```

### Template: CRM / Contact List

```json
{
  "columns": [
    { "headerName": "Name",    "type": "text" },
    { "headerName": "Email",   "type": "email" },
    { "headerName": "Company", "type": "text" },
    { "headerName": "Stage",   "type": "select" },
    { "headerName": "Tags",    "type": "multiSelect" },
    { "headerName": "Website", "type": "url" }
  ]
}
```

---

## EDIT Mode

### Reading — `read {action:"table"}` can query, not just dump

`read {action:"table", resource_id, args:{where?, where_match?, columns?, sort?, page?, page_size?}}`
returns **20 rows by default** and supports server-side filtering so you don't pull the whole table:

| Arg | Effect |
|-----|--------|
| `where` / `where_match` | Filter rows by column values |
| `columns` | Return only the columns you care about (accepts **header names**) |
| `sort` | Order rows |
| `page` / `page_size` | Paginate large tables |

Read before editing to confirm current rows and headers. **Column edits accept header names**, so
you usually don't need internal column ids at all.

### `edit {action:"table.edit"}` — one facade, op array

All row/column/cell mutations go through a single action with an `ops` list. A single op is just a
one-element list; batch related ops together.

| Change | Op |
|--------|----|
| Add records | `{op:"rows.add", rows:[…]}` |
| Remove records | `{op:"rows.delete", rowIds:[…]}` |
| Change cell data | `{op:"cells.update", updates:[{rowId, columnId, value}]}` |
| New field | `{op:"columns.add", columns:[{headerName, type}]}` |
| Remove field | `{op:"columns.delete", columnIds:[…]}` — ⚠️ **confirm** (deletes all cells in it) |
| Rename / retype | `{op:"columns.update", columns:[…]}` |

`columnId` accepts a **header name** or an internal id — pass the header name for convenience. A
`columns.delete` op returns `requires_confirmation` + a `confirm_token`; re-send the same call with
the token to execute. Singular actions (`table.rows.add`, `table.cells.update`, …) still exist if
you prefer one op per call.

### Operation Decision Tree

```
What's changing?
│
├── ROWS
│   ├── Adding records     → ops:[{op:"rows.add", rows}]
│   ├── Removing records   → ops:[{op:"rows.delete", rowIds}]
│   └── Changing cell data → ops:[{op:"cells.update", updates}]   (by rowId + columnId/header)
│
├── COLUMNS (schema)
│   ├── New field          → ops:[{op:"columns.add", columns}]
│   ├── Remove field       → ops:[{op:"columns.delete", columnIds}]   ⚠️ confirm
│   └── Rename / retype     → ops:[{op:"columns.update", columns}]
│
└── BOTH
    → put column ops before row ops in the same ops array
```

### Batch Updates

Group cell updates into a **single `cells.update` op** when possible. Avoid one op (or one call) per cell.

```json
edit({
  "action": "table.edit",
  "resource_id": "…",
  "args": {
    "ops": [
      { "op": "cells.update", "updates": [
        { "rowId": "0a2b", "columnId": "Status",   "value": "Done" },
        { "rowId": "0a2b", "columnId": "Done",     "value": true },
        { "rowId": "3f4e", "columnId": "Status",   "value": "In Progress" }
      ]}
    ]
  }
})
```

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| Pulling the whole table to edit a few rows | Wastes tokens on big tables | Use `read {action:"table", args:{where, columns, page}}` to fetch just what you need |
| Deleting a column with data | All cell values in that column are lost (not recoverable) | `columns.delete` requires a `confirm_token` — confirm with user; consider exporting first |
| Changing column `type` with incompatible data | Existing cells may become invalid | Warn user; consider new column + migrate |
| Many single-cell calls | Slow and chatty | Batch into one `cells.update` op |
| Assuming you need internal column ids | Extra lookups | `columnId` accepts the **header name** for row/cell/column ops |

## Cross-Skill Follow-Ups

- Visualize the data → `dokki-artifact` (create artifact that renders a chart from this table)
- Share the table → `dokki-workspace` `share {action:"user"}`
- Publish the table to a public site → `dokki-publish` `publish {action:"add", site_id, resource_id}`
