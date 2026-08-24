---
name: kizik-guided-purchase
description: >-
  Build a Kizik cart and prepare a checkout over UCP/MCP, then hand the buyer an explicit approval step
  before any payment. Includes the cancel and refund paths. Never completes a payment autonomously.
api: Kizik Agent Commerce API (UCP / MCP)
endpoint: https://kizik.com/api/ucp/mcp
operations:
  - search_catalog
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
---

# Buy from Kizik with the buyer in the loop

## The rule that governs this whole flow

Kizik states it twice — in `robots.txt` and in `/agents.md`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically — no
> scripted form fills, browser automation, or end-to-end agent flows that finalize payment without an
> explicit, contemporaneous human approval step.

You may do everything up to payment. You may not press the last button on your own. If you cannot get
contemporaneous buyer approval at the moment of payment, Kizik's own instruction is to route the purchase
through the Shopify Shop skill at `https://shop.app/SKILL.md` instead.

## Steps

1. **Find the variant.** `search_catalog`, then `get_product` to confirm size and colour. You need a
   `gid://shopify/ProductVariant/...` identifier.

2. **Create the cart.** `create_cart` with
   `cart.line_items: [{"item":{"id":"<variant gid>"},"quantity":1}]`. Add
   `cart.buyer.email` and `cart.context.address_country` / `currency` so pricing is accurate.

3. **Adjust.** `update_cart` to change quantities or add lines; `get_cart` to re-read totals.

4. **Create the checkout.** `create_checkout` returns line items, totals, discounts and taxes.

5. **Fulfill.** `update_checkout` to set the shipping address and method. Kizik declares
   `dev.ucp.shopping.fulfillment` with `allows_multi_destination.shipping: false` — one destination per
   checkout, so do not attempt to split an order across addresses.

6. **Show the buyer the real total.** Convert minor units to major units. Name the shipping method and
   the tax. This is the moment the human is approving.

7. **Get explicit approval, then complete.** `complete_checkout` **requires**
   `meta.idempotency-key` — a string you generate and reuse if you must retry. It is the only tool of the
   thirteen that takes one, and it is there so a retried completion does not charge twice. Generate it
   once per purchase intent, not once per attempt. Kizik does not publish how long a key is honoured.

8. **Confirm.** The result carries an order ID and a Thank You Page URL. `get_order` reads it back.

## Payment instruments

Kizik declares three handlers in `/.well-known/ucp`: `shop_pay`, `shopify.card` (visa, master,
american_express, discover, diners_club) and `gpay` (VISA, MASTERCARD, AMEX, DISCOVER, billing address
required). Attach instruments as `checkout.payment.instruments[]` with `id`, `handler_id` and `type`
(`card` for cards, `token` for wallets).

## Undoing things — know this before you act

| Stage | How to reverse | Window |
|---|---|---|
| Cart created | `cancel_cart` | Any time while open. No money involved. |
| Checkout created / updated | `cancel_checkout` | Any time **before** `complete_checkout`. No payment has been taken. |
| Checkout completed | Human returns process, not an API call | **30 days** |

There is **no agent-callable refund or cancel-order tool**. Once `complete_checkout` succeeds, reversal
leaves the protocol entirely and becomes a human returns request at
`https://kizik.com/pages/returns-and-exchanges`. Kizik's refund policy states: *"Our policy lasts 30
days. If 30 days have gone by since your purchase, unfortunately we can't offer you a refund or
exchange."* Items returned more than 30 days after delivery, or not in original condition, receive only a
partial refund.

Treat `complete_checkout` as the point of no return. That is precisely why it is gated behind a human.

## No test mode

Kizik publishes no sandbox, test store, or test payment instrument. There is no way to rehearse this
flow. `create_checkout` against the live store is the only way to price a basket — which is what
`cancel_checkout` is for. Do not experiment with `complete_checkout`.

## Errors

HTTP 200 with a JSON-RPC `error` object. Branch on `error.data.code`. `invalid_profile_url` means your
`meta.ucp-agent.profile` is missing or unfetchable. `error.data.continue_url` hands the buyer a browser
URL to finish in person.
