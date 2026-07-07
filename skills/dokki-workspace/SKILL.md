---
name: dokki-workspace
description: Browse Personal and Org workspaces, search the knowledge base, explore related entities, upload files, message workspace members, and organize resources (move, rename, tag, share, delete). Use for navigation, discovery, coordination, and workspace hygiene.
argument-hint: [action] [resource-name-or-id]
allowed-tools: mcp__dokki__list_workspaces mcp__dokki__list_resources mcp__dokki__create_workspace mcp__dokki__create_folder mcp__dokki__upload_file mcp__dokki__move_resource mcp__dokki__update_resource mcp__dokki__delete_resource mcp__dokki__tag_resource mcp__dokki__untag_resource mcp__dokki__share_resource mcp__dokki__search_workspace mcp__dokki__grep_workspace mcp__dokki__related_entities mcp__dokki__preview_resource mcp__dokki__list_channel_members mcp__dokki__send_channel_message mcp__dokki__read_channel
---

# Dokki Workspace — Browse, Search & Organize

## Mode Decision Tree

Determine which mode the user is in:

```
User request
│
├── "Show me / what's in / list…"                 → BROWSE
├── "Find / search / where is / look up / related" → SEARCH
├── "Ask / notify / confirm with a teammate"      → CHANNEL
└── "Move / rename / tag / share / delete / organize" → ORGANIZE
```

If ambiguous, ask before acting.

---

## BROWSE Mode

### Workflow

1. `list_workspaces` — Get all workspaces authorized for this MCP connection with roles.
   Results include `org_id`, `org_name`, and `org` for Org workspaces; Personal
   workspaces have `org_id: null`.
2. If user has only one workspace → use it directly. Otherwise group by Personal/Org
   and ask or match by name. If names collide, include the Org name in the choice.
3. `list_resources` with `workspace_id` → Get the tree

### Output Template

```
Personal
└── <workspace-name> (role: admin, id: a1b2c3d4)

Org: <org-name> (<org-id-prefix>)
└── <workspace-name> (role: editor, id: e5f6g7h8)

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
- `list_workspaces` is scoped by the current MCP connection. If an expected Org or workspace
  is missing, tell the user to reconnect OAuth and select that scope, or use the correct
  Org-scoped API key. Do not ask them to change the server URL to an Org-specific endpoint.

---

## SEARCH Mode

### Pick the right search tool

| Tool | Use when | Matches on |
|------|----------|-----------|
| `search_workspace` | **First choice.** Question-style / conceptual lookups ("what's our refund policy?") | Meaning (hybrid semantic + keyword) |
| `grep_workspace` | You need an **exact** string, code token, identifier, or regex | Literal text / regex, line-level excerpts |
| `related_entities` | User asks how people, companies, products, places, or concepts connect | Knowledge graph entities and relationships |

Default to `search_workspace`. Reach for `grep_workspace` only when the user wants a precise literal match (a function name, error string, exact phrase). Use `related_entities` for relationship mapping, not content lookup.

### Workflow

1. Transform natural language question into keyword-rich query:
   - "What's our refund policy?" → `"refund policy terms conditions"`
   - "How does auth work?" → `"authentication flow architecture login"`
   - Combine entity names with structural keywords (summary, timeline, spec)
2. Call `search_workspace` with `query`, optional `workspace_id`, `limit`
   (or `grep_workspace` with `pattern` for exact/regex matches)
3. If results are weak, reformulate with different angles and run again
4. To scan a single hit before a full `doc_read`, use `preview_resource` with its `resource_id`
5. For "how is X related to Y?" or "what do we know around X?", call `related_entities`
   and then read the most relevant returned resources.

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
- Search and grep results include absolute in-app `url` fields. Cite hits as markdown links
  like `[Title](https://dokki.one/...)`, not raw IDs.

---

## CHANNEL Mode

Use workspace channel tools when the user wants to notify people, ask for approval,
or wait for a teammate's answer before continuing.

### Workflow

1. Resolve the `workspace_id`. If needed, call `list_workspaces`.
2. If addressing a person by name, call `list_channel_members` first to identify members.
3. Call `send_channel_message` with clear text. Set `require_response: true` when waiting
   for approval or an answer.
4. Call `read_channel` to check recent replies before proceeding.

### Pitfalls

- Messages are posted as the authenticated Dokki user.
- For destructive or visible actions, wait for an explicit reply before continuing.
- Keep the message self-contained: what you plan to do, what answer you need, and any
  resource links or short IDs.

---

## ORGANIZE Mode

### Action Matrix

| Action | Tool | Key params | Risk |
|--------|------|-----------|------|
| Create folder | `create_folder` | name, workspace_id, parent_id? | safe |
| Rename | `update_resource` | resource_id, name | safe |
| Change icon | `update_resource` | resource_id, icon (tables/folders only) | safe |
| Upload file | `upload_file` | workspace_id, name, content_base64, mime_type?, parent_path? | safe |
| Ask/notify workspace members | `list_channel_members`, `send_channel_message`, `read_channel` | workspace_id, content, require_response? | visible to workspace |
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

### Uploads

- `upload_file` expects bytes as base64 or a `data:...;base64,...` URL, never a local path.
- For a normal file resource, pass `workspace_id`, `name`, `content_base64`, and optional
  `parent_path`.
- For an image that will be inserted into a document, set `inline_image: true`; the returned
  `node` or `url` can be used with `doc_insert`.

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
- After asking for approval, use `read_channel` before executing the approved action
- After organizing, user may want to publish → route to `dokki-publish`
