---
name: dokki
description: Entry point for working with Dokki. Interprets the user's intent, routes to the right specialized skill, and orchestrates multi-skill workflows (e.g., write + publish, search + edit, data + visualize).
argument-hint: <what-you-want-to-do>
---

# Dokki — Intent Router & Workflow Orchestrator

This is the **entry skill** for Dokki. When a user describes what they want to do in natural language, this skill maps it to one or more specialized skills and runs the full workflow.

## The facade surface

Dokki's MCP exposes **8 facade tools + `preview_resource`**. Every call is `<facade>` with
`{ action: "<action>", <top-level ids>, args: { <payload> } }`. Top-level ids are
`workspace_id`, `resource_id`, `parent_id`, `insert_after_id`, `site_id`; everything else
goes in `args`.

| Facade | What it covers |
|--------|----------------|
| `find` | list workspaces/resources, semantic search, grep, related entities |
| `read` | read a doc / table / artifact / file, or `preview_resource` for a rendered preview |
| `create` | create doc, table, artifact, folder, workspace, upload file |
| `edit` | edit doc/table/artifact content + resource ops (rename, icon, move, tag, delete) |
| `share` | share by email or set public access |
| `message` | workspace channel: members, send, read |
| `publish` | published sites, publish/unpublish resources, custom domains |
| `connect` | connect + call 1000+ external integrations (GitHub, Slack, Gmail, Notion, …) |

### Facade self-teaching conventions

The facades teach themselves — you rarely need to enumerate every arg:

- Call a facade with **no `action`** → it lists its available actions. A **partial action**
  (e.g. `edit {action:"table.columns"}`) returns that subtree. An unknown/typo action returns
  `invalid_action` with nearest matches.
- Missing or invalid args → a `missing_args` hint with the exact keys and an example. Don't
  over-specify; name the *action* and let the facade guide you.
- **Dangerous actions** (`edit {action:"resource.delete"}`, an `edit {action:"table.edit"}`
  with a `columns.delete` op, `share {action:"public"}`, `publish {action:"add"}`) return
  `requires_confirmation` + a `confirm_token`. Re-call the SAME action + args WITH the
  `confirm_token` to execute.

## Specialized Skills Map

| Intent | Skill | Core facade actions |
|--------|-------|---------------------|
| Browse, search, organize resources | `dokki-workspace` | `find {action:"workspaces"/"resources"/"search"/"grep"/"related"}`, `create {action:"file"}`, `message {…}`, `edit {action:"resource.move"/"resource.update"/"resource.tag"/"resource.untag"/"resource.delete"}`, `share {…}` |
| Write/edit rich text documents | `dokki-document` | `create {action:"doc"}`, `read {action:"doc"}`, `edit {action:"doc.edit"/"doc.rewrite"}` |
| Work with structured tabular data | `dokki-table` | `create {action:"table"}`, `read {action:"table"}`, `edit {action:"table.edit"}` |
| Build HTML/JSX UI / charts / diagrams | `dokki-artifact` | `create {action:"artifact"}`, `read {action:"artifact"}`, `edit {action:"artifact.update"/"artifact.patch"}` |
| Publish as a public site | `dokki-publish` | `publish {action:"site"/"site.create"/"site.update"/"add"/"remove"/"domain.*"}` |
| Connect & use external integrations | `dokki-workspace` (CONNECT mode) | `connect {action:"apps"/"list"/"authorize"/"disconnect"/"tools"/"call"}` |

## Intent → Skill Routing

Match the user's request to these patterns:

### Single-skill intents

| If user says… | Route to |
|---------------|----------|
| "What's in my workspace?" / "Show me my docs" | `dokki-workspace` (browse) |
| "Find docs about X" / "What do we have on Y?" | `dokki-workspace` (search) |
| "How is X related to Y?" / "Map relationships around X" | `dokki-workspace` (related entities) |
| "Move / rename / tag / share / delete X" | `dokki-workspace` (organize) |
| "Ask Alice to confirm" / "Notify the workspace" | `dokki-workspace` (channel) |
| "Upload this file/image" | `dokki-workspace` (upload), then route to `dokki-document` if inserting into a doc |
| "Write a doc about X" / "Create a note" | `dokki-document` (create) |
| "Add / edit / rewrite section in doc" | `dokki-document` (edit) |
| "Create a table of X" / "Track Y in a table" | `dokki-table` (create) |
| "Add rows / update cells" | `dokki-table` (edit) |
| "Make a chart / dashboard / widget" | `dokki-artifact` (create) |
| "Make this doc / workspace public" | `dokki-publish` |
| "Connect my GitHub / Slack / Gmail" / "Pull data from <app>" | `dokki-workspace` (CONNECT) |

### External integrations — the `connect` facade

The canonical home for this is **`dokki-workspace` CONNECT mode** — route there for the full
flow. This table is the router's quick reference for orchestrating it inside multi-skill
workflows (e.g. #8 below).

Dokki's MCP is also a **gateway to 1000+ external integrations** (via Composio: GitHub,
Slack, Gmail, Notion, Google Sheets/Drive/Calendar, Linear, and more). Connections are
**user-scoped** — the user authorizes their own accounts *through* Dokki, once, and every
skill can then read/write those systems.

| Step | Call |
|------|------|
| Discover integrations | `connect {action:"apps", args:{query?, limit?}}` |
| See what's already connected | `connect {action:"list"}` |
| Authorize a new one | `connect {action:"authorize", args:{toolkit:"github"}}` → open the returned `authorize_url`, then poll `connect {action:"list"}` until active |
| Disconnect | `connect {action:"disconnect", args:{connection_id}}` |
| List callable tools | `connect {action:"tools", args:{toolkit?}}` |
| Run a tool | `connect {action:"call", args:{tool, args}}` — the relay executes it |

When a user asks to pull data from or push data to an outside system, check `connect list`
first; if the toolkit isn't connected, walk them through `authorize` before calling tools.

### Multi-skill workflows

These are the high-value flows — chain skills in this order:

#### 1. Write & Publish
**"Write a blog post / doc and publish it"**
1. `dokki-document` → `create {action:"doc", workspace_id, args:{name, content}}`
2. `dokki-publish` →
   - `publish {action:"site", workspace_id}`; create the site if missing (`publish {action:"site.create", workspace_id, args:{slug}}`)
   - `publish {action:"add", site_id, resource_id}` with the new document (**confirm** — exposes it publicly)
3. Return the public URL

#### 2. Research & Summarize
**"Find info on X and write a summary doc"**
1. `dokki-workspace` → `find {action:"search", args:{query, limit:20-30}}` with a broad query
2. Read the top matches with `read {action:"doc", resource_id}` (default `view` mode = markdown)
3. `dokki-document` → `create {action:"doc", workspace_id, args:{name, content}}` with a synthesized summary that cites source resource IDs

#### 3. Data-Driven Visualization
**"Make a chart/dashboard of my table data"**
1. `dokki-table` → `read {action:"table", resource_id}` (use `columns`/`where`/`sort` to pull just what you need)
2. `dokki-artifact` → `create {action:"artifact", workspace_id, args:{name, source}}` with source that embeds the data inline and renders it with Recharts/Mermaid
3. Optionally place the artifact alongside the table in the same folder (`dokki-workspace` → `edit {action:"resource.move", …}`)

#### 4. Publish Site Bootstrap
**"Launch a public docs site with these documents"**
1. `dokki-workspace` → `find {action:"resources", workspace_id}` to pick which to publish
2. `dokki-publish` → `publish {action:"site.create", workspace_id, args:{slug}}`
3. `dokki-publish` → `publish {action:"add", site_id, resource_id}` for each selected resource (**confirm** each)
4. `dokki-publish` → `publish {action:"site.update", site_id, args:{settings:{home, nav_tags}}}`
5. (Optional) `dokki-publish` → `publish {action:"domain.set", …}` + `publish {action:"domain.status", …}`

#### 5. Reorganize Knowledge Base
**"Clean up my workspace / restructure my docs"**
1. `dokki-workspace` → `find {action:"resources", workspace_id}` to see the current tree
2. `dokki-workspace` → `create {action:"folder", workspace_id, args:{name}}` for new structure
3. `dokki-workspace` → `edit {action:"resource.move", resource_id, args:{new_parent_path}}` for each resource
4. `dokki-workspace` → `edit {action:"resource.tag", resource_id, args:{tag_names}}` to categorize
5. Confirm summary before executing

#### 6. Human Confirmation
**"Ask the team before doing this"**
1. `dokki-workspace` → `message {action:"members", workspace_id}` if the user names a person
2. `dokki-workspace` → `message {action:"send", workspace_id, args:{content, require_response:true}}`
3. `dokki-workspace` → `message {action:"read", workspace_id}` before proceeding

#### 7. Update Published Content
**"The published doc is stale, refresh it"**
1. `dokki-document` → edit the source resource (`edit {action:"doc.edit"/"doc.rewrite", …}`)
2. `dokki-publish` → re-call `publish {action:"add", site_id, resource_id}` (refreshes the frozen snapshot)
3. Confirm the public URL now shows the new content

#### 8. External Data → Dokki
**"Pull my open GitHub issues (or latest Slack messages) into a Dokki doc/table"**
1. `connect {action:"list"}` — confirm the toolkit is connected; if not, `connect {action:"authorize", args:{toolkit:"github"}}` and walk the user through the OAuth URL
2. `connect {action:"tools", args:{toolkit:"github"}}` to find the right tool, then `connect {action:"call", args:{tool, args}}` to fetch the data
3. `dokki-table` → `create {action:"table", workspace_id, args:{name, columns, rows}}` (structured) **or** `dokki-document` → `create {action:"doc", …}` (narrative) with the fetched data
4. (Optional) `dokki-artifact` to visualize, or `dokki-publish` to share externally

## Bootstrap: Context Resolution

Before any operation, resolve context in this order:

1. **Workspace**: If not specified and the user has only one, use it. Otherwise
   `find {action:"workspaces"}` and ask. Workspaces may be Personal (`org_id: null`) or
   Org-owned (`org_id`/`org_name`); include the Org name when disambiguating. If the desired
   Org is missing, the MCP connection may not be authorized for it.
2. **Resource**: If the user gives a name, `find {action:"resources", workspace_id}` to match
   by name. If multiple match, ask. If the name is not obvious,
   `find {action:"search", args:{query}}` as a fallback.
3. **Parent folder** (for creates): If not specified, create at workspace root.

Always report the resolved IDs back to the user so they can confirm.

## Choosing Between Skills

Some intents are genuinely ambiguous. Decision hints:

- **"Track X"** → Is X tabular (tasks, inventory)? → `dokki-table`. Is X narrative (journal, notes)? → `dokki-document`.
- **"Visualize X"** → Is data already in Dokki? → `dokki-artifact` reading from a table. Is it new data? → `dokki-table` first, then `dokki-artifact`.
- **"Share with someone"** → Single user by email → `dokki-workspace` `share {action:"user"}`. Public URL → `dokki-publish`.
- **"Show me what's there"** → Hierarchy view → `dokki-workspace` browse. Content search → `dokki-workspace` search.
- **"Get X from <external app>"** → `dokki-workspace` CONNECT mode first to fetch, then route the result to the right content skill.

## Output Conventions

When completing any workflow, always end with:

1. **What was done** — a one-sentence summary
2. **Resource IDs** — short IDs for anything created/modified
3. **URLs** — document URL, table URL, artifact URL, or public site URL
4. **Next steps** — suggested follow-up actions (publish? tag? share?)

## When to Say No

Don't invoke skills when:
- User is just asking "what is Dokki?" or asking for help — answer directly
- The operation would be clearly destructive (bulk delete, remove custom domain) without explicit confirmation — remember dangerous facade actions return a `confirm_token` you must echo back
- The required context (workspace, resource) can't be resolved — ask first
