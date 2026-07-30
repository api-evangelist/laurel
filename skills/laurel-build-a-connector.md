---
name: Build a Laurel connector from a practice-management system
description: Seed Laurel with clients, code types, codes and initiatives from a firm's billing or practice-management system using the Ingestion Service batch imports.
api: openapi/laurel-ingestion-openapi-original.json
operations:
  - ClientsController_importClientBatchV1_v1
  - CodeTypesController_importCodeTypeV1_v1
  - CodesController_importCodeBatchV1_v1
  - InitiativesController_importInitiativesConsolidatedBatchV2_v2
  - InitiativesController_setInitiativeAccessV1_v1
  - InitiativesController_setInitiativeCodesV1_v1
  - EntryValidationRulesController_setOneForInitiative_v1
  - UsersController_updateUserDelegatesBatch_v1
---

# Build a Laurel connector from a practice-management system

A Laurel connector keeps the firm's reference data — clients, matters, billing codes and who may
bill to them — in sync from the practice-management or billing system into Laurel. Everything below
runs against the **Ingestion Service** (`https://api.laurel.ai/ingestion/`) and is customer-scoped.

Authenticate first (see `laurel-authenticate.md`).

## Load order

Reference data has dependencies. Import in this order or the later imports will not resolve:

1. **Clients** — `ClientsController_importClientBatchV1_v1`
   (`POST /api/v1/customers/{customerId}/clients/batch`). The single-record sibling is
   `ClientsController_importClientV1_v1`; prefer the batch form for a sync.

2. **Code types** — `CodeTypesController_importCodeTypeV1_v1`
   (`POST /api/v1/customers/{customerId}/code-types/import`). Code types are the categories
   (task, activity, phase) that codes hang off.

3. **Codes** — `CodesController_importCodeBatchV1_v1`
   (`POST /api/v1/customers/{customerId}/codes/batch`), single-record sibling
   `CodesController_importCodeV1_v1`.

4. **Initiatives (matters/engagements)** —
   `InitiativesController_importInitiativesConsolidatedBatchV2_v2`
   (`POST /api/v2/customers/{customerId}/initiatives/consolidated-batch`). This is the current
   consolidated batch import; the v1 batch operations
   (`InitiativesController_importInitiativesBatchV1_v1`,
   `InitiativesController_importInitiativesConsolidatedBatchV1_v1`) are marked deprecated.

## Then bind access and codes to each initiative

5. **Who may bill to a matter** — `InitiativesController_setInitiativeAccessV1_v1`
   (`POST /api/v1/customers/{customerId}/initiatives/{initiativeExternalId}/access`). The batch
   form is `InitiativesController_setInitiativeAccessBatchV1_v1`; where the firm runs an ethical-
   walls system, use `InitiativesController_setInitiativeAccessBatchWithWallsV1_v1`.

6. **Which codes are valid on a matter** — `InitiativesController_setInitiativeCodesV1_v1`
   (`POST /api/v1/customers/{customerId}/initiatives/{initiativeExternalId}/codes`); remove with
   `InitiativesController_deleteInitiativeCodesV1_v1`.

7. **Client-mandated billing rules** — `EntryValidationRulesController_setOneForInitiative_v1`
   (`POST /api/v1/validations/customers/{customerId}/initiatives/{initiativeExternalId}/rules`).

8. **Delegates** — `UsersController_updateUserDelegatesBatch_v1`
   (`POST /api/v1/customers/{customerId}/users/delegates/batch`), or
   `UsersController_updateUserDelegates_v1` for one user. Delegates are assistants permitted to
   edit another timekeeper's time.

## Rules

- **Address records by external id.** Initiative and entry operations key off
  `{initiativeExternalId}` / `{entryExternalId}` — the identifier from the *source* system. Send
  the firm's own ids; do not mint new ones. Laurel's internal `{initiativeId}` is a different value.
- **Use `isInitial` on first load.** Several import operations take an `isInitial` query parameter
  to distinguish the initial bulk seed from an incremental delta sync.
- **Batch imports are asynchronous.** Several ingestion operations return `202 Accepted` rather
  than `201` — the write is queued, not complete. Do not treat the response as confirmation that
  the records are queryable; poll the Time Service to confirm.
- **Prefer the highest version.** Where v1 and v2 of an operation both exist, v1 is usually flagged
  `deprecated`. Both keep serving — Laurel versions in the URI path and retires nothing abruptly.
- The specs declare no error responses. Treat non-2xx as opaque and retry only 5xx/429 with backoff.
