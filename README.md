# sitedoctor

sitedoctor is a small local bridge to DrUrls, a hosted
service that audits websites for search visibility and
on-page health. Instead of crawling anything itself,
this package stands in front of the service's remote
MCP endpoint: it presents the upstream tool list
locally, exactly as captured, and passes each call
through, returning whatever the upstream returns.

That split matters in practice. Scores, issue groups,
visibility verdicts and fix plans all come from
DrUrls' own crawls and data, so an answer you get
through sitedoctor is the service's answer, not a
local approximation. The bridge adds no scoring of
its own and rewrites nothing on the way back.

## Tools

This build serves a **labelled subset** of the
captured surface: 10 of the 11 tools seen in the live
capture. The omitted tool is `hire_tester`, which
places a paid order for a human test; purchase
execution is out of scope for this program, so the
tool is deliberately absent, and a test asserts that
absence. `list_tester_offers` remains — browsing the
catalogue is a read, ordering from it is not.

- `list_tester_offers` — browse the human-testing
  catalogue: mobile UX on a real handset, checkout
  runs, screen-reader passes, translation review and
  similar offers, each with its credit price,
  turnaround and deliverables. No arguments.
- `list_sites` — every site in the organisation with
  its latest completed audit: score, grade, issues,
  pages crawled, last scan time, and whether a scan
  is running. No arguments.
- `get_site_report` — a deeper look at one site
  (`domain`, required): latest score, issue counts
  by severity, the most frequent issue types, and
  social-network presence.
- `find_issues` — search open issues across all
  sites, or narrow with `domain` and with
  `severity` (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`
  or `INFO`). Results are grouped by site and issue
  type, with affected-page counts and an example.
- `check_google_visibility` — whether Google can
  index each site: noindex tags, robots blocks,
  sitemap errors, canonical problems, redirect
  chains and broken internal links from the last
  crawl. Optional `domain` for a single site.
- `get_improvement_plan` — the highest-impact fixes
  right now, ranked by severity and reach, with
  how-to-fix guidance. Optional `domain`.
- `start_scan` — launch a fresh audit of a site
  (`url`, required). The site is created in the
  organisation if needed, and the upstream plan
  quota is enforced upstream.
- `get_scan_status` — what is running now plus the
  last 10 finished or failed scans, with scores and
  errors. No arguments.
- `generate_report` — compile a full SEO report for
  one site (`domain`), or, without a domain, for
  the whole organisation.
- `free_page_check` — a free single-page audit of
  any public URL (`url`, required), with no account
  needed at the upstream, plus an `upgrade` field
  describing what the authenticated tools add.

## Configuration

- `DRURLS_MCP_URL` (optional) — override the
  upstream endpoint. Defaults to
  `https://drurls.com/mcp`. (The override hook is
  this rebuild's own labelled reconstruction.)
- `DRURLS_API_KEY` (optional) — when set, every
  upstream request carries
  `Authorization: Bearer <key>`. When it is not set,
  no auth header is sent, matching the keyless
  capture this rebuild is based on. (The header
  behaviour is a labelled reconstruction; the
  registry record declares an Authorization header
  for the remote, but the capture was taken without
  one.) Account-scoped tools are served upstream
  under the caller's own credential.

### Forwarding mechanics

On the first tool call the client opens a
streamable HTTP session upstream: a JSON-RPC
`initialize` request (protocolVersion `2025-03-26`,
clientInfo name `sitedoctor`), then a
`notifications/initialized` notification, then the
`tools/call` request carrying the caller's
arguments unchanged. When the upstream answers
`initialize` with an `Mcp-Session-Id` header, that
value rides on every later request, and the session
is reused for the life of the process. Replies may
arrive as plain JSON or as server-sent events; both
are read, and the message with the matching id wins.

Successful upstream results pass through verbatim,
including any `isError` flag the upstream sets and
any extra fields it adds. Failures — a JSON-RPC
error, a non-2xx status, an unreadable response, or
a request that cannot be completed — come back as
an `isError` result whose text starts with
`sitedoctor error:` (this rendering is this
rebuild's own labelled reconstruction).

## CLI

Each tool is also a subcommand taking one JSON
argument string (omit it for tools that take none):

```bash
npx @reneturcios/sitedoctor list_sites
npx @reneturcios/sitedoctor get_site_report '{"domain":"example.com"}'
npx @reneturcios/sitedoctor free_page_check '{"url":"https://example.com"}'
npx @reneturcios/sitedoctor --version
```

Exit codes: `0` on success, `1` on an `isError`
result, `2` on a usage error.

## MCP server

```bash
npx -y @reneturcios/sitedoctor mcp
```

MCP client config:

```json
{
  "mcpServers": {
    "sitedoctor": {
      "command": "npx",
      "args": ["-y", "@reneturcios/sitedoctor", "mcp"]
    }
  }
}
```

## Docker

```bash
docker run -i ghcr.io/cloudcorprecords/sitedoctor
docker run --rm ghcr.io/cloudcorprecords/sitedoctor --version
```

The container's default command is `mcp`, so the first
form is a stdio MCP server; any other arguments reach
the CLI normally.

## Development

```bash
npm install
npm run build
npm test
npm run dev
```

The test suite runs entirely against a stubbed
JSON-RPC upstream — no live network calls.

## License

MIT — see `LICENSE`.
