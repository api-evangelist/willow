---
name: willow-agentic-checkout
description: >-
  Build a cart and complete a purchase on Willow's storefront through its UCP commerce MCP endpoint —
  including the buyer-approval rule, the one operation that requires an idempotency key, and exactly
  how far a purchase can be undone.
api: Willow Pump UCP Commerce MCP Server
endpoint: https://onewillow.com/api/ucp/mcp
operations:
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
generated: '2026-09-04'
method: generated
source: mcp/willow-tools-list.json (live tools/list probe, 2026-09-04)
---

# Buying from Willow as an agent

This is a real-money surface. Everything below is grounded in the tool schemas the endpoint returned
on 2026-09-04 and in Willow's own published agent instructions at https://onewillow.com/agents.md.

## The non-negotiable rule

> "Checkout requires human approval. Agents must not complete payment without explicit buyer
> consent." — Willow's agent instructions

If you cannot get contemporaneous buyer approval at the moment of payment, do not call
`complete_checkout`. Willow's instructions tell you to route the purchase through the Shop skill
(`https://shop.app/SKILL.md`) instead.

## Flow

1. **Find the variant** — see `willow-catalog-discovery`. You need a product *variant* ID.
2. **`create_cart`** — `cart.line_items[] = {item: {id: <variant id>}, quantity: <n>}`. Optionally
   `cart.buyer.email` / `.phone_number` and `cart.context` hints (`address_country`,
   `address_region`, `postal_code`, `language`, `currency`, `intent`).
3. **`get_cart` / `update_cart`** — read totals back; adjust quantities or line items by `id`.
4. **`create_checkout`** — turns the basket into a priced checkout. IDs look like
   `gid://shopify/Checkout/abc123`.
5. **`update_checkout`** — set `checkout.buyer`, `checkout.fulfillment.methods[]` (shipping),
   `checkout.discounts.codes[]` (case-insensitive; a new submission **replaces** the previous set —
   only prompt for a code if the buyer raised it), and `checkout.payment.instruments[]`.
   Willow's discovery document lists the payment handlers it accepts: Google Pay (`gpay`), Shopify
   card (`shopify.card`) and Shop Pay (`shop_pay`); the schema also branches on `apple-pay`.
6. **Show the buyer the total and get approval.** Convert minor units first
   (`{"amount": 2500, "currency": "USD"}` is $25.00).
7. **`complete_checkout`** — this one is different: `meta["idempotency-key"]` is **required**.
   Generate one key per intended purchase and reuse it verbatim on every retry of that same
   purchase. Never generate a fresh key for a retry — that is how a buyer gets charged twice.

## Undoing things

| Situation | What exists |
|---|---|
| Cart no longer wanted | `cancel_cart` — no published time limit. |
| Checkout started, not paid | `cancel_checkout` — no published time limit. In practice the window is "before `complete_checkout`". |
| Order already placed | **No API path.** Reversal is a human Customer Care request; Willow's refund policy states 60 days from purchase for pumps and pump accessories, 30 days for other items, original packaging and unused (https://onewillow.com/policies/refund-policy). |

Treat `complete_checkout` as the point of no return for an agent. Say so to the buyer before you
call it.

## Retry and replay

Only `complete_checkout` is replay-protected. `create_cart`, `update_cart`, `create_checkout` and
`update_checkout` accept no idempotency key, so a timeout on any of them can leave a duplicate
resource behind. Read back with `get_cart` / `get_checkout` before retrying rather than firing the
write again.

## Auth ladder

- `initialize`, `tools/list` — anonymous.
- Any `tools/call` — requires `meta["ucp-agent"].profile`, a resolvable UCP agent profile URI.
- `get_order` — additionally requires a signed agent JWT. Without one you get
  `-32000 AuthenticationRequired`.
- Buyer-account access is OAuth 2.0 / OIDC on `https://account.onewillow.com`
  (scopes `openid`, `email`, `customer-account-api:full`, `customer-account-mcp-api:full`).

## Rate limits

Per-IP, unnumbered. Back off on 429. Each response carries `shopify-complexity-score-v2` — a
`tools/call` can cost more than ten times a `tools/list`, so pace by cost, not by request count.
