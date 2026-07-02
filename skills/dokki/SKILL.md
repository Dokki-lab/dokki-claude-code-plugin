---
name: dokki
description: Entry point for working with Dokki. Interprets the user's intent, routes to the right specialized skill, and orchestrates multi-skill workflows (e.g., write + publish, search + edit, data + visualize).
argument-hint: <what-you-want-to-do>
---

# Dokki — Intent Router & Workflow Orchestrator

This is the **entry skill** for Dokki. When a user describes what they want to do in natural language, this skill maps it to one or more specialized skills and runs the full workflow.

## Specialized Skills Map

| Intent | Skill | Core tools |
|--------|-------|-----------|
| Browse, search, organize resources | `dokki-workspace` | list_*, search_workspace, grep_workspace, move, rename, tag, share, delete |
| Write/edit rich text documents | `dokki-document` | create_document, doc_read/insert/replace/delete |
| Work with structured tabular data | `dokki-table` | create_table, table_read/add/update/delete_* |
| Build interactive UI / charts / diagrams | `dokki-artifact` | create_artifact, artifact_read/update/patch |
| Publish as a public site | `dokki-publish` | *_publish_site, publish_resource, *_custom_domain |

## Intent → Skill Routing

Match the user's request to these patterns:

### Single-skill intents

| If user says… | Route to |
|---------------|----------|
| "What's in my workspace?" / "Show me my docs" | `dokki-workspace` (browse) |
| "Find docs about X" / "What do we have on Y?" | `dokki-workspace` (search) |
| "Move / rename / tag / share / delete X" | `dokki-workspace` (organize) |
| "Write a doc about X" / "Create a note" | `dokki-document` (create) |
| "Add / edit / rewrite section in doc" | `dokki-document` (edit) |
| "Create a table of X" / "Track Y in a table" | `dokki-table` (create) |
| "Add rows / update cells" | `dokki-table` (edit) |
| "Make a chart / dashboard / widget" | `dokki-artifact` (create) |
| "Make this doc / workspace public" | `dokki-publish` |

### Multi-skill workflows

These are the high-value flows — chain skills in this order:

#### 1. Write & Publish
**"Write a blog post / doc and publish it"**
1. `dokki-document` → create document with content
2. `dokki-publish` →
   - `get_publish_site`, create site if missing (`create_publish_site`)
   - `publish_resource` with the new document
3. Return the public URL

#### 2. Research & Summarize
**"Find info on X and write a summary doc"**
1. `dokki-workspace` → `search_workspace` with broad query (limit 20-30)
2. Read the top matches with `doc_read`
3. `dokki-document` → `create_document` with a synthesized summary that cites source resource IDs

#### 3. Data-Driven Visualization
**"Make a chart/dashboard of my table data"**
1. `dokki-table` → `table_read` (csv format for readability)
2. `dokki-artifact` → `create_artifact` with source code that embeds the data inline and renders it with Recharts/Mermaid
3. Optionally place artifact alongside the table in the same folder (use `dokki-workspace` `move_resource`)

#### 4. Publish Site Bootstrap
**"Launch a public docs site with these documents"**
1. `dokki-workspace` → `list_resources` to pick which to publish
2. `dokki-publish` → `create_publish_site` with slug
3. `dokki-publish` → `publish_resource` for each selected resource
4. `dokki-publish` → `update_publish_site` with `settings.home` + `nav_tags`
5. (Optional) `dokki-publish` → `set_custom_domain` + `check_domain_status`

#### 5. Reorganize Knowledge Base
**"Clean up my workspace / restructure my docs"**
1. `dokki-workspace` → `list_resources` to see current tree
2. `dokki-workspace` → `create_folder` for new structure
3. `dokki-workspace` → `move_resource` for each resource
4. `dokki-workspace` → `tag_resource` to categorize
5. Confirm summary before executing

#### 6. Update Published Content
**"The published doc is stale, refresh it"**
1. `dokki-document` → edit the source resource
2. `dokki-publish` → re-call `publish_resource` (refreshes frozen snapshot)
3. Confirm the public URL now shows the new content

## Bootstrap: Context Resolution

Before any operation, resolve context in this order:

1. **Workspace**: If not specified and user has only one, use it. Otherwise `list_workspaces` and ask.
2. **Resource**: If user gives a name, `list_resources` to match by name. If multiple match, ask. If the name is not obvious, `search_workspace` as fallback.
3. **Parent folder** (for creates): If not specified, create at workspace root.

Always report the resolved IDs back to the user so they can confirm.

## Choosing Between Skills

Some intents are genuinely ambiguous. Decision hints:

- **"Track X"** → Is X tabular (tasks, inventory)? → `dokki-table`. Is X narrative (journal, notes)? → `dokki-document`.
- **"Visualize X"** → Is data already in Dokki? → `dokki-artifact` reading from a table. Is it new data? → `dokki-table` first, then `dokki-artifact`.
- **"Share with someone"** → Single user by email → `dokki-workspace` `share_resource`. Public URL → `dokki-publish`.
- **"Show me what's there"** → Hierarchy view → `dokki-workspace` browse. Content search → `dokki-workspace` search.

## Output Conventions

When completing any workflow, always end with:

1. **What was done** — a one-sentence summary
2. **Resource IDs** — short IDs for anything created/modified
3. **URLs** — document URL, table URL, artifact URL, or public site URL
4. **Next steps** — suggested follow-up actions (publish? tag? share?)

## When to Say No

Don't invoke skills when:
- User is just asking "what is Dokki?" or asking for help — answer directly
- The operation would be clearly destructive (bulk delete, remove custom domain) without explicit confirmation
- The required context (workspace, resource) can't be resolved — ask first
