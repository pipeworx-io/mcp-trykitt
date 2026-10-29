# @pipeworx/trykitt

Find a verified work email by full name + company domain, via TryKitt.ai.
Part of the `enrich-waterfall` finder rotation (fleet #2860).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `trykitt_find_email(full_name, domain, linkedin_url?, _apiKey)` — email,
  confidence (0-1), and sources, requested in realtime mode so the HTTP
  response carries the finished result rather than a job id to poll.

## Auth

BYO only. Pass your TryKitt API key as `_apiKey` (header `x-api-key` —
**not** `Authorization`, which several third-party write-ups wrongly
assume). Get one at https://trykitt.ai. No platform key (D4).

## Data sources

- `https://api.trykitt.ai/job/find_email` — email finder (realtime mode).

## Verification notes (fleet #2860, 2026-10-08, no vendor key held)

- **Header name confirmed by edge probe, including the "missing vs wrong"
  distinction**: no auth header -> HTTP 401
  `{"message":"Auth credential not set in request","response_code":401}`;
  `x-api-key: invalidkey123` -> HTTP 401
  `{"message":"Invalid API Key","response_code":401}`. Two different
  messages for two different failures confirms the header is actually being
  read, not just any 401 default.
- **Response shape is from TryKitt's public Postman collection and
  third-party MCP wrapper READMEs** (not TryKitt's own prose docs, which sit
  behind the same auth wall as the API) — we hold no key, so no successful
  call was made. Documented shape: `{ job_id, status, result: { email,
  confidence, sources } }`. This pack treats any `status` other than
  `"completed"` as a loud error rather than a silent no-match, since
  TryKitt's job model can also run async — if a live key ever gets back a
  `"pending"` status despite `realtime: true`, that needs a poll step added,
  and the current code will say so explicitly instead of reporting
  `found: false`.
- **Not verified (no key held):** whether `realtime: true` reliably forces
  a synchronous result on every plan tier; the full set of `status` values.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "trykitt": {
      "url": "https://gateway.pipeworx.io/trykitt/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/trykitt/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

This pack takes your own API key (`_apiKey`) — we don't front one for it, so there's no curl here that would run without it. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/trykitt_find_email`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-trykitt` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/trykitt/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "trykitt": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-trykitt"]
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
ask_pipeworx({ question: "your question about Trykitt data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
