# PaperOffice API — Reference Notes

Condensed from `https://api.paperoffice.ai/latest/docs/llms.txt`. When in doubt, the live guide wins.

## Two pipelines, not one

| Goal | Endpoint | Discriminator |
|------|----------|---------------|
| Standalone OCR | `POST /job/add/paperoffice_aiocr___generate` | `ocr_mode` = `text` \| `grid` \| `complete` |
| Structured IDP (invoice, ID card, receipt, contract, custom fields…) | `POST /job/add/workflow` | `idp_collection` (e.g. `invoice`) + optional `template` |

Do not send `ocr_mode` and `idp_collection` on one slug. An IDP workflow already runs OCR and returns both `pages_idp` and `pages_aiocr` (with bounding boxes).

Handwriting, government forms and US tax forms are OCR pipeline jobs (`paperoffice_aiocr___generate`), not IDP.

## Upload field names

Multipart keys matching `file`, `files`, `file_1`, `file_2`… all work. Alternatives: `pofid` (existing document) or, for OCR and most non-workflow jobs, `source_url` / JSON `files` URL array. `/job/add/workflow` does **not** accept JSON URL arrays.

## Invoice field paths (`idp_collection=invoice`)

Keys have a leading underscore.

```
result.pages_idp[0].suggested_fields._supplier_name.value
result.pages_idp[0].suggested_fields._total_amount.value        display, e.g. "1.699,32 EUR"
result.pages_idp[0].suggested_fields._total_amount.value_raw    numeric, e.g. 1699.32
_invoice_number  _invoice_date  _net_amount  _vat_amount  _customer_name  _line_items
result.pages_aiocr.pages.{page}.bounding_boxes
```

`suggested_fields` is either an object keyed by field or an array of records (`category`/`key` = field name). Handle both. `VALUE_MISSING` means "not found" — treat as empty.

Field catalogue: `GET /documents/idp-collections-list?compact=false`.

## `model` and OCR depth

- API values are `basic-pro` / `basic-pro-max` (UI says basic-per). For `invoice`, `premium` is capped to `basic-pro-max` — expected.
- Workflow OCR depth follows `billing_tier`: `basic` → text, `premium` → grid, `ultra` → complete; omitted → complete.
- Standalone OCR infers `ocr_mode` from `model` when `ocr_mode` is omitted.

## Document management (DMS) essentials

| Task | Endpoint |
|------|----------|
| List workspaces the token can see | `GET /documents/workspaces-list` |
| Upload into a workspace | `POST /documents/document-put` multipart `file` + `workspace_id` → `results[]` |
| Full-text / hybrid search | `GET /documents/document-search` (workspace required) |
| List documents | `GET /documents/documents-list` |
| Download original | `GET /documents/document-download/{pofid}` |
| Trigger AI-DMS processing on a stored document | `POST /documents/document-process/{pofid}` |
| Delete | `POST /documents/document-delete` (`mode=trash` default where the tier has a trash; otherwise permanent) |
| Generate a PDF from Markdown/HTML | `POST /document_generation/create-from-content` |

Destructive headless calls need target confirmation: workspace delete → `confirm_delete=true` + `confirm_name`; empty trash → `confirm=true` + `confirm_count`; legal-hold release → `confirm=true` + `confirm_pofid`.

## Rate limits

Minimum on all tiers: 5/s, 30/min, 100/h, 500/day per token. Read `RateLimit-*` headers; back off on `429`.

## Free (no token, rate-limited) endpoints

`GET /health`, `GET /vat/rates`, `GET /weather`, `POST /currency_exchange/get_rates`, `POST /ip2location/*`. With a token these bill the 5-credit minimum.
