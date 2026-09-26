---
name: paperoffice-dms-documents
description: Work with documents stored in PaperOffice workspaces — find, read, upload, create, organise and delete them safely, via the PaperOffice MCP tools (po_documents_*, po_workspaces_list) or the REST API. Use when the user asks to search their documents, read a stored file, upload or file away a document, list workspaces or folders, or clean up documents in PaperOffice.
license: MIT
---

# PaperOffice Documents (DMS)

## Mental model

- **Workspace** = the container a token can see (`po_workspaces_list` / `GET /documents/workspaces-list`). A group token (`po_gt_`) sees only its group's workspaces; other workspaces answer `WORKSPACE_ACCESS_DENIED` — do not retry, tell the user.
- **Document** = identified by `pofid` (string) and `documents_id` (int). Keep the `pofid`; it is the handle for every later call.
- **Folder** = a stapled bundle of documents (`po_documents_folders_list`), not a file-system directory.
- Every MCP tool description starts with its effect: `READ-ONLY.` · `WRITES DATA.` · `DESTRUCTIVE.` Read it before calling.

## Find and read

```
po_workspaces_list                         → pick workspace_id
po_documents_search  workspace_id, query   → documents[] with pofid, file_name, date, fields
po_documents_get     pofid                 → identity, processing state, trust summary
po_documents_text_get pofid                → existing OCR / full text (no new job, no OCR credits)
```

Always ask for or infer the workspace first; searching without one fails. Prefer `po_documents_text_get` over re-running OCR when the user wants the content of a stored document. Quote the document (file name, date, pofid) when answering from its text.

REST equivalents: `GET /documents/document-search`, `GET /documents/documents-list`, `GET /documents/document-download/{pofid}`, `GET /documents/ocr-search-hits` (term → page + rectangle).

## Upload and create

MCP (three steps):
1. `po_documents_upload_url_get` (`filename`, `content_type`) → `upload_url` (single-use, valid `expires_in` s, max `max_size_bytes`), `upload_id`, a ready `curl_command`
2. `PUT` the bytes to `upload_url` (Content-Type = the file's MIME type)
3. `po_documents_upload` with `upload_id` + `workspace_id` — on lanes where it is not listed: `po_mcp_tools_call_write` with `tool_id: "po_documents_upload"`. The response carries the new `pofid`; processing follows the workspace's AI-DMS tier.

No local file? `po_documents_upload` also accepts a public `file_url`, or, without `upload_id`, returns a browser upload page for the user.

REST (one step): `POST /documents/document-put` multipart `file` + `workspace_id` → `results[]` with pofid.

Generate a new PDF from text: `po_documents_create_from_content` (Markdown or HTML, workspace_id) / `POST /document_generation/create-from-content`.

Trigger processing for an already stored file: `POST /documents/document-process/{pofid}`.

## Organise

Tags of one document: `po_documents_tags_list` (`pofid`). Folders of a workspace: `po_documents_folders_list` (`workspace_id`). Adding tags, versions (`po_documents_upload` with `version_of_pofid`), document types, retention, edit-session locks, audit trail: discover the inner tools with `po_mcp_tools_search` (e.g. "add tag", "new version", "audit trail") and call them via `po_mcp_tools_call_read` / `po_mcp_tools_call_write` — see the tool-discovery skill.

## Delete — carefully

- `po_documents_delete` / `POST /documents/document-delete`: `mode=trash` (default where the tier has a trash) or `mode=instant` (permanent). On tiers without a trash every delete is permanent. Check `_capabilities.trash_enabled` on the workspace before promising a restore.
- Workspace delete, empty trash and legal-hold release are headless-destructive and require the exact target: `confirm_name` (workspace name), `confirm_count` (documents in trash), `confirm_pofid`. Missing or wrong → `400 TARGET_CONFIRMATION_REQUIRED`.
- Never delete on a vague request. Restate what will be deleted (count, workspace, permanence) and let the user confirm.
- Documents under WORM retention or legal hold cannot be deleted; report the reason instead of forcing it.

## Answer format

When reporting documents, list: file name · date · workspace · pofid · one-line what-it-is (from fields or summary). Never expose raw tokens or presigned URLs in chat transcripts that will be shared.

Recipes: `https://github.com/paperoffice-ai/cookbook/tree/main/document-ai/dms` · MCP setup: `https://github.com/paperoffice-ai/paperoffice-mcp-setup`.
