---
name: Provision and look up Laurel users
description: Create, find, update and deactivate timekeepers in the Laurel Identity Service, and reconcile them against a firm's directory or timekeeper ids.
api: openapi/laurel-identity-openapi-original.json
operations:
  - CustomerUserController_search_v1
  - CustomerUserController_lookupUser_v1
  - CustomerUserController_lookupUsersBatch_v1
  - CustomerUserController_translateTimekeeperIdsToUserIds_v1
  - CustomerUserController_get_v1
  - CustomerUserController_updateV2_v2
  - CustomerUserController_createNewUserIdentity_v1
  - CustomerUserController_delete_v1
---

# Provision and look up Laurel users

Laurel users are the firm's timekeepers and administrators. They live in the **Identity Service**
(`https://identity.laurel.ai`) and are always scoped to a `customerId`.

Authenticate first (see `laurel-authenticate.md`).

## Find users

1. **Search** — `CustomerUserController_search_v1`
   (`GET /api/v1/customers/{customerId}/users`), tagged `Public`. Takes an optional `search`
   parameter. A v2 (`CustomerUserController_searchV2_v2`) and a trimmed
   `CustomerUserController_searchLite_v1` also exist; `searchLite` is the cheaper call when you
   only need names and ids.

2. **Look up one user by identity** — `CustomerUserController_lookupUser_v1`
   (`GET /api/v1/customers/{customerId}/users/lookup`), tagged `Public`.

3. **Look up many at once** — `CustomerUserController_lookupUsersBatch_v1`
   (`POST /api/v1/customers/{customerId}/users/batch/lookup`), tagged `Public`. Prefer this over
   looping the single lookup when reconciling a directory.

4. **Fetch by id** — `CustomerUserController_get_v1`
   (`GET /api/v1/customers/{customerId}/users/{userId}`), tagged `Public`. For several known ids
   use `CustomerUserController_getUsersByIds_v1`.

5. **Map billing timekeeper ids to Laurel user ids** —
   `CustomerUserController_translateTimekeeperIdsToUserIds_v1`
   (`POST /api/v1/customers/{customerId}/users/translate-timekeeper-ids`). This is the bridge
   between the billing system's timekeeper numbers and Laurel's user ids — use it when joining
   time data back to the practice-management system.

## Change users

6. **Update** — `CustomerUserController_updateV2_v2`
   (`PATCH /api/v2/customers/{customerId}/users/{userId}`), tagged `Public`. The v1 sibling
   `CustomerUserController_update_v1` still serves.

7. **Add an identity** (a new login/email/SSO identity for an existing user) —
   `CustomerUserController_createNewUserIdentity_v1`
   (`POST /api/v1/customers/{customerId}/users/{userId}/identities`). Read current identities with
   `CustomerUserController_lookupUserIdentity_v1`, replace with
   `CustomerUserController_updateUserIdentity_v1`.

8. **Grant super-delegate status** — `CustomerUserController_setUserSuperDelegateStatus_v1`
   (`PATCH /api/v1/customers/{customerId}/users/{userId}/super-delegate`).

9. **Delete** — `CustomerUserController_delete_v1`
   (`DELETE /api/v1/customers/{customerId}/users/{userId}`).

## Bulk provisioning and SSO

- Bulk import: `CustomerUserController_postUserDataV2_v2`
  (`POST /api/v2/customers/{customerId}/users/batch/import`). The v1 forms
  (`CustomerUserController_deprecatedImport_v1`, `CustomerUserController_postUserData_v1`) are
  marked deprecated.
- SSO connections are managed per customer: `CustomerController_createConnection_v1`,
  `CustomerController_updateConnectionDomains_v1`, and SCIM provisioning is toggled with
  `CustomerController_updateConnectionScim_v1`. Where the firm runs SCIM, let SCIM drive user
  lifecycle rather than calling the batch import.

## Rules

- Prefer the operations tagged `Public` — that is the supported integration surface. The rest are
  internal service endpoints and may change without notice.
- Prefer the highest version of an operation; superseded versions are flagged `deprecated` in the
  spec but keep serving alongside their replacement.
- **Deletion and SCIM changes are consequential.** Deleting a user or disabling SCIM affects access
  and billable history; require human confirmation rather than acting autonomously.
- No error envelope, rate limits or request-id header are documented. Retry only 5xx/429 with backoff.
