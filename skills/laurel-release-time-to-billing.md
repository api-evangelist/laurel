---
name: Release Laurel time entries into a billing system
description: Pull review-ready time entries out of Laurel, release them to the firm's billing system, and handle corrections, unrelease and overrides.
api: openapi/laurel-time-openapi-original.json
operations:
  - CustomerEntryPublicController_findManyWithPaginationV3_v3
  - CustomerEntryPublicController_bulkRelease_v1
  - CustomerEntryPublicController_unrelease_v1
  - CustomerEntryPublicController_overrideOneEntry_v1
  - CustomerEntryPublicController_deleteOneEntry_v1
  - EntriesController_getAllReleasePendingEntries_v1
  - EntriesController_createReleasedEntry_v1
  - EntriesController_updateReleasedEntry_v1
  - EntriesController_deleteReleasedEntry_v1
---

# Release Laurel time entries into a billing system

Laurel captures and enriches time; the firm's billing system remains the system of record for
invoicing. This flow moves reviewed entries out of Laurel and keeps the two sides reconciled.
It spans the **Time Service** (read/release) and the **Ingestion Service** (write-back).

Authenticate first (see `laurel-authenticate.md`).

## Pull and release

1. **Read entries** — `CustomerEntryPublicController_findManyWithPaginationV3_v3`
   (`GET /api/v3/public/customers/{customerId}/entries`). This is the current paginated public
   read. Page with `page` / `pageSize`, and narrow with the date filters (`workDate`,
   `startedBefore`, `stoppedAfter`).

2. **Find what is waiting to go** — `EntriesController_getAllReleasePendingEntries_v1`
   (`GET /api/v1/customers/{customerId}/entries/release-pending`, Ingestion Service). This is the
   queue of entries the timekeeper has approved but that have not yet reached billing.

3. **Release in bulk** — `CustomerEntryPublicController_bulkRelease_v1`
   (`POST /api/v1/public/customers/{customerId}/entries/bulk-release`). Release marks the entries
   as handed off. Single-entry release also exists on the internal surface
   (`EntryController_releaseEntryDtoById_v2`), but prefer the public bulk operation.

4. **Write the released entry back** — `EntriesController_createReleasedEntry_v1`
   (`POST /api/v1/customers/{customerId}/users/{userId}/entries/import-released`). Use this to
   record in Laurel that the entry now exists in the billing system, keyed by your
   `{entryExternalId}`.

## Corrections

- **Amend a released entry** — `EntriesController_updateReleasedEntry_v1`
  (`PATCH /api/v1/customers/{customerId}/users/{userId}/entries/{entryExternalId}`).
- **Remove a released entry** — `EntriesController_deleteReleasedEntry_v1`.
- **Pull an entry back into Laurel** — `CustomerEntryPublicController_unrelease_v1`
  (`POST /api/v1/public/customers/{customerId}/users/{userId}/entries/{entryId}/unrelease`).
- **Override a value on an entry** — `CustomerEntryPublicController_overrideOneEntry_v1`
  (`PATCH /api/v1/public/customers/{customerId}/users/{userId}/entries/{entryId}/override`).
- **Delete an entry** — `CustomerEntryPublicController_deleteOneEntry_v1`.

## Unreleased entries from another source

If time also originates outside Laurel, push it in with
`EntriesController_importUnreleasedEntry_v1`
(`POST /api/v1/customers/{customerId}/entries/import-unreleased`). Bulk amend and delete by source
system with `EntriesController_updateUnreleasedEntriesBulk_v1` and
`EntriesController_deleteUnreleasedEntriesBulk_v1`
(`/api/v1/customers/{customerId}/entries/external/{sourceSystem}`).

## Rules

- **Use the idempotency key on unreleased imports.** `EntriesController_importUnreleasedEntry_v1`
  accepts an optional `idempotencyKey` field in the request body
  (schema `ImportUnreleasedEntriesRequestDto`). Set it to a stable value per logical import so a
  retry does not double-post time. This is the only idempotency control Laurel documents — it is a
  **body field, not an `Idempotency-Key` header**, it is not available on other operations, and no
  retention window is published.
- **Mind released vs unreleased ids.** Released-entry operations key off `{entryExternalId}` (your
  system's id); the public read/override/unrelease operations key off Laurel's `{entryId}`. They
  are not interchangeable.
- **Async writes.** Import and bulk operations frequently return `202 Accepted` — queued, not
  applied. Confirm by re-reading rather than trusting the response.
- **Overrides are consequential.** Override, unrelease and delete change billable records. Treat
  them as requiring human confirmation rather than autonomous agent action; the repo's
  `agentic-access/laurel-agentic-access.yml` classifies the override and stop operations
  safety-critical with human-in-the-loop required.
- No error envelope, rate limits or request-id header are documented. Retry only 5xx/429 with backoff.
