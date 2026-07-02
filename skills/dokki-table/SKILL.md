---
name: dokki-table
description: Create tables with typed columns, and edit rows, columns, and cells. Use for structured/tabular data.
argument-hint: <action> [table-name-or-id] [details]
allowed-tools: mcp__dokki__create_table mcp__dokki__table_read mcp__dokki__table_add_rows mcp__dokki__table_delete_rows mcp__dokki__table_add_columns mcp__dokki__table_delete_columns mcp__dokki__table_update_columns mcp__dokki__table_update_cells
---

# Dokki Table — Create & Manage

## Is a table the right resource?

Use a **table** for: task trackers, inventories, directories, CRM lists, anything with repeated records sharing the same shape.

Use a **document** instead for: narrative content, tutorials, PRDs, meeting notes — even if they contain Markdown tables. Markdown tables in docs don't give you sort/filter/cell-level editing.

If data is for **visualization**, still create the table first (source of truth), then `dokki-artifact` reads from it for the chart.

---

## CREATE Mode

### Tool

```
create_table:
  name: string
  workspace_id: string
  parent_id?: string
  description?: string
  columns?: [{ headerName, type }]
  rows?: [{ <headerName>: <value>, … }]  # keyed by headerName on create only
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

### Golden Rule

**Always call `table_read` before editing** to get current column and row IDs. The compact format returns short 8-char IDs — use these in all subsequent operations.

### Operation Decision Tree

```
What's changing?
│
├── ROWS
│   ├── Adding records     → table_add_rows
│   ├── Removing records   → table_delete_rows
│   └── Changing cell data → table_update_cells (by rowId + columnId)
│
├── COLUMNS (schema)
│   ├── New field          → table_add_columns
│   ├── Remove field       → table_delete_columns (deletes all cells in it)
│   └── Rename / retype    → table_update_columns
│
└── BOTH
    → Execute column changes first, then row changes
```

### `table_update_cells` vs `table_add_rows`

- **`update_cells`** — Change values in rows that already exist. Target specific cells by `rowId` + `columnId`.
- **`add_rows`** — Insert new rows with initial values. `values` is keyed by **columnId** (not headerName) from the compact read.

### Format Choice on `table_read`

| Format | When to use |
|--------|-------------|
| `compact` (default) | Editing — get short IDs |
| `csv` | Human-readable display, export |

### Batch Updates

Group cell updates into a **single `table_update_cells` call** when possible. Avoid one call per cell.

```json
table_update_cells({
  "resource_id": "…",
  "updates": [
    { "rowId": "0a2b", "columnId": "c1", "value": "Done" },
    { "rowId": "0a2b", "columnId": "c2", "value": true },
    { "rowId": "3f4e", "columnId": "c1", "value": "In Progress" }
  ]
})
```

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| Editing without `table_read` first | Stale column/row IDs, wrong cells hit | Always read immediately before edit |
| Using headerName in update_cells | Tool wants `columnId`, not name | Get IDs from compact read |
| Using headerName in add_rows `values` | Same problem after initial create | On update path, use columnId |
| Deleting a column with data | All cell values in that column are lost (not recoverable) | Confirm with user; consider exporting first |
| Changing column `type` with incompatible data | Existing cells may become invalid | Warn user; consider new column + migrate |
| Many single-cell updates | Slow and chatty | Batch into one `update_cells` call |

## Cross-Skill Follow-Ups

- Visualize the data → `dokki-artifact` (create artifact that renders a chart from this table)
- Share the table → `dokki-workspace` `share_resource`
- Publish the table to a public site → `dokki-publish` `publish_resource`
