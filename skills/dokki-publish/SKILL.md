---
name: dokki-publish
description: Publish Dokki content as a public site — manage published sites, publish or unpublish resources, configure custom domains and site settings.
argument-hint: <action> [workspace-or-site] [details]
allowed-tools: mcp__dokki__publish mcp__dokki__find
---

# Dokki Publish — Sites, Resources & Custom Domains

> **Server note:** publishing is now the **`publish` facade on the single `dokki` MCP server**
> (`https://dokki.one/mcp/v2`) — the old separate `dokki-publish` server (`/api/publish-mcp`) has
> been folded in and should be dropped. Resource browsing/reading come from the same server via the
> `find` and `read` facades. Every call here is `publish {action, <ids>, args:{…}}`.

## Mental Model

- Each **workspace** can have **one published site** at `dokki.one/pub/{slug}`
- `publish {action:"site.create"}` defaults to `is_active: true` if omitted; pass
  `args:{is_active: false}` when staging a site before launch.
- **Resources** (documents, tables, artifacts) are published **individually** into a site — the site is a curated subset
- Published content is a **frozen snapshot** — editing the source doesn't auto-update the public version; you must re-publish
- **Custom domains** can replace the `dokki.one/pub/...` URL (e.g., `docs.example.com`)
- `publish {action:"add"}` and `share {action:"public"}` expose content publicly, so they return a
  `confirm_token` — re-send the same call with the token to execute.

---

## Action Decision Tree

```
What does the user want?
│
├── Start publishing from scratch
│     → Full Setup Flow (below)
│
├── Add a doc/table to an existing site
│     → publish {action:"add", site_id, resource_id}   ⚠️ confirm
│
├── Remove a doc/table from a site
│     → publish {action:"remove", site_id, resource_id}
│
├── Refresh an already-published doc after edits
│     → publish {action:"add", …} again (re-freezes snapshot)   ⚠️ confirm
│
├── Make site public / private
│     → publish {action:"site.update", site_id, args:{is_active}}
│
├── Change site slug, homepage, nav tags
│     → publish {action:"site.update", site_id, args:{settings|slug}}
│
├── Add / verify / remove custom domain
│     → publish {action:"domain.set"/"domain.status"/"domain.remove"}
│
└── "Is anything published?" / status check
      → publish {action:"site"} + publish {action:"resources", site_id}
```

---

## Full Setup Flow (New Site)

Use this when the user says "I want to publish these docs as a public site".

1. **Check existing state**
   `publish {action:"site", workspace_id}` — skip to step 3 if a site already exists.

2. **Create the site**
   ```
   publish:
     action: "site.create"
     workspace_id
     args:
       slug: "acme-docs"     # lowercase, hyphens only
       is_active: false      # explicit: start hidden until content is ready
   ```
   Save the returned `site_id`.

3. **Select resources to publish**
   If not specified, ask. Or call `find {action:"resources", workspace_id}` (via `dokki-workspace`) and let the user pick.

4. **Publish each resource**
   ```
   publish { action:"add", site_id, resource_id }   # confirm — exposes it publicly
   ```
   Loop per resource. It returns a `confirm_token` the first time — re-send with the token to
   execute. Slugs are auto-generated from names; override with `args:{slug}` if needed.

5. **Configure site settings (optional)**
   ```
   publish:
     action: "site.update"
     site_id
     args:
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
   publish { action:"site.update", site_id, args:{ is_active: true } }
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
publish { action:"domain.set", site_id, args:{ domain:"docs.example.com" } }
    ↓
status: pending  ← Cloudflare starts SSL provisioning
    ↓
User adds CNAME DNS record pointing docs.example.com → Dokki
    ↓
publish { action:"domain.status", site_id }  ← Poll until verified
    ↓
status: active  ← HTTPS works, proxy rewrites traffic
```

### Instructions to give the user

After `publish {action:"domain.set"}` succeeds, tell the user:

> Domain registered with status `pending`. To activate:
> 1. Go to your DNS provider (Cloudflare, Namecheap, Route53, etc.)
> 2. Add a CNAME record:
>    - Name: `docs` (or whatever subdomain you chose)
>    - Value: `cname.dokki.one` *(or the target documented by Dokki)*
> 3. Wait a few minutes for DNS propagation + SSL issuance
> 4. Run `publish {action:"domain.status"}` to verify

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

| Need | Call | Where |
|------|------|-------|
| "Send a link to Alice" | `share {action:"user", args:{email, role}}` | `dokki-workspace` |
| "Let anyone with the URL view it" | `share {action:"public", args:{public_access:"view"}}` | `dokki-workspace` |
| "Build a browsable docs website" | `publish {action:"add", site_id, resource_id}` | **this skill** |

A shared link = single-resource access, hosted on the app.
A published site = curated collection of resources on a public, navigable website (with homepage, nav, SEO, custom domain).

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| Publishing then editing, expecting auto-refresh | Published content is **frozen** — live edits invisible to the public | Re-call `publish {action:"add"}` to refresh the snapshot |
| Omitting `is_active` when creating a staging site | Defaults to live, so an empty URL may be public | Pass `args:{is_active:false}`, then flip it after publishing resources |
| Using marketing-y slug with caps/spaces | Slug is lowercased + hyphenated, may surprise user | Confirm cleaned slug with user before creating |
| Removing domain when user meant to disable site | Nukes Cloudflare hostname record — re-adding takes time | Confirm intent: disable (`site.update` is_active=false) vs `domain.remove` |
| Not waiting for SSL on custom domain | User sees cert warnings | Tell user to run `publish {action:"domain.status"}` and wait for `active` |
| Forgetting `nav_tags` are tag IDs | Passing tag names silently breaks navigation | Look up tag IDs first |
| `home.resource_id` set to a non-published resource | Homepage 404s | Ensure the homepage resource is also published |
| Ignoring the `confirm_token` on `publish {action:"add"}` | Nothing gets published | Re-send the same call with the returned token |

## Cross-Skill Follow-Ups

- User wants to update content → `dokki-document` / `dokki-table` / `dokki-artifact`, then re-`publish {action:"add"}`
- User wants to see what's publishable → `dokki-workspace` `find {action:"resources"}`
- User wants SEO for a published doc → `publish {action:"site.update"}` + consider the doc's tags and headings
