---
name: Activate Starlink service for a new site
description: Create an address, open a service line against a product, add the user terminal to the account and attach it to the line.
api: openapi/starlink-public-api-v2-openapi.json
base_url: https://starlink.com/api/public/v2
operations:
  - GET /public/v2/products
  - POST /public/v2/addresses
  - GET /public/v2/addresses/{addressReferenceId}
  - POST /public/v2/service-lines
  - PUT /public/v2/service-lines/{serviceLineNumber}/nickname
  - POST /public/v2/user-terminals
  - POST /public/v2/service-lines/{serviceLineNumber}/user-terminals
  - GET /public/v2/service-lines/{serviceLineNumber}
permissions:
  - Account information, Edit
  - Service plan, View
  - Service plan, Edit
  - Device management, Edit
generated: '2026-07-25'
method: generated
---

# Activate Starlink service for a new site

This flow **creates billable service**. There is no sandbox and no idempotency key. Confirm with
a human before the first write, and guard every POST with your own dedupe key so a retry after a
timeout does not open a second service line.

## Order of operations

Starlink enforces a strict order. A service line needs an address and a product before it exists,
and a terminal must be on the account before it can join a line.

### 1. Pick a product

`GET /public/v2/products` — requires *Service plan, View*. Returns
`SubscriptionProductResponse` objects: `productReferenceId`, `name`, `price`, `isoCurrencyCode`,
`isSla`, `maxNumberOfUserTerminals`, and the compatible `dataProducts`. Keep the
`productReferenceId`.

### 2. Create the address

`POST /public/v2/addresses` — requires *Account information, Edit*.

Send `addressLines`, `formattedAddress`, `latitude` and `longitude`. `regionCode` and
`administrativeArea` are **deprecated as of 2026-07-21**: they are accepted but ignored, so do
not rely on them. The response carries the `addressReferenceId`.

Reuse an existing address with `GET /public/v2/addresses` rather than creating a duplicate.

### 3. Create the service line

`POST /public/v2/service-lines` — requires *Service plan, Edit*. Link the `addressReferenceId`
from step 2 and the `productReferenceId` from step 1. Data blocks can be configured in the same
call. The response carries the `serviceLineNumber`.

Optionally `PUT /public/v2/service-lines/{serviceLineNumber}/nickname` to label it.

### 4. Put the terminal on the account

`POST /public/v2/user-terminals` — requires *Device management, Edit*. This adds the terminal to
the account but **does not start service**.

### 5. Attach the terminal to the line

`POST /public/v2/service-lines/{serviceLineNumber}/user-terminals` — requires *Service plan,
Edit*. The terminal must already be on the account (step 4).

### 6. Verify

`GET /public/v2/service-lines/{serviceLineNumber}` — check `active`, `startDate`,
`productReferenceId` and `addressReferenceId`.

## Teardown, in reverse

1. `DELETE /public/v2/service-lines/{serviceLineNumber}/user-terminals/{deviceId}` — removes the
   terminal from the line but leaves it on the account, and **clears every L2VPN circuit
   configured for that terminal**.
2. `DELETE /public/v2/service-lines/{serviceLineNumber}` — deactivates the line. Irreversible
   from the API.
3. `DELETE /public/v2/user-terminals/{deviceId}` — only succeeds once the terminal is off every
   service line.

## Watch for

- **No idempotency.** Retrying step 3 after a network timeout can create a second billable line.
  Re-read `GET /public/v2/service-lines` before retrying any write.
- **Proration.** Mid-cycle changes generate partial periods; read
  `GET /public/v2/service-lines/{serviceLineNumber}/billing-cycles/partial-periods` and
  <https://starlink.readme.io/docs/understanding-proration>.
- **`422`** is how this API says "your request was shaped correctly but the operation failed."
  The reason is only in `errors[].errorMessage`.
