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

> **BXGY ignores `forceApply` on itself** — multiple eligible BXGY rules all
> fire (additive stacking). Use `exclusiveWithBxgy` on a non-BXGY rule to opt
> out of stacking against BXGY freebies.

### List Rules

```graphql
query ListDiscounts($shopId: String!) {
  listDiscountRules(shopId: $shopId) {
    id name priority isActive startDate endDate usageCount
    maxUsage maxUsagePerCustomer
    targetUsage targetMembershipId targetCouponId
    discountOn
    forceApply exclusionGroupId exclusiveWithBxgy
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
  ) {
    id name forceApply exclusionGroupId
  }
}
```

### Update Discount Rule

Same fields as Create plus `$discountId: String!`:

```graphql
mutation UpdateDiscount($discountId: String!, $shopId: String!, ...) {
  updateDiscountRule(discountId: $discountId, shopId: $shopId, ...) {
    id name forceApply exclusionGroupId
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

---

## Discount Cache (read-only)

The storefront uses a cache structure to prefilter applicable rules client-side
before checkout. Read-only query:

```graphql
query DiscountCache($shopId: String!) {
  getDiscountCache(shopId: $shopId) {
    productMap         # product_id -> [discount_ids]
    discountMap        # discount_id -> GeneralDiscountRule
    globalDiscounts    # cart/shipping discounts that apply to all carts
    bxgyDiscounts      # BXGY rules
    userMembershipTierId
    userContext { userId shopId hasMembership }
  }
}
```

Use this when an agent needs to predict whether a discount will apply before the
user checks out (e.g. "tell me if my cart qualifies for free shipping"). The
authoritative truth still comes from the order-creation pipeline.

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
