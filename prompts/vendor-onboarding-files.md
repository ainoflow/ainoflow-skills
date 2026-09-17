# Sample prompt — vendor-onboarding-files

- **Skill:** `vendor-onboarding-files` ([SKILL.md](../skills/vendor-onboarding-files/SKILL.md))
- **Required MCP servers:** Files, Storage, Memory
- **Also connect (repo default):** Inbox — unused by this prompt
- **Expected handoff keys (category `vendors`):**
  - Files: `smoke-acme-supplies/other-smoke-packet`
  - Storage: `onboarding/smoke-acme-supplies` (`status: ready_for_review`)
  - Memory: `profiles/smoke-acme-supplies`

Copy everything below the line into an agent chat after MCP is connected.

---

Follow the `vendor-onboarding-files` skill. Call `files_guide`, `storage_guide`, and `memory_guide` first. Use only public Files, Storage, and Memory tools.

Act as procurement, then as compliance (same API key). Use vendor slug `smoke-acme-supplies` only.

Procurement:

1. `memory_read` `vendors` / `playbooks/onboarding`. If `NOT_FOUND`, `memory_write` a short required-doc list.
2. `storage_json_get` `vendors` / `onboarding/smoke-acme-supplies` with `allowEmpty` true. If absent, `storage_json_upsert` a checklist: `status` collecting, `required` `["other-smoke-packet"]`, `received` `{}`, `memoryProfileKey` `profiles/smoke-acme-supplies`, `handoffTo` compliance.
3. `files_upload` category `vendors`, key `smoke-acme-supplies/other-smoke-packet`, `fileName` `smoke-packet.txt`, `contentType` `text/plain`, `content` (base64) `U01PS0UgcmVjZWlwdCBzdHVi`.
4. Merge that pointer into `received` (get + upsert with `ifMatch` from `storage_json_get_metadata`, or a patch that includes the full `received` object). Then patch `status` `ready_for_review`.
5. `memory_write` `profiles/smoke-acme-supplies` linking `[[playbooks/onboarding]]` and the Storage checklist key. `files_list` prefix `smoke-acme-supplies/`.

Compliance:

6. `storage_json_list_keys` `vendors`, `prefix` `onboarding/`, `where` `{ "status": "ready_for_review" }`.
7. `storage_json_get` the checklist. `files_get_metadata` / `files_get_url` on `received.other-smoke-packet`. `memory_read` the profile.
8. Reply with the Storage handoff key, Files key, and whether compliance can review without procurement pasting the file. Stop. Do not delete keys unless I ask.
