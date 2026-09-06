---
name: acadio-buy-exam-prep-course
description: >-
  Find, cart and purchase an Acadio securities exam prep course (SIE, Series 6, Series 7, Series 63)
  through Acadio's live Universal Commerce Protocol MCP endpoint, with the buyer approving payment.
api: Acadio Storefront Agentic Commerce (UCP)
endpoint: https://acadio.com/api/ucp/mcp
transport: mcp
auth: none
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-09-06'
method: generated
source: >-
  Grounded entirely in the 13 tools returned by a live tools/list against
  https://acadio.com/api/ucp/mcp on 2026-09-06 (saved verbatim at mcp/acadio-mcp-tools.json) and the
  provider's own agent instructions at https://acadio.com/agents.md. No operation, parameter or
  behaviour here is invented.
---

# Buy an Acadio exam prep course

Acadio sells FINRA securities licensing exam prep courses (SIE, Series 6, Series 7, Series 63)
direct to learners. Its storefront is reachable by agents over MCP at
`https://acadio.com/api/ucp/mcp`, no credential required. Acadio publishes no OpenAPI, so the tool
schemas returned by `tools/list` are the contract.

## Before you start

- Every tool requires a `meta.ucp-agent.profile` URI. It is `required` on all 13 tools; a call
  without it will not validate.
- Pass `context.address_country` (ISO 3166-1 alpha-2) and `context.currency` on catalog, cart and
  checkout calls. Acadio's agent instructions state pricing and availability depend on them.
- **All money is integers in ISO 4217 minor units** paired with a currency code. `{"amount": 2500,
  "currency": "USD"}` is $25.00. Divide by 100 for two-decimal currencies before you quote a price
  to a person. Zero-decimal currencies such as JPY are already whole units.
- The endpoint is rate-limited per IP. Back off on `429`.

## Steps

1. **Find the course.** Call `search_catalog` with `catalog.query` set to what the buyer asked for
   ("Series 7 exam prep"). Narrow with `catalog.filters.categories`, `catalog.filters.price` or
   `catalog.filters.available`. Page with `catalog.pagination.cursor` and
   `catalog.pagination.limit` — this is the only pagination convention Acadio exposes.
2. **Confirm the exact item.** Call `get_product` (single) or `lookup_catalog` (several) with the
   identifiers from step 1 before quoting anything. Acadio sells several similarly named courses,
   one per FINRA exam.
3. **Build a cart.** Call `create_cart` with `cart.line_items[]` (`id`, `quantity`, `item`) and
   `cart.buyer.email`. Keep the returned cart `id`. Adjust with `update_cart`; abandon with
   `cancel_cart`.
4. **Open a checkout.** Call `create_checkout`, passing `checkout.cart_id` from step 3. The returned
   checkout `id` has the form `gid://shopify/Checkout/...`. Read totals, discounts and taxes from
   the response — do not compute them yourself.
5. **Complete the checkout details.** Call `update_checkout` with the checkout `id` to set
   `checkout.buyer`, `checkout.fulfillment.methods` and any `checkout.discounts.codes`. Re-read
   totals afterward; discounts and taxes change them.
6. **Get the buyer's approval, then complete.** Call `complete_checkout` only with the buyer's
   contemporaneous consent at the moment of payment. Acadio's published rules are explicit:
   *"Checkout requires human approval. Agents must not complete payment without explicit buyer
   consent."* If you cannot get that consent live, do not call this tool — route the purchase
   through the Shop skill Acadio's own `agents.md` recommends instead.
7. **Confirm.** Call `get_order` with the resulting order id and report it back to the buyer.

## Reversibility — read this before step 6

`cancel_cart` and `cancel_checkout` undo steps 3 through 5. **Nothing undoes step 6.** Acadio
exposes no refund, void or reverse tool for a completed checkout, and publishes no reversal window.
After `complete_checkout` succeeds, the only recourse is the human refund policy at
<https://acadio.com/policies/refund-policy>. Treat step 6 as irreversible.

## What this skill cannot do

There is no idempotency key on this surface. If a `create_checkout` or `complete_checkout` call
times out, do not blind-retry it — call `get_checkout` with the id you hold and read the actual
state first.

This skill covers the **storefront only**. The Acadio LMS platform (courses, learners, credits,
completions) has no public API; its integration surface is inbound JWT SSO and an outbound webhook
bus, described in `authentication/acadio-authentication.yml` and
`asyncapi/acadio-lms-webhooks.yml`.
