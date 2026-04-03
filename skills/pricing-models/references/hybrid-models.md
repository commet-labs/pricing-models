# Hybrid Pricing Models

Most real products combine multiple pricing models. A base plan provides the foundation, and individual features use different models depending on what they measure.

## How Hybrid Works

A plan has one **consumption model** (metered, credits, or balance) that applies to all metered features. On top of that, seats and boolean features work independently. The combination creates a hybrid pricing structure.

```
Plan: Pro ($99/mo)                          ← Base price (fixed)
  Consumption model: metered                ← Applies to all metered features
  Features:
    - api_calls: 50,000 included, $0.001/extra   ← Metered
    - storage_gb: 10 included, $0.50/extra        ← Metered
    - custom_branding: enabled                    ← Boolean
    - sso: disabled                               ← Boolean (upgrade gate)
    - team_members: 5 included, $25/extra         ← Seats
```

## Common Hybrid Patterns

### Base Plan + Metered Features

The most common pattern. Flat monthly fee for the plan, plus usage-based charges for specific features.

**Examples:** API platforms with storage, SaaS tools with compute limits.

```
Plan: Growth ($79/mo)
  - api_calls: 100,000 included, $0.002/extra
  - storage_gb: 50 included, $1.00/extra
  - webhooks: 5,000 included, $0.005/extra
```

```typescript
const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

// Track different metered features independently
await commet.usage.track({ externalId: "user_123", feature: "api_calls", value: 1 });
await commet.usage.track({ externalId: "user_123", feature: "storage_gb", value: 0.5 });
```

### Seats + Metered Usage

Per-user pricing for the team, plus usage-based charges for consumption features. Seats charge advance, metered charges at period end.

**Examples:** Project management with file storage, team collaboration with API access.

```
Plan: Team ($12/seat/mo)
  - team_members: seats, $12/seat
  - file_storage_gb: 5 included, $2.00/extra
  - api_calls: 10,000 included, $0.01/extra
```

```typescript
// Manage seats when team changes
await commet.seats.add({ externalId: "org_456", seatType: "editor", count: 1 });

// Track metered features independently
await commet.usage.track({ externalId: "org_456", feature: "file_storage_gb", value: 1 });
await commet.usage.track({ externalId: "org_456", feature: "api_calls", value: 1 });
```

### Boolean Feature Gates + Metered

Use boolean features to differentiate plans (Pro gets SSO, Enterprise gets audit logs) while metering the actual usage-based features.

**Examples:** Any tiered product with "Pro features" and usage limits.

```
Plan: Free ($0/mo)
  - api_calls: 1,000 included, overage disabled
  - custom_branding: off
  - sso: off
  - priority_support: off

Plan: Pro ($49/mo)
  - api_calls: 50,000 included, $0.002/extra
  - custom_branding: on
  - sso: off
  - priority_support: on

Plan: Enterprise ($299/mo)
  - api_calls: 500,000 included, $0.001/extra
  - custom_branding: on
  - sso: on
  - priority_support: on
```

```typescript
// Check boolean feature access
const { data } = await commet.features.check({
  code: "sso",
  externalId: "user_123",
});

if (!data.allowed) {
  // Show upgrade prompt
}

// Track metered feature independently
await commet.usage.track({ externalId: "user_123", feature: "api_calls", value: 1 });
```

### Balance (AI) + Boolean Gates

AI-powered product where the core value is AI usage (balance model) but plan tiers are differentiated by feature access.

**Examples:** AI writing assistant with advanced features on higher plans.

```
Plan: Starter ($19/mo)
  Balance: $5.00/month
  - ai_generation: balance model, 15% margin
  - advanced_models: off  (only basic models)
  - team_sharing: off
  - api_access: off

Plan: Pro ($49/mo)
  Balance: $25.00/month
  - ai_generation: balance model, 10% margin
  - advanced_models: on
  - team_sharing: on
  - api_access: off

Plan: Enterprise ($199/mo)
  Balance: $100.00/month
  - ai_generation: balance model, 0% margin (pass-through)
  - advanced_models: on
  - team_sharing: on
  - api_access: on
```

```typescript
import { tracked } from "@commet/ai-sdk";
import { anthropic } from "@ai-sdk/anthropic";

// Check if customer can use advanced models
const { data: modelAccess } = await commet.features.check({
  code: "advanced_models",
  externalId: "user_123",
});

const modelId = modelAccess.allowed ? "claude-sonnet-4-20250514" : "claude-haiku-3";

const result = await generateText({
  model: tracked(anthropic(modelId), {
    commet,
    feature: "ai_generation",
    customerId: "user_123",
  }),
  prompt: userPrompt,
});
```

### Addons on Top of Base Plans

Addons are purchasable feature extensions with their own pricing, activated mid-cycle with proration.

**Examples:** Priority support addon, extra storage addon, compliance package.

```
Plan: Pro ($49/mo)
  - api_calls: 50,000 included
  - storage_gb: 10 included

Addon: Extra Storage ($9/mo)
  - storage_gb: +50 included

Addon: Priority Support ($29/mo)
  - priority_support: enabled (boolean)

Addon: API Boost ($19/mo)
  - api_calls: +100,000 included
```

When a customer activates an addon mid-cycle, the addon cost is prorated for the remaining period. At renewal, the full addon price is charged.

## Consumption Model Constraint

Each plan uses ONE consumption model for all its metered features. You cannot mix metered and credits models within the same plan.

| Combination | Allowed |
|------------|---------|
| Metered features + seats + booleans | Yes |
| Credits features + seats + booleans | Yes |
| Balance features + seats + booleans | Yes |
| Metered features + credits features | No (pick one) |
| Balance features + metered features | No (pick one) |

If you need features from different consumption models, use addons (addon consumption model must match the plan's model or be boolean).

## Choosing Your Hybrid

| Your Product | Recommended Hybrid |
|-------------|-------------------|
| API platform with teams | Seats + metered |
| AI SaaS with tiered features | Balance + boolean gates |
| Creative tool with team plans | Seats + credits |
| Developer tool with usage limits | Base plan + metered + boolean gates |
| Enterprise product with add-ons | Base plan + boolean gates + addons |
| Collaboration tool with storage | Seats + metered + boolean gates |
