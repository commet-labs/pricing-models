# Metered Pricing

Pay for what you use, billed at the end of the billing period (true-up). The most common model for API platforms and infrastructure products.

## How It Works

1. Plan defines an **included amount** (free units per period)
2. Customer uses the feature throughout the billing period
3. At period end, usage above the included amount is charged as **overage**
4. Customer is never blocked -- usage always goes through

```
Period: March 1 - March 31
Included: 10,000 API calls
Used: 14,500 API calls
Overage: 4,500 x $0.002 = $9.00
Invoice: $29.00 (base) + $9.00 (overage) = $38.00
```

## When to Choose Metered

**Good fit:**
- Usage is highly variable between customers and between months
- Marginal cost per unit is real (API calls, bandwidth, compute)
- Customers expect to pay proportionally to their usage
- You want low friction -- no hard limits, no purchase gates

**Bad fit:**
- Customers need hard spending caps (use credits or balance instead)
- Usage is too small to meaningfully bill (use boolean)
- You need real-time cost control (use balance)

## Key Concepts

### Included Amount

Free units bundled with the plan base price. Set this high enough that most small customers never hit overage. This creates a natural upgrade path: "You are using 90% of your included API calls."

### Overage Unit Price

Price per unit above the included amount. Uses **rate scale** (10000 = $1.00) to support sub-cent pricing.

| Rate Value | Price Per Unit |
|-----------|---------------|
| 10000 | $1.00 |
| 1000 | $0.10 |
| 100 | $0.01 |
| 20 | $0.002 |
| 1 | $0.0001 |

### Unlimited Flag

Set `unlimited: true` to remove any cap. The customer still pays for overage, but there is no artificial limit. Alternatively, if `overageEnabled: false`, the included amount acts as a hard cap (but this is uncommon for metered -- consider credits if you need hard limits).

## Implementation

### Track usage events

Every billable action sends a usage event:

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

// Track a single API call
await commet.usage.track({
  externalId: "user_123",
  feature: "api_calls",
  value: 1,
  idempotencyKey: `req_${requestId}`,
});
```

### Track in batches

For high-volume usage, batch events to reduce API calls:

```typescript
await commet.usage.trackBatch({
  events: [
    { externalId: "user_123", feature: "api_calls", value: 1 },
    { externalId: "user_456", feature: "api_calls", value: 1 },
    { externalId: "user_789", feature: "api_calls", value: 3 },
  ],
});
```

### Check current usage

```typescript
const { data } = await commet.features.get({
  code: "api_calls",
  externalId: "user_123",
});

// data.current   = 8,500  (used this period)
// data.included  = 10,000 (free units)
// data.remaining = 1,500  (before overage kicks in)
// data.overage   = 0      (overage units so far)
```

### Scoped API for cleaner code

```typescript
const customer = commet.customer("user_123");

await customer.usage.track("api_calls", 1);
const { data } = await customer.features.get("api_calls");
```

## Idempotency

Always pass an `idempotencyKey` to prevent duplicate billing on retries. Use the request ID, job ID, or any unique identifier for the action being tracked.

```typescript
// If this request is retried, the second call is a no-op
await commet.usage.track({
  externalId: "user_123",
  feature: "api_calls",
  value: 1,
  idempotencyKey: `req_${request.id}`,
});
```

## Plan Configuration Example

```
Plan: Starter ($29/mo)
  Feature: api_calls
    type: metered
    includedAmount: 10,000
    overageEnabled: true
    overageUnitPrice: 20  (rate scale: $0.002/call)

Plan: Pro ($99/mo)
  Feature: api_calls
    type: metered
    includedAmount: 100,000
    overageEnabled: true
    overageUnitPrice: 10  (rate scale: $0.001/call)

Plan: Enterprise ($499/mo)
  Feature: api_calls
    type: metered
    unlimited: true
    overageUnitPrice: 5  (rate scale: $0.0005/call)
```

## Billing Behavior

| Event | What Happens |
|-------|-------------|
| Period end | Overage calculated and invoiced |
| Mid-period upgrade | Usage carries over, new included amount and overage price apply |
| Mid-period downgrade | Takes effect at renewal, current period uses current plan |
| Cancellation | Final overage charged on last invoice |
| Price increase by founder | Takes effect at renewal (does not hurt customer mid-period) |
| Included amount increase by founder | Immediate (benefits customer) |
