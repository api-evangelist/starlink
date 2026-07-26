---
name: Authenticate to the Starlink API and inventory an account
description: Mint a bearer token from a Starlink V2 service account, then walk the account, service lines and user terminals to build a full inventory.
api: openapi/starlink-public-api-v2-openapi.json
base_url: https://starlink.com/api/public/v2
operations:
  - POST https://starlink.com/api/auth/connect/token
  - GET /public/v2/account
  - GET /public/v2/service-lines
  - GET /public/v2/user-terminals
  - GET /public/v2/routers/{routerId}
  - GET /public/v2/addresses
permissions:
  - Account information, View
  - Service plan, View
  - Device management, View
generated: '2026-07-25'
method: generated
---

# Authenticate to the Starlink API and inventory an account

The Starlink Public API V2 has no API keys and no test mode. Every call runs against a real,
billable enterprise account. Read `sandbox/starlink-sandbox.yml` before you write anything.

## 1. Get credentials

A human with the **Admin** or **Service Account Management** role creates a V2 service account at
<https://www.starlink.com/account/settings> and hands you a `clientId` and a `secret`. The service
account is bound to a Starlink **account**, not to a user, and carries a fixed permission set. A
service account cannot be granted a permission its creator does not hold.

## 2. Mint a token

```
POST https://starlink.com/api/auth/connect/token
Content-Type: application/x-www-form-urlencoded

client_id=<clientId>&client_secret=<secret>&grant_type=client_credentials
```

Take `access_token` from the response and send `Authorization: Bearer <access_token>` on every
call. Tokens live about **15 minutes**.

**Cache the token.** The token endpoint is limited to 1000 requests per 15 minutes per client IP.
Do not mint a token per request. Re-mint only on a `401`, then retry the original call **once**.

## 3. Walk the account

1. `GET /public/v2/account` — accountNumber, regionCode, accountName, activeSuspensions.
2. `GET /public/v2/addresses` — paginated with `page`; each address carries an `addressReferenceId`.
3. `GET /public/v2/service-lines` — paginated; each carries `serviceLineNumber`,
   `addressReferenceId`, `productReferenceId`, `active`, `startDate`/`endDate`, `dataBlocks`.
4. `GET /public/v2/user-terminals` — paginated at 100 per page. Filter with `serviceLineNumbers`,
   `userTerminalIds`, `hasServiceLine`, or `searchString` (partial match on terminal id, serial
   number or kit serial number). Each terminal carries its `routers` and `l2VpnCircuits`.
5. `GET /public/v2/routers/{routerId}` for any router detail you need.

## 4. Paginate correctly

Every collection returns `content.{pageIndex, limit, isLastPage, totalCount, results}`. `page` is
**zero-indexed**. Loop until `isLastPage` is true — do not trust `totalCount` alone, and do not
assume stable ordering across pages (a September 2025 incident produced duplicated and missing
terminals mid-iteration; re-fetch and reconcile by `userTerminalId` rather than by position).

## 5. Respect the budget

250 requests per minute **per Starlink account**, shared with every other integration on that
account. On `429`, back off. Starlink's own guidance is to sync this data into your own database
on a schedule rather than querying the API for every read.

## 6. Read errors

Responses are never RFC 9457. Success and failure share one envelope:

```json
{"errors":[{"memberNames":[],"errorMessage":"not_found"}],"warnings":[],"information":[],"isValid":false}
```

- `401` — token expired; re-mint and retry once.
- `403` — the service account lacks the permission named in the operation description. This is a
  configuration problem, not a retryable one. Stop and ask for the permission.
- `422` — the dominant failure code. Read `errors[].errorMessage`; it is free text.
- `404` — either a bad identifier or a sunset endpoint. All V1 paths and every
  `web-api.starlink.com/enterprise/` path now return 404.

See `errors/starlink-problem-types.yml` and `conventions/starlink-conventions.yml`.
