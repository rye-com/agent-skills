# Rye API Reference

Complete API reference for Rye's Product Data API and Universal Checkout API.

## Authentication

All requests require a Basic auth header:

```
Authorization: Basic YOUR_API_KEY
```

Get your key from the Rye Console → Account page.

## Environments

| Environment | Console | Base URL |
|-------------|---------|----------|
| **Staging** | staging.console.rye.com | `https://staging.api.rye.com/api/v1/` |
| **Production** | console.rye.com | `https://api.rye.com/api/v1/` |

Staging uses Stripe test cards (`tok_visa`). No real orders or charges. Always start here.

## Endpoints

### Product Lookup

```
GET /api/v1/products/lookup?url={productUrl}
```

Returns structured product data for any supported product URL. Use this to verify a product's price, availability, and details before creating a checkout intent.

**Query parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `url` | string | Yes | The product URL to look up |

**curl example:**

```bash
curl --request GET \
  --url "https://staging.api.rye.com/api/v1/products/lookup?url=https://flybyjing.com/collections/shop/products/the-big-boi" \
  --header "Authorization: Bearer $RYE_API_KEY"
```

**Response (200):**

```json
{
  "id": "flybyjing.com:the-big-boi",
  "name": "The Big Boi",
  "url": "https://flybyjing.com/collections/shop/products/the-big-boi",
  "description": "Product description text",
  "brand": "Fly By Jing",
  "sku": "FBJ-BB-001",
  "isPurchasable": true,
  "availability": "in_stock",
  "price": {
    "currencyCode": "USD",
    "amountSubunits": 4900
  },
  "images": [
    {
      "url": "https://cdn.shopify.com/...",
      "isFeatured": true
    }
  ]
}
```

**Response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique product identifier (e.g., `"amazon.com:B0DFC9MT8Q"`) |
| `name` | string | Product title |
| `url` | string | Product URL |
| `description` | string or null | Product description |
| `brand` | string or null | Manufacturer/brand name |
| `sku` | string or null | Stock keeping unit |
| `isPurchasable` | boolean | Whether the product can be purchased through Rye |
| `availability` | enum | `in_stock`, `out_of_stock`, `preorder`, `backorder`, or `unknown` |
| `price.currencyCode` | string | ISO currency code (e.g., `"USD"`) |
| `price.amountSubunits` | integer | Price in smallest currency unit (e.g., `4900` = $49.00) |
| `images` | array | Product images, each with `url` and `isFeatured` |

**Error codes:** 401 (auth failed), 404 (product not found), 422 (validation error), 500 (server error)

**Note:** Product Lookup uses `Authorization: Bearer` (not `Basic`). The same API key works for both.

---

### Create Checkout Intent

```
POST /api/v1/checkout-intents
```

Creates a multi-step checkout intent. Poll until `awaiting_confirmation`, then confirm with payment.

**Request body:**

```json
{
  "buyer": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "phone": "212-333-2121",
    "address1": "123 Main St",
    "address2": "",
    "city": "New York",
    "province": "NY",
    "postalCode": "10001",
    "country": "US"
  },
  "productUrl": "https://flybyjing.com/collections/shop/products/the-big-boi",
  "quantity": 1,
  "promoCodes": ["SAVE10"],
  "discoverPromoCodes": false,
  "variantSelections": [
    { "label": "Size", "value": "Large" }
  ]
}
```

**Required fields:** `buyer` (firstName, lastName, email, phone, address1, city, province, postalCode, country), `productUrl`, `quantity`

**Optional fields:** `promoCodes`, `discoverPromoCodes`, `variantSelections`, `address2`

**curl example:**

```bash
curl --request POST \
  --url https://staging.api.rye.com/api/v1/checkout-intents \
  --header "Authorization: Basic $RYE_API_KEY" \
  --header 'Content-Type: application/json' \
  --data '{
    "buyer": {
      "firstName": "John",
      "lastName": "Doe",
      "email": "john.doe@example.com",
      "phone": "212-333-2121",
      "address1": "123 Main St",
      "city": "New York",
      "province": "NY",
      "postalCode": "10001",
      "country": "US"
    },
    "productUrl": "https://flybyjing.com/collections/shop/products/the-big-boi",
    "quantity": "1"
  }'
```

### Retrieve Checkout Intent

```
GET /api/v1/checkout-intents/{id}
```

Poll this endpoint to check intent status. Poll every 10 seconds. Allow up to 45 minutes for `retrieving_offer` to transition.

**Response includes:** status, pricing (subtotal, shipping, tax, total), product details, failure reason (if failed).

### Confirm Checkout Intent

```
POST /api/v1/checkout-intents/{id}/confirm
```

Confirms an intent that is in `awaiting_confirmation` status. Triggers order placement.

**Request body:**

```json
{
  "paymentMethod": {
    "stripeToken": "tok_visa",
    "type": "stripe_token"
  }
}
```

**curl example:**

```bash
curl --request POST \
  --url https://staging.api.rye.com/api/v1/checkout-intents/{id}/confirm \
  --header "Authorization: Basic $RYE_API_KEY" \
  --header 'Content-Type: application/json' \
  --data '{
    "paymentMethod": {
      "stripeToken": "tok_visa",
      "type": "stripe_token"
    }
  }'
```

In staging, use `tok_visa`. In production, generate tokens via Stripe Elements using Rye's publishable key.

### Single-Step Purchase

```
POST /api/v1/checkout-intents/purchase
```

One-call fire-and-forget checkout. Rye handles offer retrieval, payment, and order placement asynchronously.

**Request body:**

```json
{
  "buyer": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "phone": "212-333-2121",
    "address1": "123 Main St",
    "city": "New York",
    "province": "NY",
    "postalCode": "10001",
    "country": "US"
  },
  "productUrl": "https://flybyjing.com/collections/shop/products/the-big-boi",
  "quantity": 1,
  "paymentMethod": {
    "stripeToken": "tok_visa",
    "type": "stripe_token"
  },
  "constraints": {
    "maxShippingPrice": 1000,
    "maxTotalPrice": 5000
  }
}
```

**Additional fields:** `paymentMethod` (required), `constraints` (optional)

## Checkout Intent Lifecycle

```
Create Intent → retrieving_offer → awaiting_confirmation → Confirm → placing_order → completed
                       ↓                    ↓                                ↓
                    failed               failed                           failed
```

| Status | Description |
|--------|-------------|
| `retrieving_offer` | Rye is fetching pricing from the merchant. Poll every 10s. |
| `awaiting_confirmation` | Offer ready. Review pricing and confirm with payment. |
| `placing_order` | Payment accepted, order being placed with merchant. |
| `completed` | Order placed successfully. Confirmation sent to buyer email. |
| `failed` | Something went wrong. Check `failureReason`. |

## SDKs

### TypeScript (`checkout-intents` on npm)

```bash
npm install checkout-intents
```

```typescript
import CheckoutIntents from 'checkout-intents';

const client = new CheckoutIntents({ apiKey: process.env.RYE_API_KEY });

// Look up product details before buying
const product = await client.products.lookup({
  url: 'https://example.com/product',
});
// product.isPurchasable, product.price, product.availability

// Create and poll until awaiting_confirmation
const intent = await client.checkoutIntents.createAndPoll({
  buyer: { /* ... */ },
  productUrl: 'https://example.com/product',
  quantity: 1,
});

// Confirm and poll until completed/failed
const result = await client.checkoutIntents.confirmAndPoll(intent.id, {
  paymentMethod: { stripeToken: 'tok_visa', type: 'stripe_token' },
});

// Single-step purchase
const purchase = await client.checkoutIntents.purchase({
  buyer: { /* ... */ },
  productUrl: 'https://example.com/product',
  quantity: 1,
  paymentMethod: { stripeToken: 'tok_visa', type: 'stripe_token' },
});
```

**Helper methods:**
- `products.lookup()` — Look up product details by URL
- `createAndPoll()` — Creates intent and polls until `awaiting_confirmation` or `failed`
- `confirmAndPoll()` — Confirms intent and polls until `completed` or `failed`
- `purchase()` — Single-step fire-and-forget checkout

### Python (`checkout-intents` on PyPI)

```bash
pip install checkout-intents
```

```python
import os
from checkout_intents import CheckoutIntents

client = CheckoutIntents(os.environ["RYE_API_KEY"])

# Look up product details before buying
product = client.products.lookup(url="https://example.com/product")
# product.is_purchasable, product.price, product.availability

# Create and poll
intent = client.checkout_intents.create_and_poll(
    buyer={ "first_name": "John", "last_name": "Doe", ... },
    product_url="https://example.com/product",
    quantity=1,
)

# Confirm and poll
result = client.checkout_intents.confirm_and_poll(
    intent.id,
    payment_method={"stripe_token": "tok_visa", "type": "stripe_token"},
)

# Single-step purchase
purchase = client.checkout_intents.purchase(
    buyer={ ... },
    product_url="https://example.com/product",
    quantity=1,
    payment_method={"stripe_token": "tok_visa", "type": "stripe_token"},
)
```

**Helper methods:**
- `products.lookup()` — Look up product details by URL
- `create_and_poll()` — Creates intent and polls until `awaiting_confirmation` or `failed`
- `confirm_and_poll()` — Confirms intent and polls until `completed` or `failed`
- `purchase()` — Single-step fire-and-forget checkout

### Ruby (`checkout-intents` gem)

```bash
gem install checkout-intents
```

### Java (`com.rye:checkout-intents` on Maven Central)

Add to your build tool from Maven Central.

## Additional Features

### Promo codes

Pass an array of codes when creating an intent. The first valid code is applied automatically.

```json
{
  "promoCodes": ["SAVE10", "WELCOME", "FREESHIP"]
}
```

- Up to 16 codes per intent
- Alphanumeric only, max 32 characters each

### Automatic promo code discovery

Set `discoverPromoCodes: true` and Rye automatically finds and applies the best available promo code for the merchant. Codes are aggregated from multiple sources and cached. Discovered codes merge with any manual `promoCodes` you provide.

```json
{
  "discoverPromoCodes": true
}
```

### Variant selection

For Amazon and Shopify products, use a deep-link URL pointing to the exact variant. For other merchants, use the `variantSelections` field with exact `label`/`value` pairs matching the product page:

```json
{
  "variantSelections": [
    { "label": "Size", "value": "8.5" },
    { "label": "Color", "value": "Black" }
  ]
}
```

### Price constraints (single-step only)

Cap costs on single-step purchases:

```json
{
  "constraints": {
    "maxShippingPrice": 1000,
    "maxTotalPrice": 5000
  }
}
```

Values are in cents. The purchase will fail if costs exceed these limits.

## Limitations

- **US-only shipping** — International addresses cause checkout intents to fail
- **Physical products only** — Digital goods are not supported
- **One product per checkout** — Multiple quantities OK, multi-product carts not supported
- **No post-purchase webhooks** — Order tracking goes to buyer's email
- **Checkout intents are immutable** — To change details, create a new intent
- **45-minute confirmation window** — Intents expire after creation
- **Rate limits** — 5 requests/second, 50 requests/day (contact Rye to increase)
- **Polling** — Every 10 seconds; allow up to 45 minutes for offer retrieval
- **Rye is merchant of record** — Payment is tokenized via Stripe through Rye

## Test product URLs (staging)

- Shopify: `https://flybyjing.com/collections/shop/products/the-big-boi`
- Shopify (variant): `https://www.raakachocolate.com/products/blueberry-lemon?variant=41038993227863`
- Amazon: `https://www.amazon.com/Apple-MX532LL-A-AirTag/dp/B0CWXNS552/`

## Documentation links

- Product Lookup: https://docs.rye.com/api-v2-experimental/api-reference/products/lookup-product
- Quickstart (multi-step): https://docs.rye.com/api-v2/example-flows/simple-checkout
- Single-step checkout: https://docs.rye.com/api-v2/example-flows/single-step-checkout
- API Reference: https://docs.rye.com/api-v2-experimental/api-reference
- Checkout Intent Lifecycle: https://docs.rye.com/api-v2/checkout-intent-lifecycle
- Payment Setup (Stripe): https://docs.rye.com/api-v2/payment-flow-setup
- Error Handling: https://docs.rye.com/api-v2/errors
- Go Live Checklist: https://docs.rye.com/api-v2/go-live
- Variants Guide: https://docs.rye.com/api-v2/variants
- Promo Codes: https://docs.rye.com/api-v2/promo-codes
- API Limitations: https://docs.rye.com/api-v2/developer-notes
- SDKs: https://docs.rye.com/api-v2/sdk
- Full docs for LLMs: https://docs.rye.com/llms-full.txt
