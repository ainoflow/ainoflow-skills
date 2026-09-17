# Sample prompt — inbox-triage-to-storage

- **Skill:** `inbox-triage-to-storage` ([SKILL.md](../skills/inbox-triage-to-storage/SKILL.md))
- **Required MCP servers:** Inbox, Storage, Memory (Files only if you copy attachments)
- **Inbox hook:** replace `YOUR_TRIAGE_INBOX_EMAIL` (or use `AINOFLOW_TRIAGE_INBOX_EMAIL`). If you have no hook, **stop** and skip this prompt (Smoke B is optional).
- **Expected handoff keys (category `inbox-triage`):**
  - Storage: `{lane}/{yyyy-mm-dd}/{message-id}` where `lane` is `support`, `billing`, or `sales`, `status: open`
  - Memory (optional): `log/{yyyy-mm-dd}`

Copy everything below the line into an agent chat after MCP is connected.

---

Follow the `inbox-triage-to-storage` skill. Call `inbox_guide`, `storage_guide`, and `memory_guide` first. Stay on Inbox + Storage + Memory. Do not invent Shield or Guard tools.

If `YOUR_TRIAGE_INBOX_EMAIL` is still a placeholder, stop and say Smoke B is skipped.

Triage:

1. `inbox_messages` email `YOUR_TRIAGE_INBOX_EMAIL`, `isHandled` false, `limit` 0 (read `totalCount`). Then page with `limit` 5, `sortBy` `ReceivedAt`, `sortOrder` `Desc`. Do not send `page` and `offset` together.
2. If `totalCount` is 0, say so and stop.
3. Classify one message: billing (invoice/payment/refund), sales (demo/pricing/quote), else support. If two lanes fit, prefer billing over sales over support and set `needsHuman` true.
4. `storage_json_upsert` category `inbox-triage`, key `{lane}/{yyyy-mm-dd}/{message-id}` with `status` open, inbox metadata, a 400-character `excerpt`, `handoffTo` the specialist. `data` is a JSON string.
5. `inbox_handle_message` that `messageId`. Do not `inbox_delete_message`.
6. Optional: `memory_edit` append on `inbox-triage` / `log/{yyyy-mm-dd}`.

Specialist (same API key, your lane only):

7. `storage_json_list_keys` `inbox-triage`, `prefix` `{lane}/`, `where` `{ "status": "open" }`.
8. `storage_json_get` the smoke key. Reply with the Storage handoff key, lane, and subject. Stop.
