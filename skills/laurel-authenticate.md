---
name: Authenticate to the Laurel APIs
description: Exchange machine-to-machine client credentials for a Bearer JWT and call the Laurel Time, Identity and Ingestion services.
api: openapi/laurel-identity-openapi-original.json
operations:
  - OauthController_createServiceToken_v1
  - CustomerController_getCustomerRegion_v1
---

# Authenticate to the Laurel APIs

Every Laurel API call is authenticated with a Bearer JWT obtained from a machine-to-machine
client-credentials exchange. There is no self-service signup: Laurel's solutions delivery team
issues your `customerId`, `clientId`, `clientSecret`, and your **regional** Time Service endpoint.

## Prerequisites

Collect from Laurel before you start:

- `customerId` — scopes nearly every request (it is a path parameter, not a claim you can infer)
- `clientId` and `clientSecret`
- your regional Time Service base URL

## Steps

1. **Exchange credentials for a token** — `OauthController_createServiceToken_v1`
   (`POST https://identity.laurel.ai/api/v1/oauth/token`). Send a JSON body:

   ```json
   {
     "audience": "https://timeautomation.com",
     "grant_type": "client_credentials",
     "client_id": "<clientId>",
     "client_secret": "<clientSecret>"
   }
   ```

   The response carries `access_token`, `token_type: Bearer`, and `expires_in: 86400`.

2. **Send the token** on every subsequent request as `Authorization: Bearer <access_token>`.
   The OpenAPI calls this scheme `ApiBearerAuth` (`http` / `bearer`, `bearerFormat: JWT`).

3. **Cache and refresh.** Tokens last 86400 seconds (24h). Cache the token for its lifetime and
   re-run step 1 before expiry — do not exchange credentials per request. There is no refresh
   token in the client-credentials flow.

4. **Resolve the customer region if you need it** — `CustomerController_getCustomerRegion_v1`
   (`GET /api/v1/customers/{customerId}/region`). Laurel deploys regionally and holds data in the
   customer's jurisdiction, so the Time Service host differs per customer. Note this operation is
   marked `deprecated` in the spec; prefer the regional endpoint Laurel issued you directly.

## Base URLs

| Service   | Base URL                                                   |
|-----------|------------------------------------------------------------|
| Identity  | `https://identity.laurel.ai`                               |
| Time      | `https://api.laurel.ai/time/` (regional variant per customer) |
| Ingestion | `https://api.laurel.ai/ingestion/`                         |

## Rules

- Always scope calls with the issued `customerId`; most paths nest under `/customers/{customerId}/`.
- Prefer operations tagged `Public` in the OpenAPI — that is the surface Laurel supports for
  customer and partner integrations. Untagged operations are internal service endpoints.
- The specs declare no 4xx/5xx responses and Laurel publishes no error reference, so do not assume
  an error envelope shape. Treat any non-2xx as opaque: log the raw status and body, and retry only
  on 5xx and 429 with backoff.
- No rate limits are published. Be conservative and back off on any non-2xx.
