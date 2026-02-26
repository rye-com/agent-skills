---
name: rye-universal-checkout
description: "Use this skill when a developer wants to add product purchasing to their application using Rye's Universal Checkout API, or when an AI agent needs to programmatically buy a physical product from any URL. Triggers on: integrating checkout or purchasing into an app, buying a product on behalf of a user, building a shopping assistant, dropshipping integration, chat commerce, or any mention of Rye checkout. Do NOT use for digital goods, international shipping, or multi-product cart scenarios."
license: MIT
metadata:
  author: rye
  version: "1.0.0"
---

# Rye Universal Checkout

Rye's Universal Checkout API turns any product URL into a completed purchase. You provide a product URL and buyer info — Rye handles pricing, tax, shipping, and order placement across merchants (Shopify, Amazon, and thousands more).

The **Product Data API** complements checkout by letting you look up product details (price, availability, images, description) from any URL before committing to a purchase.

## When to use this skill

**Two primary use cases:**

1. **Integrate (B2B/Developer)** — A developer wants to add purchasing capabilities to their app (marketplace, shopping assistant, dropshipping tool, gifting platform, etc.)
2. **Buy (Agent Commerce)** — An AI agent needs to programmatically purchase a product on behalf of a user

## Which reference to load

Determine the task type, then load the appropriate reference:

| Task | Load |
|------|------|
| Developer wants to integrate Rye into their codebase | `references/integrate.md` |
| Agent needs to buy a product right now | `references/buy.md` |
| Need full API details, SDK docs, or endpoint reference | `references/api-reference.md` |

**Load `references/api-reference.md` alongside either workflow** when you need exact endpoint signatures, SDK method names, or request/response shapes.

## Key constraints (always apply)

Before proceeding with ANY Rye operation, be aware of these hard constraints:

- **US-only shipping** — International addresses will cause the checkout intent to fail
- **Physical products only** — Digital goods, subscriptions, and gift cards are not supported
- **One product per checkout** — Multiple quantities of the same product are fine, but multi-product carts are not supported
- **45-minute window** — Checkout intents expire 45 minutes after creation; they are immutable (create a new one to change details)
- **Rate limits** — 5 requests/second, 50 requests/day by default (contact Rye to increase)
- **Polling** — Poll `GET /checkout-intents/{id}` every 10 seconds; allow up to 45 minutes for offer retrieval
- **Payment** — Rye is merchant of record; payment is tokenized via Stripe through Rye

## How it works (overview)

The recommended two-call workflow:

1. **Look up the product** — `GET /api/v1/products/lookup?url=...` returns structured product data (price, availability, images, description) so you can verify the product is purchasable before committing
2. **Create a checkout intent** — `POST /api/v1/checkout-intents` with the product URL and buyer details; Rye handles offer resolution, tax, shipping, payment, and order confirmation

Every checkout flows through a **Checkout Intent** lifecycle:

```
Product Lookup → Create Intent → retrieving_offer → awaiting_confirmation → Confirm → placing_order → completed
  (optional)                            ↓                    ↓                                ↓
                                     failed               failed                           failed
```

**Two checkout paths:**

- **Multi-step** (`POST /checkout-intents` → poll → confirm) — Show the buyer final costs before charging. Use for user-facing flows.
- **Single-step** (`POST /checkout-intents/purchase`) — One call, fire-and-forget. Use for automated workflows that don't need buyer approval.

## Authentication

All requests require:

```
Authorization: Basic YOUR_API_KEY
```

Get your key from the Rye Console → Account page.

| Environment | Console | Base URL |
|-------------|---------|----------|
| **Staging** (no real orders) | staging.console.rye.com | `https://staging.api.rye.com/api/v1/` |
| **Production** (real orders) | console.rye.com | `https://api.rye.com/api/v1/` |

Always start with staging. Use Stripe test token `tok_visa` for payments.

## SDKs

| Language | Package | Install |
|----------|---------|---------|
| TypeScript | `checkout-intents` | `npm install checkout-intents` |
| Python | `checkout-intents` | `pip install checkout-intents` |
| Ruby | `checkout-intents` | `gem install checkout-intents` |
| Java | `com.rye:checkout-intents` | Maven Central |

SDKs provide `createAndPoll()` and `confirmAndPoll()` helpers that handle polling automatically.

## Test product URLs (staging)

- Shopify: `https://flybyjing.com/collections/shop/products/the-big-boi`
- Shopify (variant): `https://www.raakachocolate.com/products/blueberry-lemon?variant=41038993227863`
- Amazon: `https://www.amazon.com/Apple-MX532LL-A-AirTag/dp/B0CWXNS552/`

## Documentation links

- Quickstart: https://docs.rye.com/api-v2/example-flows/simple-checkout
- Single-step checkout: https://docs.rye.com/api-v2/example-flows/single-step-checkout
- API Reference: https://docs.rye.com/api-v2-experimental/api-reference
- Lifecycle: https://docs.rye.com/api-v2/checkout-intent-lifecycle
- Errors: https://docs.rye.com/api-v2/errors
- Go Live: https://docs.rye.com/api-v2/go-live
