---
name: Poll for parsed documents and export them
description: Windowed polling without a webhook, and exporting parsed documents as CSV, XLSX or QBO.
api: openapi/parseforme-openapi-original.json
operations: [V1DocumentsController_list, V1DocumentsController_get, V1DocumentsController_exportOne]
generated: '2026-09-03'
method: generated
---

# Poll and export with ParseForMe

1. **Poll with a window, not a cursor** — `GET /v1/documents` (`V1DocumentsController_list`) returns
   `items` newest-first and nothing else (no cursor). Filter `status`, `kind`, `since` (ISO 8601),
   `limit` (1–100, default 20). `since` filters `createdAt` — a document uploaded last week but parsed
   today will NOT resurface in a `status=parsed&since=` poll, so keep the window wide enough to cover a
   slow parse and deduplicate on `id`. Poll on `status`; documents have no `updatedAt`.
2. **Fetch the fields** — `GET /v1/documents/{id}` (`V1DocumentsController_get`) once `status` is
   `parsed`.
3. **Export** — `GET /v1/documents/{id}/export` (`V1DocumentsController_exportOne`) with `format`
   `csv`, `xlsx` or `qbo`, optionally `templateId` for a saved column layout (`format=qbo` REQUIRES a
   `templateId` or it answers `400 QBO_NEEDS_TEMPLATE`). The response is `{url, expiresAt, format,
   filename}` — the URL is short-lived, download promptly — or add `redirect=1` for a 302 straight to
   the file.
