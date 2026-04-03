# Balance / Prepaid Pricing

A dollar-denominated balance that depletes in real-time as the customer uses features. Best for products where cost varies per action, like AI token billing.

## How It Works

1. Plan includes a monthly **dollar balance** (e.g., $10.00 of AI usage)
2. Each action's cost is calculated in real-time and deducted from the balance
3. When balance reaches zero, behavior depends on configuration:
   - `blockOnExhaustion = true` -- Customer is blocked, must top up or wait for renewal
   - `blockOnExhaustion = false` -- Usage continues, overage charged at period end
4. Balance resets to `includedBalance` at each renewal

```
Plan: Pro ($49/mo)
  - $10.00 AI balance included
  - blockOnExhaustion: false

Customer uses:
  - 50,000 input tokens (claude-sonnet): $0.15
  - 10,000 output tokens (claude-sonnet): $0.60
  - 20,000 input tokens (gpt-4o): $0.05
  Total consumed: $0.80
  Remaining balance: $9.20
```

## When to Choose Balance

**Good fit:**
- Cost per action varies (different AI models, different operation types)
- You need real-time cost tracking and deduction
- Customers want to see a dollar balance, not an abstract credit count
- You are building on top of variable-cost APIs (AI providers, cloud services)

**Bad fit:**
- All actions cost the same amount (use credits -- simpler mental model)
- You do not need real-time deduction (use metered -- simpler to implement)
- Customers want to count discrete actions, not dollars (use credits)

## Balance vs Credits

| Dimension | Balance | Credits |
|-----------|---------|---------|
| Denomination | Dollars ($) | Abstract units |
| Cost per action | Variable (calculated per action) | Fixed (`creditsPerUnit`) |
| Mental model | "I have $7.50 left to spend" | "I have 35 credits left" |
| Sub-cent pricing | Yes (rate scale) | No |
| Best for | Variable-cost APIs, AI models | Fixed-cost discrete actions |
| Deduction timing | Real-time | Real-time |
| Top-up | Dollar amount | Credit pack |

## Key Concepts

### Included Balance

Monthly dollar amount bundled with the plan. Uses **rate scale** (10000 = $1.00).

```
includedBalance: 100000  → $10.00/month
includedBalance: 500000  → $50.00/month
```

The balance resets to this value at each renewal. Unused balance does not roll over.

### Block on Exhaustion

Configurable per plan:

| Setting | Behavior | Best For |
|---------|----------|----------|
| `false` (default) | Usage continues, overage billed at period end | Revenue-maximizing, less friction |
| `true` | Customer blocked at $0.00, must top up | Cost control, budget-sensitive customers |

### Top-ups

Customers can add to their balance via the Customer Portal at any time. Top-ups are applied immediately. At renewal, the balance resets to `includedBalance` (top-up amount is not preserved separately).

### Rate Scale

Balance and costs use rate scale (10000 = $1.00) for sub-cent precision. This matters for token-based billing where costs per unit are fractions of a cent.

## Implementation

### AI Token Billing (Automatic)

The easiest way to bill for AI usage. Wrap any AI SDK model with `tracked()`:

```typescript
import { Commet } from "@commet/node";
import { tracked } from "@commet/ai-sdk";
import { anthropic } from "@ai-sdk/anthropic";
import { generateText } from "ai";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

const result = await generateText({
  model: tracked(anthropic("claude-sonnet-4-20250514"), {
    commet,
    feature: "ai_generation",
    customerId: "user_123",
  }),
  prompt: "Explain quantum computing",
});
// Tokens counted, cost calculated from AI model catalog, balance deducted.
```

### Manual Token Tracking

For cases where you are not using `@commet/ai-sdk`:

```typescript
await commet.usage.track({
  externalId: "user_123",
  feature: "ai_generation",
  model: "anthropic/claude-3-opus",
  inputTokens: 1000,
  outputTokens: 500,
  cacheReadTokens: 100,
  cacheWriteTokens: 50,
});
```

When `model` is provided, `inputTokens` and `outputTokens` are required. Costs are calculated from the AI model catalog with configurable margins per feature per plan.

### Value-Based Balance Tracking

For non-AI features on a balance model, track with a value:

```typescript
await commet.usage.track({
  externalId: "user_123",
  feature: "data_processing",
  value: 500,  // 500 units processed
});
// Cost = 500 x overageUnitPrice, deducted from balance
```

### Check Balance

```typescript
const { data } = await commet.features.get({
  code: "ai_generation",
  externalId: "user_123",
});

// data.current   = consumed amount
// data.remaining = balance remaining
```

### Check Before Using (for blockOnExhaustion)

```typescript
const { data } = await commet.features.canUse({
  code: "ai_generation",
  externalId: "user_123",
});

if (!data.allowed) {
  // Balance exhausted and blocking is enabled
  return { error: "insufficient_balance" };
}
```

## Cost Calculation (AI Models)

When tracking AI token usage, costs are calculated from the model catalog:

```
inputCost    = ceil(inputTokens    x inputPricePerMillionTokens    / 1,000,000)
outputCost   = ceil(outputTokens   x outputPricePerMillionTokens   / 1,000,000)
subtotal     = inputCost + outputCost + cacheReadCost + cacheWriteCost
marginAmount = ceil(subtotal x margin / 10000)
total        = subtotal + marginAmount
```

All values in rate scale. Margin is configured per feature per plan in basis points (2000 = 20% markup). All costs use `Math.ceil()` to prevent underbilling.

## Plan Configuration Example

```
Plan: Starter ($29/mo)
  Balance: $5.00/month (includedBalance: 50000)
  blockOnExhaustion: true
  Feature: ai_generation
    type: metered
    pricingMode: ai_model
    margin: 2000  (20% markup on AI costs)

Plan: Pro ($49/mo)
  Balance: $25.00/month (includedBalance: 250000)
  blockOnExhaustion: false
  Feature: ai_generation
    type: metered
    pricingMode: ai_model
    margin: 1500  (15% markup)

Plan: Enterprise ($199/mo)
  Balance: $100.00/month (includedBalance: 1000000)
  blockOnExhaustion: false
  Feature: ai_generation
    type: metered
    pricingMode: ai_model
    margin: 0  (pass-through cost)
```

## Billing Behavior

| Event | Balance |
|-------|---------|
| Renewal | Reset to `includedBalance` |
| Upgrade | Reset to new plan's `includedBalance` |
| Downgrade | At renewal, reset to new value |
| Top-up | Added immediately to current balance |
| Cancellation | Balance lost |
| Reactivation | Reset to `includedBalance` |
| `includedBalance` increase (founder) | Immediate (benefits customer) |
| `includedBalance` decrease (founder) | At renewal |
| Overage unit price change (founder) | At renewal |
