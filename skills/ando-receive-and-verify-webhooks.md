---
name: Receive and verify Ando webhooks
description: Create an Ando webhook endpoint, verify signed payloads, and test/replay deliveries.
api: openapi/ando-public-api-v1-openapi.json
operations: [createWebhookEndpoint, sendWebhookEndpointTestEvent, listWebhookDeliveries, replayWebhookDelivery, rotateWebhookEndpointSecret]
auth: Workspace API key (x-api-key header) to manage endpoints; signing secret to verify events
base_url: https://api.ando.so/v1
---

# Receive and verify Ando webhooks

Use this skill to durably receive Ando product events over HTTP POST. Delivery is at-least-once and unordered — receivers must be idempotent.

## Auth
Manage endpoints with a workspace API key (`x-api-key: ando_sk_...`). Verify inbound events with the endpoint **signing secret** (not an API key).

## Steps
1. **Create an endpoint** — call `createWebhookEndpoint` (`POST /webhook-endpoints`) with `{ name, url, enabled_events: [...] }`. Subscribe only to events you handle (e.g. `message.created`, `message.updated`, `webhook.test`). The response returns `data.signing_secret` **exactly once** — store it in a secret manager.
2. **Verify signatures** — on each delivery, compute `HMAC_SHA256(signing_secret, "<Ando-Timestamp>.<raw body>")` and compare the lowercase hex digest (constant-time) against any `v1` value in the `Ando-Signature` header (`t=<unix>,v1=<hex>`). Reject timestamps outside a 300s tolerance window. Use the raw request bytes — verify before JSON parsing.
3. **Ack fast** — persist/enqueue the event and return a 2xx within 10 seconds; process business logic asynchronously. Dedupe by `Ando-Event-Id`.
4. **Test** — call `sendWebhookEndpointTestEvent` (`POST /webhook-endpoints/{endpointId}/test`) to emit a `webhook.test` event.
5. **Inspect and replay** — call `listWebhookDeliveries` (`GET /webhook-deliveries`) with `endpoint_id`/status filters; call `replayWebhookDelivery` (`POST /webhook-deliveries/{deliveryId}/replay`) to re-send (a new delivery id, same event id).
6. **Rotate** — call `rotateWebhookEndpointSecret` (`POST /webhook-endpoints/{endpointId}/rotate-secret`) to roll the secret (returned once; both old and new `v1` values may sign during rotation).

## Conventions
- Non-2xx responses, network errors, and timeouts are retried — handlers must be idempotent (dedupe by `Ando-Event-Id`). See `asyncapi/ando-webhooks.yml` for the full event catalog and delivery headers.
- The `@andocorp/sdk/webhooks` subpath ships `verifyAndoWebhookSignature` and receiver helpers.
