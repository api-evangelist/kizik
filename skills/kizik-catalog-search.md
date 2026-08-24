---
name: kizik-catalog-search
description: >-
  Search and read the Kizik hands-free shoe catalog over Kizik's anonymous UCP/MCP endpoint, with correct
  price handling and buyer localization. Read-only — takes no action that costs money.
api: Kizik Agent Commerce API (UCP / MCP)
endpoint: https://kizik.com/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
---

# Search the Kizik catalog

Kizik sells hands-free step-in shoes. Its catalog is readable by any agent with no credentials.

## Before you call

Every tool call needs `meta.ucp-agent.profile` — a URI resolving to **your own** published UCP agent
profile. Without it the server answers HTTP 200 with a JSON-RPC error, `code: -32001`,
`data.code: invalid_profile_url`. This is checked before the method name and before the arguments, so a
missing profile masks every other error.

## Steps

1. **Confirm the surface.** `GET https://kizik.com/.well-known/ucp`. Check `ucp.version` is a version you
   speak (currently `2026-04-08`; `2026-01-23` is also served). The response header
   `x-shopify-ucp-mcp-api-version` on later calls tells you which one was applied.

2. **Search.** POST to `https://kizik.com/api/ucp/mcp`:

   ```json
   {"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_catalog","arguments":{
     "meta":{"ucp-agent":{"profile":"https://your-agent.example/ucp-profile.json"}},
     "catalog":{
       "query":"hands-free sneakers",
       "context":{"address_country":"US","currency":"USD"},
       "filters":{"available":true,"price":{"max":15000}},
       "pagination":{"limit":10}
     }
   }}}
   ```

   Pass `context.address_country` and `context.currency` or the prices you get back may not be the ones
   the buyer would actually pay.

3. **Page.** `catalog.pagination.limit` defaults to 10, minimum 1. Carry `catalog.pagination.cursor`
   forward for the next page.

4. **Narrow.** `filters.categories` is an array combined with OR. `filters.price.min` / `.max` are in
   **minor units** — `15000` is $150.00. `filters.available` defaults to `true` (sale-ready items only).

5. **Resolve detail.** Use `get_product` for one product by identifier, or `lookup_catalog` for several
   products or variants at once. Identifiers are Shopify global IDs, e.g.
   `gid://shopify/ProductVariant/123`.

## Price handling — get this right

Every price is an integer in ISO 4217 **minor units** paired with a currency code.
`{"amount": 600, "currency": "USD"}` is **$6.00**, not $600. Divide by 100 for two-decimal currencies
before you quote anything to a person. Zero-decimal currencies such as JPY are already whole units.

## Errors

Errors come back with **HTTP 200**. Read `error` in the body, not the status code. Branch on
`error.data.code`; `error.data.continue_url` is a URL a human can open to carry on in a browser.

## Rate limits

The endpoint is rate limited per IP and returns 429 when exhausted. No `RateLimit-*` or `Retry-After`
headers are returned on success, so you cannot see the limit approaching — back off on 429 and do not
poll. `shopify-complexity-score` on each response tells you how expensive that call was.
