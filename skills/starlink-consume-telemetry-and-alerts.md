---
name: Consume the Starlink telemetry stream and device alerts
description: Poll the telemetry stream correctly, decode column-indexed rows, resolve alert codes, and pick the cache endpoint when only current state is needed.
api: asyncapi/starlink-telemetry-asyncapi.yml
base_url: https://starlink.com/api/public/v2
operations:
  - POST /public/v2/telemetry/stream
  - POST /public/v2/telemetry/query
permissions:
  - Device telemetry, View
generated: '2026-07-25'
method: generated
---

# Consume the Starlink telemetry stream and device alerts

Both operations are documented in the API reference but are **absent from the downloadable V2
OpenAPI**, so there is no spec-backed input schema. Everything below comes from
<https://starlink.readme.io/docs/telemetry-api> and
<https://starlink.readme.io/docs/alerts>, captured in `asyncapi/starlink-telemetry-asyncapi.yml`.

## Choose the right endpoint

- **`POST /public/v2/telemetry/query`** — most recent typed values per device, in a convenient
  keyed dictionary. Use this if you only need current state. It does not consume the stream.
- **`POST /public/v2/telemetry/stream`** — the full time series. Use this only if you are
  building a continuous consumer with somewhere to put the data.

## Poll the stream

```json
{"batchSize": 1000, "maxLingerMs": 15000}
```

`batchSize` defaults to 1000 (max 65000). `maxLingerMs` defaults to 15000 (max 65000). The
recommended cadence is those defaults, roughly 4 requests per minute.

This is a hot loop, not a cron job. Starlink keeps a **read index per {service account, account}**
and advances it on every successful response. Consequences:

- If your process crashes mid-batch, the unprocessed entries are **gone** — they are not
  re-delivered. Persist the batch before you acknowledge it to yourself.
- If you poll slower than the account produces data you fall permanently behind and keep
  receiving the oldest retained rows.
- Retention is **8 hours**. A consumer offline longer than that loses everything in between.
- Each service account is an independent consumer. Create a second service account to read the
  same stream from a second environment; that is the only isolation Starlink offers.

An empty response is normal — telemetry is not present in every response. Keep polling.

## Decode the rows

The response is optimised for transport, not for reading:

```json
{"data":{"columnNamesByDeviceType":{"u":["DeviceType","UtcTimestampNs","DeviceId", "..."]},
         "values":[["u",1681879752000000000,"ut12345678-a12b3456-123456c1", "..."]]},
 "metadata":{"enums":{"DeviceType":{"u":"UserTerminal","r":"Router","i":"IpAllocs"}}}}
```

- Element 0 of every row is the device-type code: `u` user terminal, `r` router, `i` IP allocations.
- **Re-parse `columnNamesByDeviceType` on every response.** Column order is explicitly subject to
  change. Resolve a metric by looking up its index in the column-name array, never by a hardcoded
  offset.
- Types are not published. Infer them from the values; a field's type is stable once introduced,
  and all fields should be treated as nullable.
- No filtering: you always receive every device on the account.

Cadence: user terminals and routers aggregate every 15 seconds. IP allocation rows arrive at
least every 5 minutes and immediately on change; an empty `Ipv4`/`Ipv6Ue`/`Ipv6Cpe` row is a
tombstone meaning the terminal no longer holds public IPs.

## Resolve alerts

Active alerts arrive as an array of integer codes in the `ActiveAlert` column. Resolve them
against `metadata.enums.AlertsByDeviceType` **in the same response** — Starlink may add codes,
remove deprecated ones, or re-point a code at a different name, and the metadata block is the
declared source of truth. Do not hardcode the mapping; the snapshot in
`asyncapi/starlink-telemetry-asyncapi.yml` is a reference, not a contract.

An alert fires if its condition held at least once in the previous 15 seconds and persists while
it is active. The API is stateless: it reports current state only, so build your own edge
detection if you need "alert opened" and "alert cleared" events.

Codes worth routing on: `55` `disabled_no_active_account`, `56`
`disabled_too_far_from_service_address`, `80` `thermal_shutdown`, `84`
`disabled_data_usage_exceeded_quota`, `97` `data_overage_rate_limited`, `62` `pop_change` (a new
public IP is coming), `79` `software_update_reboot_pending` (currently **known to misfire** on
device software at or above 2025.06.05).

## Errors

`401` means the token expired mid-loop — re-mint and continue rather than restarting the
consumer. `403` means the service account is missing *Device telemetry, View*. Telemetry has its
own incident history; see `lifecycle/starlink-lifecycle.yml`.

## There are no webhooks

Starlink pushes nothing. The only push channel is proactive alert **email** to account contacts,
which has no API to subscribe, list or acknowledge.
