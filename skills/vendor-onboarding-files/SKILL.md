---
name: vendor-onboarding-files
description: Use when procurement must collect vendor contracts and packets in Files, track a Storage checklist, and hand Memory profiles to compliance or AP.
---

# Vendor onboarding (Files + Storage + Memory)

Procurement agent collects onboarding documents (contract, W-9 / tax form, insurance certificate, bank letter). Files holds the binaries. Storage holds the checklist and status. Memory holds the vendor profile with `[[wiki-links]]` so compliance or AP can recall “who is this vendor?” without knowing the Files keys.

Public contract: [Files](https://www.ainoflow.io/docs/mcp/files), [Storage](https://www.ainoflow.io/docs/mcp/storage), [Memory](https://www.ainoflow.io/docs/mcp/memory). Start with `files_guide`, `storage_guide`, and `memory_guide`.

## Shared addresses

| Surface | Category | Key convention |
| --- | --- | --- |
| Files | `vendors` | `{vendor-slug}/{doc-type}` e.g. `acme-supplies/contract`, `acme-supplies/tax-form` |
| Storage | `vendors` | `onboarding/{vendor-slug}` |
| Memory | `vendors` | `profiles/{vendor-slug}`, `playbooks/onboarding` |

`doc-type` values this skill uses: `contract`, `tax-form`, `insurance`, `bank`, `other-{label}`.

Files `files_upload` with a supplied `key` is create-only: a second upload to the same address returns `CONFLICT`. For a replacement, `files_delete` then `files_upload`, or choose a versioned key `{vendor-slug}/{doc-type}/v{n}`.

## Steps (procurement agent)

1. **Guides.** `files_guide`, `storage_guide`, `memory_guide`.

2. **Playbook once.** `memory_read` `vendors` / `playbooks/onboarding`. If missing, `memory_write` the required doc list and who reviews it (compliance vs AP).

3. **Create or load the checklist.** `storage_json_get` `vendors` / `onboarding/{vendor-slug}` with `allowEmpty: true`. If absent, `storage_json_upsert`:

```json
{
  "status": "collecting",
  "vendorSlug": "acme-supplies",
  "vendorName": "Acme Supplies",
  "required": ["contract", "tax-form", "insurance"],
  "received": {},
  "memoryProfileKey": "profiles/acme-supplies",
  "handoffTo": "compliance"
}
```

4. **Store each document.** `files_upload`:
   - `category`: `vendors`
   - `key`: `{vendor-slug}/{doc-type}`
   - `fileName`, `contentType`, `content` (base64)
   - Read `filesFileSizeBytes` from the guide; the limit applies to decoded bytes

   Then `storage_json_patch` `onboarding/{vendor-slug}` merging `received`:

```json
{
  "received": {
    "contract": {
      "category": "vendors",
      "key": "acme-supplies/contract",
      "fileName": "msa.pdf",
      "uploadedAt": "2026-09-04T15:00:00Z"
    }
  }
}
```

   JSON Merge Patch replaces the `received` object you send — include **all** received docs so you do not wipe earlier pointers. Safer pattern: `storage_json_get`, merge locally, `storage_json_upsert` with `ifMatch` from `storage_json_get_metadata`.

5. **Vendor profile.** `memory_write` `vendors` / `profiles/{vendor-slug}`:

```markdown
---
status: collecting
type: vendor-profile
---
# Acme Supplies

Onboarding playbook: [[playbooks/onboarding]]

Storage checklist key: `onboarding/acme-supplies` (category `vendors`).
Files live under category `vendors`, prefix `acme-supplies/`.
```

6. **When required docs are present.** `storage_json_patch` `{"status":"ready_for_review","handoffTo":"compliance"}`. Update Memory frontmatter `status: ready_for_review` with `memory_write` (search filters use the parsed frontmatter).

7. **List what you stored.** `files_list` category `vendors`, `prefix`: `{vendor-slug}/` to confirm keys before handing off.

## Steps (compliance or AP agent)

1. Same API key. Guides first.
2. Find work:
   - `storage_json_list_keys` category `vendors`, `prefix`: `onboarding/`, `where`: `{ "status": "ready_for_review" }`, or
   - `memory_search` category `vendors`, `q`: vendor name, `prefix`: `profiles/`, `frontmatter`: `{ "status": "ready_for_review" }`.
3. `storage_json_get` the checklist. For each `received.*` pointer, `files_get_metadata` and `files_get_url` (do not assume an old upload URL is still valid).
4. `memory_read` the profile and `playbooks/onboarding`.
5. Approve or request docs:
   - Approve: `storage_json_patch` `{"status":"approved","reviewedBy":"compliance"}` and `memory_write` / `memory_edit` the profile (`status: active`, link `[[profiles/...]]` from any AP note).
   - Gaps: `{"status":"collecting","reviewNote":"Insurance expired"}` so procurement resumes.
6. Do not `files_delete` approved packets unless policy says so. Deleting is permanent on the Files surface.

## Multi-agent handoff

| Who | Writes | Reads |
| --- | --- | --- |
| **Agent A — procurement** | Files `{vendor-slug}/{doc-type}`; Storage `onboarding/{vendor-slug}`; Memory `profiles/{vendor-slug}` + playbook | Playbook; existing checklist |
| **Agent B — compliance / AP** | Storage status; Memory profile status / comments | Storage `onboarding/` + `ready_for_review`; Files via pointers; Memory profile |

Handoff token: Storage key `onboarding/{vendor-slug}` in category `vendors`. Files keys are listed on that JSON. Memory is how a later agent finds the vendor by name when nobody pasted the Storage key.

## Failures (public codes)

- Files: `CONFLICT` on a taken upload key; `ITEM_TOO_LARGE` over the decoded size limit; `SCHEMA_VALIDATION_FAILED` for illegal key segments (empty, `.`, `..`, trailing `/`).
- Storage / Memory: `NOT_FOUND`, `PRECONDITION_FAILED`, `RATE_LIMITED` (stored/live document allowance is a count, not a monthly reset).
- `files_get_url` when you need a fresh download. Upload responses already include a `downloadUrl` for immediate use.
