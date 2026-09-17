# Sample prompt — expense-tracking-handoff

- **Skill:** `expense-tracking-handoff` ([SKILL.md](../skills/expense-tracking-handoff/SKILL.md))
- **Required MCP servers:** Storage (`https://mcp.ainoflow.io/mcp/v1/storage/json`), Files (`https://mcp.ainoflow.io/mcp/v1/files`), Memory (`https://mcp.ainoflow.io/mcp/v1/memory`)
- **Also connect (repo default):** Inbox — unused by this prompt
- **Expected handoff keys (category `expenses`):**
  - Files: `receipts/2026-09/smoke-tester/smoke-berlin/1-stub`
  - Storage: `reports/2026-09/smoke-tester/smoke-berlin` (`status: submitted`)
  - Memory: `summaries/2026-09/smoke-tester/smoke-berlin`

Copy everything below the line into an agent chat after MCP is connected.

---

Follow the `expense-tracking-handoff` skill. Call `storage_guide`, `files_guide`, and `memory_guide` first. Use only public Storage, Files, and Memory tools.

Act as the employee agent, then as the finance agent (same API key).

Employee:

1. `files_upload` category `expenses`, key `receipts/2026-09/smoke-tester/smoke-berlin/1-stub`, `fileName` `smoke-receipt.txt`, `contentType` `text/plain`, `content` (base64) `U01PS0UgcmVjZWlwdCBzdHVi`. If `CONFLICT`, `files_get_metadata` and continue.
2. `storage_json_upsert` category `expenses`, key `reports/2026-09/smoke-tester/smoke-berlin` with `status` submitted, one line (Smoke Cafe, meals, EUR 1.00, date 2026-09-17), `receipt` pointing at that Files key, `memoryKey` `summaries/2026-09/smoke-tester/smoke-berlin`, `handoffTo` finance. `data` must be a JSON string (object).
3. `memory_write` the summary page with frontmatter `status: submitted` and a pointer to the Storage report key. If `policies/expense-policy` is missing, write a one-line policy page first and `[[wiki-link]]` it.

Finance (do not reuse receipt bytes from chat — pull from Ainoflow):

4. `storage_json_list_keys` category `expenses`, `prefix` `reports/`, `where` `{ "status": "submitted" }`.
5. `storage_json_get` the smoke report. `files_get_metadata` and `files_get_url` on the receipt key. `memory_read` the summary.
6. Reply with the Storage key, Files key, Memory key, receipt `size`, and whether finance can review without anything pasted from the employee. Optional: `storage_json_patch` `{"status":"approved","reviewedBy":"finance"}`. Stop. Do not delete keys unless I ask.
