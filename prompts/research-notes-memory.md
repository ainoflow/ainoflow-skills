# Sample prompt — research-notes-memory

- **Skill:** `research-notes-memory` ([SKILL.md](../skills/research-notes-memory/SKILL.md))
- **Required MCP servers:** Memory (`https://mcp.ainoflow.io/mcp/v1/memory`), Storage (`https://mcp.ainoflow.io/mcp/v1/storage/json`)
- **Also connect (repo default):** Files and Inbox — unused by this prompt
- **Expected handoff keys (category `research`):**
  - Memory: `briefs/smoke-q3-pricing`, `notes/smoke-q3-pricing/packaging`, `outlines/smoke-q3-pricing`
  - Storage: `sessions/smoke-q3-pricing` (`readyForDraft: true`)

Copy everything below the line into an agent chat after MCP is connected.

---

Follow the `research-notes-memory` skill. Call `memory_guide` and `storage_guide` first. Use only public Memory and Storage tools. Do not invent tool names.

Act as the researcher agent, then immediately as the writer agent (same API key).

Researcher:

1. `memory_write` category `research`, key `briefs/smoke-q3-pricing` — a short brief on Q3 packaging for a writer, with a `[[outlines/smoke-q3-pricing]]` wiki-link and frontmatter `status: active`.
2. `storage_json_upsert` category `research`, key `sessions/smoke-q3-pricing` with JSON (string `data`): `status` researching, `briefKey` `briefs/smoke-q3-pricing`, `outlineKey` `outlines/smoke-q3-pricing`, `readyForDraft` false, `handoffTo` writer.
3. `memory_search` category `research`, `q` `packaging`, `prefix` `notes/smoke-q3-pricing/` (`q` matches title/prose, not tags alone). Then `memory_write` `notes/smoke-q3-pricing/packaging` linking `[[briefs/smoke-q3-pricing]]` and putting the words **pricing packaging** in the body.
4. `memory_write` `outlines/smoke-q3-pricing` listing that note.
5. `storage_json_patch` the session to `status` `ready_for_draft`, `readyForDraft` true, `updatedNoteKeys` including `notes/smoke-q3-pricing/packaging`.

Writer (new role, same keys — do not rely on chat memory of the bodies):

6. `storage_json_get` `research` / `sessions/smoke-q3-pricing`.
7. `memory_read` the brief and outline keys. `memory_search` + `memory_read` the note. Optionally `memory_context` on the brief (`depth` 2).
8. Reply with: the Storage session JSON, the three Memory keys you read, and one sentence you would put in a draft. Stop. Do not delete keys unless I ask.
