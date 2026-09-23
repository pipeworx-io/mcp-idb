# @pipeworx/idb

Inter-American Development Bank (IDB) project procurement — live bidding
notices, contract awards, and a 5-multilateral-development-bank cross-debarment
check, all sourced from IDB's keyless CKAN open-data portal.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1663+ live data sources.

## Tools

- `idb_search_notices(...)` — search live IDB procurement notices (bids,
  expressions of interest, general/specific notices, award notifications)
  across 26 LAC countries; defaults to open (deadline not passed) notices.
- `idb_get_notice(notice_id)` — full record for one notice.
- `idb_search_awards(...)` — search ~159k awarded IDB project contracts by
  country, firm, sector, or year.
- `idb_check_mdb_debarment(name, ...)` — check a name against IDB's own
  debarment list plus the cross-debarment lists of the World Bank Group, Asian
  Development Bank, African Development Bank, and EBRD (one shared feed
  answering "is this vendor debarred" across five banks).

## Auth

Keyless.

## Data sources

- <https://data.iadb.org/dataset/project-procurement-bidding-notices-and-notification-of-contract-awards> — bidding notices + award notifications (~37,000 rows).
- <https://data.iadb.org/dataset/idb-project-procurement-contract-awards-data> — award-level contract detail (~159,000 rows; the dataset page offers a ~70MB CSV, but the same data is exposed via CKAN's Datastore, so every tool here queries it live with server-side SQL rather than downloading the file).
- <https://data.iadb.org/dataset/dataset-of-sanctioned-firms-and-individuals> — IDB debarment + WBG/ADB/AfDB/EBRD cross-debarment (~1,300 rows).

All three are queried through `data.iadb.org/api/3/action/datastore_search_sql`
(CKAN Datastore, keyless, no auth). Notes for anyone touching this pack:

- **Every resource here is CKAN Datastore-active** — `package_show` on each
  dataset reports `datastore_active: true`, which is what makes live SQL
  querying possible instead of downloading and caching the 70MB awards file
  ourselves. Check `datastore_active` before assuming a new IDB dataset needs
  downloading.
- **The plain `/files/download/<id>` URLs sit behind an AWS WAF bot challenge**
  (HTTP 202, `x-amzn-waf-action: challenge`, on a bare request) — irrelevant
  here since nothing calls them, but don't reach for them if extending this
  pack; the Datastore API is unaffected.
- **The upstream's own full-text search (`q=` on `datastore_search`) is
  accent-SENSITIVE.** `q=Gonzalez` (unaccented) returns zero rows against a
  debarment table full of "González" (with the accent) — verified live.
  `idb_check_mdb_debarment` never uses `q=`; it fetches the whole (small,
  ~1,300-row) debarment table and matches client-side with NFD-normalized,
  diacritic-stripped, token-substring comparison.
- **`type` in the notices table has inconsistent trailing whitespace** —
  `SELECT DISTINCT "type"` returns `"AWARD"`, `"AWARD "`, and `"AWARD   "` as
  three separate values. Always `TRIM()` before comparing, and match with a
  prefix `ILIKE`, never `=`.
- **`datastore_search_sql` has no bind-parameter support** — every string
  argument is quote-escaped (`'` → `''`) before interpolation. CKAN's SQL
  endpoint itself only accepts `SELECT` statements against a read replica.
- Many `idb-project-procurement-contract-awards-data` rows record
  `awarded_firm_name` as `"Not Available"` — this is by design for
  individual-consultant selections (`contract_type: "Individual Consultants"`),
  not a data gap.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "idb": {
      "url": "https://gateway.pipeworx.io/idb/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/idb/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1663+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/idb_search_notices \
  -H 'Content-Type: application/json' \
  -d '{"country":"Peru","open_only":true,"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/idb_search_notices`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "idb": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-idb"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-idb
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Idb data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
