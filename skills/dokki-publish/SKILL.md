---
name: dokki-publish
description: Publish Dokki content as a public site — manage published sites, publish or unpublish resources, configure custom domains and site settings.
argument-hint: <action> [workspace-or-site] [details]
allowed-tools: mcp__dokki-publish__get_publish_site mcp__dokki-publish__create_publish_site mcp__dokki-publish__update_publish_site mcp__dokki-publish__publish_resource mcp__dokki-publish__unpublish_resource mcp__dokki-publish__list_published_resources mcp__dokki-publish__set_custom_domain mcp__dokki-publish__remove_custom_domain mcp__dokki-publish__check_domain_status
---

# Dokki Publish — Sites, Resources & Custom Domains

> **Server note:** publishing tools live on a separate MCP server (`dokki-publish` →
> `/api/publish-mcp`), distinct from the core `dokki` server (`/api/mcp`). This plugin
> wires both. If you run the skills standalone, add **both** servers (see the plugin
> README). Resource browsing/reading still comes from the core `dokki` server.

## Mental Model

- Each **workspace** can have **one published site** at `dokki.one/pub/{slug}`
- The site is **inactive by default** — not publicly reachable until `is_active = true`
- **Resources** (documents, tables, artifacts) are published **individually** into a site — the site is a curated subset
- Published content is a **frozen snapshot** — editing the source doesn't auto-update the public version; you must re-publish
- **Custom domains** can replace the `dokki.one/pub/...` URL (e.g., `docs.example.com`)

---

## Action Decision Tree

```
What does the user want?
│
├── Start publishing from scratch
│     → Full Setup Flow (below)
│
├── Add a doc/table to an existing site
│     → publish_resource
│
├── Remove a doc/table from a site
│     → unpublish_resource
│
├── Refresh an already-published doc after edits
│     → publish_resource again (re-freezes snapshot)
│
├── Make site public / private
│     → update_publish_site with is_active
│
├── Change site slug, homepage, nav tags
│     → update_publish_site with settings
│
├── Add / verify / remove custom domain
│     → set_custom_domain / check_domain_status / remove_custom_domain
│
└── "Is anything published?" / status check
      → get_publish_site + list_published_resources
```

---

## Full Setup Flow (New Site)

Use this when the user says "I want to publish these docs as a public site".

1. **Check existing state**
   `get_publish_site(workspace_id)` — skip to step 3 if a site already exists.

2. **Create the site**
   ```
   create_publish_site:
     workspace_id
     slug          # e.g. "acme-docs"; lowercase, hyphens only
     is_active: false  # start hidden until content is ready
   ```
   Save the returned `site_id`.

3. **Select resources to publish**
   If not specified, ask. Or call `list_resources` (via `dokki-workspace`) and let the user pick.

4. **Publish each resource**
   ```
   publish_resource(site_id, resource_id)
   ```
   Loop per resource. Slugs are auto-generated from names; override with `slug:` if needed.

5. **Configure site settings (optional)**
   ```
   update_publish_site:
     site_id
     settings:
       home:
         title: "Acme Documentation"
         description: "Everything about our product"
         resource_id: <id>     # optional: pin a specific doc as homepage
       nav_tags: [<tag_id>, …]  # up to 5
       site_url: "https://acme.com"  # logo click destination
   ```

6. **Flip it live**
   ```
   update_publish_site(site_id, is_active: true)
   ```

7. **(Optional) Custom domain**
   See flow below.

8. **Report**
   - Public URL: `https://dokki.one/pub/<slug>` (+ custom domain if set)
   - Number of resources published
   - Next steps (add custom domain? share URL with team?)

---

## Custom Domain Flow

```
set_custom_domain(site_id, "docs.example.com")
    ↓
status: pending  ← Cloudflare starts SSL provisioning
    ↓
User adds CNAME DNS record pointing docs.example.com → Dokki
    ↓
check_domain_status(site_id)  ← Poll until verified
    ↓
status: active  ← HTTPS works, middleware rewrites traffic
```

### Instructions to give the user

After `set_custom_domain` succeeds, tell the user:

> Domain registered with status `pending`. To activate:
> 1. Go to your DNS provider (Cloudflare, Namecheap, Route53, etc.)
> 2. Add a CNAME record:
>    - Name: `docs` (or whatever subdomain you chose)
>    - Value: `cname.dokki.one` *(or the target documented by Dokki)*
> 3. Wait a few minutes for DNS propagation + SSL issuance
> 4. Run `check_domain_status` to verify

### Domain Statuses

| Status | Meaning |
|--------|---------|
| `none` | No domain configured |
| `pending` | Cloudflare created hostname, waiting for CNAME + SSL |
| `active` | Verified and serving HTTPS |
| `error` | Something failed — check Cloudflare dashboard |

---

## Publishing vs Sharing

Easy to confuse — they're different:

| Need | Tool | Where |
|------|------|-------|
| "Send a link to Alice" | `share_resource` (email, role) | `dokki-workspace` |
| "Let anyone with the URL view it" | `share_resource` (public_access: view) | `dokki-workspace` |
| "Build a browsable docs website" | `publish_resource` on a `published_site` | **this skill** |

A shared link = single-resource access, hosted on the app.
A published site = curated collection of resources on a public, navigable website (with homepage, nav, SEO, custom domain).

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| Publishing then editing, expecting auto-refresh | Published content is **frozen** — live edits invisible to the public | Re-call `publish_resource` to refresh the snapshot |
| Creating site with `is_active: true` before adding content | Empty public URL → bad first impression | Start with `is_active: false`, flip it after publishing resources |
| Using marketing-y slug with caps/spaces | Tool lowercases + hyphenates, may surprise user | Confirm cleaned slug with user before creating |
| Removing domain when user meant to disable site | Nukes Cloudflare hostname record — re-adding takes time | Confirm intent: disable (is_active=false) vs remove domain |
| Not waiting for SSL on custom domain | User sees cert warnings | Tell user to run `check_domain_status` and wait for `active` |
| Forgetting `nav_tags` are tag IDs | Passing tag names silently breaks navigation | Look up tag IDs first |
| `home.resource_id` set to a non-published resource | Homepage 404s | Ensure the homepage resource is also published |

## Cross-Skill Follow-Ups

- User wants to update content → `dokki-document` / `dokki-table` / `dokki-artifact`, then re-`publish_resource`
- User wants to see what's publishable → `dokki-workspace` `list_resources`
- User wants SEO for a published doc → `update_publish_site` + consider the doc's tags and headings
