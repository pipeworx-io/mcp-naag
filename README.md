# @pipeworx/naag

Multistate **state attorney-general** enforcement — the coalition layer of US enforcement, from the National Association of Attorneys General (naag.org). Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1573+ live data sources.

`enforcement-actions` covers the federal layer (DOJ, SEC). This covers what groups of state AGs do together: 819 multistate cases, 63 coalition policy letters, and NAAG's own announcements.

**What it does not cover, stated plainly because routing depends on it:** a single state's formal AG opinions, that state's own docket, and anything one state did alone are not in NAAG's database and this pack cannot answer for them.

## Tools

- `naag_multistate_cases(query?, state?, lead_state?, case_type?, industry?, resolution?, federal_partner?, filed_from?, filed_to?, limit?, offset?)` — the multistate case database (819 records back to 2020, mostly antitrust and consumer protection). Returns case name, date, participating and lead states, conduct type, relief, industry, resolution, federal co-plaintiff and a complaint summary.
- `naag_case_detail(case)` — one case by numeric id or naag.org URL, with the untruncated summary and every classification.
- `naag_policy_letters(query?, state?, sent_from?, sent_to?, limit?, offset?)` — coalition comment letters to Congress, agencies and companies, with the signing states.
- `naag_press_releases(query?, topic?, published_from?, published_to?, limit?, offset?)` — NAAG press releases and Attorney General Journal articles.

State arguments accept a two-letter code or a full name (`"NY"` or `"New York"`). Category arguments accept the slug or the display name, and fall back to substring matching; the terms actually used come back in `terms_matched`.

## Auth

None. naag.org exposes a public WordPress REST API.

## Upstream behaviour worth knowing

**`X-WP-Total` and `X-WP-TotalPages` are wrong on this site, and wrong small.** Every custom post type reports `X-WP-Total: 2`, `X-WP-TotalPages: 1`, at any `per_page` — for `multistate-case` (really 819 records), `policy-letter` (really 63) and `publication` (really 15). A client that pages the documented way stops after one page and reports two results as the complete answer to "how many multistate cases name this company". It never errors, and the two records it returns are real. This pack never reads those headers: `total_matching` is counted by paging until a short page, and `total_is_exact` says whether that count is bounded.

**The state taxonomies have separate term ids for the same state.** `plaintiff-lead-state` calls New York 796; `participating-state` calls it 797. Resolving a state against the wrong vocabulary returns a different state's cases with a clean 200, so every lookup names its taxonomy.

**`type-of-relief` (259 terms) and `related-industry` (262) are free-tagged** and hold near-duplicates — "Injunction", "injunctive relief" and "Injunctive" are three distinct terms. An exact filter on one will miss cases tagged with another, so matching is substring-based across the vocabulary and the response reports which terms were used.

**Cost of an honest number.** naag.org answers in ~1.3s whatever you ask it, and `per_page` is hard-capped at 100, so an exact count is ten round trips. They run concurrently (bounded at 5) alongside the taxonomy fetch, which keeps a cold call to ~4-5s; a filtered query whose whole match set fits in one window costs a single request. The gateway caches results for a day — the case record is historical and gains a handful of rows a month.

**Filters are honest**, which is worth recording because it is not the norm: an unknown term id returns 0 rather than the unfiltered set, `offset` past the end returns empty rather than the first page, and a nonsense `search` returns 0. A zero here really is a zero.

## Data sources

- WordPress REST API: `https://naag.org/wp-json/wp/v2/` (`multistate-case`, `policy-letter`, `posts`, plus the taxonomy endpoints)
- Publisher: National Association of Attorneys General, <https://www.naag.org/>

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "naag": {
      "url": "https://gateway.pipeworx.io/naag/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/naag/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1573+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "naag": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-naag"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-naag
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Naag data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
