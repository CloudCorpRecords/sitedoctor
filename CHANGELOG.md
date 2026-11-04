# Changelog

## 0.1.0 (2026-10-27)

- First release: a forwarding client for the DrUrls remote MCP server, serving the 10 captured tools locally for tools/list and forwarding every tools/call upstream over a streamable-HTTP JSON-RPC session (initialize, notifications/initialized, tools/call, honouring the Mcp-Session-Id header when issued).
- Tools: `list_tester_offers`, `list_sites`, `get_site_report`, `find_issues`, `check_google_visibility`, `get_improvement_plan`, `start_scan`, `get_scan_status`, `generate_report`, `free_page_check`. This is a labelled subset of the captured 11 tools: `hire_tester` is omitted under the SAFETY rule because it places a paid order.
- When `DRURLS_API_KEY` is set, an `Authorization: Bearer <key>` header is sent on every upstream request (labelled reconstruction; the capture was keyless).
- Upstream results pass through verbatim; upstream failures surface as `sitedoctor error:` text results.
- One bin serves the CLI (one subcommand per tool, plus `--version`) and the stdio MCP server (`mcp` subcommand). Upstream endpoint overridable via `DRURLS_MCP_URL`.
