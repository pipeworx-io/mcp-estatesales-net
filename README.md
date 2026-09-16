# @pipeworx/estatesales-net

Upcoming estate sales, moving sales and estate auctions near a US ZIP code,
from EstateSales.NET — the largest US estate-sale listing site — plus the
companies that run them.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `estatesales_search(postal_code)` — the upcoming sales nearest a ZIP: sale
  ID, type, city/state/ZIP, coordinates, distance, schedule, start/end dates.
- `estatesales_sale(sale_id)` — one sale in full: name, description, terms,
  every scheduled date, coordinates, picture count, and the company running it
  with phone and page. Street address only once the site has released it (see
  below).
- `estatesales_companies(postal_code)` — estate-sale businesses working that
  ZIP: name, business address, phone, coordinates, listing tier.

## Auth

Keyless. No account, no token, no registration.

### The official EstateSales.NET API is a different thing, and it is not usable here

EstateSales.NET does publish a documented beta REST API
(<https://github.com/vintage-software/EstateSales.NET-Api>), and the obvious
reading is that this pack should use it. It should not. That API is
**seller-side only** — every endpoint is scoped to your own `OrgId`:

```
GET    /api/public-sales/org/{OrgId}      your org's sales
GET    /api/public-sales/{saleId}         403 without your org's token
POST   /api/public-sales/                 create a sale
PUT    /api/public-sales/{saleId}         edit a sale
POST   /api/public-sales/{saleId}/publish
POST   /api/sale-dates/ · /api/sale-pictures/
```

It is how an estate-sale **company** lists its own sales. There is no search
endpoint in it at all, its keys are issued from an estate-sale company account
(`/account/company/api-keys`), and a key would only ever return our own empty
org. It cannot answer a buyer's question, so no `PLATFORM_ESTATESALES_KEY`
exists and none would help.

What this pack reads instead is the site's own public buyer-facing data — the
structured JSON the website itself renders from, no credentials involved.

## Addresses are embargoed by the publisher, and this pack honors that

An estate sale is usually held at a private home, so EstateSales.NET withholds
the street address until shortly before the sale (`showAddress`,
`utcShowAddressAfter`, `showAddressType`). `estatesales_sale` returns the
address **only when the site has already published it**; otherwise it returns
`address_published: false`, the release time in `address_visible_after_utc`,
and a note. Coordinates and city/ZIP are always present, since the site
publishes those from the start.

Do not route around this. The embargo is the publisher's decision about
someone's home, not a technical quirk.

## No prices

EstateSales.NET lists **upcoming sales**. It publishes no hammer prices and no
realized-sale data, so nothing here is a comp. Every response says so in a
`price_data` field.

## Data sources

- `https://www.estatesales.net/{ST}/{City}/{ZIP}` — city sale listing.
- `https://www.estatesales.net/{ST}/{City}/{ZIP}/{saleId}` — sale detail.
- `https://www.estatesales.net/companies/{ST}/{City}/{ZIP}` — company listing.

Things worth not rediscovering:

- **The pages carry their data as JSON, not markup.** Each embeds a `script`
  tag with `id="estatesales-net-state"` and `type="application/json"`, holding
  `{"NGRX_STATE": …}` — Angular transfer state. Sales are at
  `feature.cityViewState.filteredSales`, a sale at
  `feature.traditionalSaleViewState.entitiesById[saleId]`, companies at
  `feature.companiesCityViewState.companies`. Anchor on the script tag's `id`:
  matching `type="application/json"` alone also hits the analytics blocks.
- **The site canonicalises its own location URLs**, which is why every tool
  here needs only a ZIP or a sale ID. `/XX/x/64111` redirects to
  `/MO/Kansas-City/64111`, and `/XX/x/00000/{saleId}` redirects to the sale's
  real path. Never hand-build the city slug.
- **A city page lists 20 sales** regardless of how many exist;
  `filteredSaleCount` and `totalSaleCount` report the real depth (e.g. 64111:
  20 listed, 61 filtered, 97 total). There is no page parameter — the site
  loads the rest client-side. Report the counts rather than implying 20 is all.
- Dates arrive wrapped as `{"_type":"DateTime","_value":"…"}`; this pack
  unwraps them to ISO strings.
- `GET /api/sales/{id}` **is** public and keyless, but returns only
  bookkeeping (org id, type, timestamps) — no address, dates, description or
  pictures. The page state has all of those, which is why the pack uses it.
  `GET /api/public-sales/{id}` (the official API's path) is 403 without a
  seller token.
- `/api/sales?postalCodeNumber=…` answers HTTP 400
  *"Permissions denied on filter"* — the filter is not public. That error means
  the endpoint exists, not that the site is down.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "estatesales-net": {
      "url": "https://gateway.pipeworx.io/estatesales-net/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/estatesales-net/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "estatesales-net": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-estatesales-net"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-estatesales-net
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Estatesales Net data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
