---
name: Receive parse results by webhook
description: Register a signed webhook endpoint, verify deliveries, test it, and rotate its secret safely.
api: openapi/parseforme-openapi-original.json
operations: [V1WebhooksController_create, V1WebhooksController_test, V1WebhooksController_list, V1WebhooksController_rotate, V1WebhooksController_remove, V1CatalogController_meta]
generated: '2026-09-03'
method: generated
---

# Receive ParseForMe results by webhook

1. **Register** — `POST /v1/webhooks` (`V1WebhooksController_create`) with `url` (https on port 443,
   public hostname, no credentials, no IP literal), `events` (`document.parsed`, `document.failed`),
   optionally `includeResult: true` and a static `authHeader`. The response carries the signing secret
   (`pfm_whsec_...`) ONCE — store it now. Max 10 endpoints per workspace; every registration emails the
   workspace owners/admins by design.
2. **Allowlist the source** — `GET /v1/meta` (`V1CatalogController_meta`) returns `webhookEgressIps`;
   generate the firewall rule from it rather than hard-coding.
3. **Verify every delivery** — `X-ParseForMe-Signature: t=<unix>,v1=<hex>` is HMAC-SHA256 over
   `t + "." + RAW body`. Verify before parsing JSON, compare constant-time, reject `|now - t| > 300s`.
   During rotation two `v1` entries appear — either matching is valid. Use `X-ParseForMe-Delivery` as
   your idempotency key; it sits inside the signed bytes as the envelope `id`.
4. **Acknowledge fast** — answer 2xx within 10 seconds, then do the work. Retries: 5xx/408/425/429/no
   answer → up to 12 attempts over ~13 hours. A 410 disables the endpoint immediately; 50 consecutive
   failures disable it too. Delivery is at-most-once and unordered — keep `GET /v1/documents?since=` as
   the backstop and refetch the document rather than trusting event order.
5. **Test it** — `POST /v1/webhooks/{id}/test` (`V1WebhooksController_test`) sends a synthetic signed
   `document.parsed` with `test: true` and the nil-UUID document id. 5/min per workspace.
6. **Rotate** — `POST /v1/webhooks/{id}/rotate` (`V1WebhooksController_rotate`): new secret returned
   once; the old one keeps signing until `previousSecretExpiresAt` (24 h). Deploy, then drop the old.
   **Clean up** — `GET /v1/webhooks` to see what a key left behind; `DELETE /v1/webhooks/{id}`
   (`V1WebhooksController_remove`) stops deliveries immediately.
