---
name: hubble-network-onboard-devices-and-pull-packets
description: Register Bluetooth devices on the Hubble Network and pull their decrypted packet data by polling the Cloud API with continuation-token pagination.
api: openapi/hubble-network-platform-openapi.yml
generated: '2026-08-04'
method: generated
source: openapi/hubble-network-platform-openapi.yml + conventions/hubble-network-conventions.yml
operations:
  - validate-api-key
  - list-key-scopes
  - register-new-devices
  - list-devices
  - get-device
  - batch-update-devices
  - retrieve-organization-packets
  - get-packet-metrics
---

# Onboard devices and pull packets — Hubble Network

Use this when the goal is "get my Bluetooth devices onto Hubble and read their data by polling".
If the goal is push delivery instead, use `hubble-network-configure-packet-webhook`.

## 0. Preconditions

- Base URL is `https://api.hubble.com`. Every path is scoped to your organization UUID (`org_id`).
- Auth is a JWT bearer API key: `Authorization: Bearer <token>`. Issue it in the Hubble Dashboard
  under **Developer Tools > API Tokens**; it cannot be retrieved again after creation.
- Required scopes for this flow: `read-devices`, `write-devices`, `read-packets`,
  `read-platform-metrics`. A key created with no scopes gets all 16 (admin) — prefer least privilege.

## 1. Verify the credential before doing anything else

Call `validate-api-key` (`GET /v1/org/{org_id}/check`). A 401 means the key is missing, expired, or
belongs to a different organization. Call `list-key-scopes` (`GET /v1/org/{org_id}/key_scopes`) to
confirm the key carries the scopes above — a 403 later in this flow is almost always a missing scope,
not a bad key.

## 2. Register devices

Call `register-new-devices` (`POST /v2/org/{org_id}/devices`) — note this is the **v2** path; the rest
of the flow is v1. Register in batches rather than one call per device: the per-endpoint limit is
3 requests/second.

Set device `tags` at registration. Two kinds exist:
- **custom tags** — your own key/values, used later for filtering.
- **platform tags** — Hubble-reserved, prefixed `_`. `"_env": "sandbox"` keeps a device in the
  100-device non-billable Sandbox; `"_env": "production"` puts it on the live network.

Start in Sandbox, prove the device transmits, then promote by flipping `_env` with
`batch-update-devices` (`PATCH /v1/org/{org_id}/devices`).

For satellite devices, also set `install_location` (`lat`, `long`, `location_name`) — Hubble uses it
for pass prediction. Send `0,0` to clear it.

## 3. Confirm the device is really beaconing

Before polling the cloud, verify locally with the first-party CLI:

```
pip install pyhubblenetwork
hubblenetwork ble scan --payload-format hex
```

Then call `get-device` (`GET /v1/org/{org_id}/devices/{device_id}`) and read `most_recent_packet` —
it carries separate `terrestrial` and `satellite` sub-objects, each with a timestamp and location.
It updates only periodically, so treat a stale value as "not yet confirmed", not "broken".

## 4. Poll for packets

Call `retrieve-organization-packets` (`GET /v1/org/{org_id}/packets`).

Pagination is **cursor-in-a-header**, which most generated clients get wrong:
- The response returns up to 1,000 packets and a `Continuation-Token` **response header**.
- Send that value back as a `Continuation-Token` **request header** on the next call to the same
  endpoint.
- Keep going until no `Continuation-Token` comes back. That is the only end-of-data signal.

Optional query filters: `start` / `end` timestamps (max 30 days of lookback) and `device_id`.
Results are ordered by ingestion timestamp.

Each packet carries `device.payload` (Base64, decrypted), `device.rssi`, `device.timestamp`,
`device.sequence_number`, `device.counter`, a `location` fix, `network_type`
(`TERRESTRIAL` | `SATELLITE`), and — only for self-provided packets — `gateway`.

## 5. Rules the API will enforce on you

- **Rate limits**: 3 requests/second per endpoint, 15 requests/second per organization, leaky bucket.
  A 429 is the *only* signal — there are no `RateLimit-*` or `Retry-After` headers. Implement
  exponential backoff yourself.
- **No idempotency contract**: there is no `Idempotency-Key`. A retried write is a second write.
  Make writes safe by checking with `list-devices` / `get-device` before re-issuing a registration.
- **Errors** are a custom envelope, not RFC 9457 problem+json:
  `{"code": 400, "name": "Bad Request", "description": "..."}` served as `application/json`.
  Branch on the HTTP status and the `name` enum. `description` is explicitly documented as changeable
   — never parse it.
- **Tracing**: send an `X-Request-ID` header; it is echoed on the response and logged. Include it in
  any support ticket.

## 6. Watch the surface

`get-packet-metrics` (`GET /v1/org/{org_id}/packet_metrics`) and `get-device-metrics`
(`GET /v1/org/{org_id}/device_metrics`) give totals and interval breakdowns — use them to spot a
device that stopped reporting instead of scanning packets yourself.
