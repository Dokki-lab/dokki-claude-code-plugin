---
name: dokki-document
description: Create new documents or edit existing ones using compact node-based operations. Use for writing and modifying rich-text content.
argument-hint: <title-or-resource-id> [instruction]
allowed-tools: mcp__dokki__create_document mcp__dokki__upload_file mcp__dokki__doc_read mcp__dokki__doc_insert mcp__dokki__doc_replace mcp__dokki__doc_delete mcp__dokki__doc_rewrite
---

# Dokki Document — Create & Edit

## Mode Decision

```
Does the resource exist?
├── No  → CREATE mode
└── Yes → EDIT mode (always doc_read first)
```

---

## CREATE Mode

### Tool

`create_document` — Creates document with initial Markdown content.

```
create_document:
  name: string            # Document title (stored separately, NOT in content)
  workspace_id: string
  parent_id?: string      # Folder ID, omit for workspace root
  content?: string        # Markdown body — NO H1 title
```

### Content Authoring Rules

1. **NO H1** in content — title goes in `name` field
2. Start body with intro paragraph or H2 heading
3. Use Markdown syntax:
   - `## Heading 2`, `### Heading 3`
   - `- bullet`, `1. numbered`, `- [ ] task`
   - `` ```language `` code blocks
   - `> quote`, `| table | syntax |`
   - `**bold**`, `*italic*`, `[link](url)`, `` `code` ``
4. Keep paragraphs focused; use headings for scannable sections
5. For local images, call `upload_file` with `inline_image: true` first, then insert
   the returned image `node` with `doc_insert` or use its `url` as the image `src`.

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

### Golden Rule

**Always call `doc_read` immediately before any edit.** Node IDs are short 8-char prefixes that change when the document is modified; stale IDs cause failures or edit the wrong nodes.

### Operation Decision Tree

```
What's the edit?
│
├── Add NEW content (section, paragraph, list)
│     └── doc_insert  (anchor node_id + position before/after)
│
├── Modify EXISTING content
│     ├── Single node (one paragraph / one heading)
│     │     └── doc_replace (node_id only)
│     └── Multiple consecutive nodes (rewrite a section)
│           └── doc_replace (node_id + end_node_id for range)
│
├── Remove content
│     └── doc_delete (node_ids[])
│
└── Replace the ENTIRE document body
      └── doc_rewrite (resource_id + full node list; no doc_read needed)
```

### Full Rewrite

To replace the entire document body, use `doc_rewrite` (resource_id + the full
`nodes` list). It bypasses `doc_replace`'s range cap and does **not** require a prior
`doc_read`, so it's the right tool for "rewrite this whole doc" / "regenerate from
scratch". Returns before/after node counts.

Reserve `doc_replace` with a `node_id`…`end_node_id` range for rewriting a *section*,
not the whole document.

### Node Format Reference

```json
// Paragraph
{ "type": "paragraph", "text": "Hello **world**. See [docs](https://…)." }

// Heading (level 1-6; but avoid level 1 — reserved for title)
{ "type": "heading", "level": 2, "text": "Section" }

// Code block
{ "type": "codeBlock", "language": "typescript", "text": "const x = 1;" }

// Bullet / Ordered list
{ "type": "bulletList", "items": [{ "text": "item one" }, { "text": "item two" }] }
{ "type": "orderedList", "items": [{ "text": "step one" }, { "text": "step two" }] }

// Task list
{ "type": "taskList", "items": [
  { "text": "done task", "checked": true },
  { "text": "pending task", "checked": false }
] }

// Table (pass full Markdown table as text)
{ "type": "table", "text": "| Name | Age |\n| --- | --- |\n| Alice | 30 |" }

// Blockquote
{ "type": "blockquote", "text": "Less is more." }

// Image
{ "type": "image", "src": "https://example.com/img.png", "alt": "Description" }
// Image returned from upload_file inline_image mode
{ "type": "image", "src": "<returned url>", "alt": "<returned alt text>" }

// Horizontal rule
{ "type": "horizontalRule" }

// Inline SVG (raw <svg> markup in `code`; optional `title`)
{ "type": "svg", "code": "<svg xmlns=\"http://www.w3.org/2000/svg\" viewBox=\"0 0 100 100\"><circle cx=\"50\" cy=\"50\" r=\"40\" fill=\"tomato\"/></svg>", "title": "Red circle" }
```

> SVG markup is stored verbatim in `code` and sanitized at render time
> (`<script>` and event handlers are stripped). Put the full
> `<svg>…</svg>` element in `code`, not in `text`.

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| Editing without `doc_read` first | Node IDs are stale, operations target wrong content | Always read immediately before edit |
| Including H1 in content | Title duplicated (once in `name`, once in body) | Put title in `name`, start content with H2+ |
| Using full UUIDs everywhere | Noisy, wastes tokens | Use short 8-char IDs from `doc_read` output |
| Massive single-node rewrites | Loses change granularity | Break into multiple `doc_replace` calls by section |
| `doc_insert` with a wrong anchor | Content ends up in wrong place | Read the node and one before/after to verify anchor |
| Forgetting `end_node_id` on range replace | Only first node gets replaced | For multi-node rewrites, always set `end_node_id` |

## Cross-Skill Follow-Ups

- Need to publish the doc? → `dokki-publish`
- Need to tag, move, or share? → `dokki-workspace`
- Need to insert a local image? → `upload_file` with `inline_image: true`, then `doc_insert`
- Need to embed a chart? → `dokki-artifact` (create artifact), then link it in the doc
