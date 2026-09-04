---
name: expense-tracking-handoff
description: Use when an employee agent files an expense report with receipts and a finance agent must review the same Storage state, Files, and Memory summary.
---

# Expense tracking handoff

Employee agent (submitter) writes one expense report as Storage JSON, stores receipt binaries in Files, and leaves a Memory summary. Finance agent (approver) reads those exact addresses. Both sides share one Ainoflow API key and the `expenses` namespace.

Public contract: [Storage](https://www.ainoflow.io/docs/mcp/storage), [Files](https://www.ainoflow.io/docs/mcp/files), [Memory](https://www.ainoflow.io/docs/mcp/memory). Start with `storage_guide`, `files_guide`, and `memory_guide`.

## Shared addresses

| Surface | Category | Key convention |
| --- | --- | --- |
| Storage | `expenses` | `reports/{yyyy-mm}/{employee-slug}/{report-id}` |
| Files | `expenses` | `receipts/{yyyy-mm}/{employee-slug}/{report-id}/{n}-{label}` |
| Memory | `expenses` | `summaries/{yyyy-mm}/{employee-slug}/{report-id}` and `policies/expense-policy` |

`report-id` is a stable id the employee agent chooses (for example `trip-berlin-2026-09`). Do not use `storage_json_create` for the report itself — an auto-generated UUID is harder for finance to guess. Create is fine for ephemeral scratch keys if you then write the UUID into the report.

## Steps (employee agent)

1. **Guides.** `storage_guide`, `files_guide`, `memory_guide`.

2. **Policy (optional).** `memory_search` category `expenses`, `q`: `receipt required mileage per diem`, `prefix`: `policies/`. Then `memory_read` `expenses` / `policies/expense-policy`. If missing, ask the user for the policy text and `memory_write` it once so later reports can `[[policies/expense-policy]]`.

3. **Collect line items** from the user: date, merchant, category (meals, travel, lodging, other), amount, currency, business purpose. One receipt file per line when a receipt exists.

4. **Upload receipts.** For each file, `files_upload`:
   - `category`: `expenses`
   - `key`: `receipts/{yyyy-mm}/{employee-slug}/{report-id}/{n}-{label}` (e.g. `receipts/2026-09/alex-nguyen/trip-berlin-2026-09/1-hotel`)
   - `fileName`, `contentType` (`image/jpeg`, `application/pdf`, …), `content` (base64)
   - Supply `key` so a retry of the same receipt fails with `CONFLICT` instead of creating a second UUID file

   Keep the returned `category`, `key`, `fileName`, and `size` for the report JSON. Size limit is decoded bytes (`filesFileSizeBytes` on the guide).

5. **Write workflow state.** `storage_json_upsert`:
   - `category`: `expenses`
   - `key`: `reports/{yyyy-mm}/{employee-slug}/{report-id}`
   - `data`:

```json
{
  "status": "submitted",
  "employeeSlug": "alex-nguyen",
  "employeeName": "Alex Nguyen",
  "reportId": "trip-berlin-2026-09",
  "period": "2026-09",
  "currency": "EUR",
  "total": 612.4,
  "lines": [
    {
      "n": 1,
      "date": "2026-09-02",
      "merchant": "Hotel Spree",
      "category": "lodging",
      "amount": 480.0,
      "purpose": "Customer workshop",
      "receipt": {
        "category": "expenses",
        "key": "receipts/2026-09/alex-nguyen/trip-berlin-2026-09/1-hotel"
      }
    }
  ],
  "memoryKey": "summaries/2026-09/alex-nguyen/trip-berlin-2026-09",
  "handoffTo": "finance"
}
```

6. **Memory summary** (searchable; not a substitute for the JSON). `memory_write` category `expenses`, key `summaries/{yyyy-mm}/{employee-slug}/{report-id}`:

```markdown
---
status: submitted
type: expense-summary
employee: alex-nguyen
period: 2026-09
---
# Trip Berlin 2026-09

Submitted by Alex Nguyen. Policy: [[policies/expense-policy]]

Storage report key: `reports/2026-09/alex-nguyen/trip-berlin-2026-09` in category `expenses`.
Total EUR 612.40. Receipts live under Files `expenses` / `receipts/2026-09/alex-nguyen/trip-berlin-2026-09/`.
```

7. Tell the user the **Storage key**. That is what finance needs.

## Steps (finance agent)

1. Call the same guides.
2. Discover work:
   - `storage_json_list_keys` category `expenses`, `prefix`: `reports/`, `where`: `{ "status": "submitted" }`, or
   - `memory_search` category `expenses`, `q`: employee or trip name, `frontmatter`: `{ "status": "submitted" }` (containment filter; hits have no body).
3. `storage_json_get` the report key. For each `lines[].receipt`, `files_get_metadata` then `files_get_url` (or use a still-valid upload URL).
4. `memory_read` the `memoryKey` and `policies/expense-policy`.
5. Approve, reject, or request changes with `storage_json_patch` (JSON Merge Patch). Examples:
   - Approve: `{"status":"approved","reviewedBy":"finance","reviewedAt":"2026-09-04T12:00:00Z"}`
   - Changes: `{"status":"changes_requested","reviewNote":"Need itemized hotel folio"}`
6. `memory_edit` the summary (`operation`: `append` or `str_replace`) and keep frontmatter `status` in sync via a full `memory_write` if you need `memory_search` filters to change. Frontmatter is parsed at write time.

Optional TTL: if the report is a draft, `storage_json_upsert` may set `expiresMs` or `expiresAt` (mutually exclusive; expiry must be in the future). Do not expire a submitted report that finance still needs.

## Multi-agent handoff

| Who | Writes | Reads |
| --- | --- | --- |
| **Agent A — employee** | Files receipts; Storage report `status: submitted`; Memory summary with `[[policies/expense-policy]]` | Policy Memory page |
| **Agent B — finance** | Storage patch (`status`, reviewer, notes); Memory summary update | Storage `reports/` + `where status=submitted`; Files receipts; Memory summary + policy |

Handoff token: Storage key `reports/{yyyy-mm}/{employee-slug}/{report-id}` in category `expenses`. Same API key, same categories. The employee does not email receipts to finance — finance pulls Files from the pointers on the JSON.

## Failures (public codes)

- Files `CONFLICT` on a reused upload key — treat as “receipt already stored” and keep going if metadata matches.
- Storage / Memory: `ITEM_TOO_LARGE`, `NOT_FOUND`, `PRECONDITION_FAILED` (stale `ifMatch`), `RATE_LIMITED` when the scope is full of documents (free a key with `storage_json_delete` / `memory_delete`; waiting does not reset the stored-document allowance).
- `storage_json_patch` has **no** ETag/TTL. Use `storage_json_upsert` with `ifMatch` when two reviewers might collide.
