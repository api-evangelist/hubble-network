---
name: hubble-cloud-api
description: This skill should be used when the user asks to integrate with the Hubble Network Cloud API — "register devices on Hubble", "retrieve Hubble packets", "stream packet data", "configure a Hubble webhook", "rotate a Hubble API key", "batch update devices", "list Hubble metrics", "Hubble billing/invoices", or mentions Hubble Network, Hubble IoT, api.hubble.com, dev_eui, or the HTTP-X-HUBBLE-TOKEN header.
---

# Hubble Cloud API Integration

## Overview

Hubble Network connects off-the-shelf Bluetooth chips to a global network of 90M+ gateways. This skill covers integration with the Hubble Cloud API for device management, packet retrieval, webhooks, metrics, and organization administration.

- **Base URL**: `https://api.hubble.com`
- **Versioning**: rolling release, continuous deployment, backward compatible

## Domain Model

**Device** — a Bluetooth hardware endpoint registered on Hubble. Identified by a UUID (`device.id`), with encryption keys and custom/platform tags.

**Packet** — Base64-encoded Bluetooth data received via gateways. Includes encrypted payload, sequence numbers, auth tags, and metadata (RSSI, SNR, gateway info, timestamps).

**Webhook** — an HTTP endpoint that receives real-time packet batches via push. Configurable batch size (10–1000), automatic retries, delivery metrics.

**Organization** — top-level container (UUID) for devices, users, API keys, webhooks, billing.

**API Key** — JWT bearer token with granular scopes.

### Data flow

1. Device broadcasts a Bluetooth packet.
2. Gateway forwards it to Hubble Network.
3. Platform processes and decrypts.
4. Data is delivered via real-time webhook (push), API streaming (continuation-token pagination), or metrics endpoints.

## Authentication

Send the JWT bearer token in the `Authorization` header on every request:

```
Authorization: Bearer <jwt-token>
```

Generate tokens from Hubble Dashboard → Developer → API Tokens. Tokens cannot be retrieved after creation — store them in a secret manager or environment variable.

### Scopes

Scopes follow the `<resource>:<read|write>` pattern across 8 resources: `devices`, `packets`, `webhooks`, `organization`, `users`, `invitations`, `api_keys`, `metrics`, plus `billing:read`. Apply least privilege — request only the scopes a given integration needs. For the complete scope catalog with descriptions, see `references/api-reference.md` ("API Key Management").

## Rate Limits

- **Per endpoint**: 3 req/s
- **Per organization**: 15 req/s
- **Exceeded**: HTTP 429 with `Retry-After` header

Response headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) expose current state. On 429, honor `Retry-After` and back off exponentially. Prefer batch endpoints and webhook push over polling to stay under limits.

## API Endpoint Map

For request/response schemas and every parameter, see `references/api-reference.md`. For step-by-step implementation guides, see `references/workflows.md`. For runnable Python examples, see `references/examples.md`.

### Device Management

- `POST /api/v2/org/{org_id}/devices` — batch register (up to 1,000/request)
- `GET /api/org/{org_id}/devices` — list with filtering/sorting
- `GET /api/org/{org_id}/devices/{device_id}` — retrieve one
- `PATCH /api/org/{org_id}/devices/{device_id}` — update one
- `PATCH /api/org/{org_id}/devices` — batch update (up to 1,000)
- `DELETE /api/org/{org_id}/devices` — batch delete (up to 1,000)

Registration auto-generates device IDs and encryption keys. Encryption options: `AES-256-CTR`, `AES-128-CTR`, `NONE`.

### Packet Retrieval

- `GET /api/org/{org_id}/packets` — stream with continuation-token pagination

Parameters: `start_time`, `end_time` (default last 7 days), `device_id`, `platform_tag` (e.g. `hubnet.platform=LoRaWAN`).

### Webhook Management

- `POST /api/org/{org_id}/webhooks` — register endpoint
- `GET /api/org/{org_id}/webhooks` — list
- `PATCH /api/org/{org_id}/webhooks/{webhook_id}` — update
- `DELETE /api/org/{org_id}/webhooks/{webhook_id}` — remove

Configure URL, display name, and max batch size (10–1000; default 100).

### Metrics

- `GET /api/org/{org_id}/api_metrics` — API request metrics (hourly)
- `GET /api/org/{org_id}/packet_metrics` — packet volume
- `GET /api/org/{org_id}/webhook_metrics` — delivery success/failure
- `GET /api/org/{org_id}/device_metrics` — active/registered counts

Configurable `days_back` (1–365) and `interval` (hour/day/month).

### Organization, Users, Invitations

- `GET|PATCH /api/org/{org_id}` — read/update org
- `GET|POST /api/org/{org_id}/users`, `PATCH|DELETE .../{user_id}` — manage users (roles: Admin, Member)
- `GET|POST|DELETE /api/org/{org_id}/invitations` — manage invites

### API Keys

- `GET /api/org/{org_id}/check` — validate current key
- `GET|POST /api/org/{org_id}/key`, `PATCH|DELETE .../{key_id}` — manage keys
- `GET /api/org/{org_id}/key_scopes` — list available scopes

### Billing

- `GET /api/org/{org_id}/billing/invoices` — list invoices
- `GET /api/org/{org_id}/billing/invoices/{invoice_id}/pdf` — download PDF
- `GET /api/org/{org_id}/billing/usage` — device usage

## Packet Structure (Critical)

The packets endpoint returns a **nested** structure. Field locations differ from what naive callers expect, and getting them wrong is the most common integration bug.

```json
{
  "location": {
    "timestamp": 1765598212.181298,
    "latitude": 47.61421,
    "longitude": -122.31929,
    "altitude": 829,
    "horizontal_accuracy": 29,
    "vertical_accuracy": 29
  },
  "device": {
    "id": "bc17a947-7a4f-4cff-9127-340cc4005272",
    "name": "Device Display Name",
    "tags": ["tag1", "tag2"],
    "payload": "SGVsbG8gV29ybGQ=",
    "timestamp": "2025-01-15T10:30:45Z",
    "rssi": -85,
    "sequence_number": 42,
    "counter": 123
  },
  "network_type": "bluetooth"
}
```

Field locations:
- Device ID: `packet.device.id` — **not** `packet.device_id` or `packet.dev_eui`
- Device name: `packet.device.name`
- RSSI: `packet.device.rssi` — **not** `packet.rssi`
- Sequence number: `packet.device.sequence_number`
- GPS: `packet.location.latitude`, `packet.location.longitude`
- Packet timestamp: `packet.location.timestamp` — Unix epoch **seconds** (multiply by 1000 for JS `Date`)

## Pagination

Large-dataset endpoints (notably `/packets`) use the `Continuation-Token` **header**, not offset/cursor query params:

1. Issue the initial request.
2. Read up to 1,000 items from the response.
3. Read the `Continuation-Token` response header. If absent, streaming is complete.
4. If present, reissue the same request with `Continuation-Token: <value>` set in request headers.

For a complete streaming loop implementation, see `references/examples.md` ("Packet Retrieval Examples").

## Data Encoding

Binary data crosses JSON as Base64: device encryption keys on registration, and packet payloads on retrieval. A missing/bad Base64 encoding on device keys produces a 400 with "Invalid deviceKey format". Decode payloads with the standard library (`base64.b64decode` / `Buffer.from(s, 'base64')`).

## Webhooks

Hubble POSTs JSON batches to the configured URL. Every request carries an `HTTP-X-HUBBLE-TOKEN` header — validate it against the webhook secret before processing, and return 2xx only on successful receipt (any non-2xx triggers automatic retry with exponential backoff).

Request body shape:

```json
{ "packets": [ { "device_id": "...", "payload": "...", "timestamp": "...", "metadata": { } } ] }
```

For a Flask/Express receiver with signature validation, see `references/examples.md` ("Webhook Examples"). For delivery-failure debugging, see `references/troubleshooting.md` ("Webhook Problems").

## Error Handling

| Status | Meaning | Action |
|--------|---------|--------|
| 200/201 | Success | — |
| 400 | Bad request | Check payload shape; common: unencoded device key |
| 401 | Missing/invalid token | Verify `Bearer` prefix and token validity |
| 403 | Insufficient scope | Check key's scopes; confirm org ID |
| 404 | Not found | Validate IDs |
| 429 | Rate limited | Honor `Retry-After`, back off |
| 500 | Server error | Retry with backoff |

Every response includes an `X-Request-ID` header — capture it in logs and include it in support requests. For full error catalog and fixes, see `references/troubleshooting.md`.

## Quick Diagnostics

Validate the API key and its scopes:

```bash
curl -H "Authorization: Bearer $HUBBLE_API_TOKEN" \
  https://api.hubble.com/api/org/$HUBBLE_ORG_ID/check

curl -H "Authorization: Bearer $HUBBLE_API_TOKEN" \
  https://api.hubble.com/api/org/$HUBBLE_ORG_ID/key_scopes
```

## Best Practices

- Prefer batch endpoints (up to 1,000 items) over N single-item calls.
- Push over poll: use webhooks for real-time data.
- Back off on 429/5xx; never retry tightly.
- Rotate keys via create-new → cut over → delete-old; never edit in place.
- Treat device encryption keys as secrets — Hubble does not store recoverable copies.
- Log `X-Request-ID` on every request for traceability.

## Additional Resources

- **`references/api-reference.md`** — every endpoint with parameters, responses, and examples
- **`references/workflows.md`** — multi-step guides (device onboarding, packet streaming, webhook setup, key rotation, batch operations)
- **`references/examples.md`** — runnable Python code for auth, device management, packets, webhooks, metrics
- **`references/troubleshooting.md`** — symptom-indexed fixes for auth, registration, pagination, Base64, rate-limit, and webhook issues
- **`resources/hubble-openapi.yaml`** — machine-readable OpenAPI spec
- **Hubble API docs**: https://docs.hubble.com/docs/api-specification/hubble-cloud-api
- **Support**: support@hubble.com
