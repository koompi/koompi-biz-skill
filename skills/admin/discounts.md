# Coupons & Discount Rules

## Coupons

### List Coupons
```graphql
query ListCoupons($shopId: String!) {
  listCoupons(shopId: $shopId) {
    id code description isActive shopId createdAt
  }
}
```

### Get Coupon by Code
```graphql
query CouponByCode($couponCode: String!, $shopId: String!) {
  getCouponByCode(couponCode: $couponCode, shopId: $shopId) {
    id code description isActive
  }
}
```

### Create Coupon
```graphql
mutation CreateCoupon($shopId: String!, $code: String!, $description: String) {
  createCoupon(shopId: $shopId, code: $code, description: $description) {
    id code isActive
  }
}
```

### Update Coupon
```graphql
mutation UpdateCoupon($couponId: String!, $code: String!, $description: String, $isActive: Boolean!) {
  updateCoupon(couponId: $couponId, code: $code, description: $description, isActive: $isActive) {
    id code isActive
  }
}
```

### Delete Coupon
> ⚠️ **CONFIRMATION REQUIRED** — Show coupon code and confirm before deleting.
```graphql
mutation DeleteCoupon($couponId: String!) {
  deleteCoupon(couponId: $couponId)
}
```

---

## Discount Rules

Discount rules are evaluated by the platform's discount engine for every cart.
A rule has a **scope** (`discountOn`) — `LINE_ITEM`, `TOTAL_CART`, `SHIPPING`, or
`BUY_X_GET_Y` — and a **target audience** (`targetUsage`) — everyone, a
specific membership tier, or a specific coupon code.

### Engine semantics (must read before authoring rules)

The discount engine picks the **best-saving** rule per scope by default, but
merchants can override that with two knobs:

- **`forceApply`** — when `true`, this rule *always wins* in its scope (even if
  another eligible rule would give a larger saving). Multiple `forceApply` rules
  are tie-broken by `priority` (lower wins), then `_id`. NOT supported on
  `BUY_X_GET_Y` (BXGY rules naturally stack additively — there is no "winner"
  to force).
- **`exclusionGroupId`** — a free-form string. At most one rule per
  `(shopId, exclusionGroupId)` fires per cart. Use this to make rules mutually
  exclusive (e.g. a flash-sale group where any rule in the group cancels the
  others). Resolved by `priority` ascending, ties broken by `_id`.
- **`exclusiveWithBxgy`** — when `true`, this non-BXGY rule is suppressed for
  any order where a BXGY rule fires. Use it to prevent stacking sale prices on
  top of a BXGY freebie.
- **`maxUsagePerCustomer`** — caps redemptions per individual customer (phone
  number). The engine fails closed on the RPC fetch — if usage can't be
  verified, capped rules are denied rather than over-applied.
- **`priority`** — integer, lower number = higher priority. Used as the
  tiebreaker for forceApply collisions and exclusion-group resolution.
- **`recurringSchedule`** — recurring / calendar-based activation rules
  (specific weekdays, day-of-month ranges, time-of-day windows, specific
  dates). ANDs with the absolute `startDate`/`endDate` window. See "Recurring
  Schedules" below.
- **`firstOrderOnly`** — `Some(true)` → only fires on the customer's first
  order. Evaluated against the customer's actual order history at checkout
  (works for guests via normalized phone). Defaults to `null` (no gate).
- **`winBackDays`** — `Some(N)` → only fires if the customer's last
  non-cancelled order was at least N days ago. Never-ordered customers do
  NOT qualify (use `firstOrderOnly` for those). Defaults to `null`.

> **BXGY ignores `forceApply` on itself** — multiple eligible BXGY rules all
> fire (additive stacking). Use `exclusiveWithBxgy` on a non-BXGY rule to opt
> out of stacking against BXGY freebies.

> **`firstOrderOnly` and `winBackDays` are mutually exclusive in practice** —
> pick one customer-segment condition per rule (the admin UI surfaces them
> as a single "Everyone / First order only / Win-back" selector).

> **Browse is conservative.** On the storefront product listing, anonymous /
> unknown browsers see `isFirstOrder = false` / `daysSinceLastOrder = null`
> so first-order and win-back discounts stay HIDDEN. They appear at
> checkout once the guest types their phone (debounced). A returning guest's
> cart price can only go DOWN at checkout, never up.

### List Rules

```graphql
query ListDiscounts($shopId: String!) {
  listDiscountRules(shopId: $shopId) {
    id name priority isActive startDate endDate usageCount
    maxUsage maxUsagePerCustomer
    targetUsage targetMembershipId targetCouponId
    discountOn
    forceApply exclusionGroupId exclusiveWithBxgy
    firstOrderOnly winBackDays
    recurringSchedule {
      daysOfWeek
      dayOfMonthRanges { from to }
      timeWindow { from { hour minute } to { hour minute } }
      specificDates
      tzOffsetMinutes
    }
    lineItems    { productId discountType discountAmount moq }
    cartDiscount { discountType discountAmount }
    shippingDiscount { discountType discountAmount minCartTotal }
    bxgyDiscount {
      buyProductId buyQuantity
      getProductId getQuantity
      getDiscountType getDiscountValue
    }
  }
}
```

### Get Single Rule

```graphql
query GetDiscountRule($discountId: String!) {
  getDiscountRule(discountId: $discountId) {
    id name priority isActive startDate endDate usageCount
    maxUsage maxUsagePerCustomer
    targetUsage targetMembershipId targetCouponId
    discountOn
    forceApply exclusionGroupId exclusiveWithBxgy
    lineItems    { productId discountType discountAmount moq }
    cartDiscount { discountType discountAmount }
    shippingDiscount { discountType discountAmount minCartTotal }
    bxgyDiscount {
      buyProductId buyQuantity getProductId getQuantity
      getDiscountType getDiscountValue
    }
  }
}
```

### Create Discount Rule

```graphql
mutation CreateDiscount(
  $shopId: String!
  $name: String!
  $priority: Int!
  $startDate: ScalarDateTimeUtc
  $endDate: ScalarDateTimeUtc
  $isActive: Boolean!
  $usageCount: Int!                    # always pass 0 on create
  $maxUsage: Int                       # null = unlimited
  $maxUsagePerCustomer: Int            # null = unlimited per customer
  $targetUsage: UsageTarget!
  $targetMembershipId: String
  $targetCouponId: String
  $discountOn: DiscountScope!
  $lineItems: [LineItemDiscountInput!]
  $shippingDiscount: ShippingDiscountInput
  $cartDiscount: CartDiscountInput
  $bxgyDiscount: BxgyDiscountInput
  $exclusiveWithBxgy: Boolean          # default false
  $forceApply: Boolean                 # default false — merchant override
  $exclusionGroupId: String            # null = no exclusion grouping
  $recurringSchedule: RecurringScheduleInput  # null = always on
  $firstOrderOnly: Boolean             # null = no first-order gate
  $winBackDays: Int                    # null = no win-back gate
) {
  createDiscountRule(
    shopId: $shopId name: $name priority: $priority
    startDate: $startDate endDate: $endDate isActive: $isActive
    usageCount: $usageCount maxUsage: $maxUsage
    maxUsagePerCustomer: $maxUsagePerCustomer
    targetUsage: $targetUsage
    targetMembershipId: $targetMembershipId
    targetCouponId: $targetCouponId
    discountOn: $discountOn
    lineItems: $lineItems
    shippingDiscount: $shippingDiscount
    cartDiscount: $cartDiscount
    bxgyDiscount: $bxgyDiscount
    exclusiveWithBxgy: $exclusiveWithBxgy
    forceApply: $forceApply
    exclusionGroupId: $exclusionGroupId
    recurringSchedule: $recurringSchedule
    firstOrderOnly: $firstOrderOnly
    winBackDays: $winBackDays
  ) {
    id name forceApply exclusionGroupId
    firstOrderOnly winBackDays
  }
}
```

### Update Discount Rule

Same fields as Create plus `$discountId: String!`:

```graphql
mutation UpdateDiscount($discountId: String!, $shopId: String!, ...) {
  updateDiscountRule(discountId: $discountId, shopId: $shopId, ...) {
    id name forceApply exclusionGroupId firstOrderOnly winBackDays
  }
}
```

### Delete Discount Rule

> ⚠️ **CONFIRMATION REQUIRED** — Show discount name and scope before deleting.

```graphql
mutation DeleteDiscount($shopId: String!, $discountId: String!) {
  deleteDiscountRule(shopId: $shopId, discountId: $discountId)
}
```

---

## Input Shapes

### `LineItemDiscountInput` — per-product discount
```json
{ "productId": "...", "discountType": "PERCENTAGE", "discountAmount": 10, "moq": 1 }
```
`discountType`: `PERCENTAGE` | `FIXED` | `LOCKED_PRICE` (overrides the price
entirely). `moq` = minimum order quantity required to trigger this line.

### `CartDiscountInput` — total-cart discount
```json
{ "discountType": "FIXED", "discountAmount": 5.0 }
```
`discountType`: `PERCENTAGE` | `FIXED` | `LOCKED_PRICE`.

### `ShippingDiscountInput` — shipping discount
```json
{ "discountType": "FREE", "discountAmount": 0, "minCartTotal": 30.0 }
```
`discountType`: `FIXED` | `PERCENTAGE` | `FREE`. `minCartTotal` optional —
discount only fires if subtotal ≥ threshold.

### `BxgyDiscountInput` — Buy X Get Y
```json
{
  "buyProductId": "...",
  "buyQuantity": 2,
  "getProductId": "...",
  "getQuantity": 1,
  "getDiscountType": "FREE",
  "getDiscountValue": 0
}
```
`getDiscountType`: `FREE` | `PERCENTAGE` | `FIXED_AMOUNT_OFF`. `discountOn`
must be `BUY_X_GET_Y` for these rules.

### `RecurringScheduleInput` — calendar / recurring activation

Optional. When `null` the discount is "always on" (within `startDate`/`endDate`).

```json
{
  "daysOfWeek": ["SAT", "SUN"],
  "dayOfMonthRanges": [{ "from": 25, "to": 31 }],
  "specificDates": ["2026-12-25", "2027-01-01"],
  "timeWindow": {
    "from": { "hour": 18, "minute": 0 },
    "to":   { "hour": 21, "minute": 0 }
  },
  "tzOffsetMinutes": 420
}
```

**Semantics (must read):**

- `daysOfWeek`, `dayOfMonthRanges`, `specificDates` are **OR'd** together —
  a day matches if it satisfies ANY set condition. If all three are unset,
  every day qualifies.
- `timeWindow` **ANDs** on top, gating the active hours within a matching day.
- The whole schedule **ANDs** with the absolute `startDate`/`endDate` window
  and `isActive`.
- `tzOffsetMinutes`: minutes east of UTC. **Defaults to 420** (Cambodia,
  UTC+7) when unset. The schedule is evaluated in this local time.
- End-of-month ranges clamp to the real last day: `{from: 25, to: 31}` matches
  Feb 25–28 / Feb 25–29 in a leap year.
- `timeWindow` wraps past midnight: `{ from: 22:00, to: 02:00 }` is active
  22:00–01:59 of the next day. The `to` end is **EXCLUSIVE** —
  14:55–14:57 is active 14:55:00–14:56:59 and turns off at 14:57:00.
- A zero-width window (`from == to`) is never active.

**`Day` enum:** `MON | TUE | WED | THU | FRI | SAT | SUN`.
**`TimeInput`:** `{ hour: Int!, minute: Int! }`.

### Common preset schedules

```json
// "Every Sunday"
{ "daysOfWeek": ["SUN"] }

// "Every weekend"
{ "daysOfWeek": ["SAT", "SUN"] }

// "Every workday"
{ "daysOfWeek": ["MON", "TUE", "WED", "THU", "FRI"] }

// "Early month" (1st–7th)
{ "dayOfMonthRanges": [{ "from": 1, "to": 7 }] }

// "End of month" (25th–31st, auto-clamps short months)
{ "dayOfMonthRanges": [{ "from": 25, "to": 31 }] }

// "Happy hour" daily 18:00–21:00 ICT
{ "timeWindow": { "from": {"hour":18,"minute":0}, "to": {"hour":21,"minute":0} } }

// "Christmas day only"
{ "specificDates": ["2026-12-25"] }
```

---

## Common Discount Recipes

### 10% off a specific product
```json
{
  "name": "Summer Sale - T-Shirt",
  "priority": 10,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "LINE_ITEM",
  "lineItems": [{ "productId": "<id>", "discountType": "PERCENTAGE", "discountAmount": 10, "moq": 1 }]
}
```

### $5 off entire cart, max 100 total redemptions
```json
{
  "name": "Welcome Discount",
  "priority": 10,
  "isActive": true,
  "usageCount": 0,
  "maxUsage": 100,
  "maxUsagePerCustomer": 1,
  "targetUsage": "EVERY_ONE",
  "discountOn": "TOTAL_CART",
  "cartDiscount": { "discountType": "FIXED", "discountAmount": 5.0 }
}
```

### Free shipping on orders over $30
```json
{
  "name": "Free Shipping Over $30",
  "priority": 10,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "SHIPPING",
  "shippingDiscount": { "discountType": "FREE", "discountAmount": 0, "minCartTotal": 30.0 }
}
```

### Coupon-gated 15% off cart
First create the coupon, then target it:
```json
{
  "name": "VIP15 Coupon",
  "priority": 5,
  "isActive": true,
  "usageCount": 0,
  "maxUsage": 50,
  "targetUsage": "COUPON",
  "targetCouponId": "<coupon_id>",
  "discountOn": "TOTAL_CART",
  "cartDiscount": { "discountType": "PERCENTAGE", "discountAmount": 15 }
}
```

### Membership-only 20% off
```json
{
  "name": "Gold tier 20% off",
  "priority": 5,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "MEMBERSHIP",
  "targetMembershipId": "<tier_id>",
  "discountOn": "TOTAL_CART",
  "cartDiscount": { "discountType": "PERCENTAGE", "discountAmount": 20 }
}
```

### Buy 2 get 1 free
```json
{
  "name": "B2G1 Tees",
  "priority": 10,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "BUY_X_GET_Y",
  "bxgyDiscount": {
    "buyProductId": "<id>",
    "buyQuantity": 2,
    "getProductId": "<id>",
    "getQuantity": 1,
    "getDiscountType": "FREE",
    "getDiscountValue": 0
  }
}
```

### Force-apply a smaller cart discount over a larger one (merchant override)
The merchant wants the branded "Anniversary" sale (10%) to always show on the
receipt even if a 15% rule would technically save more. Set `forceApply: true`:
```json
{
  "name": "Anniversary 10% (forced)",
  "priority": 1,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "TOTAL_CART",
  "cartDiscount": { "discountType": "PERCENTAGE", "discountAmount": 10 },
  "forceApply": true
}
```

### Make two rules mutually exclusive via exclusion group
Two flash-sale rules — one per category — that should never both fire on the
same cart. Give them the same `exclusionGroupId`; the engine keeps only the
highest-priority (lowest number) one that's eligible.
```json
{
  "name": "Flash sale - Bags",
  "priority": 5,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "LINE_ITEM",
  "lineItems": [{ "productId": "<bag_id>", "discountType": "PERCENTAGE", "discountAmount": 25, "moq": 1 }],
  "exclusionGroupId": "flash_sale_2026_q2"
}
```
The "Flash sale - Shoes" rule uses the same `exclusionGroupId`.

### Prevent stacking a percentage sale on top of a BXGY freebie
The merchant runs a B2G1 promo and doesn't want a separate 10%-off-cart rule to
*also* apply on the same order. Set `exclusiveWithBxgy: true` on the non-BXGY
rule:
```json
{
  "name": "10% off cart (paused during BXGY)",
  "priority": 10,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "TOTAL_CART",
  "cartDiscount": { "discountType": "PERCENTAGE", "discountAmount": 10 },
  "exclusiveWithBxgy": true
}
```

### Happy-hour 20% off cart (weekends only, 18:00–21:00 ICT)
Use a recurring schedule on top of a normal rule:
```json
{
  "name": "Happy hour",
  "priority": 5,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "TOTAL_CART",
  "cartDiscount": { "discountType": "PERCENTAGE", "discountAmount": 20 },
  "recurringSchedule": {
    "daysOfWeek": ["SAT", "SUN"],
    "timeWindow": {
      "from": { "hour": 18, "minute": 0 },
      "to":   { "hour": 21, "minute": 0 }
    }
  }
}
```

### First-order welcome — 15% off the customer's first order only
Works for guests too (evaluated against normalized phone at checkout):
```json
{
  "name": "Welcome — first order",
  "priority": 5,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "TOTAL_CART",
  "cartDiscount": { "discountType": "PERCENTAGE", "discountAmount": 15 },
  "firstOrderOnly": true
}
```

### Win-back lapsed customers — free shipping if last order ≥ 60 days ago
```json
{
  "name": "We miss you — free shipping",
  "priority": 8,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "SHIPPING",
  "shippingDiscount": { "discountType": "FREE", "discountAmount": 0 },
  "winBackDays": 60
}
```

### End-of-month flash on Christmas day at midnight
Stack `dayOfMonthRanges` for late-month + `specificDates` for an exact day:
```json
{
  "name": "Christmas drop",
  "priority": 3,
  "isActive": true,
  "usageCount": 0,
  "targetUsage": "EVERY_ONE",
  "discountOn": "TOTAL_CART",
  "cartDiscount": { "discountType": "PERCENTAGE", "discountAmount": 25 },
  "recurringSchedule": {
    "specificDates": ["2026-12-25"]
  }
}
```

---

## Discount Cache (read-only)

The storefront uses a cache structure to prefilter applicable rules client-side
before checkout. Read-only query:

```graphql
query DiscountCache($shopId: String!, $guestPhone: String) {
  getDiscountCache(shopId: $shopId, guestPhone: $guestPhone) {
    productMap         # product_id -> [discount_ids]
    discountMap        # discount_id -> GeneralDiscountRule (includes recurringSchedule, firstOrderOnly, winBackDays)
    globalDiscounts    # cart/shipping discounts that apply to all carts
    bxgyDiscounts      # BXGY rules
    userMembershipTierId
    userContext {
      userId shopId hasMembership
      isFirstOrder daysSinceLastOrder
    }
  }
}
```

`guestPhone` (optional, anonymous callers): when an authenticated token isn't
available, pass the phone the guest typed at checkout. The backend normalizes
it identically to the guest-order pipeline so first-order / win-back discounts
preview correctly. Omit when authenticated — the token id wins.

Use this when an agent needs to predict whether a discount will apply before the
user checks out (e.g. "tell me if my cart qualifies for free shipping"). The
authoritative truth still comes from the order-creation pipeline. Recurring
schedules are NOT pre-filtered in the cache — the cache carries the schedule
and the client checks it at render time (so a 6pm happy hour switches on
without a cache refetch).

---

## Order Receipt: which discounts fired?

The order response exposes the applied discounts plus `appliedViaForce` so
merchants can audit overrides:

```graphql
query OrderDiscounts($orderId: String!) {
  getOrder(orderId: $orderId) {
    id
    appliedDiscounts {
      discountId discountName discountType
      amount percentage couponCode
      originalAmount discountedAmount
      appliedViaForce
      suppressedCompetingRuleIds   # rules the engine skipped because of this one
    }
  }
}
```

---

## Agent Notes

- **Coupons are containers.** A coupon itself is just a code. The actual logic
  lives in a Discount Rule with `targetUsage: COUPON` and `targetCouponId`
  pointing at the coupon.
- **`priority`**: Lower number = higher priority. Used both as the human-facing
  "ranking" *and* as the tie-breaker for `forceApply` collisions and
  exclusion-group resolution.
- **`moq`**: Minimum order quantity per line item. The line discount only
  applies if the customer orders ≥ `moq` of that product.
- **Dates**: `startDate` and `endDate` are optional. Omit both and the rule is
  always active (subject to `isActive`).
- **`usageCount`**: Always pass `0` on create. The engine increments it on each
  successful redemption.
- **`maxUsagePerCustomer`**: Caps redemptions per individual customer (keyed by
  normalized phone number for guests). Engine fails closed if the count RPC
  errors — capped rules are denied, not over-applied.
- **`forceApply` ⚠️**: Use with care — it overrides best-saving-wins, so a
  smaller discount can win over a larger one. Reserve for cases where the
  merchant explicitly wants this rule to "always show" (anniversary sales,
  branded promo codes). Not supported on `BUY_X_GET_Y`.
- **`exclusionGroupId`**: Pick a stable string per campaign (e.g.
  `"flash_sale_2026_q2"`). At most one rule per group fires per cart.
- **`exclusiveWithBxgy`**: Toggle on non-BXGY rules to opt out of stacking
  against any BXGY freebie firing on the same order.
- **`recurringSchedule`**: Always evaluated in **Cambodia time (ICT, UTC+7)**
  unless you set `tzOffsetMinutes`. Day-level conditions OR together
  (`daysOfWeek` OR `dayOfMonthRanges` OR `specificDates`); `timeWindow` ANDs
  on top. The `to` end of `timeWindow` is exclusive.
- **`firstOrderOnly` vs `winBackDays`**: pick at most one per rule. They're
  customer-segment gates, not stacking rules — the admin UI surfaces them
  as a single "Everyone / First order only / Win-back N days" selector.
- **Schedules + customer gates compose with everything else.** A rule with
  `forceApply: true` + `recurringSchedule` only fires AND wins during its
  window. A `firstOrderOnly` rule with `exclusionGroupId` still respects the
  group. Don't worry about ordering — the engine applies them all in one pass.
