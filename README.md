# Dokki for Claude Code

> Bring your [Dokki](https://dokki.one) workspace into Claude Code — create and edit
> documents, build tables and artifacts, publish public sites, and search everything
> you've written, without leaving the terminal.

This plugin bundles:

- **Two connectors** (`.mcp.json`) — Dokki's hosted MCP servers:
  - **`dokki`** (`/api/mcp`) — the core toolset: `create_document`, `doc_*`, `table_*`,
    `artifact_*`, `search_workspace` / `grep_workspace`, and workspace management.
  - **`dokki-publish`** (`/api/publish-mcp`) — public sites & custom domains, backing the
    `dokki-publish` skill only.
- **Six skills** that orchestrate those tools into real workflows (routing, decision
  trees, templates), so natural-language requests map to the right tool sequence.

| Skill | Command | Scope |
|-------|---------|-------|
| Entry / router | `/dokki:dokki <intent>` | Interprets intent, routes to the right skill, orchestrates multi-skill workflows |
| Workspace | `/dokki:dokki-workspace` | Browse, search, organize (`list_*`, `search_workspace`, `grep_workspace`, `preview_resource`, move, tag, share, delete) |
| Document | `/dokki:dokki-document` | Rich-text docs (`create_document`, `doc_read/insert/replace/delete/rewrite`) |
| Table | `/dokki:dokki-table` | Structured data (`create_table`, `table_*`) |
| Artifact | `/dokki:dokki-artifact` | Interactive JSX / charts (`create_artifact`, `artifact_*`) |
| Publish | `/dokki:dokki-publish` | Public sites & custom domains (`*_publish_site`, `*_custom_domain`) |

## Install

From the community marketplace:

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install dokki@claude-community
```

On first use, Claude Code connects to `https://dokki.one/api/mcp` and
`https://dokki.one/api/publish-mcp`, then runs Dokki's OAuth flow in your browser — no
API key to paste. Both servers share the same discovery, so it's one sign-in.

## Authentication

Both servers support three auth methods:

1. **OAuth** (default) — nothing to configure; Dokki's `.well-known` discovery endpoints
   drive the browser sign-in.
2. **API key** — set an `Authorization: Bearer dk_...` header on each server entry if you
   prefer static credentials:
   ```json
   {
     "mcpServers": {
       "dokki": {
         "type": "http",
         "url": "https://dokki.one/api/mcp",
         "headers": { "Authorization": "Bearer dk_your_api_key_here" }
       },
       "dokki-publish": {
         "type": "http",
         "url": "https://dokki.one/api/publish-mcp",
         "headers": { "Authorization": "Bearer dk_your_api_key_here" }
       }
     }
   }
   ```
3. **Connector token** — workspace-scoped tokens for embedded/integration use.

For an organization workspace, point the core URL at
`https://dokki.one/api/mcp/org/<orgId>`.

## Local development

```bash
# Load the plugin without installing:
claude --plugin-dir .

# Validate before submitting:
claude plugin validate .
```

Try it:

```
/dokki:dokki Write a PRD about rate limiting and publish it to our docs site
/dokki:dokki-document "API Guide"
```

## Self-hosting

If you run your own Dokki instance, change both `url`s in `.mcp.json` to your origin's
`/api/mcp` and `/api/publish-mcp` endpoints.

## License

[MIT](LICENSE)
