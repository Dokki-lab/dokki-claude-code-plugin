---
name: dokki-document
description: Create new documents or edit existing ones using compact node-based operations. Use for writing and modifying rich-text content.
argument-hint: <title-or-resource-id> [instruction]
allowed-tools: mcp__dokki__create mcp__dokki__read mcp__dokki__edit
---

# Dokki Document — Create & Edit

Documents live behind three facade actions: `create {action:"doc"}`, `read {action:"doc"}`, and
`edit {action:"doc.edit"/"doc.rewrite"}`. Each is `<facade> {action, <ids>, args:{…}}`.

## Mode Decision

```
Does the resource exist?
├── No  → CREATE mode
└── Yes → EDIT mode
```

---

## CREATE Mode

### Action

`create {action:"doc", …}` — Creates a document with initial Markdown content.

```
create:
  action: "doc"
  workspace_id: string
  parent_id?: string      # Folder ID, omit for workspace root
  args:
    name: string          # Document title (stored separately, NOT in content)
    content?: string      # Markdown body — NO H1 title
    metadata?: object     # Structured JSON on the resource, e.g.
                          # {kind:"api_endpoint", http_method:"GET", path:"/api/v1/credits"}
```

`args.metadata` attaches structured JSON to the document resource (distinct from its Markdown
body) — useful for classifying docs, e.g. an API-endpoint catalog. Set it later or change it with
`edit {action:"resource.update", resource_id, args:{metadata}}` (replaces the object wholesale).

### Content Authoring Rules

1. **NO H1** in content — title goes in `args.name`
2. Start body with intro paragraph or H2 heading
3. Use Markdown syntax:
   - `## Heading 2`, `### Heading 3`
   - `- bullet`, `1. numbered`, `- [ ] task`
   - `` ```language `` code blocks
   - `> quote`, `| table | syntax |`
   - `**bold**`, `*italic*`, `[link](url)`, `` `code` ``
4. Keep paragraphs focused; use headings for scannable sections
5. For local images, upload first via `dokki-workspace` → `create {action:"file", args:{inline_image:true}}`,
   then insert the returned image with an `edit {action:"doc.edit"}` `insert` op, or use its `url` as the image `src`.

### Templates

<details>
<summary>Meeting Notes</summary>

```markdown
## Attendees
- Alice, Bob, Carol

## Agenda
1. Status updates
2. Blockers
3. Next steps

## Discussion
…

## Action Items
- [ ] Alice: draft spec by Friday
- [ ] Bob: review PR #42
```
</details>

<details>
<summary>PRD / Design Doc</summary>

```markdown
## Problem
<What are we solving and why now?>

## Goals
- <G1>
- <G2>

## Non-Goals
- <Things explicitly out of scope>

## Proposal
<High-level approach>

## Detailed Design
### Component A
…

## Open Questions
- <Q1>

## Risks & Mitigations
| Risk | Mitigation |
|------|-----------|
| … | … |
```
</details>

<details>
<summary>Tutorial</summary>

```markdown
## Prerequisites
- <What the reader needs before starting>

## Overview
<What they'll build / learn>

## Step 1: <first step>
…

## Step 2: <next step>
…

## Troubleshooting
…
```
</details>

---

## EDIT Mode

### Reading first — `read {action:"doc"}` has modes

`read {action:"doc", resource_id, args:{mode?, page?, locator?}}` supports three modes — pick the
lightest one for the job:

| Mode | Returns | Use for |
|------|---------|---------|
| `view` (default) | Markdown of the doc — **no node ids** | Understanding current content before an edit |
| `outline` | Headings only | Orienting in a large doc, finding the section to target |
| `edit` | Node ids for a **located region** | When you specifically need node ids for a precise op |

Large docs paginate — pass `args.page`. Use `outline` to locate a section cheaply, then read just
that region in `edit` mode with a `locator` if you need node ids.

### Golden Rule — anchor/section beats node ids

The `doc.edit` op-array can **locate by anchor or section**, so most edits need **no node ids and
no prior read** of the raw structure:

- Locate by `anchor:{text}` — matches on visible text near where you want to act.
- Locate by `section` — targets a heading and its body.
- Only fall back to `node_id` (from `read {action:"doc", args:{mode:"edit"}}`) when anchor/section
  can't pin the spot uniquely.

When you do use `node_id`, read immediately before the edit — node IDs are short 8-char prefixes
that change when the document is modified.

### Operation Decision Tree

```
What's the edit?
│
├── Add NEW content (section, paragraph, list)
│     └── edit {action:"doc.edit", args:{ops:[{op:"insert", anchor|section|node_id, position?, markdown}]}}
│
├── Modify EXISTING content
│     └── edit {action:"doc.edit", args:{ops:[{op:"replace", anchor|section|node_id, markdown}]}}
│
├── Remove content
│     └── edit {action:"doc.edit", args:{ops:[{op:"delete", anchor|section|node_id}]}}
│
└── Replace the ENTIRE document body
      └── edit {action:"doc.rewrite", args:{markdown}}   # no read, no node ids
```

A single op is just a **one-element `ops` list**. Batch several ops into one `ops` array when
making related edits.

### `doc.edit` op shape

```json
edit({
  "action": "doc.edit",
  "resource_id": "…",
  "args": {
    "ops": [
      { "op": "insert",  "anchor": { "text": "## Risks" }, "position": "after", "markdown": "### New risk\nDetails here." },
      { "op": "replace", "section": "Overview", "markdown": "## Overview\nRewritten intro." },
      { "op": "delete",  "anchor": { "text": "Deprecated note" } }
    ]
  }
})
```

- `markdown` is accepted directly — write natural Markdown, not compact node JSON.
- `position` (`before` | `after`) applies to `insert` relative to the located anchor/section.
- Singular actions `doc.insert` / `doc.replace` / `doc.delete` still exist if you prefer one op per call.

### Full Rewrite

To replace the entire document body, use `edit {action:"doc.rewrite", resource_id, args:{markdown}}`
(or `args:{nodes}`). It's the **lowest-fragility path**: no prior `read`, no node ids — the right
tool for "rewrite this whole doc" / "regenerate from scratch". Returns before/after node counts.

Reserve a `doc.edit` `replace` op over a `section` for rewriting a *section*, not the whole document.

### Markdown authoring reference

Everything is plain Markdown inside `content` / `markdown`:

```markdown
## Section
Paragraph with **bold**, *italic*, a [link](https://…), and `inline code`.

- bullet one
- bullet two

1. step one
2. step two

- [x] done task
- [ ] pending task

> Less is more.

| Name | Age |
| --- | --- |
| Alice | 30 |

​```typescript
const x = 1;
​```

![Description](https://example.com/img.png)
```

For an image returned from an upload (`create {action:"file", args:{inline_image:true}}`), use the
returned `url` as the image `src` in your Markdown, or insert the returned image `node` via a
`doc.edit` `insert` op.

> Raw inline SVG is supported — pass an `svg` node (with the `<svg>…</svg>` markup in its `code`
> field) via a `doc.edit` op using `nodes` instead of `markdown`. SVG is sanitized at render time
> (`<script>` and event handlers are stripped).

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| Reaching for node ids by default | Extra reads, stale-id failures | Locate by `anchor:{text}` or `section` — no read needed |
| Using a `node_id` without reading right before | Node IDs go stale after any edit | Read in `edit` mode immediately before, or use anchor/section |
| Including H1 in content | Title duplicated (once in `name`, once in body) | Put title in `args.name`, start content with H2+ |
| Reading a huge doc in full to make one edit | Wastes tokens | `read` in `outline` mode to locate, then edit by section |
| Massive single-op rewrites of a section | Loses change granularity | Split into multiple `ops` by section, or use `doc.rewrite` for the whole doc |
| `insert` op with a wrong/ambiguous anchor | Content lands in the wrong place | Use a distinctive anchor phrase or a `section`; verify with an `outline` read |
| Passing compact node JSON where markdown is expected | Confusing / rejected | Prefer `markdown` in `doc.edit` ops; use `nodes` only for special node types (e.g. svg) |

## Cross-Skill Follow-Ups

- Need to publish the doc? → `dokki-publish`
- Need to tag, move, or share? → `dokki-workspace`
- Need to insert a local image? → `dokki-workspace` `create {action:"file", args:{inline_image:true}}`, then a `doc.edit` `insert` op
- Need to embed a chart? → `dokki-artifact` (create artifact), then link it in the doc
