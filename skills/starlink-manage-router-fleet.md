---
name: Manage a Starlink WiFi router fleet
description: Create and assign router configs, manage TLS certificates and local content, run client sandboxing with heartbeats, and reboot hardware safely.
api: openapi/starlink-public-api-v2-openapi.json
base_url: https://starlink.com/api/public/v2
operations:
  - GET /public/v2/routers/configs
  - POST /public/v2/routers/configs
  - GET /public/v2/routers/configs/{configId}
  - PUT /public/v2/routers/configs/{configId}
  - PUT /public/v2/routers/configs/assign
  - GET /public/v2/routers/configs/default
  - PUT /public/v2/routers/configs/default
  - GET /public/v2/routers/configs/tls
  - POST /public/v2/routers/configs/tls
  - DELETE /public/v2/routers/configs/tls
  - POST /public/v2/routers/local-content
  - GET /public/v2/routers/local-content
  - GET /public/v2/routers/sandbox/clients
  - POST /public/v2/routers/sandbox/clients
  - PUT /public/v2/routers/sandbox/heartbeat
  - POST /public/v2/routers/{routerId}/reboot
  - POST /public/v2/user-terminals/{deviceId}/reboot
  - PUT /public/v2/user-terminals/configs/assign
permissions:
  - Device command and configuration, View
  - Device command and configuration, Edit
  - Device configuration assignment, Edit
generated: '2026-07-25'
method: generated
---

# Manage a Starlink WiFi router fleet

These operations act on **physical hardware in the field**. Assignments propagate to online
routers within 1-2 minutes and to offline routers when they next connect. Treat every write here
as a change-managed action.

## Config lifecycle

1. `POST /public/v2/routers/configs` — create a config (`nickname` + `routerConfigJson`).
2. `GET /public/v2/routers/configs` — paginated list; `GET .../configs/{configId}` for one.
3. `PUT /public/v2/routers/configs/{configId}` — update. **Every router already assigned to this
   config receives the update immediately if online**, or on next connect. There is no staged
   rollout; edit a config only when you intend to change every router carrying it.
4. `PUT /public/v2/routers/configs/assign` — assign a config (or none) to a set of routers.
   Atomic in the sense that **on error no assignment occurs**.
5. `PUT /public/v2/routers/configs/default` — the config every *new* router on the account
   inherits. Send an empty string to clear the default. Existing routers are unaffected.

User terminals have the parallel operation `PUT /public/v2/user-terminals/configs/assign`; note
that terminal `configId`s are currently only visible on the Starlink website, not through the API.

## TLS configs

`POST /public/v2/routers/configs/tls` stores a certificate + key pair so router configs can
reference the certificate alone and have the matching key filled in on save. The **base64-encoded
certificate string is the identifier** and must be unique on the account.

`DELETE /public/v2/routers/configs/tls` removes it from the reusable set only — router configs
already saved with that certificate are unaffected. Rotate by creating the new pair first,
re-saving the configs that reference it, then deleting the old one. Read `notBefore`/`notAfter`
from `GET /public/v2/routers/configs/tls` and rotate before expiry; an expired certificate breaks
the router's local HTTPS API.

## Local content

`POST /public/v2/routers/local-content` uploads the HTML page a router serves from its local
HTTPS server. Constraints: **HTML only, under 4 MB, filename under 100 characters**, sent as
`multipart/form-data`. Files land in a public bucket for configured routers to download — do not
put anything sensitive in them. `GET /public/v2/routers/local-content` lists them with
`fileContentId` and `fileContentHash`.

## Client sandboxing

"Sandbox" here means a captive-portal walled garden for WiFi clients, **not** a developer test
environment.

- `POST /public/v2/routers/sandbox/clients` — batch update sandbox state. When a client is
  duplicated in the batch, the record with the latest expiry wins.
- `GET /public/v2/routers/sandbox/clients` — returns only clients unsandboxed through the API
  whose access has not yet expired, 1000 per page. Clients allowed out because sandboxing is
  disabled are not returned.
- `PUT /public/v2/routers/sandbox/heartbeat` — **this is a dead-man switch.** If Starlink stops
  receiving heartbeats for the account, it instructs every router under that account to disable
  sandboxing until reboot, i.e. the walled garden **fails open**. Run the heartbeat on its own
  independent schedule, and alert on failures. Two published incidents (2025-09-06, 2026-03-01)
  are exactly this failure mode.

## Reboots

`POST /public/v2/routers/{routerId}/reboot` drops every client on that router.
`POST /public/v2/user-terminals/{deviceId}/reboot` drops the site. Both are physical actions with
no dry run and no undo. Never issue them in a loop over a fleet without a change window, and
never as an automated remediation without a human in the loop.

A manual reboot also applies a pending software update, which is the documented workaround when
alert `79` `software_update_reboot_pending` would otherwise reboot at 3 AM local time.

## Budget

250 requests per minute per account covers this fleet work *and* every other integration on the
account. Batch where the API allows it (`configs/assign` and `sandbox/clients` both take sets)
rather than iterating device by device.
