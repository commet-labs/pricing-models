---
name: pricing-models
description: Use when choosing a pricing model for a SaaS product — metered (pay per use), credits (block when exhausted), balance (prepaid spend), seats (per-user), or boolean (feature flags). Covers decision frameworks, implementation patterns, and when to use each model.
license: Apache-2.0
metadata:
  author: commet
  version: "1.0.0"
  homepage: https://commet.co
  source: https://github.com/commet-labs/pricing-models
references:
  - metered.md
  - credits.md
  - balance.md
  - seats.md
  - hybrid-models.md
  - choosing-a-model.md
---

# Pricing Models

## Decision Matrix

| Question | Metered | Credits | Balance | Seats |
|----------|---------|---------|---------|-------|
| Is usage predictable? | No | Somewhat | Somewhat | Yes |
| Need hard limits? | No | Yes | Optional | N/A |
| Per-user value? | No | No | No | Yes |
| Multi-feature spend? | No | No | Yes | No |
| Sub-cent pricing? | Yes | No | Yes | No |
| Customer wants cost control? | Low | High | Medium | High |

## Quick Comparison

| Model | Charges When | Blocks on Limit | Best For |
|-------|-------------|-----------------|----------|
| **Metered** | Period end (true-up) | Never | API calls, bandwidth, storage |
| **Credits** | Upfront (blocks) | Yes, hard stop | Image generation, exports, compute jobs |
| **Balance** | Real-time deduction | Configurable | AI token usage, multi-feature platforms |
| **Seats** | Period start (advance) + true-up | N/A | Team tools, per-user licenses |
| **Boolean** | Included in plan base | N/A | Feature flags, plan differentiation |

## Code Examples

### Metered -- Track API usage

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

await commet.usage.track({
  externalId: "user_123",
  feature: "api_calls",
  value: 1,
  idempotencyKey: "req_abc123",
});
```

### Credits -- Check before consuming

```typescript
const { data } = await commet.features.canUse({
  code: "image_generations",
  externalId: "user_123",
});

if (!data.allowed) {
  // Customer exhausted credits -- prompt to buy a credit pack
  const { data: portalData } = await commet.portal.getUrl({ externalId: "user_123" });
  return redirect(portalData.portalUrl);
}

await commet.usage.track({
  externalId: "user_123",
  feature: "image_generations",
  value: 1,
});
```

### Balance -- AI token billing

```typescript
import { tracked } from "@commet/ai-sdk";
import { anthropic } from "@ai-sdk/anthropic";
import { generateText } from "ai";

const result = await generateText({
  model: tracked(anthropic("claude-sonnet-4-20250514"), {
    commet,
    feature: "ai_generation",
    customerId: "user_123",
  }),
  prompt: "Explain quantum computing",
});
// Tokens tracked, cost calculated, balance deducted automatically.
```

### Seats -- Manage team members

```typescript
await commet.seats.add({
  externalId: "org_456",
  seatType: "editor",
  count: 3,
});

const { data } = await commet.seats.getBalance({
  externalId: "org_456",
  seatType: "editor",
});
// data.current = 3
```

### Boolean -- Check feature access

```typescript
const { data } = await commet.features.check({
  code: "custom_branding",
  externalId: "user_123",
});

if (!data.allowed) {
  // Feature not included in their plan
}
```

## Detailed References

| Need | Reference |
|------|-----------|
| **Choosing between models** | [choosing-a-model.md](references/choosing-a-model.md) -- Decision framework, real-world examples |
| **Metered pricing** | [metered.md](references/metered.md) -- Pay-per-use, overage, included amounts |
| **Credit-based pricing** | [credits.md](references/credits.md) -- Block purchases, hard limits, credit packs |
| **Balance / prepaid** | [balance.md](references/balance.md) -- Prepaid spend, AI billing, top-ups |
| **Seat-based pricing** | [seats.md](references/seats.md) -- Per-user, advance + true-up, seat types |
| **Combining models** | [hybrid-models.md](references/hybrid-models.md) -- Base + usage, seats + metered, addons |
