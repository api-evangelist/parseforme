---
name: Parse a document and read back its fields
description: Upload a document (file or URL), wait for the parse, and read the typed, confidence-scored fields.
api: openapi/parseforme-openapi-original.json
operations: [V1MeController_me, V1CatalogController_documentTypes, V1DocumentsController_create, V1DocumentsController_get]
generated: '2026-09-03'
method: generated
---

# Parse a document with ParseForMe

Auth: every request carries `Authorization: Bearer pfm_live_...` (a workspace API key). All failures are
an opaque 401. Base URL: `https://api.parseforme.com` (paths already carry `/v1`).

1. **Connection test** — `GET /v1/me` (`V1MeController_me`) returns the workspace, key label and token
   balance. Parsing spends ~1 token per page; a zero balance means `402 INSUFFICIENT_BALANCE` later.
2. **Know the fields first** — `GET /v1/document-types` (`V1CatalogController_documentTypes`) lists the
   nine kinds with their dotted field paths and `lineItemPath`. Build any mapping against this, not
   against one sample document.
3. **Send the document** — `POST /v1/documents` (`V1DocumentsController_create`) with EITHER a multipart
   `file` OR a JSON `{"sourceUrl": "https://..."}` — exactly one, or `400 INVALID_SOURCE`. Accepted:
   PDF, DOCX, PNG, JPEG, TIFF, WebP up to 20 MB. Omit `kind` to auto-detect.
   - ALWAYS send an `Idempotency-Key` header (`^[A-Za-z0-9_\-:.]{1,128}$`) built from something stable
     in your trigger: a replay returns the SAME document instead of spending tokens twice. There is no
     way to un-spend a parse.
   - Optional `?wait=1..25` holds the response up to that many seconds for a terminal status.
4. **Read the result** — `GET /v1/documents/{id}` (`V1DocumentsController_get`). `result` appears once
   `status` is `parsed`; `result.fields` and `result.confidence` are parallel maps keyed by dotted
   paths. Route low-confidence values to human review. Terminal failures: `failed`, `rejected`,
   `infected` (see `failureReason`).
5. **Respect the limits** — 60 req/min, 10 document-creates/min, 500/day per key; on 429 honour
   `Retry-After` and read the applied limit from `RateLimit-Policy` (`q=`). Errors always come as
   `{ "error": { "code", "message", "requestId" } }` — quote `requestId` (also the `X-Request-Id`
   header) to support.
