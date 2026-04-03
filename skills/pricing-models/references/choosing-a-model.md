# Choosing a Pricing Model

## The Three Questions

Before picking a model, answer these:

### 1. How predictable is your customer's usage?

| Usage Pattern | Best Model |
|---------------|-----------|
| Highly variable, spiky | Metered |
| Roughly predictable with occasional spikes | Balance |
| Discrete, countable actions | Credits |
| Scales linearly with team size | Seats |

### 2. Do your customers need cost control?

| Need | Best Model |
|------|-----------|
| "I want to pay only for what I use" | Metered |
| "I need a hard cap so I never overspend" | Credits (hard stop on exhaustion) |
| "I want to prepay and watch my balance" | Balance |
| "I want a predictable per-user cost" | Seats |

### 3. What is your value metric?

The value metric is what your customer perceives as the unit of value. This is the single most important pricing decision you will make.

| Value Metric | Model | Example |
|-------------|-------|---------|
| API requests | Metered | Stripe, Twilio |
| Compute time | Metered | AWS Lambda |
| AI generations | Credits or Balance | Midjourney (credits), OpenAI (balance) |
| Team members | Seats | Slack, Notion |
| Data processed | Metered | Snowflake |
| Feature access | Boolean | "Pro features" tier gating |

## Model Comparison

| Dimension | Metered | Credits | Balance | Seats | Boolean |
|-----------|---------|---------|---------|-------|---------|
| **Billing timing** | Period end | Upfront | Real-time | Period start | Plan base |
| **Blocks on limit** | Never | Yes | Configurable | N/A | N/A |
| **Revenue predictability** | Low | Medium | Medium | High | High |
| **Customer cost clarity** | Low | High | Medium | High | High |
| **Implementation complexity** | Low | Medium | Medium | Low | Trivial |
| **Sub-cent pricing** | Yes | No | Yes | No | N/A |
| **Multi-feature** | Per feature | Per feature | Shared pool | Per seat type | Per flag |
| **Overage handling** | Auto-charged | Buy more packs | Top-up or overage | Prorated add | N/A |

## Real-World Examples

### API Platforms -- Metered

Twilio, Stripe, SendGrid. Customers make API calls, get billed at the end of the month for what they used above included amounts.

**Why metered works:** Usage is highly variable. A startup might send 100 emails one month and 10,000 the next. Charging upfront would either overprice small users or underprice large ones.

**Pattern:** Generous free tier (included amount) + per-unit overage.

```
Plan: Starter ($29/mo)
  - API Calls: 10,000 included, $0.002/extra
  - Webhooks: 1,000 included, $0.005/extra
```

### AI Applications -- Balance

OpenAI, Anthropic API, AI-powered SaaS tools. Customers get a dollar balance that depletes as they use AI features, with costs varying by model and token count.

**Why balance works:** Different AI models cost different amounts. A single "credit" system would either over-abstract the cost or require constant rebalancing. A dollar balance maps naturally to per-token pricing.

**Pattern:** Monthly included balance + optional top-ups. Block or allow overage on exhaustion.

```
Plan: Pro ($49/mo)
  - $10.00 AI balance included
  - blockOnExhaustion: false (overage charged at period end)
```

### Creative Tools -- Credits

Midjourney, Canva, export-heavy tools. Customers buy blocks of credits. Each action costs a defined number of credits. Hard stop when exhausted.

**Why credits work:** Each generation or export is a discrete, visible action. Customers want to know "I have 50 generations left" not "I have $3.42 of balance." Credits create a tangible unit customers can count.

**Pattern:** Monthly included credits + purchasable credit packs. Credits reset each period. Purchased credits never expire.

```
Plan: Creator ($19/mo)
  - 100 credits/month included
  - Image generation: 2 credits each
  - Video export: 10 credits each
Credit Pack: 50 credits for $9.99
```

### Team Collaboration -- Seats

Slack, Notion, Linear, Figma. Each user costs a fixed amount per month. The more people on the team, the more you pay.

**Why seats work:** Value scales linearly with team size. A 50-person company gets 10x the value of a 5-person team. Per-seat pricing captures this directly.

**Pattern:** Included seats in plan base + per-seat overage. Different seat types for different roles.

```
Plan: Team ($99/mo)
  - 5 editors included ($25/extra)
  - Unlimited viewers (free)
```

### Feature-Gated Products -- Boolean

Plan differentiation through feature access. "Pro gets custom branding, Enterprise gets SSO." No metering, no counting -- just on or off.

**Why boolean works:** Some features have near-zero marginal cost but high perceived value. SSO, custom domains, white-labeling. You want them to drive upgrade decisions, not be metered.

**Pattern:** Boolean features that are off on lower plans and on for higher plans.

```
Plan: Free       → custom_branding: off, sso: off, api_access: off
Plan: Pro        → custom_branding: on,  sso: off, api_access: on
Plan: Enterprise → custom_branding: on,  sso: on,  api_access: on
```

## Decision Flowchart

```
Does value scale with team size?
├─ YES → Seats
└─ NO
   │
   Is each action discrete and countable?
   ├─ YES
   │   │
   │   Do customers need hard spending limits?
   │   ├─ YES → Credits
   │   └─ NO → Metered
   └─ NO
       │
       Do costs vary by action type (e.g., different AI models)?
       ├─ YES → Balance
       └─ NO
           │
           Is the feature just on/off access?
           ├─ YES → Boolean
           └─ NO → Metered (default for measurable consumption)
```

## Common Mistakes

### 1. Choosing credits when you mean balance

Credits work when every action costs the same number of credits. If you find yourself saying "this action costs 2 credits but that one costs 7 based on the AI model used," you need balance (dollar-denominated), not credits.

### 2. Metering things that should be boolean

If the marginal cost of a feature is near zero and you would never actually charge per-use, make it boolean. Metering SSO logins or dashboard views adds complexity without value.

### 3. Ignoring hybrid models

Most real products combine models. A project management tool with per-seat pricing (seats) and a file storage limit (metered) and an AI assistant (balance) is perfectly normal. See [hybrid-models.md](hybrid-models.md).

### 4. Picking the model that is easiest to implement

Implementation difficulty should not drive pricing strategy. The right value metric matters more. A metered model that captures value will outperform a flat-rate model that is simpler to build.
