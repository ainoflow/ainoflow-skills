# Smoke-test skills against public MCP

Use a real Ainoflow API key from your own account. These steps call only the public MCP tools on [Memory](https://www.ainoflow.io/docs/mcp/memory), [Storage](https://www.ainoflow.io/docs/mcp/storage), [Files](https://www.ainoflow.io/docs/mcp/files), and [Inbox](https://www.ainoflow.io/docs/mcp/inbox). Start every surface with its `*_guide` tool. If a guide and a docs page disagree, the guide is current.

Do not commit the key. Copy [`.env.example`](.env.example) to `.env` and keep `YOUR_API_KEY` / hook-email placeholders in git.

| Smoke | Skill | Surfaces | Required? |
| --- | --- | --- | --- |
| A | `research-notes-memory` | Memory, Storage | Yes |
| B | `invoice-intake` or `inbox-triage-to-storage` | Inbox + Storage (+ Files/Memory) | Only if you have an Inbox hook email |
| C | `expense-tracking-handoff` | Files, Storage, Memory | Yes |

Install the skill folder under `.cursor/skills/<name>/SKILL.md` (or `~/.cursor/skills/`) before the run. Sample prompts: [`prompts/`](prompts/).

All smoke writes use keys that contain `smoke-` (or the Inbox message id on Smoke B). Delete **only** those keys in [Cleanup](#cleanup).

## Prerequisites

1. Connect the four category-agnostic servers (`mcp.ainoflow.io`, not `api.ainoflow.io`) with `Authorization: Bearer YOUR_API_KEY`. Snippets: [README — Connect MCP](README.md#connect-mcp).
2. Confirm tools exist: `memory_guide`, `storage_guide`, `files_guide`, `inbox_guide`.
3. Same API key for every agent in a handoff (same scope/project).

`storage_json_*` `data` and `patch` arguments are **JSON strings** (object or array, never a bare string). `files_upload` `content` is **base64**. Inbox tools take the hook `email`, not a category.

## Smoke A — `research-notes-memory` (no Inbox)

Skill: [`skills/research-notes-memory`](skills/research-notes-memory/SKILL.md). Prompt: [`prompts/research-notes-memory.md`](prompts/research-notes-memory.md).

Namespace: category `research`. Brief slug: `smoke-q3-pricing`.

### A1. Researcher — guides, brief, session

1. `memory_guide` — no arguments.
2. `storage_guide` — no arguments.
3. `memory_write`

```text
category: research
key: briefs/smoke-q3-pricing
content:
---
status: active
type: brief
---
# Smoke Q3 pricing

Audience: writer agent. Due: smoke test.
Outline: [[outlines/smoke-q3-pricing]]
```

4. `storage_json_upsert`

```text
category: research
key: sessions/smoke-q3-pricing
data: {"status":"researching","briefKey":"briefs/smoke-q3-pricing","updatedNoteKeys":[],"outlineKey":"outlines/smoke-q3-pricing","handoffTo":"writer","readyForDraft":false}
```

### A2. Researcher — note + outline + ready

5. `memory_search` — `category`: `research`, `q`: `pricing packaging`, `prefix`: `notes/smoke-q3-pricing/` (expect zero hits the first time).
6. `memory_write`

```text
category: research
key: notes/smoke-q3-pricing/packaging
content:
---
status: active
type: note
brief: smoke-q3-pricing
tags:
  - pricing
  - smoke
---
# Packaging note

Tied to [[briefs/smoke-q3-pricing]]. Competitors bundle usage + support.
```

7. `memory_write` `research` / `outlines/smoke-q3-pricing` listing `[[notes/smoke-q3-pricing/packaging]]`.
8. `storage_json_patch`

```text
category: research
key: sessions/smoke-q3-pricing
patch: {"status":"ready_for_draft","readyForDraft":true,"updatedNoteKeys":["notes/smoke-q3-pricing/packaging"]}
```

### A3. Writer — handoff read path

9. `storage_json_get` `research` / `sessions/smoke-q3-pricing` (or `storage_json_list_keys` with `prefix`: `sessions/`, `where`: `{ "readyForDraft": true }`).
10. `memory_read` `research` / `briefs/smoke-q3-pricing` and `research` / `outlines/smoke-q3-pricing`.
11. `memory_search` `category`: `research`, `q`: `pricing packaging`, `prefix`: `notes/smoke-q3-pricing/`, `sort`: `blended`. Then `memory_read` the hit key. Hits have **no** body.
12. Optional: `memory_context` `category`: `research`, `key`: `briefs/smoke-q3-pricing`, `depth`: `2`.

### A — pass

- [ ] Guides returned without error.
- [ ] `memory_read` of `briefs/smoke-q3-pricing` returns the brief markdown.
- [ ] `memory_search` finds `notes/smoke-q3-pricing/packaging` (snippet/title, not the full body).
- [ ] `storage_json_get` of `sessions/smoke-q3-pricing` shows `readyForDraft: true`.
- [ ] Writer can draft from those keys without any chat transcript from the researcher.

## Smoke B — Inbox (optional)

Skip this smoke unless you have a real Inbox hook email (`AINOFLOW_INVOICE_INBOX_EMAIL` or `AINOFLOW_TRIAGE_INBOX_EMAIL` in `.env`). Creating an inbox is REST-only ([Inbox — beyond the tools](https://www.ainoflow.io/docs/mcp/inbox)); MCP cannot provision one.

If both placeholders are still `YOUR_*_INBOX_EMAIL` or empty: **skip, mark N/A**, continue to Smoke C. Do not invent a hook domain.

Pick **one** skill:

### B-invoice — `invoice-intake`

Skill: [`skills/invoice-intake`](skills/invoice-intake/SKILL.md). Prompt: [`prompts/invoice-intake.md`](prompts/invoice-intake.md).

1. `inbox_guide`, `files_guide`, `storage_guide`, `memory_guide`.
2. `inbox_messages` — `email`: your hook, `isHandled`: `false`, `limit`: `0` (read `totalCount`). Then `limit`: `5`.
3. If `totalCount` is `0`: send any email with a small PDF (or skip attachments) to the hook, wait, list again. If you cannot send mail, **skip the rest of B** (N/A).
4. `inbox_handle_message` `messageId` of an invoice-like message (idempotent; returns attachments).
5. If there is an attachment: use `downloadUrl` or `inbox_attachment_url` `attachmentId` (URL lasts 1 hour). Download locally. Inbox tools never return raw bytes.
6. `files_upload` `category`: `invoices`, `key`: `originals/smoke-acme/SMOKE-001`, `fileName`: original name, `contentType`: `application/pdf` (or `text/plain` for a stub), `content`: base64.
7. `storage_json_upsert` `invoices` / `queue/smoke-acme/SMOKE-001` with `"status":"pending_review"`, `files.key` pointing at that Files key, `handoffTo`: `ap-reviewer`.
8. `memory_write` `invoices` / `vendors/smoke-acme` and `invoices` / `notes/smoke-acme/SMOKE-001` with `[[wiki-links]]`.
9. Reviewer: `storage_json_list_keys` `invoices`, `prefix`: `queue/`, `where`: `{ "status": "pending_review" }`. Then `storage_json_get`, `files_get_metadata` / `files_get_url`, `memory_read`.
10. Do **not** `inbox_delete_message` after a successful queue. Leave the message handled.

### B-triage — `inbox-triage-to-storage`

Skill: [`skills/inbox-triage-to-storage`](skills/inbox-triage-to-storage/SKILL.md). Prompt: [`prompts/inbox-triage-to-storage.md`](prompts/inbox-triage-to-storage.md).

1. `inbox_guide`, `storage_guide`, `memory_guide`.
2. `inbox_messages` — `email`: your triage hook, `isHandled`: `false`, `limit`: `0`, then page `limit`: `5` (`sortBy`: `ReceivedAt`, `sortOrder`: `Desc`). Do not send `page` and `offset` together.
3. Classify one message (`support` / `billing` / `sales`).
4. `storage_json_upsert` `inbox-triage` / `{lane}/{yyyy-mm-dd}/{message-id}` with `"status":"open"` and the inbox metadata.
5. `inbox_handle_message` `messageId`. Do **not** `inbox_delete_message`.
6. Specialist: `storage_json_list_keys` `inbox-triage`, `prefix`: `{lane}/`, `where`: `{ "status": "open" }`.

### B — pass or skip

- [ ] **N/A (skipped)** — no Inbox hook email, **or** hook exists but `inbox_messages` `totalCount` is 0 and you cannot send mail.
- [ ] **Pass** — one Storage queue key exists (`invoices` / `queue/smoke-acme/SMOKE-001` **or** `inbox-triage` / `{lane}/{date}/{message-id}`), the message is handled, and a second agent can list it with `prefix` + `where`.

## Smoke C — `expense-tracking-handoff`

Skill: [`skills/expense-tracking-handoff`](skills/expense-tracking-handoff/SKILL.md). Prompt: [`prompts/expense-tracking-handoff.md`](prompts/expense-tracking-handoff.md).

Tiny Files stub (decoded text `SMOKE receipt stub`, `contentType`: `text/plain`):

```text
content: U01PS0UgcmVjZWlwdCBzdHVi
```

A one-page PDF stub is fine too — still base64, still a smoke key.

### C1. Employee agent

1. `storage_guide`, `files_guide`, `memory_guide`.
2. `files_upload`

```text
category: expenses
key: receipts/2026-09/smoke-tester/smoke-berlin/1-stub
fileName: smoke-receipt.txt
contentType: text/plain
content: U01PS0UgcmVjZWlwdCBzdHVi
```

On `CONFLICT`, treat the receipt as already stored if `files_get_metadata` matches.

3. `storage_json_upsert`

```text
category: expenses
key: reports/2026-09/smoke-tester/smoke-berlin
data: {"status":"submitted","employeeSlug":"smoke-tester","employeeName":"Smoke Tester","reportId":"smoke-berlin","period":"2026-09","currency":"EUR","total":1.0,"lines":[{"n":1,"date":"2026-09-17","merchant":"Smoke Cafe","category":"meals","amount":1.0,"purpose":"Public MCP smoke test","receipt":{"category":"expenses","key":"receipts/2026-09/smoke-tester/smoke-berlin/1-stub"}}],"memoryKey":"summaries/2026-09/smoke-tester/smoke-berlin","handoffTo":"finance"}
```

4. `memory_write` `expenses` / `summaries/2026-09/smoke-tester/smoke-berlin` with frontmatter `status: submitted` and a `[[policies/expense-policy]]` link (write a one-line policy page first if `memory_read` is `NOT_FOUND`).

### C2. Finance agent read path

5. `storage_json_list_keys` `expenses`, `prefix`: `reports/`, `where`: `{ "status": "submitted" }` — the smoke report key must appear.
6. `storage_json_get` `expenses` / `reports/2026-09/smoke-tester/smoke-berlin`.
7. For `lines[0].receipt`: `files_get_metadata` then `files_get_url` (do not assume the upload URL is still valid).
8. `memory_read` `expenses` / `summaries/2026-09/smoke-tester/smoke-berlin`.
9. Optional: `storage_json_patch` `{"status":"approved","reviewedBy":"finance"}`.

### C — pass

- [ ] `files_get_metadata` finds `receipts/2026-09/smoke-tester/smoke-berlin/1-stub`.
- [ ] Storage report `status` is `submitted` (or `approved` if you patched).
- [ ] Finance read path works from the Storage key alone — no receipt emailed in chat.

## Pass / fail checklist

| Check | A | B | C |
| --- | --- | --- | --- |
| Guides first (`*_guide`) | | N/A ok | |
| No invented tool names | | | |
| Handoff key exists and is listable | | skip ok | |
| Second-role read path (get / search / files URL) | | skip ok | |
| Cleanup ran on smoke keys only | | skip ok | |

**Pass:** A and C pass; B passes **or** is N/A.  
**Fail:** any required smoke cannot create or re-read its handoff key with the public tools above.

## Cleanup

Delete **only** the smoke addresses below. Do not `storage_json_delete` / `memory_delete` / `files_delete` other keys in these categories. Inbox: do **not** `inbox_delete_message` unless you created the mail only for this test.

```text
memory_delete     category: research      key: briefs/smoke-q3-pricing
memory_delete     category: research      key: notes/smoke-q3-pricing/packaging
memory_delete     category: research      key: outlines/smoke-q3-pricing
storage_json_delete category: research    key: sessions/smoke-q3-pricing

files_delete      category: expenses      key: receipts/2026-09/smoke-tester/smoke-berlin/1-stub
storage_json_delete category: expenses    key: reports/2026-09/smoke-tester/smoke-berlin
memory_delete     category: expenses      key: summaries/2026-09/smoke-tester/smoke-berlin
memory_delete     category: expenses      key: policies/expense-policy
  (only if you created the policy page for this smoke)

files_delete      category: invoices      key: originals/smoke-acme/SMOKE-001
storage_json_delete category: invoices    key: queue/smoke-acme/SMOKE-001
memory_delete     category: invoices      key: vendors/smoke-acme
memory_delete     category: invoices      key: notes/smoke-acme/SMOKE-001

storage_json_delete category: inbox-triage key: {lane}/{yyyy-mm-dd}/{message-id}
memory_delete     category: inbox-triage  key: log/{yyyy-mm-dd}
  (only the daily log you appended for this smoke)
```

`NOT_FOUND` on a delete means that key is already gone — still a clean pass for that row.

## Public tool inventory (do not invent names)

Verified against [www.ainoflow.io/docs/mcp](https://www.ainoflow.io/docs/mcp) and the Memory / Storage / Files / Inbox pages (also published as `https://docs.ainoflow.io/llms.mdx/docs/mcp/{memory,storage,files,inbox}/content.md`).

- Memory: `memory_guide`, `memory_write`, `memory_read`, `memory_edit`, `memory_move`, `memory_delete`, `memory_list`, `memory_context`, `memory_search`
- Storage: `storage_guide`, `storage_json_create`, `storage_json_get`, `storage_json_upsert`, `storage_json_patch`, `storage_json_delete`, `storage_json_list_keys`, `storage_json_get_metadata`, `storage_json_list_categories`
- Files: `files_guide`, `files_upload`, `files_list`, `files_get_metadata`, `files_get_url`, `files_delete`, `files_list_categories`
- Inbox: `inbox_guide`, `inbox_messages`, `inbox_attachment_url`, `inbox_handle_message`, `inbox_unhandle_message`, `inbox_delete_message`
