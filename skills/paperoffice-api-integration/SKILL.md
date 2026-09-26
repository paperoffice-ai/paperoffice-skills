---
name: paperoffice-api-integration
description: Integrate the PaperOffice AI REST API into an application — authentication, the job/add → job/get lifecycle, Start-SLA lanes, response envelopes, error handling and credits. Use whenever code is written against api.paperoffice.ai, when the user mentions PaperOffice, a po_ut_/po_gt_/po_sk_ token, OCR/IDP/DMS via REST, or asks how to call a PaperOffice endpoint.
license: MIT
---

# PaperOffice API Integration

## Read the guide first

Before writing any code, read the machine-readable API guide completely:

```
https://api.paperoffice.ai/latest/docs/llms.txt
```

Use only endpoints, parameters and field names that appear there. For exact request/response samples of a single endpoint, consult the live Postman collection `https://api.paperoffice.ai/latest/docs/postman`. Do not invent endpoints; PaperOffice is not a REST-CRUD API (mutations are `POST` with action paths such as `/documents/document-delete`).

## Setup

| Item | Value |
|------|-------|
| Base URL | `https://api.paperoffice.ai/latest` |
| Header | `Authorization: Bearer <token>` |
| Token in code | read from the environment variable `PAPEROFFICE_API_KEY`, never hard-code it, never ship it to a browser |
| Get a token | free account at `https://app.paperoffice.ai/en/register/`, then **Account → API** |

Token types: `po_ut_` user token (sees the whole account), `po_gt_` group token (limited to one group's workspaces), `po_sk_` system key (server-to-server, REST only), `po_pk_` publishable key (browser only, origin-locked). Recommend `po_ut_` for backend code.

## The one pattern: job/add → job/get

Long-running work (OCR, IDP, PDF tools, TTS…) goes through the job pipeline.

```
POST /job/add/<pipeline>      multipart: file=@doc.pdf  + parameters
  → 200  { status:"success", job_id, result | job_result }   finished inline
  → 202  { job_id, poll_url, max_wait_seconds }              still running
GET  /job/get/<job_id>
  → 200  { job_status: queued|processing|completed|failed, result | job_result }
```

Rules:

1. `202` is **not** an error. Poll `GET /job/get/{job_id}` every 5–10 s until `job_status` is `completed` or `failed`. Polling is free.
2. Always resolve the payload as `body.result ?? body.job_result` (workflow/IDP → `result`, OCR → `job_result`).
3. The pipeline slug uses triple underscores: `paperoffice_aiocr___generate`, `workflow`, `pdfstudio___compress_pdf`. Dot notation returns `400 JOB_CONFIG_INVALID`.
4. Use `processing_lane` to choose the Start-SLA: `no_sla` (×1, default) · `sla_24h` (×1.5) · `sla_12h` (×2) · `sla_6h` (×3) · `sla_1h` (×4) · `instant` (×5). The multiplier applies to credits; the guarantee is the *start* of processing. For interactive tools use `instant`; for batch use `no_sla` or `sla_24h`.
5. Default `client_wait=true` holds the connection (typically 20 s to several minutes). Set `client_wait=false` to get a `job_id` immediately.
6. `GET /job/get` defaults to `compact=true`; pass `compact=false` when full OCR tables/layout are needed.

Reference implementation of the polling helper in three languages: `https://github.com/paperoffice-ai/cookbook/tree/main/getting-started/async-job-polling`.

## Credits

Every billable call with a token costs at least 5 credits; rejected requests cost nothing. Free: `GET /health`, `GET /job/get`, `/billing/*`, `/docs/*`. Check `_billing.job.credits_billed` on billed responses. Prices: `GET /billing/plans` or `https://paperoffice.ai/en/pricing/`.

## Error handling

| HTTP | Code | What to do |
|------|------|-----------|
| 401 | `NOT_AUTHENTICATED` | no Bearer header sent |
| 401 | `TOKEN_NOT_FOUND` / `INVALID_TOKEN` | token wrong, revoked or malformed |
| 402 | `INSUFFICIENT_CREDITS` | account out of credits; `BUDGET_EXHAUSTED` = this token's own budget |
| 403 | `GROUP_RESTRICTION` | the token's group lacks that module; use a user token |
| 403 | `WORKSPACE_ACCESS_DENIED` | workspace belongs to another group; do not retry |
| 400 | `TARGET_CONFIRMATION_REQUIRED` | destructive call needs `confirm_name` / `confirm_count` / `confirm_pofid` |
| 404 | `DOCUMENT_NOT_FOUND` | wrong pofid / documents_id |
| 429 | `RATE_LIMIT_EXCEEDED` | wait for `Retry-After` |
| 500 | `INTERNAL_ERROR` | retry with exponential backoff |

Full table: llms.txt → "Standard Error Codes".

## Checklist before handing over code

- [ ] `llms.txt` read; every endpoint used exists there
- [ ] Token from `PAPEROFFICE_API_KEY`, no literal in code or logs
- [ ] `202` handled by polling, `result ?? job_result` resolved
- [ ] `processing_lane` chosen deliberately and documented
- [ ] Error codes above mapped to user-facing messages
- [ ] Destructive calls (delete workspace, empty trash, legal-hold release) carry the target confirmation

## Additional resources

- Endpoint families, invoice field paths and OCR modes: [reference.md](reference.md)
- Copy-paste recipes (Python, Node, curl): `https://github.com/paperoffice-ai/cookbook`
