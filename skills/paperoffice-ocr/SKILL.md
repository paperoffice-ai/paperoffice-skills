---
name: paperoffice-ocr
description: Run AI-OCR on PDFs and images with PaperOffice — plain text, line boxes (grid) or complete layout with tables — and pick the right mode, model and lane. Use when the user needs text from a scan or PDF, bounding boxes, table detection, handwriting or form recognition, or a searchable PDF, and PaperOffice is the OCR engine.
license: MIT
---

# PaperOffice AI-OCR

## Endpoint

```
POST https://api.paperoffice.ai/latest/job/add/paperoffice_aiocr___generate
  multipart: file=@scan.pdf            (or source_url=https://… or JSON files:[…])
             ocr_mode=text|grid|complete
             processing_lane=instant | no_sla
  Authorization: Bearer $PAPEROFFICE_API_KEY
```

Result: `payload = body.result ?? body.job_result` → `payload.pages[]` (or `payload.output.pages`), each with `text`, `bounding_boxes`, in `complete` also `table_data` / `layout_data`.

Handwriting, filled customer forms, government forms and US tax forms (W-2, 1040, 1099) run on this same pipeline. Invoices, ID cards, receipts and other **structured** documents do **not** — those are IDP (`/job/add/workflow` with `idp_collection`), see the invoice skill.

## Choose the mode

| `ocr_mode` | Returns | Use for |
|-----------|---------|---------|
| `text` | plain text per page, boxes | full-text search, feeding an LLM, cheapest |
| `grid` | text + precise line bounding boxes | highlighting, redaction, click-to-source UIs |
| `complete` | text + boxes + tables + layout | table extraction, layout-faithful conversion |

Bounding boxes exist in every mode; `complete` adds the table/layout extras. If `ocr_mode` is omitted, it is inferred from `model` (`basic` → text, `premium` → grid, `ultra` → complete).

## Choose the lane

`processing_lane` is the Start-SLA and credit factor: `no_sla` ×1 (default, batch) · `sla_24h` ×1.5 · `sla_12h` ×2 · `sla_6h` ×3 · `sla_1h` ×4 · `instant` ×5 (someone is waiting). Guarantee = start of processing.

## Handle the response

- HTTP `200` → done inline. HTTP `202` → poll `GET /job/get/{job_id}` every 5–10 s until `job_status` is `completed` (free) or `failed`.
- OCR results on `job/get` live in **`job_result`**, not `result`.
- `job/get` defaults to `compact=true` — for full tables/layout use `compact=false`.
- Multi-file: keys `file`, `file_1`, `file_2`… or `files`.

## Searchable PDF

To get a text-layer PDF back instead of JSON, use the PDF Studio pipeline documented in llms.txt (`/job/add/pdfstudio___…`); recipe: `https://github.com/paperoffice-ai/cookbook/tree/main/document-ai/ocr/searchable-pdf`. Download via `GET /job/download/{token}` when the payload contains a download token.

## Documents already in PaperOffice

For a stored document pass `pofid` instead of a file, or read the existing text layer without a new job: REST `GET /documents/ocr-search-hits` (term → page + rectangle) or MCP `po_documents_text_get`. Re-OCR only when the text layer is missing or the user asks for a different mode.

## Do not

- Use OCR for invoices/receipts/IDs when structured fields are wanted (IDP does OCR + fields in one call).
- Loop on `202` without a delay; 5–10 s between polls.
- Assume `result` for OCR polls; it is `job_result`.

Recipes: `https://github.com/paperoffice-ai/cookbook/tree/main/document-ai/ocr` and `getting-started/first-ocr`.
