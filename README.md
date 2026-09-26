# PaperOffice Agent Skills

Skills teach an AI agent **how to work** with PaperOffice — which pipeline to call, how to read the result, when to ask before deleting. The [PaperOffice MCP server](https://github.com/paperoffice-ai/paperoffice-mcp-setup) supplies the tools; these skills supply the procedure.

Format: [Agent Skills](https://agentskills.io) — a folder with a `SKILL.md` (YAML frontmatter + instructions). Works in Claude Code, claude.ai, Cursor and every client that reads `skills/*/SKILL.md`.

## Skills

| Skill | Teaches | Trigger |
|-------|---------|---------|
| [`paperoffice-api-integration`](skills/paperoffice-api-integration/) | REST integration: auth, `job/add → job/get`, Start-SLA lanes, envelopes, error codes, credits | writing code against `api.paperoffice.ai` |
| [`paperoffice-invoice-extraction`](skills/paperoffice-invoice-extraction/) | IDP for invoices: the `workflow` pipeline, `_supplier_name` / `_total_amount` field paths, validation, batch CSV, the regional and e-invoice collection variants | invoices, accounts payable, DATEV |
| [`paperoffice-ocr`](skills/paperoffice-ocr/) | AI-OCR modes `text` / `grid` / `complete`, handwriting and forms, searchable PDF, `job_result` | text or boxes from scans and PDFs |
| [`paperoffice-dms-documents`](skills/paperoffice-dms-documents/) | Documents in workspaces via MCP: search, read, three-step upload, organise, delete with confirmation | "find / read / file / delete my documents" |
| [`paperoffice-mcp-tool-discovery`](skills/paperoffice-mcp-tool-discovery/) | Reaching all 300+ tools from a short lane: `po_mcp_tools_search → schema → call_read / call_write`, effect annotations | a capability is not in `tools/list` |

Every fact in the skills is taken from the live API guide `https://api.paperoffice.ai/latest/docs/llms.txt` and checked against the production MCP server.

## Install

**Claude Code** (as a plugin):

```
/plugin marketplace add paperoffice-ai/paperoffice-skills
/plugin install paperoffice-skills@paperoffice
```

**Claude Code** (single project): copy the folders you need into `.claude/skills/` of your project.

**claude.ai**: Settings → Capabilities → Skills → upload a zip of one skill folder (the folder must contain `SKILL.md` at its root).

**Cursor**: the same five skills ship inside the [PaperOffice Cursor plugin](https://github.com/paperoffice-ai/paperoffice-cursor-plugin) (`skills/`). Manually: copy a folder to `.cursor/skills/` in your project or `~/.cursor/skills/` for all projects.

**Other agents**: any runtime that reads `skills/<name>/SKILL.md` can load this repository directly.

## Skills need tools

Skills describe the procedure. For the agent to *act*, connect the MCP server or give your code an API token:

| Client | How |
|--------|-----|
| Claude Desktop / claude.ai | MCP `https://mcp.paperoffice.ai/claude` (OAuth sign-in) |
| Claude Code | `claude mcp add --transport http paperoffice https://mcp.paperoffice.ai/dms` |
| Cursor / Windsurf | plugin or MCP `https://mcp.paperoffice.ai/cursor` with `Authorization: Bearer <po_ut_…>` |
| ChatGPT | connector `https://mcp.paperoffice.ai/chatgpt` |
| Your code | `PAPEROFFICE_API_KEY=po_ut_…` and the API-integration skill |

Free account: [app.paperoffice.ai/en/register/](https://app.paperoffice.ai/en/register/). Token: **Account → API**. Full client configs: [paperoffice-mcp-setup](https://github.com/paperoffice-ai/paperoffice-mcp-setup).

## Related

- [Cookbook](https://github.com/paperoffice-ai/cookbook) — 38 copy-paste recipes in Python, Node and curl
- [API guide for agents (llms.txt)](https://api.paperoffice.ai/latest/docs/llms.txt)
- [MCP documentation](https://paperoffice.ai/en/developer/mcp/)
- [Help & FAQ](https://help.paperoffice.ai/) · [Support](https://paperoffice.ai/en/support/) · [Issues](https://github.com/paperoffice-ai/paperoffice-skills/issues)

## Contributing

Skills must stay factual: every endpoint, field name and error code has to exist in `llms.txt` or in the live `tools/list`. Open an issue with the transcript when a skill led the agent astray.

## License

[MIT](LICENSE)
