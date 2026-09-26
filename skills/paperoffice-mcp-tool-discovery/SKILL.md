---
name: paperoffice-mcp-tool-discovery
description: Reach any of the 300+ PaperOffice API/MCP tools from a lane that lists only a short tool set, using the discovery tools po_mcp_tools_search → po_mcp_tools_schema → po_mcp_tools_call_read / po_mcp_tools_call_write. Use when a PaperOffice MCP server is connected and the needed capability (e-signature, tagging, versions, workflows, PDF tools, translation, analytics…) is not among the listed tools, or when the user asks "can PaperOffice do X".
license: MIT
---

# PaperOffice MCP Tool Discovery

## Why this exists

PaperOffice keeps `tools/list` short on purpose (14 tools on `/dms`, `/cursor`, `/grok`; 9 on `/chatgpt`) so the context stays small. The rest of the catalogue is reached through four discovery tools that are present on every lane:

| Tool | Effect | Purpose |
|------|--------|---------|
| `po_mcp_tools_search` | READ-ONLY | natural-language search over the catalogue → compact hits with `tool_id`, effect, one-liner |
| `po_mcp_tools_schema` | READ-ONLY | full input schema for one `tool_id` |
| `po_mcp_tools_call_read` | READ-ONLY | execute a read-only `tool_id`; write tools are rejected |
| `po_mcp_tools_call_write` | DESTRUCTIVE | execute a writing or destructive `tool_id` |

Only `/mcp-full` lists everything directly.

## The three-step loop

```
1. po_mcp_tools_search   { query: "send document for e-signature" }
      → matches[]: tool_id, one_liner, execute_via, annotations, annotation_reason, compact_schema
2. po_mcp_tools_schema   { tool_id: "<from step 1>" }
      → full inputSchema with required fields, enums, descriptions
3. po_mcp_tools_call_read  { tool_id, arguments }     when execute_via is po_mcp_tools_call_read
   po_mcp_tools_call_write { tool_id, arguments }     when execute_via is po_mcp_tools_call_write
```

Each match tells you how to run it: `execute_via` names the right call tool, `annotations.readOnlyHint` / `destructiveHint` / `openWorldHint` describe the effect, `annotation_reason` explains it in one sentence (e.g. "external delivery: delivers a document to an external recipient e-mail"). `compact_schema` lists the required arguments; step 2 gives the full schema.

Rules:

- Step 2 may be skipped only when `compact_schema` already contains every argument you intend to send. Argument names come from the schema, not from memory.
- Pick the match whose effect matches the intent. If the user wants to *look*, choose a `readOnlyHint: true` tool even when a writing tool also matches.
- `openWorldHint: true` means the call reaches outside PaperOffice (e-mail, SMS, signature request to a person). Confirm recipient and content with the user first.
- Before `po_mcp_tools_call_write` with a DESTRUCTIVE tool (delete, revoke, cancel, send to a person), restate the target and get the user's explicit confirmation. Some tools additionally require `confirm_name` / `confirm_count` / `confirm_pofid`; a missing value returns `TARGET_CONFIRMATION_REQUIRED` — fill it from the user's confirmation, never guess it.
- Search phrasing: describe the outcome ("mark document as final version", "who changed this document"), not the assumed tool name.
- If a search returns nothing useful, try one rephrasing, then tell the user the capability is not available on this lane instead of improvising with other tools.

## Results

Inner tool results come back as `structuredContent` plus a text summary. Billing appears as `_billing_summary` (credits billed, remaining) — mention credits only when the user asks or a call is unusually expensive. A `job_id` in the result means an asynchronous job: poll with `po_job_get` until `job_status` is `completed`.

## Lane quick reference

| Lane | Tools listed | Note |
|------|--------------|------|
| `/dms` `/cursor` `/grok` | 14 | Documents Operations catalogue + discovery |
| `/claude` | Directory set | no TTS |
| `/chatgpt` (`/openai`) | 9 | 5 core + 4 discovery, business documents only |
| `/mcp-full` | 300+ | everything, no discovery needed |

Auth on all lanes: OAuth 2.1 sign-in, or `Authorization: Bearer po_ut_… / po_gt_…`. `po_sk_` and `po_pk_` are rejected.

Setup files per client: `https://github.com/paperoffice-ai/paperoffice-mcp-setup`.
