---
name: dokki-workspace
description: Browse Personal and Org workspaces, search the knowledge base, explore related entities, upload files, message workspace members, and organize resources (move, rename, tag, share, delete). Use for navigation, discovery, coordination, and workspace hygiene.
argument-hint: [action] [resource-name-or-id]
allowed-tools: mcp__dokki__find mcp__dokki__read mcp__dokki__create mcp__dokki__edit mcp__dokki__share mcp__dokki__message mcp__dokki__preview_resource
---

# Dokki Workspace — Browse, Search & Organize

This skill drives the `find`, `edit`, `share`, `message`, and `create` facades. Every call is
`<facade> {action, <top-level ids>, args:{…}}`. Facades are self-teaching: call one with **no
`action`** to list its actions, and dangerous actions return a `confirm_token` you re-send to
execute.

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

1. `find {action:"workspaces"}` — Get all workspaces authorized for this MCP connection with
   roles. Results include `org_id`, `org_name`, and `org` for Org workspaces; Personal
   workspaces have `org_id: null`.
2. If user has only one workspace → use it directly. Otherwise group by Personal/Org
   and ask or match by name. If names collide, include the Org name in the choice.
3. `find {action:"resources", workspace_id}` → Get the tree. Pass `args.filter {tags, type, dates}`
   to return a flat, filtered list instead of the full tree.

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

- `find {action:"resources"}` only returns non-archived resources
- IDs shown to the user should be short (8-char prefix) — full UUIDs are visually noisy
- If the user has many workspaces (>5), don't dump the full tree of all of them; ask which one
- `find {action:"workspaces"}` is scoped by the current MCP connection. If an expected Org or
  workspace is missing, tell the user to reconnect OAuth and select that scope, or use the
  correct Org-scoped API key. Do not ask them to change the server URL to an Org-specific endpoint.

---

## SEARCH Mode

### Pick the right search action

| Action | Use when | Matches on |
|--------|----------|-----------|
| `find {action:"search", args:{query}}` | **First choice.** Question-style / conceptual lookups ("what's our refund policy?") | Meaning (hybrid semantic + keyword) |
| `find {action:"grep", args:{pattern, regex?}}` | You need an **exact** string, code token, identifier, or regex | Literal text / regex, line-level excerpts |
| `find {action:"related", args:{query}}` | User asks how people, companies, products, places, or concepts connect | Knowledge graph entities and relationships |

Default to `find {action:"search"}`. Reach for `find {action:"grep"}` only when the user wants a
precise literal match (a function name, error string, exact phrase). Use `find {action:"related"}`
for relationship mapping, not content lookup.

### Workflow

1. Transform natural language question into a keyword-rich query:
   - "What's our refund policy?" → `"refund policy terms conditions"`
   - "How does auth work?" → `"authentication flow architecture login"`
   - Combine entity names with structural keywords (summary, timeline, spec)
2. Call `find {action:"search", args:{query, limit?}}` (optionally with a top-level `workspace_id`),
   or `find {action:"grep", args:{pattern, regex?}}` for exact/regex matches
3. If results are weak, reformulate with different angles and run again
4. To scan a single hit before a full `read {action:"doc"}`, use `preview_resource {resource_id}`
5. For "how is X related to Y?" or "what do we know around X?", call
   `find {action:"related", args:{query}}` and then read the most relevant returned resources.

### Query Strategy

| Question style | Limit | Tip |
|----------------|-------|-----|
| Specific lookup ("Where is X?") | 5-10 | Narrow query, entity name + type |
| Broad exploration ("What do we have on Y?") | 20-30 | Multiple keyword combinations |
| Exact string / identifier / regex | — | Use `find {action:"grep"}`, not `find {action:"search"}` |
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
- The user usually wants an *answer*, not a list of links. Read top hits with
  `read {action:"doc", resource_id}` (delegate to `dokki-document`) and synthesize if the intent
  is "find out what X is"
- Search and grep results include absolute in-app `url` fields. Cite hits as markdown links
  like `[Title](https://dokki.one/...)`, not raw IDs.

---

## CHANNEL Mode

Use the `message` facade when the user wants to notify people, ask for approval,
or wait for a teammate's answer before continuing.

### Workflow

1. Resolve the `workspace_id`. If needed, call `find {action:"workspaces"}`.
2. If addressing a person by name, call `message {action:"members", workspace_id}` first to
   identify members.
3. Call `message {action:"send", workspace_id, args:{content, require_response?}}` with clear
   text. Set `require_response: true` when waiting for approval or an answer.
4. Call `message {action:"read", workspace_id, args:{limit?}}` to check recent replies before
   proceeding.

### Pitfalls

- Messages are posted as the authenticated Dokki user.
- For destructive or visible actions, wait for an explicit reply before continuing.
- Keep the message self-contained: what you plan to do, what answer you need, and any
  resource links or short IDs.

---

## ORGANIZE Mode

### Action Matrix

| Action | Facade call | Key params | Risk |
|--------|-------------|-----------|------|
| Create workspace | `create {action:"workspace"}` | args:{name, org_id?} — new top-level workspace; you become admin | safe |
| Create folder | `create {action:"folder"}` | workspace_id, args:{name}, parent_id? | safe |
| Rename | `edit {action:"resource.update"}` | resource_id, args:{name} | safe |
| Change icon | `edit {action:"resource.update"}` | resource_id, args:{icon} — emoji **or** `lucide:<name>` | safe |
| Upload file | `create {action:"file"}` | workspace_id, args:{name, content_base64, mime_type?}, parent_path? | safe |
| Download / read a file | `read {action:"file"}` | resource_id, args:{format?} — signed URL by default, base64 for small files | safe |
| Ask/notify workspace members | `message {action:"members"/"send"/"read"}` | workspace_id, args:{content, require_response?} | visible to workspace |
| Move | `edit {action:"resource.move"}` | resource_id, args:{new_parent_path}, insert_after_id? | safe |
| Delete (archive) | `edit {action:"resource.delete"}` | resource_id | ⚠️ **confirm** — soft, recoverable |
| Add tags | `edit {action:"resource.tag"}` | resource_id, args:{tag_names[]} | safe |
| Remove tags | `edit {action:"resource.untag"}` | resource_id, args:{tag_names[]} | ⚠️ destructive |
| Share by email | `share {action:"user"}` | resource_id, args:{email, role?} | ⚠️ visible to others |
| Set public access | `share {action:"public"}` | resource_id, args:{public_access} | ⚠️ **confirm** — visible publicly |

### Resource icons (any resource type)

`edit {action:"resource.update", resource_id, args:{icon}}` accepts either an **emoji** or a
**Lucide icon** as `lucide:<kebab-name>` (e.g. `lucide:rocket`, `lucide:file-text`). Unlike the
old surface, this works on **any** resource type — documents, tables, artifacts, and folders.

### `resource.move` path format

- Root: `"/"` or `""` or omit `new_parent_path`
- Into a folder: `new_parent_path:"/<folderId>"` (use the folder's short or full ID)
- Ordering: combine with top-level `insert_after_id` to place after a specific sibling; omit to place at the top

### Batch Operations

For "clean up my workspace":

1. `find {action:"resources", workspace_id}` → show current tree
2. Propose a reorganization plan (which folders to create, where each resource goes)
3. **Get user confirmation** before executing
4. Execute: `create {action:"folder"}` first, then `edit {action:"resource.move"}` per item, then `edit {action:"resource.tag"}`
5. Show final tree

### Uploads & downloads

- `create {action:"file"}` expects bytes as base64 or a `data:...;base64,...` URL in
  `args.content_base64`, never a local path.
- For a normal file resource, pass `workspace_id`, `args:{name, content_base64}`, and optional
  `parent_path`.
- For an image that will be inserted into a document, set `args:{inline_image: true}`; the returned
  `node` or `url` can be used with `edit {action:"doc.edit"}` (an `insert` op).
- To read a file resource back, `read {action:"file", resource_id}` returns a signed URL by
  default (base64 for small files); pass `args:{format:"base64"}` to force inline bytes.

### New workspaces

`create {action:"workspace", args:{name, org_id?}}` spins up a **new top-level workspace** (you
become its admin). Pass `org_id` to create it under an Organization you belong to; omit it for a
Personal workspace. This is a global action — a workspace-scoped connector can't call it.

### Pitfalls

- `edit {action:"resource.delete"}` is a soft delete (`is_archived = true`), so resources don't
  actually disappear — it still returns a `confirm_token` you must echo back, so confirm intent first
- `edit {action:"resource.update"}` `icon` now applies to **any** resource type (emoji or `lucide:<name>`)
- Share needs **admin or editor** role on the workspace
- Public share levels: `view | comment | edit | none`; user-level roles: `viewer | commenter | editor | admin`
- `share {action:"public"}` requires confirmation (echo the `confirm_token`) — it exposes the resource publicly
- When sharing, always confirm the target email address before calling

---

## Cross-Skill Follow-Ups

- After browsing, user may want to `read {action:"doc"}` or edit → route to `dokki-document`
- After search, user may want to read full content → route to `dokki-document`
- After asking for approval, use `message {action:"read"}` before executing the approved action
- After organizing, user may want to publish → route to `dokki-publish`
