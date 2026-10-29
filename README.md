# @pipeworx/alabama-code

Code of Alabama 1975 (Alabama state statutes) — full section text by
citation, real full-text topic search, and amendment history.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `al_statute(citation)` — full current text of one Code of Alabama section
  (e.g. `"13A-6-2"`, murder), with the section title, the Legislature's own
  amendment-history citation list (`amendment_history`), repealed/transferred
  disposition, previous/next-section neighbors, and — when an Act has
  already been enacted but is not yet in force — a forward-looking
  `pending_version` carrying its future effective date and text.
- `al_search(query, limit?)` — REAL full-text search over actual section
  body text (e.g. `"landlord"`, `"concealed pistol permit"`), run live
  against the Alabama Legislature's own search index. Unlike several sibling
  state-code packs, this is not limited to captions/catchlines.

## Auth

Keyless.

## Data sources

- `https://alison.legislature.state.al.us/graphql` — the Alabama
  Legislature's own public GraphQL API, confirmed keyless and reachable from
  Cloudflare Worker egress (edge-probed, not just laptop-tested). Both tools
  call it live, per request; nothing is baked.
  - `query codeOfAlabamaSection($displayId: String!)` — one section's current
    (and, when present, pending-future) text, title, and amendment-history
    string, plus `codeOfAlabamaNeighbors` for previous/next citation.
  - `query searchCodeOfAlabama($query: String!, $limit: Int, $offset: Int)` —
    full-text search (`fullTextQuery` argument) over section body `content`,
    not just titles; returns a `count` of total matches independent of
    `limit`.

**Trap avoided: there are TWO graphql-shaped hosts, only one of which serves
statute data.** `cms.alison.legislature.state.al.us` also answers POST
`/graphql` (it serves the site's marketing/CMS content — page copy, the
logo asset, etc.) and returns a *plausible* 400 (`Cannot query field
"codesOfAlabama" on type "Query"`) for every statute query, because it's a
different schema on a different service, not an error about this pack's
query shape. The real statute API lives on the main site's own domain,
`alison.legislature.state.al.us/graphql` (found in a plain inline URL
reference inside the page's own HTML — not something a build-config
guess would find — and confirmed by pulling the query text for
`codeOfAlabamaSection`/`searchCodeOfAlabama` straight out of the Next.js
chunk bundle that defines them).

**Trap avoided: the GraphQL endpoint answers HTTP 200 on a query-level
error.** A malformed or schema-mismatched query comes back as a 200 with a
body-level `errors` array — not a 4xx. `al_statute`/`al_search` read the body
and check for `errors` explicitly and throw a LOUD error naming the GraphQL
message; neither tool ever reshapes that into `found: false`.

## Capabilities — three available, one is forward-looking instead of backward

- **Citation lookup** — `al_statute`.
- **Topic/full-text search** — `al_search`, over real section body text
  (verified live: `"landlord"` matches 88 sections, including ones whose
  title never says "landlord").
- **Amendments/enactment history** — AVAILABLE. Every in-force section's
  `history` field from the API is the Legislature's own Acts citation list
  (e.g. `"(Acts 1977, No. 607, p. 812, §2005; Act 2016-29, §2.)"`), returned
  as `amendment_history` with `&sect;`/`&amp;` entities decoded.
- **Historical version** (the text as it read before an amendment) — NOT
  AVAILABLE going backward. ALISON publishes the CURRENT in-force text only:
  no per-year archive, no "view as of" date picker, and `codeOfAlabamaSection`
  never returns more than one *past* row for a citation (checked live on
  several amended sections). It DOES sometimes return a *future* row — when
  the Legislature has already enacted an amendment that is not yet in force,
  the same query returns a second version whose `effectiveDate` is the date
  it takes over. `al_statute` surfaces that as `pending_version` (verified
  live on `26-2-2`, effective 2027-01-01 under Act 2026-488) and says
  explicitly that this is forward-looking, not a historical archive.

## Source and shape — read this before touching the parser

alison.legislature.state.al.us ("ALISON") is a Next.js single-page app — the
August state-law survey (`docs/state-law-probe.md`) correctly flagged the
HTML shell as "client-rendered or bot-walled" (it carries no statute text at
all), and stopped one layer too early, the same way the survey was wrong on
South Dakota and Utah. The SPA is fed by a keyless, public **GraphQL** API
(not REST, unlike South Dakota/North Dakota) — found by downloading the
site's own `/_next/static/chunks/*.js` bundles and grepping for
`query codeOfAlabamaSection`, which is where the exact query text used in
this pack's `SECTION_QUERY`/`SEARCH_QUERY` constants comes from verbatim.

The citation shape is title-chapter-section (`13A-6-2`); titles can carry a
trailing letter (`13A`, `26-2A`). `al_statute`'s regex accepts 2 or 3
dash-separated segments plus an optional decimal suffix, matching the
sibling packs' validation shape.

No baked index exists for this pack, unlike `mcps/south-dakota-code` or
`mcps/north-dakota-code`: ALISON's own `searchCodeOfAlabama` query is a real
full-text search over section body content, so there is nothing to bake —
building an offline index would only add staleness (the Code gains pending
future-effective rows on its own legislative schedule) with no accuracy
benefit over the live call.

## Known gaps

- No per-year historical archive — see "Historical version" above.
- `al_search` is full-text over body content but still keyword-shaped (not a
  phrase/boolean query language) — it is the Legislature's own search
  backend, not Pipeworx's.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "alabama-code": {
      "url": "https://gateway.pipeworx.io/alabama-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/alabama-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/al_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"13A-6-2"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/al_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-alabama-code` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/alabama-code/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "alabama-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-alabama-code"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Alabama Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
