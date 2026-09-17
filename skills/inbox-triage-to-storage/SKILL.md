---
name: inbox-triage-to-storage
description: Use when inbound email must be classified into support, billing, or sales Storage queues and handed to specialist agents.
---

# Inbox triage → Storage

A triage agent lists an Ainoflow Inbox, classifies each unhandled message, writes a queue record in Storage, optionally logs the decision in Memory, then marks the message handled. Specialist agents (support, billing, sales) pull only their prefix.

Public contract: [Inbox](https://www.ainoflow.io/docs/mcp/inbox), [Storage](https://www.ainoflow.io/docs/mcp/storage), [Memory](https://www.ainoflow.io/docs/mcp/memory). Start with `inbox_guide`, `storage_guide`, and `memory_guide`.

This skill does **not** need Shield or Guard. Stay on Inbox + Storage + Memory.

## Shared addresses

| Surface | Category | Key convention |
| --- | --- | --- |
| Inbox | hook email | `YOUR_TRIAGE_INBOX_EMAIL` |
| Storage | `inbox-triage` | `{lane}/{yyyy-mm-dd}/{message-id}` where `lane` is `support`, `billing`, or `sales` |
| Memory | `inbox-triage` | `log/{yyyy-mm-dd}` (daily decision log), `playbooks/{lane}` |

Lanes are Storage key prefixes, not separate categories. Specialists filter with `prefix`: `support/`, `billing/`, or `sales/`.

If a message has attachments the specialist will need later, the triage agent may also `files_upload` into category `inbox-triage` with key `attachments/{message-id}/{file-name}` after `inbox_attachment_url`. Keep that pointer on the Storage record. See [Files](https://www.ainoflow.io/docs/mcp/files).

## Classification rules (keep them boring)

Read `subject` + `bodyText` (and attachment names). Assign exactly one lane:

| Lane | Signals |
| --- | --- |
| `billing` | invoice, payment, refund, receipt, PO, wire, tax ID |
| `sales` | demo, pricing, quote, trial, partnership, “who is the right person” |
| `support` | default for how-to, outage, bug, access, “it doesn’t work” |

If two lanes fit, prefer `billing` over `sales` over `support`, and set `"needsHuman": true` on the JSON.

## Steps (triage agent)

1. **Guides.** `inbox_guide`, `storage_guide`, `memory_guide` (and `files_guide` if you will copy attachments).

2. **Count, then page.** `inbox_messages` with `email`: `YOUR_TRIAGE_INBOX_EMAIL`, `isHandled`: `false`, `limit`: `0` to read `totalCount` without bodies. Then page with `limit`: `5`–`10` (`sortBy`: `ReceivedAt`, `sortOrder`: `Desc` unless you want oldest-first `Asc`).

   Do not send `page` and `offset` together. Wrong-typed filters are refused (`-32602`); they are not dropped.

3. **For each message:**
   1. Classify the lane.
   2. `storage_json_upsert` category `inbox-triage`, key `{lane}/{yyyy-mm-dd}/{message-id}`:

```json
{
  "status": "open",
  "lane": "support",
  "needsHuman": false,
  "inbox": {
    "email": "YOUR_TRIAGE_INBOX_EMAIL",
    "messageId": "MESSAGE_ID",
    "fromAddress": "user@example.com",
    "subject": "Cannot reset password",
    "receivedAt": "2026-09-04T10:15:00Z",
    "hasAttachments": false
  },
  "excerpt": "First 400 characters of bodyText…",
  "attachments": [],
  "handoffTo": "support-specialist"
}
```

   3. Optional Files copy: `inbox_attachment_url` → download → `files_upload` → push `{ category, key }` into `attachments`.
   4. `inbox_handle_message` with `messageId` so the next triage pass skips it. Idempotent.
   5. Do **not** `inbox_delete_message` during triage. Deleting drops the only copy of the mail. Specialists can delete later if policy says so.

4. **Decision log (optional but useful for audits).** `memory_read` `inbox-triage` / `log/{yyyy-mm-dd}`. If `NOT_FOUND`, `memory_write` a new daily page. Otherwise `memory_edit` `operation`: `append` a bullet: time, lane, subject, Storage key. Link playbooks: `[[playbooks/support]]`.

5. **Playbooks.** Once, `memory_write` short pages at `playbooks/support`, `playbooks/billing`, `playbooks/sales` so specialists can `memory_search` / `memory_read` the same guidance.

## Steps (specialist agent)

1. Same API key. Call guides.
2. `storage_json_list_keys` category `inbox-triage`, `prefix`: `{your-lane}/`, `where`: `{ "status": "open" }`.
3. `storage_json_get` each key. For the full mail, call `inbox_handle_message` with the stored `inbox.messageId` (idempotent on an already-handled message; returns body and attachment metadata). Use `excerpt` + Files pointers when you do not need the original.
4. Close the item with `storage_json_patch`: `{"status":"closed","resolvedBy":"support-specialist"}`.
5. If triage was wrong, patch `{"lane":"billing","status":"open"}` **and** `storage_json_upsert` a new key under the correct prefix, then `storage_json_delete` the old key (or leave the old key with `"status":"moved"`). Prefix filters will not see a record that still lives under `support/` even if `lane` changed — **key prefix is the queue**.

## Multi-agent handoff

| Who | Writes | Reads |
| --- | --- | --- |
| **Agent A — triage** | Storage `inbox-triage` / `{lane}/{date}/{message-id}`; optional Files attachments; Memory daily log + playbooks; Inbox handle | Inbox `isHandled: false` |
| **Agent B — support** | Storage patch/delete on `support/…` | `prefix: support/` + `status: open` |
| **Agent C — billing** | Same on `billing/…` | `prefix: billing/` |
| **Agent D — sales** | Same on `sales/…` | `prefix: sales/` |

Handoff token: Storage key `{lane}/{yyyy-mm-dd}/{message-id}` in category `inbox-triage`. Specialists do not need the Inbox hook if the record plus Files copies are complete.

Copy-paste try-it prompt: [prompts/inbox-triage-to-storage.md](../../prompts/inbox-triage-to-storage.md).

## Failures (public codes)

- Inbox tools on this surface declare `NOT_FOUND` for unknown ids. Monthly receive limits are reported by `inbox_guide` (`inboxMessagesMonthly`) but are charged at ingest — tools here do not return `RATE_LIMITED`.
- Storage: `RATE_LIMITED` when the stored-document allowance is full; `SCHEMA_VALIDATION_FAILED` for over-long category/key.
- Re-queue a mistaken handle with `inbox_unhandle_message` (`messageId` only).
