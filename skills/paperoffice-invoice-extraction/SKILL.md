---
name: paperoffice-invoice-extraction
description: Extract structured fields (supplier, totals, invoice number, dates, VAT, line items) from invoice PDFs and scans with PaperOffice IDP, validate them and hand them to accounting or a CSV. Use when the user wants invoice data extraction, accounts-payable automation, DATEV export, or asks PaperOffice to "read" an invoice — via REST or the PaperOffice MCP tool po_extraction_invoice.
license: MIT
---

# PaperOffice Invoice Extraction

## Which pipeline

Invoices are **structured IDP**, not plain OCR. One call does OCR and extraction together:

```
POST https://api.paperoffice.ai/latest/job/add/workflow
  multipart: file=@invoice.pdf
             idp_collection=invoice
             processing_lane=instant        (interactive) | no_sla (batch)
  Authorization: Bearer $PAPEROFFICE_API_KEY
```

Via MCP (lanes `/dms`, `/cursor`, `/claude`, `/grok`): `po_extraction_invoice` with **one** source — `pofid` (document already stored; find it with `po_documents_search`), `upload_id` (from `po_documents_upload_url_get` after the `PUT`), or a public `file_url`. Optional `idp_collection`, `processing_lane`, `client_wait`. The tool starts a job; when the answer carries a `job_id`, poll with `po_job_get` until `job_status` is `completed`.

`invoice` is the full 22-field collection and the default choice. Pick a variant only when the user's context calls for it:

| `idp_collection` | When |
|------------------|------|
| `invoice_quick_capture` | only vendor, total, date, number needed — cheapest |
| `invoice_eu_vat` / `invoice_ch_vat` / `invoice_at_b2b` / `invoice_ae_vat` | regional VAT rules |
| `invoice_de_xrechnung` / `invoice_de_zugferd` / `invoice_int_en16931` | e-invoice standards (31–33 fields) |
| `invoice_fr_facturx` / `invoice_es_facturae` / `invoice_it_fatturapa` | national e-invoice formats |
| `invoice_3way_matching` | match against purchase order and delivery note |
| `accounting_datev` | booking record for DATEV |
| `hotel_invoice` / `receipt` / `cash_receipt` | not a commercial invoice |

The live catalogue (69 collections, field lists) is `GET /documents/idp-collections-list?compact=false`; read it before naming a collection the user has not mentioned.

## Read the result

Resolve `payload = body.result ?? body.job_result`, then:

```
page = payload.pages_idp[0]                     # one entry per page
f    = page.suggested_fields                    # object keyed by field OR array of records
supplier = f._supplier_name.value
total    = f._total_amount.value_raw            # numeric (1699.32); .value is the display string
number   = f._invoice_number.value
date     = f._invoice_date.value
net, vat = f._net_amount.value_raw, f._vat_amount.value_raw
items    = f._line_items                        # table
boxes    = payload.pages_aiocr.pages["1"].bounding_boxes   # for highlighting
```

- Field keys start with an underscore. `vendor` / `total_amount` do not exist.
- If `suggested_fields` is an array, find the record whose `category` or `key` equals the field name.
- `VALUE_MISSING` = not found. Store empty, never as a business value.
- `model` in the response reads `basic-pro-max` even when `premium` was requested — expected for the `invoice` collection.

## Validate before booking

1. `net + vat ≈ total` (tolerance 0.01). If not, flag the document for human review instead of correcting it silently.
2. Currency from `_total_amount.value` (e.g. `EUR`); do not assume.
3. Duplicate check: same `_supplier_name` + `_invoice_number` already processed → skip and report.
4. Confidence: when the payload carries `confidence` per field, treat < 0.7 as "review".

Present flagged documents to the user in a table (file, supplier, number, total, reason) and continue with the rest.

## Batch pattern

- Submit with `client_wait=false`, collect `job_id`s, poll `GET /job/get/{job_id}` every 5–10 s (free).
- Lane for batches: `no_sla` (×1) or `sla_24h` (×1.5). `instant` (×5) only for a human waiting at a screen.
- Write one CSV row per invoice: `file, supplier, invoice_number, invoice_date, net, vat, total, currency, status`.
- Keep the `pofid` from the response so the source document stays linked to the booking.

Reference implementation: `https://github.com/paperoffice-ai/cookbook/tree/main/document-ai/idp/invoice` (Python, Node, curl).

## Do not

- Run a separate OCR job "to be sure" — the workflow already returns `pages_aiocr` with boxes.
- Send `ocr_mode` together with `idp_collection`.
- Guess field names; the catalogue is `GET /documents/idp-collections-list?compact=false`.
- Post totals to accounting when the arithmetic check fails.
