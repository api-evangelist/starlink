---
name: Reconcile Starlink billing and data usage
description: Pull invoices, balance and data usage, manage recurring and top-up data blocks, and handle priority-data opt-in across a fleet of service lines.
api: openapi/starlink-public-api-v2-openapi.json
base_url: https://starlink.com/api/public/v2
operations:
  - GET /public/v2/billing/invoices
  - GET /public/v2/billing/invoices/{invoiceId}
  - GET /public/v2/billing/balance
  - POST /public/v2/data-usage/query
  - GET /public/v2/service-lines/{serviceLineNumber}/billing-cycles/partial-periods
  - PUT /public/v2/service-lines/{serviceLineNumber}/data/recurring
  - POST /public/v2/service-lines/{serviceLineNumber}/data/top-up
  - POST /public/v2/service-lines/{serviceLineNumber}/data/opt-in
  - POST /public/v2/service-lines/{serviceLineNumber}/data/opt-out
  - PUT /public/v2/service-lines/{serviceLineNumber}/product
  - GET /public/v2/data-pools
  - GET /public/v2/data-pools/usage
permissions:
  - Financial, View
  - Service plan, View
  - Service plan, Edit
generated: '2026-07-25'
method: generated
---

# Reconcile Starlink billing and data usage

Read operations here need *Financial, View* (invoices and balance) or *Service plan, View*
(usage). Every write costs money.

## Invoices and balance

- `GET /public/v2/billing/invoices` — `InvoiceSummaryResponse`: `invoiceId`, `description`,
  `currency`, `amount`, `dueAmount`, `invoiceDate`, `dueDate`, `status`.
- `GET /public/v2/billing/invoices/{invoiceId}` — adds `invoiceLines`, each with
  `productReferenceId`, `serviceLineNumbers`, `quantity`, `unitPrice`, `taxAmount`, `subTotal`
  and the service period. This is how you attribute a bill back to specific service lines.
- `GET /public/v2/billing/balance` — **one balance per currency**. When everything is settled you
  get a single zero entry in the account's default currency, not an empty array. Do not treat a
  single-element response as "one currency in use".

These three endpoints were added on 2026-07-08; older integrations will not know about them.

## Data usage

`POST /public/v2/data-usage/query` — *Service plan, View*.

```json
{"serviceLineNumbers": [], "previousBillingCycles": 1, "activeServiceLinesOnly": true}
```

An empty `serviceLineNumbers` means the whole account. Behaviour changed on 2026-04-06, so pin
your expectations to the current contract:

- Data blocks are now **always** returned, including when `blocksCount` is 0.
- `servicePlan.dataCategoryMapping` **always returns `{}`** — it carries no information; ignore it.
- Overage lines with no usage are no longer returned, and a mid-cycle plan change produces
  **separate** overage lines. Sum overage lines rather than assuming one per cycle.
- Billing cycles with no recorded usage carry no overage lines at all.

For mid-cycle changes, cross-check
`GET /public/v2/service-lines/{serviceLineNumber}/billing-cycles/partial-periods`, which returns
`productReferenceId`, `periodStart` and `periodEnd` per partial period. See
<https://starlink.readme.io/docs/understanding-proration>.

## Data blocks

Both write operations require the service line to be on a **top-up plan** and require *Service
plan, Edit*.

- `PUT /public/v2/service-lines/{serviceLineNumber}/data/recurring` — set the recurring block
  count. `ApplyToCurrentMonth` (added 2026-04-02) defaults to **false**, meaning changes apply to
  future cycles only. Set it to `true` and the delta is added to the current cycle immediately:
  the extra blocks expire at the end of the current cycle and bill next month. **Reducing** the
  count never claws back current-month data.
- `POST /public/v2/service-lines/{serviceLineNumber}/data/top-up` — a one-time block. Charged
  immediately, not idempotent. A duplicated retry is a duplicated charge; re-read usage before
  retrying.

## Priority data opt-in

`POST .../data/opt-in` lets the line keep using priority data past plan capacity;
`POST .../data/opt-out` makes it fall back to standard data instead. Only applies to some
products, and the response carries `isInOptInCoolDown` — respect the cooldown rather than
retrying the toggle.

Pair this with telemetry alert `97` `data_overage_rate_limited` and `84`
`disabled_data_usage_exceeded_quota` to drive the decision from real device state.

## Data pools (pre-release)

`GET /public/v2/data-pools`, `GET /public/v2/data-pools/usage`,
`POST /public/v2/data-pools/{dataPoolId}/set-automatic-top-up` and
`PATCH /public/v2/service-lines/{serviceLineNumber}/consume-from-pool` are marked
**"available for select audiences only"**. Feature-detect: a `403` here may mean the feature is
not enabled for the account rather than that a permission is missing. Do not build a billing
reconciliation that hard-depends on them.

## Plan changes

`PUT /public/v2/service-lines/{serviceLineNumber}/product` changes the subscription. This is a
billing event with proration consequences — confirm with a human, and re-read
`partial-periods` afterwards to see what it actually generated.
