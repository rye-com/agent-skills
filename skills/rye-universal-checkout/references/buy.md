# Rye Agent Commerce Workflow

Use this reference when an AI agent needs to programmatically purchase a physical product on behalf of a user.

## When to use

- AI shopping assistants that buy products for users
- Chat commerce bots executing purchase commands
- Automated purchasing workflows (price monitoring, restocking, gifting)
- Any agent that needs to turn a product URL into a completed order

## Prerequisites

Before buying:
- You have the user's **shipping address** (US only)
- You have a **product URL** from a supported merchant
- You have a **payment method** (Stripe token)
- The product is **physical** (not digital)
- The user has **authorized the purchase**

## Step 0: Look up the product (recommended)

Before committing to a checkout, verify the product is purchasable and check its price. This prevents wasting a checkout intent on an out-of-stock or unsupported product.

**TypeScript:**

```typescript
import CheckoutIntents from 'checkout-intents';

const client = new CheckoutIntents({ apiKey: process.env.RYE_API_KEY });

const product = await client.products.lookup({ url: productUrl });

if (!product.isPurchasable) {
  // Product can't be purchased — inform the user
}
if (product.availability !== 'in_stock') {
  // Product is out of stock, on backorder, etc.
}
// product.price.amountSubunits gives the price in cents (e.g., 4900 = $49.00)
// product.name, product.description, product.images are available for display
```

**Python:**

```python
product = client.products.lookup(url=product_url)

if not product.is_purchasable:
    # Product can't be purchased
if product.availability != "in_stock":
    # Product is out of stock
# product.price.amount_subunits gives the price in cents
```

**curl:**

```bash
curl --request GET \
  --url "https://staging.api.rye.com/api/v1/products/lookup?url=$PRODUCT_URL" \
  --header "Authorization: Bearer $RYE_API_KEY"
```

Use this data to confirm the product with the user before proceeding to checkout.

---

## Multi-step flow (recommended)

Use this when you need to show the user pricing before charging. This is the safer default.

### Flow: lookup → create → poll → review → confirm → poll → done

**TypeScript:**

```typescript
import CheckoutIntents from 'checkout-intents';

const client = new CheckoutIntents({ apiKey: process.env.RYE_API_KEY });

// Step 1: Create intent and wait for offer
const intent = await client.checkoutIntents.createAndPoll({
  buyer: {
    firstName: 'Jane', lastName: 'Smith',
    email: 'jane@example.com', phone: '555-123-4567',
    address1: '456 Oak Ave', city: 'San Francisco',
    province: 'CA', postalCode: '94102', country: 'United States',
  },
  productUrl: productUrl,  // The URL the user wants to buy
  quantity: 1,
});

// Step 2: Review the offer — present to user
const offer = {
  productName: intent.productTitle,
  price: intent.subtotal,
  shipping: intent.shippingTotal,
  tax: intent.taxTotal,
  total: intent.total,
};
// Show `offer` to the user and get confirmation

// Step 3: Confirm the purchase
const result = await client.checkoutIntents.confirmAndPoll(intent.id, {
  paymentMethod: { stripeToken: paymentToken, type: 'stripe_token' },
});

if (result.status === 'completed') {
  // Order placed — confirmation sent to buyer's email
} else {
  // Order failed — check result.failureReason
}
```

**Python:**

```python
import os
from checkout_intents import CheckoutIntents

client = CheckoutIntents(os.environ["RYE_API_KEY"])

# Step 1: Create intent and wait for offer
intent = client.checkout_intents.create_and_poll(
    buyer={
        "first_name": "Jane", "last_name": "Smith",
        "email": "jane@example.com", "phone": "555-123-4567",
        "address1": "456 Oak Ave", "city": "San Francisco",
        "province": "CA", "postal_code": "94102", "country": "US",
    },
    product_url=product_url,
    quantity=1,
)

# Step 2: Review the offer
offer = {
    "product": intent.product_title,
    "price": intent.subtotal,
    "shipping": intent.shipping_total,
    "tax": intent.tax_total,
    "total": intent.total,
}
# Present offer to user and get confirmation

# Step 3: Confirm
result = client.checkout_intents.confirm_and_poll(
    intent.id,
    payment_method={"stripe_token": payment_token, "type": "stripe_token"},
)
```

## Single-step flow (fire-and-forget)

Use this for automated workflows where the user has pre-authorized the purchase and doesn't need to review pricing.

**TypeScript:**

```typescript
import CheckoutIntents from 'checkout-intents';

const client = new CheckoutIntents({ apiKey: process.env.RYE_API_KEY });

const purchase = await client.checkoutIntents.purchase({
  buyer: {
    firstName: 'Jane', lastName: 'Smith',
    email: 'jane@example.com', phone: '555-123-4567',
    address1: '456 Oak Ave', city: 'San Francisco',
    province: 'CA', postalCode: '94102', country: 'United States',
  },
  productUrl: productUrl,
  quantity: 1,
  paymentMethod: { stripeToken: paymentToken, type: 'stripe_token' },
  // Optional: set price constraints to avoid overpaying
  constraints: {
    maxTotalPrice: 5000,     // cents — refuse if total exceeds $50
    maxShippingPrice: 1000,  // cents — refuse if shipping exceeds $10
  },
});

// purchase.id gives you the intent ID to track status
// Order confirmation goes to buyer's email
```

**Python:**

```python
purchase = client.checkout_intents.purchase(
    buyer={
        "first_name": "Jane", "last_name": "Smith",
        "email": "jane@example.com", "phone": "555-123-4567",
        "address1": "456 Oak Ave", "city": "San Francisco",
        "province": "CA", "postal_code": "94102", "country": "US",
    },
    product_url=product_url,
    quantity=1,
    payment_method={"stripe_token": payment_token, "type": "stripe_token"},
    constraints={
        "max_total_price": 5000,
        "max_shipping_price": 1000,
    },
)
```

## Choosing between flows

| Scenario | Flow | Why |
|----------|------|-----|
| User says "buy this for me" | Lookup → Multi-step | Verify availability, show price before charging |
| User says "buy if under $50" | Lookup → Single-step with constraints | Check price first, then price cap handles approval |
| Automated restocking/gifting | Single-step | No human in the loop |
| Price-sensitive purchase | Lookup → Multi-step | User reviews exact costs |
| "Is this product available?" | Lookup only | No checkout needed |

**Always start with Product Lookup** when the agent has time to verify — it's a fast, read-only call that prevents wasting checkout intents on unavailable products.

## Handling promo codes

Pass promo codes to reduce the purchase price:

```typescript
const intent = await client.checkoutIntents.createAndPoll({
  // ... buyer and product details
  promoCodes: ['SAVE10', 'WELCOME'],  // First valid code is applied
});
```

Or let Rye find the best code automatically:

```typescript
const intent = await client.checkoutIntents.createAndPoll({
  // ... buyer and product details
  discoverPromoCodes: true,  // Rye finds and applies the best available code
});
```

## Test product URLs (staging)

Use these URLs for testing in the staging environment:

- `https://flybyjing.com/collections/shop/products/the-big-boi`
- `https://www.raakachocolate.com/products/blueberry-lemon?variant=41038993227863`
- `https://www.amazon.com/Apple-MX532LL-A-AirTag/dp/B0CWXNS552/`

Use `tok_visa` as the Stripe test token. No real charges in staging.

## Error handling

Always check for failures:

```typescript
if (result.status === 'failed') {
  switch (result.failureReason) {
    case 'out_of_stock':
      // Product unavailable — inform user, suggest alternatives
      break;
    case 'address_invalid':
      // Bad address — ask user to correct
      break;
    case 'payment_declined':
      // Payment issue — ask for different payment method
      break;
    default:
      // Unknown failure — log and inform user
      break;
  }
}
```

Common failures:
- Product out of stock or discontinued
- Invalid or non-US shipping address
- Payment method declined
- Intent expired (45-minute window exceeded)
- Rate limit hit (5 req/s, 50 req/day)
