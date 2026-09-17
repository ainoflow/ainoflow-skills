---
name: research-notes-memory
description: Use when a researcher and a writer must share wiki-linked Memory notes plus Storage session progress for the same brief.
---

# Research notes in Memory

Researcher agent writes and links markdown notes. Writer agent recalls them by meaning (`memory_search`) and by address (`memory_read`), then drafts from the same graph. Storage holds session/progress keys the writer can poll without searching prose.

Public contract: [Memory](https://www.ainoflow.io/docs/mcp/memory), [Storage](https://www.ainoflow.io/docs/mcp/storage). Start with `memory_guide` and `storage_guide`.

Use Memory for knowledge you may need to **find**. Use Storage for state you will **look up by key**. That split is the published product guidance.

## Shared addresses

| Surface | Category | Key convention |
| --- | --- | --- |
| Memory | `research` | `briefs/{brief-slug}`, `sources/{brief-slug}/{source-slug}`, `notes/{brief-slug}/{note-slug}`, `outlines/{brief-slug}` |
| Storage | `research` | `sessions/{brief-slug}` |

Default Memory category in the product docs is `main`. This skill uses `research` so briefs do not mix with unrelated notes. Category is 1–100 characters; keys are path-like, 1–1024 characters.

## Memory conventions

- Every note is markdown. `memory_write` stores the body verbatim and keeps the previous body as a revision.
- Put YAML frontmatter at the top. It is parsed at write time and is what `memory_search` filters on (`tags`, `frontmatter` containment).
- Link related pages with `[[wiki-links]]` using the **key** (example: `[[briefs/q3-pricing]]`, `[[sources/q3-pricing/gartner-note]]`).
- `memory_search` returns ranked hits (`title`, `frontmatter`, `section`, `snippet`, `score`, `referenceCount`) and **never the body**. Always `memory_read` to draft.
- `memory_read` returns raw markdown in the text block; `category`, `key`, `version`, `etag` travel as metadata.
- `memory_context` walks the `[[wiki-link]]` neighborhood (no bodies). Use it before a move/delete.
- To keep only current notes in search: `frontmatter`: `{ "status": "active" }`. Supersede with `{ "status": "superseded", "supersededBy": "notes/..." }` and optional `followFrontmatter`: `supersededBy`.

Suggested frontmatter:

```markdown
---
status: active
type: note
brief: q3-pricing
tags:
  - pricing
  - competitors
---
# Competitor packaging

Tied to [[briefs/q3-pricing]]. Source: [[sources/q3-pricing/public-pricing-pages]]
```

## Steps (researcher agent)

1. **Guides.** `memory_guide`, `storage_guide`.

2. **Brief page.** `memory_write` `research` / `briefs/{brief-slug}` with the question, audience, due date, and links to `[[outlines/{brief-slug}]]`.

3. **Session pointer.** `storage_json_upsert` category `research`, key `sessions/{brief-slug}`:

```json
{
  "status": "researching",
  "briefKey": "briefs/q3-pricing",
  "updatedNoteKeys": [],
  "outlineKey": "outlines/q3-pricing",
  "handoffTo": "writer",
  "readyForDraft": false
}
```

4. **Sources and notes.** For each source, `memory_write` `sources/{brief-slug}/{source-slug}` (URL, accessed date, claim). For each synthesis, `memory_write` `notes/{brief-slug}/{note-slug}` with `[[briefs/...]]` and `[[sources/...]]`. Prefer `memory_edit` (`append` / `str_replace` / `insert`) for small updates so history stays incremental. `str_replace` needs `oldText` to occur exactly once.

5. **Dedupe before writing.** `memory_search` category `research`, `q`: the claim, `prefix`: `notes/{brief-slug}/`. Read hits before creating a near-duplicate.

6. **Outline.** When the graph is good enough, `memory_write` `outlines/{brief-slug}` listing the note keys in draft order.

7. **Mark ready.** `storage_json_patch` `sessions/{brief-slug}` with `{"status":"ready_for_draft","readyForDraft":true,"updatedNoteKeys":["notes/..."]}`. Optionally `expiresMs` only on scratch Storage keys, not on the session the writer still needs.

## Steps (writer agent)

1. Same API key. Guides first.
2. `storage_json_get` `research` / `sessions/{brief-slug}` (or `storage_json_list_keys` with `prefix`: `sessions/` and `where`: `{ "readyForDraft": true }`).
3. `memory_read` `briefKey` and `outlineKey`.
4. `memory_search` `q`: the brief question, `prefix`: `notes/{brief-slug}/`, `sort`: `blended` (mixes match, inbound links, freshness). Then `memory_read` each useful key.
5. `memory_context` on the brief key (`depth`: `2`) to see linked sources/notes you missed.
6. Draft in the writer’s editor. When the draft is based on a note, keep the `[[wiki-link]]` in a “Sources” section of a new Memory page `drafts/{brief-slug}` if you want the researcher to review.
7. `storage_json_patch` the session: `{"status":"drafting"}` then `{"status":"draft_ready","handoffTo":"researcher"}`.
8. Researcher reviews with `memory_read` on `drafts/{brief-slug}` and `memory_edit` comments.

Do not store the full draft only in chat. If the draft must survive a new conversation, `memory_write` it.

## Multi-agent handoff

| Who | Writes | Reads |
| --- | --- | --- |
| **Agent A — researcher** | Memory briefs, sources, notes, outline with `[[wiki-links]]`; Storage `sessions/{brief-slug}` (`readyForDraft`) | Existing notes via `memory_search` / `memory_read` |
| **Agent B — writer** | Memory `drafts/{brief-slug}`; Storage session status | Storage session key; Memory outline + search + `memory_context` |
| **Agent A again** | Memory edits on draft or notes | Draft page + session `draft_ready` |

Handoff token: Storage key `sessions/{brief-slug}` in category `research`, plus Memory key `briefs/{brief-slug}`. The writer does not need a transcript from the researcher — search and wiki-links are the index.

Copy-paste try-it prompt (smoke keys under `research`): [prompts/research-notes-memory.md](../../prompts/research-notes-memory.md).

## Failures (public codes)

- `ITEM_TOO_LARGE` if a note exceeds the Memory size in force (default 100 KB). Split the page instead of stuffing a source dump.
- `PRECONDITION_FAILED` when `ifMatch` / `expectedVersion` does not match — re-read, then edit.
- `RATE_LIMITED` when the live document count is full (`memory_delete` or expire frees a slot).
- `memory_move` if you rename a key; history and references are preserved. Walk `memory_context` first so you know what still points at the old key.
