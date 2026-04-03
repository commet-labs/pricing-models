# Credit-Based Pricing

Buy blocks of credits, consume them per action, hard stop when exhausted. Best for discrete, countable actions where customers want clear spending limits.

## How It Works

1. Plan includes a monthly **credit allowance** (resets each period)
2. Each action consumes a defined number of credits (`creditsPerUnit`)
3. When credits reach zero, the customer is **blocked** until they buy more or wait for renewal
4. Customers can purchase **credit packs** for immediate top-up

```
Plan: Creator ($19/mo)
  - 100 credits/month included
  - Image generation: 2 credits each
  - Video export: 10 credits each

Customer uses: 40 generations (80 credits) + 2 exports (20 credits)
Remaining: 0 credits → blocked until renewal or credit pack purchase
```

## When to Choose Credits

**Good fit:**
- Each action is discrete and visible to the customer (generation, export, transformation)
- Customers want to know exactly how many actions they have left
- You want hard limits to prevent runaway costs
- The "credits remaining" counter drives engagement and urgency

**Bad fit:**
- Costs vary significantly per action (use balance -- dollar-denominated)
- You need real-time dollar-level cost tracking (use balance)
- Customers expect unlimited usage within their plan (use metered)
- The feature is always-on access, not per-action (use boolean)

## Key Concepts

### Plan Credits vs Purchased Credits

Two separate balances with different lifecycles:

| Type | Source | Resets at Renewal | Survives Cancellation |
|------|--------|-------------------|----------------------|
| Plan credits | Included in plan | Yes (reset to `includedCredits`) | No |
| Purchased credits | Bought via credit packs | No (never expire) | Yes (preserved) |

Consumption order: **plan credits first**, then purchased credits. This ensures purchased credits are spent last.

### Credits Per Unit

Each metered feature defines how many credits one unit of that feature consumes:

```
Image generation: creditsPerUnit = 2  → 1 generation costs 2 credits
HD video export:  creditsPerUnit = 10 → 1 export costs 10 credits
PDF report:       creditsPerUnit = 1  → 1 report costs 1 credit
```

### Credit Packs

Purchasable bundles available in the Customer Portal:

```
Pack: Small  → 50 credits for $9.99
Pack: Medium → 150 credits for $24.99
Pack: Large  → 500 credits for $69.99
```

Credit packs are applied immediately to `purchasedCredits`. Since purchased credits never expire, they act as a permanent safety net.

## Implementation

### Check before consuming

Always check availability before performing the action:

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

const { data } = await commet.features.canUse({
  code: "image_generations",
  externalId: "user_123",
});

if (!data.allowed) {
  // Customer has no credits left
  // Show a "buy more credits" prompt or redirect to portal
  const { data: portalData } = await commet.portal.getUrl({
    externalId: "user_123",
  });
  return { error: "insufficient_credits", portalUrl: portalData.portalUrl };
}

// Perform the action, then track
const image = await generateImage(prompt);

await commet.usage.track({
  externalId: "user_123",
  feature: "image_generations",
  value: 1,
});
```

### Check remaining balance

```typescript
const { data } = await commet.features.get({
  code: "image_generations",
  externalId: "user_123",
});

// data.current   = 80  (credits consumed this period)
// data.included  = 100 (plan credits per period)
// data.remaining = 20  (credits left)
```

### Scoped API

```typescript
const customer = commet.customer("user_123");

const { data } = await customer.features.canUse("image_generations");
if (data.allowed) {
  await customer.usage.track("image_generations", 1);
}
```

### List available credit packs

```typescript
const { data: packs } = await commet.creditPacks.list();
// [{ id, name, credits, price, currency }]
```

## Plan Configuration Example

```
Plan: Free ($0/mo)
  Credits: 10/month
  Feature: image_generations
    type: metered
    creditsPerUnit: 2
    includedAmount: 5  (10 credits / 2 per unit)

Plan: Creator ($19/mo)
  Credits: 100/month
  Feature: image_generations
    type: metered
    creditsPerUnit: 2
    includedAmount: 50  (100 credits / 2 per unit)
  Feature: video_exports
    type: metered
    creditsPerUnit: 10
    includedAmount: 10  (100 credits / 10 per unit)

Credit Packs:
  - 50 credits: $9.99
  - 150 credits: $24.99
  - 500 credits: $69.99
```

## Billing Behavior

| Event | Plan Credits | Purchased Credits |
|-------|-------------|-------------------|
| Renewal | Reset to `includedCredits` | Intact |
| Upgrade | Reset to new plan's value | Intact |
| Downgrade | At renewal, reset to new value | Intact |
| Cancellation | Set to 0 | Preserved |
| Reactivation | Reset to `includedCredits` | Restored |
| Pause | Set to 0 | Preserved |
| Resume | Reset to `includedCredits` | Restored |

### Credits Per Unit Changes

When the founder changes `creditsPerUnit` for a feature:

| Change | Effect |
|--------|--------|
| Increase (costs more credits) | Immediate for new customers. **At renewal** for existing. |
| Decrease (costs fewer credits) | **Immediate** for everyone (benefits customer). |

### Credit Pack Price Changes

Changes apply immediately for new purchases. Previously purchased credits are not affected.
