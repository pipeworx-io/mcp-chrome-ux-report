# @pipeworx/chrome-ux-report

Real-user Core Web Vitals for any site with enough traffic, from Google's Chrome
UX Report — the 75th-percentile Largest Contentful Paint, Interaction to Next
Paint and Cumulative Layout Shift that actual Chrome visitors experienced over
the trailing 28 days, plus the distribution behind each number and ~40 weeks of
history.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1863+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

This is the field dataset Google Search uses for its Core Web Vitals assessment,
so it answers "is this site passing Core Web Vitals" — which no lab tool,
Lighthouse included, can.

## Tools

- `crux_query_origin(origin, form_factor?, metrics?)` — the whole site.
- `crux_query_url(url, form_factor?, metrics?)` — one page.
- `crux_history(origin? | url?, form_factor?, metrics?)` — ~40 weekly points.

## Auth

BYO with platform fallback: `_apiKey`, or `PLATFORM_GOOGLE_API_KEY` when that
secret is set on the gateway. **There is no keyless path** — the API answers an
unkeyed request with HTTP 403 "Method doesn't allow unregistered callers".

A key is free, instant and self-service, with **no billing account required**:
create one at <https://console.cloud.google.com/apis/credentials> and enable the
Chrome UX Report API at
<https://console.cloud.google.com/apis/library/chromeuxreport.googleapis.com>.
Quota is 150 queries/minute.

The same `PLATFORM_GOOGLE_API_KEY` serves the `pagespeed-insights` pack: one
Google Cloud key with both APIs enabled covers both.

## Data sources

- `https://chromeuxreport.googleapis.com/v1/records:queryRecord`
- `https://chromeuxreport.googleapis.com/v1/records:queryHistoryRecord`

## Traps, verified live 2026-09-17

- **The three credential failures look nothing alike.** Missing key → HTTP 403
  `PERMISSION_DENIED` "Method doesn't allow unregistered callers". Wrong key →
  HTTP 400 `INVALID_ARGUMENT` "API key not valid". Key whose project has not
  enabled the API → a third shape (`SERVICE_DISABLED`). Collapsing them into
  "auth failed" sends the caller after the wrong problem; each is reported
  separately.
- **`origin` and `url` are different questions and exactly one may be sent.**
  Both together is a 400. An origin must have no path.
- **HTTP 404 is the normal answer for most of the web.** CrUX only publishes a
  page or origin once it has enough opted-in Chrome traffic to be statistically
  sound and non-identifying. That is the dataset's answer, not an outage, and
  this pack returns it as `found: false, reason: "insufficient_traffic"`. A URL
  can be absent while its origin is present. Narrower form factors have
  separate, higher thresholds.
- **The headline number is p75, not the mean.** Search assesses at the 75th
  percentile; averaging the histogram gives a different, wrong number.
- **CLS is unitless; every other metric is milliseconds.** CrUX returns CLS p75
  as a decimal *string* while timings are integers. (The PageSpeed Insights API
  returns the same metric as an integer ×100 — the two Google APIs disagree on
  the encoding of the same number.)
- **`crux_history` points are 28-day trailing windows.** Consecutive weekly
  points overlap by 27 days, so a move in the series is a move in a rolling
  average and a one-week regression shows up damped across four points.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "chrome-ux-report": {
      "url": "https://gateway.pipeworx.io/chrome-ux-report/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/chrome-ux-report/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1863+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/crux_query_origin \
  -H 'Content-Type: application/json' \
  -d '{"origin":"https://www.shopify.com"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/crux_query_origin`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-chrome-ux-report` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/chrome-ux-report/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "chrome-ux-report": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-chrome-ux-report"]
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
ask_pipeworx({ question: "your question about Chrome Ux Report data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
