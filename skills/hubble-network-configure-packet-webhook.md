---
name: hubble-network-configure-packet-webhook
description: Register, secure, test and monitor a Hubble Network packet webhook so decrypted Bluetooth packet batches are pushed to your endpoint instead of polled.
api: openapi/hubble-network-platform-openapi.yml
generated: '2026-08-04'
method: generated
source: openapi/hubble-network-platform-openapi.yml + asyncapi/hubble-network-packet-webhooks.yml
operations:
  - validate-api-key
  - create-webhook-endpoint
  - list-registered-webhooks
  - test-webhook-endpoint
  - update-webhook-endpoint
  - delete-webhook-endpoint
  - get-webhook-metrics
  - packet-webhook-example
---

# Configure a packet webhook — Hubble Network

Push delivery is the lower-latency alternative to polling `retrieve-organization-packets`. Hubble
POSTs batches of decrypted packets to an HTTPS endpoint you own.

## 0. Preconditions

- `Authorization: Bearer <token>` with scopes `read-webhooks` and `write-webhooks`.
- Verify the key first with `validate-api-key` (`GET /v1/org/{org_id}/check`).
- An organization may have at most **2** webhook endpoints registered at once. That cap exists so you
  can stand up a replacement receiver and cut over — do not treat it as a fan-out mechanism.

## 1. Build the receiver first

`packet-webhook-example` (`POST /v1/webhook/testBatch`) in the OpenAPI **is the contract for your
endpoint**, not an endpoint you call. Implement exactly that request shape:

- Body: `packetBatch` — `{ "packets": [ ... ] }`, each packet carrying `device` (id, name, tags,
  payload, rssi, timestamp, counter, sequence_number), `location`, `network_type`, and optionally
  `gateway`.
- Batch size varies between 1 and the endpoint's configured `max_batch_size`.

Your handler must:
- **Validate the `HTTP-X-HUBBLE-TOKEN` header** against the secret Hubble generated for this endpoint.
  This is a static shared secret — there is no HMAC body signature and no timestamp/replay window, so
  the token only proves sender knowledge, not payload integrity. Compare in constant time and never
  log it.
- **Return 2xx within 10 seconds.** Anything else is treated as a failure. Enqueue and acknowledge;
  do not process inline.
- **Be idempotent.** Delivery is at-least-once with exponential-backoff retries for up to 3 hours, so
  duplicates are expected. De-duplicate on `device.id` + `sequence_number` + `timestamp`.
- **Expect no filtering.** Every packet for every device in the organization is delivered. Filter on
  your side using `device.tags`.

## 2. Register the endpoint

Call `create-webhook-endpoint` (`POST /v1/org/{org_id}/webhooks`) with `name`, `url` (HTTPS, max 2000
chars) and `max_batch_size`. `max_batch_size` accepts 10–1000 and defaults to 100 — raise it to cut
HTTP overhead on high-volume fleets, lower it to cut per-request processing time.

The response is the only place the webhook secret appears. Store it in a secret manager immediately.

## 3. Test before you trust it

Call `test-webhook-endpoint` (`POST /v1/org/{org_id}/webhooks/{webhook_id}/test`). It sends an example
packet batch to the configured URL **in the same request format as live delivery**, which exercises
connectivity, TLS, and your token check in one shot. Do this before pointing production devices at it.

Confirm what is registered with `list-registered-webhooks` (`GET /v1/org/{org_id}/webhooks`).

## 4. Migrate or rotate

To move receivers: register the second endpoint (you now have 2 of 2), verify it with
`test-webhook-endpoint`, confirm traffic in metrics, then `delete-webhook-endpoint`
(`DELETE /v1/org/{org_id}/webhooks/{webhook_id}`) on the old one.

`update-webhook-endpoint` (`PATCH /v1/org/{org_id}/webhooks/{webhook_id}`) changes name, URL or batch
size in place — use it for secret rotation and small changes, not for cutovers.

## 5. Monitor

`get-webhook-metrics` (`GET /v1/org/{org_id}/webhook_metrics`) reports total webhook requests, success
rate, and interval breakdowns. A falling success rate means your receiver is 4xx/5xx-ing or timing
out and Hubble is burning its 3-hour retry budget — after that, packets are only recoverable by
polling `retrieve-organization-packets` within the 30-day history window.

## 6. Errors

Management calls return the standard envelope `{"code", "name", "description"}` as
`application/json` — not RFC 9457. Branch on the status and the `name` enum; `description` is
documented as subject to change. A 403 here means the key lacks `write-webhooks`.
