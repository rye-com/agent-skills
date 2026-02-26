# Rye Integration Workflow

Follow these steps in order to integrate Rye's Universal Checkout into a developer's application. Do not skip ahead.

## Step 1: Analyze the codebase

Examine the project and identify:

- **Language and framework** (e.g., Next.js, Python/Flask, Ruby on Rails, Java/Spring)
- **Existing payment or checkout code** — look for Stripe, PayPal, or other payment integrations
- **API patterns** — REST clients, fetch wrappers, SDK usage, service layers
- **Environment variable conventions** — `.env`, `.env.local`, config files
- **Test infrastructure** — test framework, existing test patterns, CI setup

## Step 2: Propose an integration plan

Based on your analysis, present a plan covering:

1. **SDK choice** — Pick the SDK matching the project language (TypeScript, Python, Ruby, Java) or use raw HTTP if the language isn't covered
2. **Product lookup** — Whether to use the Product Data API (`GET /products/lookup`) to verify product details before checkout (recommended for user-facing flows)
3. **Checkout type** — Multi-step if the UI shows pricing to the user before charging; single-step for automated/background purchases
4. **Architecture** — Where checkout logic will live (API route, server action, service layer, background job)
5. **Lifecycle handling** — Use SDK helpers (`createAndPoll`, `confirmAndPoll`) when available; otherwise implement polling manually (every 10 seconds, up to 45 minutes)
6. **Environment setup** — Staging API key from https://staging.console.rye.com/account, env var naming matching project conventions

Format the plan as a numbered list. **Ask the developer to approve before proceeding.**

## Step 3: Implement

After approval, implement step by step:

### 3.1 Install the SDK

```bash
# TypeScript
npm install checkout-intents

# Python
pip install checkout-intents

# Ruby
gem install checkout-intents

# Java — add to pom.xml or build.gradle from Maven Central
```

### 3.2 Configure environment variables

Add to the project's env file (matching existing conventions):

```
RYE_API_KEY=your_staging_api_key_here
```

Get the staging key from https://staging.console.rye.com/account.

### 3.3 Add product lookup (recommended)

Use the Product Data API to verify product details before creating a checkout intent. This is a fast, read-only call that prevents wasting intents on unavailable products.

```typescript
// Look up product details before checkout
const product = await client.products.lookup({
  url: 'https://flybyjing.com/collections/shop/products/the-big-boi',
});

// Check availability before proceeding
if (!product.isPurchasable || product.availability !== 'in_stock') {
  // Handle unavailable product
}

// product.name, product.price, product.images, product.description
// are available to display to the user
```

```python
product = client.products.lookup(
    url="https://flybyjing.com/collections/shop/products/the-big-boi"
)

if not product.is_purchasable or product.availability != "in_stock":
    # Handle unavailable product
```

### 3.4 Create the checkout integration

**Multi-step checkout (TypeScript example):**

```typescript
import CheckoutIntents from 'checkout-intents';

const client = new CheckoutIntents({ apiKey: process.env.RYE_API_KEY });

// Create intent and poll until offer is ready
const intent = await client.checkoutIntents.createAndPoll({
  buyer: {
    firstName: 'John', lastName: 'Doe',
    email: 'john.doe@example.com', phone: '212-333-2121',
    address1: '123 Main St', city: 'New York',
    province: 'NY', postalCode: '10001', country: 'United States',
  },
  productUrl: 'https://flybyjing.com/collections/shop/products/the-big-boi',
  quantity: 1,
});

// intent.status is now 'awaiting_confirmation'
// intent contains pricing, shipping, and tax details
// Show these to the user, then confirm:

const confirmed = await client.checkoutIntents.confirmAndPoll(intent.id, {
  paymentMethod: { stripeToken: 'tok_visa', type: 'stripe_token' },
});

// confirmed.status is now 'completed' or 'failed'
```

**Multi-step checkout (Python example):**

```python
import os
from checkout_intents import CheckoutIntents

client = CheckoutIntents(os.environ["RYE_API_KEY"])

intent = client.checkout_intents.create_and_poll(
    buyer={
        "first_name": "John", "last_name": "Doe",
        "email": "john.doe@example.com", "phone": "212-333-2121",
        "address1": "123 Main St", "city": "New York",
        "province": "NY", "postal_code": "10001", "country": "US",
    },
    product_url="https://flybyjing.com/collections/shop/products/the-big-boi",
    quantity=1,
)

# Show pricing to user, then confirm
confirmed = client.checkout_intents.confirm_and_poll(
    intent.id,
    payment_method={"stripe_token": "tok_visa", "type": "stripe_token"},
)
```

### 3.5 Add error handling

Check `failureReason` when an intent reaches `failed` status:

```typescript
if (confirmed.status === 'failed') {
  console.error('Checkout failed:', confirmed.failureReason);
  // Handle the failure — show message to user, retry with new intent, etc.
}
```

Common failure reasons:
- Product out of stock or unavailable
- Invalid shipping address (non-US, invalid zip, etc.)
- Payment declined
- Intent expired (45-minute window)

### 3.6 Follow project conventions

- Match existing file structure, naming patterns, and code style
- Use the project's existing HTTP client or error handling patterns where appropriate
- Add the SDK to the project's dependency manifest (package.json, requirements.txt, Gemfile, etc.)
- Use `tok_visa` as the Stripe test token in staging

## Step 4: Verify

After implementation, verify correctness:

### 4.1 Lifecycle walkthrough

Confirm the code handles the full workflow:

```
product lookup (optional) → create → poll → awaiting_confirmation → confirm → poll → completed/failed
```

Check that:
- Product lookup verifies availability before checkout (if implemented)
- The intent is created with all required buyer fields
- Polling continues until status leaves `retrieving_offer`
- Offer details (price, shipping, tax) are accessible before confirmation
- Confirmation includes a valid payment method
- Both `completed` and `failed` terminal states are handled

### 4.2 Test with staging

Suggest running a test with a staging product URL:

- Shopify: `https://flybyjing.com/collections/shop/products/the-big-boi`
- Amazon: `https://www.amazon.com/Apple-MX532LL-A-AirTag/dp/B0CWXNS552/`

Use `tok_visa` for payment. No real charges in staging.

### 4.3 Edge cases

Point out any missing handling for:
- Network errors or timeouts during polling
- Intent expiration (45-minute window)
- Rate limit responses (429)
- Invalid product URLs or unsupported merchants
