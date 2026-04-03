# Seat-Based Pricing

Per-user pricing where each team member costs a fixed amount per billing period. Charged advance at period start with prorated true-up for mid-period additions.

## How It Works

1. Plan defines **included seats** (free seats bundled in base price) and a **per-seat overage price**
2. At period start, customer is charged for all current seats beyond the included amount
3. If seats are added mid-period, the additional cost is **prorated** for the remaining days
4. Seat reductions are not refunded -- they take effect at the next period

```
Plan: Team ($99/mo)
  - 5 editors included
  - $25/extra editor

Day 1:  7 editors → charge 2 extra x $25 = $50 advance
Day 15: +3 editors → 3 x $25 x (15/30) = $37.50 prorated true-up
Day 20: -1 editor → no refund, reduction takes effect next period
```

## When to Choose Seats

**Good fit:**
- Value scales linearly with team size
- Each user represents a distinct, identifiable license
- Customers think in terms of "people on my team"
- You have different permission levels that warrant different prices

**Bad fit:**
- Usage does not correlate with user count (use metered or credits)
- Single-user product (seats add complexity for no reason)
- Users are anonymous or shared (use metered based on actions)

## Key Concepts

### Advance + True-up

Seats use a hybrid billing model:

| Component | When Charged | What It Covers |
|-----------|-------------|----------------|
| Advance | Period start | All current seats above included amount |
| True-up | Mid-period (when seats added) | Prorated cost of new seats for remaining days |

This ensures the customer always pays proportionally, and you never wait until period end to collect seat revenue.

### Seat Types

Define different license tiers with different prices:

```
Seat Types:
  - admin:  $50/month
  - editor: $25/month
  - viewer: $0/month (free)
```

Each seat type is tracked independently. A customer might have 2 admins, 10 editors, and unlimited viewers.

### Proration

Daily proration for mid-period changes:

```
Day 1:  5 seats (already paid at period start)
Day 15: +2 seats added
True-up: 2 x $25 x (15 remaining days / 30 total days) = $25.00
```

### No Refunds for Reductions

Removing seats mid-period does not generate a refund. The reduced count takes effect at the next billing period. This prevents gaming (add 50 seats for one day, remove them, get a refund).

## Implementation

### Add seats

```typescript
import { Commet } from "@commet/node";

const commet = new Commet({ apiKey: process.env.COMMET_API_KEY! });

// Add 3 editor seats
await commet.seats.add({
  externalId: "org_456",
  seatType: "editor",
  count: 3,
});
```

### Remove seats

```typescript
await commet.seats.remove({
  externalId: "org_456",
  seatType: "editor",
  count: 2,
});
```

### Set to exact count

```typescript
// Set editors to exactly 10, regardless of current count
await commet.seats.set({
  externalId: "org_456",
  seatType: "editor",
  count: 10,
});
```

### Set all seat types at once

```typescript
await commet.seats.setAll({
  externalId: "org_456",
  seats: { admin: 2, editor: 10, viewer: 50 },
});
```

### Check current seats

```typescript
const { data } = await commet.seats.getBalance({
  externalId: "org_456",
  seatType: "editor",
});
// data.current = 10
// data.asOf = "2026-03-15T10:00:00Z"

// Get all seat type balances
const { data: all } = await commet.seats.getAllBalances({
  externalId: "org_456",
});
// { admin: { current: 2, asOf: "..." }, editor: { current: 10, asOf: "..." } }
```

### Scoped API

```typescript
const customer = commet.customer("org_456");

await customer.seats.add("editor", 3);
await customer.seats.remove("editor", 1);
await customer.seats.set("editor", 10);
const { data } = await customer.seats.getBalance("editor");
```

### Check if more seats are allowed

```typescript
const { data } = await commet.features.canUse({
  code: "team_members",
  externalId: "org_456",
});

if (!data.allowed) {
  // Plan does not allow more seats (e.g., at plan limit and overageEnabled is false)
}

if (data.willBeCharged) {
  // Adding a seat will incur a prorated charge
}
```

## Plan Configuration Example

```
Plan: Starter ($49/mo)
  Feature: team_members
    type: seats
    includedAmount: 3
    overageEnabled: true
    overageUnitPrice: 150000  (rate scale: $15/seat)
  Seat Types:
    - editor: $15/mo

Plan: Team ($99/mo)
  Feature: team_members
    type: seats
    includedAmount: 10
    overageEnabled: true
    overageUnitPrice: 250000  (rate scale: $25/seat)
  Seat Types:
    - admin: $50/mo
    - editor: $25/mo
    - viewer: $0/mo

Plan: Enterprise ($499/mo)
  Feature: team_members
    type: seats
    unlimited: true
    overageUnitPrice: 200000  (rate scale: $20/seat)
  Seat Types:
    - admin: $40/mo
    - editor: $20/mo
    - viewer: $0/mo
```

## Billing Behavior

| Event | What Happens |
|-------|-------------|
| Period start | Advance charge for all current seats above included |
| Seats added mid-period | Prorated true-up charged immediately |
| Seats removed mid-period | No refund. Reduction applies at next period. |
| Upgrade | New included seats and overage price apply immediately |
| Downgrade | At renewal, new values apply |
| Price increase (founder) | At renewal (does not hurt customer mid-period) |
| Included seats increase (founder) | Immediate (benefits customer) |

## Common Patterns

### Sync seats with your user database

When a team member is added or removed from your application, update the seat count:

```typescript
async function onTeamMemberAdded(organizationId: string) {
  await commet.seats.add({
    externalId: organizationId,
    seatType: "editor",
    count: 1,
  });
}

async function onTeamMemberRemoved(organizationId: string) {
  await commet.seats.remove({
    externalId: organizationId,
    seatType: "editor",
    count: 1,
  });
}
```

### Gate invitations on seat availability

```typescript
async function canInviteMember(organizationId: string): Promise<boolean> {
  const { data } = await commet.features.canUse({
    code: "team_members",
    externalId: organizationId,
  });
  return data.allowed;
}
```
