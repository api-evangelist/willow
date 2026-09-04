---
name: willow-catalog-discovery
description: >-
  Search and read Willow's wearable-breast-pump catalog through the store's anonymous UCP commerce
  MCP endpoint — find products, filter by price and availability, and pull full product detail with
  correctly-converted prices, without creating a cart or spending anything.
api: Willow Pump UCP Commerce MCP Server
endpoint: https://onewillow.com/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-09-04'
method: generated
source: mcp/willow-tools-list.json (live tools/list probe, 2026-09-04)
---

# Discovering products in Willow's catalog

Willow (onewillow.com) sells wearable breast pumps and accessories. Its storefront exposes a
Universal Commerce Protocol MCP endpoint that answers `tools/list` with no credentials, so an agent
can browse the catalog before it knows anything about the buyer.

## Before you start

- **Endpoint** — `POST https://onewillow.com/api/ucp/mcp`, `Content-Type: application/json`,
  `Accept: application/json, text/event-stream`. JSON-RPC 2.0.
- **Identity is required on every `tools/call`.** Each tool's `inputSchema` requires
  `meta["ucp-agent"].profile` — an https URI that resolves to your UCP agent profile document. A
  missing profile fails with `-32001 / invalid_profile_url`; one that resolves to HTML fails with
  `-32001 / profile_malformed`. Discovery (`initialize`, `tools/list`) needs no profile.
- **Nothing here needs an API key.** Willow issues none.

## Steps

1. **Confirm the surface.** `GET https://onewillow.com/.well-known/ucp.json` and check
   `ucp.version` (currently `2026-08-25`) and that `services["dev.ucp.shopping"]` lists
   `transport: mcp`. Willow also serves `2026-04-08` and `2026-01-23` at versioned discovery URLs.
2. **Search.** Call `search_catalog` with `catalog.query` set to the buyer's words. Pass
   `catalog.context.address_country` and `catalog.context.currency` — Willow's own agent
   instructions say pricing and availability are wrong without them. Narrow with
   `catalog.filters.categories`, `catalog.filters.price.min` / `.max` (integer minor units), and
   `catalog.filters.available` (defaults to true — sale-ready items only).
3. **Resolve identifiers in bulk.** `lookup_catalog` takes `catalog.ids[]` when you already hold
   product IDs from a previous turn or a saved list.
4. **Read one product properly.** `get_product` with `catalog.id`, plus `catalog.selected[]`
   (`{name, label}` option pairs) when the buyer has chosen a size or colour, returns the single
   product with the variant resolved.

## Quoting prices correctly

Every amount is an integer in the currency's ISO 4217 **minor** units paired with a currency code:
`{"amount": 2500, "currency": "USD"}` is **$25.00**. Divide by 100 for two-decimal currencies before
saying a number to a person; zero-decimal currencies such as JPY are already whole units. Quoting
the raw integer is the most likely mistake on this API.

## Read-only alternatives

If you only need catalog data and cannot speak MCP, Willow's agent instructions publish plain HTTP
reads that need no auth at all: `/products.json`, `/collections/{handle}/products.json`,
`/products/{handle}.json`, and `/search?q={query}&type=product`.

## Errors you will actually see

| JSON-RPC code | `data.code` | What to do |
|---|---|---|
| -32001 | `invalid_profile_url` | Add `meta["ucp-agent"].profile`. |
| -32001 | `profile_malformed` | Serve the profile URI as JSON, not HTML. |
| — (HTTP 429) | — | Back off; the endpoint is rate limited per IP. Watch `shopify-complexity-score-v2` on successful calls — a `search_catalog` costs roughly ten times a `tools/list`. |

Every error also carries `data.continue_url`, a human-openable storefront URL. Hand it to your user
when you cannot proceed.
