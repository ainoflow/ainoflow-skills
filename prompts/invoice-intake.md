# Sample prompt — invoice-intake

- **Skill:** `invoice-intake` ([SKILL.md](../skills/invoice-intake/SKILL.md))
- **Required MCP servers:** Inbox, Files, Storage, Memory
- **Inbox hook:** replace `YOUR_INVOICE_INBOX_EMAIL` (or use `AINOFLOW_INVOICE_INBOX_EMAIL`). If you have no hook, **stop** and skip this prompt (Smoke B is optional).
- **Expected handoff keys (category `invoices`):**
  - Files: `originals/smoke-acme/SMOKE-001`
  - Storage: `queue/smoke-acme/SMOKE-001` (`status: pending_review`)
  - Memory: `vendors/smoke-acme`, `notes/smoke-acme/SMOKE-001`

Copy everything below the line into an agent chat after MCP is connected.

---

Follow the `invoice-intake` skill. Call `inbox_guide`, `files_guide`, `storage_guide`, and `memory_guide` first. Use only published Inbox / Files / Storage / Memory tools (`inbox_guide`, `inbox_messages`, `inbox_attachment_url`, `inbox_handle_message`, `inbox_unhandle_message`, `inbox_delete_message` — do not delete after a successful queue).

If `YOUR_INVOICE_INBOX_EMAIL` is still a placeholder, stop and say Smoke B is skipped.

Intake:

1. `inbox_messages` email `YOUR_INVOICE_INBOX_EMAIL`, `isHandled` false, `limit` 0. Then list with `limit` 5.
2. If there are no messages, say so and stop (I must send mail to the hook first).
3. `inbox_handle_message` on one invoice-like `messageId`. Prefer an existing `downloadUrl`; otherwise `inbox_attachment_url`. Download bytes yourself — Inbox never returns raw attachment content.
4. `files_upload` category `invoices`, key `originals/smoke-acme/SMOKE-001`, base64 `content`, supply `fileName` and `contentType`.
5. `storage_json_upsert` `invoices` / `queue/smoke-acme/SMOKE-001` with `status` pending_review, totals you can extract, `files` pointer, `handoffTo` ap-reviewer. `data` is a JSON string.
6. `memory_search` / `memory_write` vendor + note pages under `invoices` with `[[wiki-links]]`.

AP reviewer (same API key):

7. `storage_json_list_keys` `invoices`, `prefix` `queue/`, `where` `{ "status": "pending_review" }`.
8. `storage_json_get` the smoke queue key. `files_get_metadata` / `files_get_url`. `memory_read` vendor and note.
9. Reply with the Storage handoff key and whether the original file URL works. Do not `inbox_delete_message`. Stop.
