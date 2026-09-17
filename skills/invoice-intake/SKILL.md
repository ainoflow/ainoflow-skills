---
name: invoice-intake
description: Use when vendor invoices arrive by email or PDF and an AP reviewer must pick up the same queue, originals, and vendor notes.
---

# Invoice intake

Intake agent (AP clerk / mailbox watcher) turns inbound invoices into shared Ainoflow records. An AP reviewer agent — another chat, app, or user with the **same API key** — reads those records without a private handoff channel.

Public contract: [Inbox](https://www.ainoflow.io/docs/mcp/inbox), [Files](https://www.ainoflow.io/docs/mcp/files), [Storage](https://www.ainoflow.io/docs/mcp/storage), [Memory](https://www.ainoflow.io/docs/mcp/memory). Call `inbox_guide`, `files_guide`, `storage_guide`, and `memory_guide` before the first write so limits and tool schemas are current.

## Shared addresses

| Surface | Category | Key convention |
| --- | --- | --- |
| Inbox | (hook email, not a category) | List with `YOUR_INVOICE_INBOX_EMAIL` |
| Files | `invoices` | `originals/{vendor-slug}/{invoice-number}` |
| Storage | `invoices` | `queue/{vendor-slug}/{invoice-number}` |
| Memory | `invoices` | `vendors/{vendor-slug}`, `policies/ap-intake`, `notes/{vendor-slug}/{invoice-number}` |

Keys are path-like. Files keys cannot contain empty segments, `.`, or `..`. Storage `data` must be a JSON object or array (never a bare string). Memory bodies are markdown, max 100 KB by default.

## Steps (intake agent)

1. **Guides.** Call `inbox_guide`, `files_guide`, `storage_guide`, `memory_guide`.

2. **Find unhandled mail.** `inbox_messages` with:
   - `email`: `YOUR_INVOICE_INBOX_EMAIL` (the Inbox hook address from your Ainoflow inbox)
   - `isHandled`: `false`
   - `limit`: `10` or smaller — each item includes full `bodyText` and attachment metadata

   If invoices are uploaded as PDFs instead of emailed, skip Inbox: `files_upload` the PDF into `invoices` / `originals/...` and continue at step 5.

3. **Claim a message.** For a mail that looks like an invoice (subject/body/PDF attachment), call `inbox_handle_message` with `messageId`. The call is idempotent and returns the full message, including attachment ids and short-lived `downloadUrl` values when present.

4. **Attachment bytes.** Prefer the `downloadUrl` already on the message. If it expired, call `inbox_attachment_url` with `attachmentId` (URL is valid for 1 hour). Download the file locally. Inbox tools never return raw attachment bytes.

5. **Store the original.** `files_upload`:
   - `category`: `invoices`
   - `key`: `originals/{vendor-slug}/{invoice-number}` (supply `key` so a second upload of the same invoice fails with `CONFLICT` instead of silently creating a UUID)
   - `fileName`: original PDF name
   - `contentType`: `application/pdf`
   - `content`: base64 of the file (size limit is on **decoded** bytes; read `filesFileSizeBytes` from `files_guide`)

   The upload result already includes `downloadUrl`. Use `files_get_url` later if that URL expired.

6. **Extract fields** from the PDF/body: vendor name, invoice number, issue date, due date, currency, total, line items if present, PO number if present.

7. **Queue the structured record.** `storage_json_upsert`:
   - `category`: `invoices`
   - `key`: `queue/{vendor-slug}/{invoice-number}`
   - `data` (JSON string), for example:

```json
{
  "status": "pending_review",
  "vendorSlug": "acme-supplies",
  "vendorName": "Acme Supplies",
  "invoiceNumber": "INV-1042",
  "issueDate": "2026-09-01",
  "dueDate": "2026-09-30",
  "currency": "USD",
  "total": 1840.5,
  "poNumber": null,
  "source": {
    "inboxMessageId": "MESSAGE_ID",
    "inboxEmail": "YOUR_INVOICE_INBOX_EMAIL"
  },
  "files": {
    "category": "invoices",
    "key": "originals/acme-supplies/INV-1042"
  },
  "memory": {
    "category": "invoices",
    "noteKey": "notes/acme-supplies/INV-1042",
    "vendorKey": "vendors/acme-supplies"
  },
  "handoffTo": "ap-reviewer"
}
```

   Use `storage_json_get` with `allowEmpty: true` first if you need to detect a duplicate: an absent key returns `{}` plus `_meta["io.ainoflow/found"] = false`.

8. **Vendor + policy notes.** `memory_search` in category `invoices` with `q` = vendor name (optionally `prefix`: `vendors/`). Hits do not include the body — `memory_read` the chosen key.

   - If no vendor page exists, `memory_write` `invoices` / `vendors/{vendor-slug}` with frontmatter and `[[wiki-links]]`:

```markdown
---
status: active
type: vendor
---
# Acme Supplies

AP policy: [[policies/ap-intake]]

Latest invoice: [[notes/acme-supplies/INV-1042]]
```

   - `memory_write` `invoices` / `notes/{vendor-slug}/{invoice-number}` linking `[[vendors/{vendor-slug}]]` and `[[policies/ap-intake]]`.
   - Optional: `memory_context` on the vendor key to confirm the link neighborhood.

9. **Do not delete the Inbox message** after a successful queue. Leave it handled so a retry can see `isHandled: true`. Use `inbox_unhandle_message` only if intake failed after handle and must be retried.

## AP reviewer agent

1. Call the same guides.
2. `storage_json_list_keys` on `invoices` with `prefix`: `queue/` and `where`: `{ "status": "pending_review" }` (top-level field equality only).
3. `storage_json_get` the key. Follow `files.key` with `files_get_metadata` / `files_get_url`. Follow `memory.noteKey` / `memory.vendorKey` with `memory_read` (raw markdown) or `memory_search`.
4. After approve/reject, `storage_json_patch` the queue key with a JSON Merge Patch string, e.g. `{"status":"approved","reviewedBy":"ap-reviewer"}`. A `null` field removes that field. Patch creates the document if the key is empty — do not patch a wrong key.
5. `memory_edit` the vendor or note page (`append` or `str_replace`) to record the decision. `str_replace` requires `oldText` to occur exactly once.

## Multi-agent handoff

| Who | Writes | Reads |
| --- | --- | --- |
| **Agent A — intake** | Inbox handle; Files `invoices` / `originals/...`; Storage `invoices` / `queue/...` with `status: pending_review`; Memory vendor + note pages with `[[wiki-links]]` | Inbox unhandled list; existing Memory vendor pages |
| **Agent B — AP reviewer** | Storage patch (`status`, reviewer); Memory decision append | Storage queue (prefix + `where`); Files original; Memory vendor/policy/note |

Both agents must use the **same API key** (same scope/project) and the **same category names**. Tell the reviewer the Storage key (`queue/{vendor-slug}/{invoice-number}`) or have them list `pending_review`. That key is the handoff token.

## Failures (public codes)

- Inbox: `NOT_FOUND` if `messageId` / `attachmentId` is outside the scope.
- Files: `CONFLICT` if `files_upload` reuses a taken `key`; `ITEM_TOO_LARGE` if decoded bytes exceed the guide limit.
- Storage / Memory: `ITEM_TOO_LARGE`, `SCHEMA_VALIDATION_FAILED`, `CONFLICT`, `PRECONDITION_FAILED`, `NOT_FOUND`, `RATE_LIMITED`. Pass `ifMatch` / `expectedVersion` on upserts when two agents might write the same key.

Do not invent extra Inbox or Files tools. Published Inbox MCP tools: `inbox_guide`, `inbox_messages`, `inbox_attachment_url`, `inbox_handle_message`, `inbox_unhandle_message`, `inbox_delete_message`. This skill uses handle / unhandle / attachment URL — not `inbox_delete_message` on a successful queue.

Copy-paste try-it prompt: [prompts/invoice-intake.md](../../prompts/invoice-intake.md).
