# @pipeworx/peru-sunat-ruc

Peru's SUNAT (tax authority) taxpayer register, "Padron reducido del RUC": every
11-digit company RUC with the registered name (razon social), taxpayer status (active,
deregistered, suspended ...), domicile condition (habido / no habido ...) and,
for legal entities, the registered address with its INEI ubigeo code.
About 2.37 million company and government-office RUCs (individuals excluded) as of the 2026-10-29 edition.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1863+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

`latam-validate` checks RUC *format* only; this pack answers whether the RUC is
in SUNAT's register and what state it is in.

## Tools

- `pe_ruc(ruc)` — look up one taxpayer by 11-digit RUC. Returns `found: false`
  with a `reason` (`bad_format`, or `not_in_register` plus `check_digit_valid`)
  and `data_as_of` — never an empty object.
- `pe_ruc_search(name, ubigeo?, active_only?, limit?)` — accent-insensitive name
  search over **legal entities** (every word of 3+ letters must appear),
  active registrations first, optional ubigeo prefix (2 = department, 4 =
  province, 6 = district).

Every response carries `data_as_of` and `source`. Status and domicile condition
are returned as SUNAT published them (the publisher truncates its labels to 20
characters, e.g. `BAJA PROV. POR OFICI`) beside a best-effort English gloss.

## Natural persons are excluded

This pack covers **companies and organisations only** (RUC prefix `20`, plus
government offices, prefix `00`; about 2.37M registrations). Individuals (RUCs
beginning `10`, `15`, `17`, about 16.1M of the register) are **not stored, not
looked up and not searchable** — Bruce ruling, 2026-10-29. `pe_ruc` on an
individual's RUC answers `found: false, reason: "natural_person_not_served"`
from the RUC prefix alone, without reading the database and without disclosing
a name, status or whether the number exists. The loader drops those prefixes
before writing.

## Auth

Keyless. The tools work only through the Pipeworx gateway (the register lives
in a gateway-side database); a standalone install returns a loud
`StoreUnavailableError`, never an empty result.

## Data source & reuse terms

- SUNAT (Superintendencia Nacional de Aduanas y de Administracion Tributaria,
  Peru), "Padron reducido del RUC", published as a daily-updated bulk file
  (page stamp "Actualizado al DD/MM/YYYY"). `data_as_of` on every response is
  that stamp for the loaded edition. Fetch mechanics live only in
  `scripts/peru-sunat-ruc/`, which does not publish (CLAUDE.md step 9b).
- **Reuse terms: none are published.** The publisher's download page (read
  2026-10-29) carries only the title "Padron Reducido", the stamp "(Actualizado
  al 29/10/2026)" and two download links; there is no licence, terms-of-use or
  redistribution statement on it. It is Peruvian public tax-register data
  served to anyone without credentials, and nothing forbids redistribution, so
  it is served — flagged here as *no explicit reuse grant*.

## Refresh

`bash scripts/peru-sunat-ruc/refresh.sh load` downloads the edition, builds a
local SQLite file (ISO-8859-1, `|`-delimited, `\|` escapes a literal pipe;
fails loudly on an unknown status, a malformed row, or a truncated download),
replaces the D1 tables, fills the trigram name index in batches and asserts it
is complete. `refresh.sh verify` is read-only; `verify --self-test` proves the
completeness gate can fail.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "peru-sunat-ruc": {
      "url": "https://gateway.pipeworx.io/peru-sunat-ruc/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/peru-sunat-ruc/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/pe_ruc \
  -H 'Content-Type: application/json' \
  -d '{"ruc":"20100070970"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/pe_ruc`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-peru-sunat-ruc` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/peru-sunat-ruc/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "peru-sunat-ruc": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-peru-sunat-ruc"]
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
ask_pipeworx({ question: "your question about Peru Sunat Ruc data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
