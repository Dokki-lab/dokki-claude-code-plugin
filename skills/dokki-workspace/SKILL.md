---
name: dokki-workspace
description: Browse workspaces, search the knowledge base, and organize resources (move, rename, tag, share, delete). Use for navigation, discovery, and workspace hygiene.
argument-hint: [action] [resource-name-or-id]
allowed-tools: mcp__dokki__list_workspaces mcp__dokki__list_resources mcp__dokki__create_workspace mcp__dokki__create_folder mcp__dokki__move_resource mcp__dokki__update_resource mcp__dokki__delete_resource mcp__dokki__tag_resource mcp__dokki__untag_resource mcp__dokki__share_resource mcp__dokki__search_workspace mcp__dokki__grep_workspace mcp__dokki__preview_resource
---

# Dokki Workspace — Browse, Search & Organize

## Mode Decision Tree

Determine which mode the user is in:

```
User request
│
├── "Show me / what's in / list…"        → BROWSE
├── "Find / search / where is / look up" → SEARCH
└── "Move / rename / tag / share / delete / organize" → ORGANIZE
```

If ambiguous, ask before acting.

---

## BROWSE Mode

### Workflow

1. `list_workspaces` — Get all accessible workspaces with roles
2. If user has only one workspace → use it directly. Otherwise ask or match by name.
3. `list_resources` with `workspace_id` → Get the tree

### Output Template

```
🏢 <workspace-name> (role: admin)

📁 Projects/
├── 📄 PRD v2 (a1b2c3d4)
├── 📊 Roadmap (e5f6g7h8)
└── 📁 Archive/
    └── 📄 Old spec (i9j0k1l2)
📄 README (m3n4o5p6)
🎨 Dashboard (q7r8s9t0)

Total: 3 documents, 1 table, 1 artifact, 2 folders
```

Icons: 📄 document, 📊 table, 🎨 artifact, 📁 folder

### Pitfalls

- `list_resources` only returns non-archived resources
- IDs shown to the user should be short (8-char prefix) — full UUIDs are visually noisy
- If the user has many workspaces (>5), don't dump the full tree of all of them; ask which one

---

## SEARCH Mode

### Pick the right search tool

| Tool | Use when | Matches on |
|------|----------|-----------|
| `search_workspace` | **First choice.** Question-style / conceptual lookups ("what's our refund policy?") | Meaning (hybrid semantic + keyword) |
| `grep_workspace` | You need an **exact** string, code token, identifier, or regex | Literal text / regex, line-level excerpts |

Default to `search_workspace`. Reach for `grep_workspace` only when the user wants a precise literal match (a function name, error string, exact phrase).

### Workflow

1. Transform natural language question into keyword-rich query:
   - "What's our refund policy?" → `"refund policy terms conditions"`
   - "How does auth work?" → `"authentication flow architecture login"`
   - Combine entity names with structural keywords (summary, timeline, spec)
2. Call `search_workspace` with `query`, optional `workspace_id`, `limit`
   (or `grep_workspace` with `pattern` for exact/regex matches)
3. If results are weak, reformulate with different angles and run again
4. To scan a single hit before a full `doc_read`, use `preview_resource` with its `resource_id`

### Query Strategy

| Question style | Limit | Tip |
|----------------|-------|-----|
| Specific lookup ("Where is X?") | 5-10 | Narrow query, entity name + type |
| Broad exploration ("What do we have on Y?") | 20-30 | Multiple keyword combinations |
| Exact string / identifier / regex | — | Use `grep_workspace`, not `search_workspace` |
| Multi-hop ("Find X then find things mentioning it") | Iterate | Run multiple searches, synthesize |

### Output Template

```
🔍 "<query>" — N results

1. 📄 **PRD v2** (a1b2c3d4, score: 0.91)
   > …relevant excerpt highlighting the match…

2. 📄 **Design Spec** (e5f6g7h8, score: 0.84)
   > …
```

### Pitfalls

- Don't dump raw search results — summarize and pick the most relevant
- The user usually wants an *answer*, not a list of links. Read top hits with `doc_read` (delegate to `dokki-document`) and synthesize if the intent is "find out what X is"

---

## ORGANIZE Mode

### Action Matrix

| Action | Tool | Key params | Risk |
|--------|------|-----------|------|
| Create folder | `create_folder` | name, workspace_id, parent_id? | safe |
| Rename | `update_resource` | resource_id, name | safe |
| Change icon | `update_resource` | resource_id, icon (tables/folders only) | safe |
| Move | `move_resource` | resource_id, new_parent_path, insert_after_id? | safe |
| Delete (archive) | `delete_resource` | resource_id | ⚠️ soft — recoverable |
| Add tags | `tag_resource` | resource_id, tag_names[] | safe |
| Remove tags | `untag_resource` | resource_id, tag_names[] | ⚠️ destructive |
| Share by email | `share_resource` | resource_id, email, role | ⚠️ visible to others |
| Set public access | `share_resource` | resource_id, public_access | ⚠️ visible publicly |

### `move_resource` path format

- Root: `"/"` or `""` or omit
- Into a folder: `"/<folderId>"` (use the folder's short or full ID)
- Ordering: combine with `insert_after_id` to place after a specific sibling; omit to place at the top

### Batch Operations

For "clean up my workspace":

1. `list_resources` → show current tree
2. Propose a reorganization plan (which folders to create, where each resource goes)
3. **Get user confirmation** before executing
4. Execute: `create_folder` first, then `move_resource` per item, then `tag_resource`
5. Show final tree

### Pitfalls

- `delete_resource` is a soft delete (`is_archived = true`), so resources don't actually disappear — don't hesitate to delete, but still confirm
- `update_resource` → `icon` only applies to tables and folders (documents and artifacts ignore it)
- Share needs **admin or editor** role on the workspace
- Public share levels: `view | comment | edit | none`; user-level roles: `viewer | commenter | editor | admin`
- When sharing, always confirm the target email address before calling

---

## Cross-Skill Follow-Ups

- After browsing, user may want to `doc_read` or edit → route to `dokki-document`
- After search, user may want to read full content → route to `dokki-document`
- After organizing, user may want to publish → route to `dokki-publish`
