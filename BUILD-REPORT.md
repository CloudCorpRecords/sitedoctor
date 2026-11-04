# BUILD REPORT — sitedoctor

Rebuild of `com.drurls/drurls` (remote-only) as
`@reneturcios/sitedoctor` 0.1.0 / `io.github.CloudCorpRecords/sitedoctor`.

## Fixture conflicts

**None.** The brief, the spec prose, and the fixture agree on all 10
tool names and every required field:

- `list_tester_offers` []; `list_sites` []; `get_site_report` [`domain`];
  `find_issues` []; `check_google_visibility` [];
  `get_improvement_plan` []; `start_scan` [`url`]; `get_scan_status` [];
  `generate_report` []; `free_page_check` [`url`].

The fixture's tool literals (name, description, inputSchema — including
the `severity` enum and every property description) are served verbatim
from `src/forward.ts`. The fixture carries no `title` fields for these
tools, and none were invented. Captured annotations are not reproduced,
per the brief (parity compares inputSchema).

## Labelled subset / omission

The live capture held 11 tools. This rebuild serves 10: `hire_tester`
(required: `offer_id`, `target_url`, `brief`; "Places an order for one
test from the tester catalogue") is **omitted under the SAFETY rule**
because it places a paid order. It was not added back. Registration
integrity is enforced by tests in two places: `TOOL_DEFS` /
`TOOL_NAMES` are asserted not to contain `hire_tester`, and
`handleCallTool` is asserted to reject `hire_tester` at the MCP layer
before any network hop. The subset and the omission are also stated in
the README and the CHANGELOG. `list_tester_offers` stays: browsing the
catalogue is a read.

## What was built

All 19 factory files under `builds/sitedoctor/`:

- `package.json` (`@reneturcios/sitedoctor` 0.1.0, bin
  `sitedoctor` → `dist/cli.js`, `mcpName`
  `io.github.CloudCorpRecords/sitedoctor`, deps exactly
  `@modelcontextprotocol/sdk ^1` + `zod ^3`; devDeps `@types/node ^20`,
  `tsx ^4`, `typescript ^5`)
- `server.json` (description exactly the prescribed 93-char string
  "SEO site audits, issue search, Google visibility checks and
  improvement plans for your sites."; npm entry carries
  `"version": "0.1.0"`; OCI entry
  `ghcr.io/cloudcorprecords/sitedoctor:0.1.0` carries **no** version
  field)
- `Dockerfile` (node:20-alpine both stages; label
  `io.modelcontextprotocol.server.name="io.github.CloudCorpRecords/sitedoctor"`,
  `ENTRYPOINT ["node", "/app/dist/cli.js"]`, `CMD ["mcp"]`)
- `tsconfig.json` (fleet-identical, 318 bytes), `.gitignore`,
  `package-lock.json`, `README.md` (fresh prose, including the labelled
  subset + `hire_tester` omission), `LICENSE` (MIT, Copyright (c) 2026
  Rene Turcios), `CHANGELOG.md` (single 0.1.0 entry, 2026-10-27)
- `src/index.ts`, `src/cli.ts` (`mcp` dispatch; `--version` → `0.1.0`;
  unknown → usage on stderr, exit 2), `src/mcp.ts`, `src/types.ts`,
  `src/forward.ts` (core; exports `VERSION = "0.1.0"`, `TOOL_DEFS`,
  `createClient`, `isKnownTool`)
- `test/forward.test.ts`
- `.github/workflows/{ci.yml, publish.yml, publish-container.yml,
  registry-publish.yml}` (container workflow: IMAGE
  `ghcr.io/cloudcorprecords/sitedoctor`, smoke assertion on server name
  `sitedoctor`)

## Design

- Forwarding client to the pinned `https://drurls.com/mcp`,
  overridable via `DRURLS_MCP_URL` (labelled reconstruction).
  tools/call is forwarded as a streamable-HTTP JSON-RPC client:
  initialize (protocolVersion `2025-03-26`, clientInfo name
  `sitedoctor`) → notifications/initialized → tools/call, honouring
  `Mcp-Session-Id` when issued; the session is established lazily and
  reused per client. Plain-JSON and SSE replies are both parsed; the
  matching-id message wins.
- When env `DRURLS_API_KEY` is set, `Authorization: Bearer <key>` is
  sent on every upstream request (labelled reconstruction; the capture
  was keyless); otherwise no auth header is sent. This is covered by
  paired tests (header present on all 3 session requests when set,
  absent on all when unset).
- Upstream results pass through verbatim (including `isError` and
  extra fields such as `structuredContent`). Upstream failures
  (JSON-RPC error, non-2xx status, malformed body, throwing or
  timing-out fetch) surface as `sitedoctor error:` text results
  (labelled reconstruction).
- Registration via the MCP SDK **low-level** `Server` API
  (`ListToolsRequestSchema` / `CallToolRequestSchema` handlers)
  serving the fixture literals exactly; no `McpServer` + zod
  registration. `src/mcp.ts` exports `createServer` /
  `startMcpServer` / `handleCallTool`; an unknown tool is rejected
  with `McpError` MethodNotFound and non-object arguments with
  `McpError` InvalidParams, before any network hop.
- Core takes injectable dependencies (`ForwardDeps`: `fetchFn`,
  `env`, `url`, `timeoutMs`, default 30000), so all tests run against
  a stubbed JSON-RPC upstream. No live network calls in tests; no
  live call claimed anywhere.

## Lockfile (Route 3, clean from birth)

Generated in a temp dir containing only the new `package.json`:
`npm install --package-lock-only --ignore-scripts --offline`, then
placed as `package-lock.json`. Verified: parses,
`name == @reneturcios/sitedoctor`, **128 package entries** (sha256
`2aa1f898c74b1977cee09e67d0011685ebd2f7a7ea00cff35a9a85f0465a39c3`,
identical for the temp-dir original and the placed copy). Then
`npm install --offline --ignore-scripts` in the build dir for
`node_modules`. No write-guard failure occurred; no /tmp staging
fallback for in-tree writes was needed beyond the Route 3 temp-dir
generation itself.

## Verification (all in the build dir)

- `npx tsc --noEmit` → exit 0
- `npm run build` → real `dist/` (cli.js 2346 B, forward.js 15415 B,
  mcp.js 2693 B, index.js 91 B, types.js 376 B, all non-empty, with
  `.d.ts` + source maps)
- `node dist/cli.js --version` → `0.1.0`; unknown subcommand → usage
  on stderr, exit 2
- `npm test` → **35 tests, 35 pass, 0 fail** (7 suites: registration
  integrity — served tools deep-equal the fixture's 10 tool objects
  in order, required fields per tool, `hire_tester` absent; session
  establishment/reuse/session-id handling; env URL override + Bearer
  header when the key is set / absent when unset; verbatim argument
  forwarding; verbatim result pass-through incl. upstream isError and
  SSE; JSON-RPC/init/malformed/HTTP/throw/timeout failures →
  `sitedoctor error:`; parseRpcBody; MCP-layer validation incl. a
  `hire_tester` rejection; CLI --version / unknown exit 2 / bad-JSON
  exit 2)
- Parity: `python3 harness/parity.py --original
  harness/fixtures/drurls-tools-original.json --rebuild
  "stdio:node dist/cli.js mcp" --out parity-out` (run in the build
  dir, original given as its absolute path, following serieslib's
  invocation shape) → **PASS, parity 100.0, 10/10 identical, no
  missing/extra/schema diffs** (see `parity-out/parity.json`)
- Consistency sweep: package.json version = server.json top-level =
  npm-entry version = OCI tag = core `VERSION` = CHANGELOG heading
  (0.1.0); package.json `mcpName` = server.json `name` = Dockerfile
  label; server.json description measured at 93 chars; OCI entry
  confirmed to have no `version` field.

Clean-room: all prose and code written fresh; only the fixture's tool
literals are served verbatim. Nothing published or pushed; NAMES,
SCOREBOARD and WALK were not edited.
