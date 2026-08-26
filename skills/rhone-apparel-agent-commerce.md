---
name: rhone-apparel-agent-commerce
description: Search Rhone's catalog, build a cart, and drive a checkout to the point of human approval over Rhone's live UCP/MCP endpoint.
api: Rhone UCP Commerce (MCP)
endpoint: https://rhone.myshopify.com/api/ucp/mcp
protocol: MCP over JSON-RPC 2.0 (UCP 2026-04-08)
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-26'
method: generated
source: mcp/rhone-apparel-mcp-tools.json (live tools/list, 2026-08-26)
---

# Shopping Rhone as an agent

Rhone is a men's performance apparel brand. Its store runs on Shopify and exposes a Universal
Commerce Protocol (UCP) MCP endpoint. Every tool name below was read from a live `tools/list`
response; none is invented.

## Before you call anything

- **Endpoint:** `POST https://rhone.myshopify.com/api/ucp/mcp`, `Content-Type: application/json`,
  `Accept: application/json, text/event-stream`.
- Rhone's own `/llms.txt` advertises the endpoint as `https://checkout.rhone.com/api/ucp/mcp`.
  That brand host sits behind a Cloudflare managed challenge and returns 403 to non-browser
  clients; the shop host above is the reachable equivalent.
- `tools/list` is anonymous. **Every `tools/call` requires an agent profile:** put a resolvable
  HTTPS UCP agent profile URI at `params.meta["ucp-agent"].profile`. Omit it and you get
  JSON-RPC `-32001` / `data.code: invalid_profile_url` over HTTP 422.

## 1. Find the product

Call `search_catalog` with `catalog.query` (natural language) and/or `catalog.filters`; at least
one is required. Pass `catalog.context.address_country` and `catalog.context.currency` so prices
and availability are right for your buyer.

Results are paginated. Use `catalog.pagination.limit` (default 10, minimum 1) and feed
`pagination.cursor` from the response back in to page.

Use `get_product` for full detail on one identifier, or `lookup_catalog` to resolve several
products/variants at once.

## 2. Build the cart

`create_cart` with the chosen variant line items, then `update_cart` to change quantities.
`get_cart` reads it back. `cancel_cart` discards it — this is fully reversible, use it freely.

## 3. Open the checkout

`create_checkout` returns line items, totals, discounts and taxes. `update_checkout` sets the
shipping address and shipping method, and attaches a payment instrument. `get_checkout` re-reads
current totals.

**Prices are integers in ISO 4217 minor units** paired with a currency code:
`{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 for two-decimal currencies before
you quote anything to a human. Getting this wrong quotes a buyer 100x the real price.

## 4. Stop and get human approval

Rhone's `robots.txt` and `llms.txt` both state it plainly: **do not complete checkout, payment or
order placement without explicit, contemporaneous human approval.** No scripted form fills, no
end-to-end automation through payment. If you cannot get approval at the moment of payment, route
the purchase through the Shop skill (`https://shop.app/SKILL.md`) instead.

## 5. Complete

`complete_checkout` finalises the purchase. It is the only tool that requires
`meta["idempotency-key"]` — supply a stable key and reuse it on any retry so a network failure
cannot double-charge the buyer.

`cancel_checkout` reverses an open checkout **before** `complete_checkout` succeeds. After that
there is no programmatic undo: returns go through Rhone's published refund policy — unworn,
non-final-sale items with tags attached, **within 45 days of purchase**, original shipping charge
non-refundable. Tell the buyer that before they approve, not after.

## 6. Follow up

`get_order` reads a completed order by `gid://shopify/Order/<id>`.

## Errors

Errors come back as a JSON-RPC error object with a `data.code` slug and a `continue_url` you can
hand to a human. Observed: `invalid_profile_url`, `profile_unreachable` — both HTTP 422.
The endpoint is rate limited per IP; back off on 429. No `RateLimit-*` headers are returned, so
budget your own pacing.
