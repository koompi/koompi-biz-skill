# Riverbase AI Skill

> Complete GraphQL reference for AI agents managing shops on the Riverbase platform via Telegram.

## Endpoint & Auth

| Key | Value |
|---|---|
| **GraphQL** | `https://lite-api.riverbase.org/graphql` |
| **Upload** | `https://lite-api.riverbase.org/uploads` (multipart POST, returns URL path) |
| **Auth** | `Authorization: Bearer <USER_TOKEN>` header on all requests |
| **Method** | `POST` with `Content-Type: application/json` body `{"query":"...","variables":{...}}` |

---

## Skill Router

Load **only** the file relevant to the user's request:

### Catalog (Products, Categories, Brands)
| File | Use when user says… |
|---|---|
| [catalog/products.md](skills/catalog/products.md) | "add product", "update price", "archive item", "search products" |
| [catalog/categories.md](skills/catalog/categories.md) | "create category", "list categories", "add subcategory" |
| [catalog/brands.md](skills/catalog/brands.md) | "add brand", "list brands" |

### Orders & Payments
| File | Use when user says… |
|---|---|
| [orders/lifecycle.md](skills/orders/lifecycle.md) | "pending orders", "confirm order", "complete order", "cancel" |
| [orders/queries.md](skills/orders/queries.md) | "show orders", "order stats", "dashboard", "search order" |
| [orders/payments.md](skills/orders/payments.md) | "mark as paid", "process payment", "payment status" |

### Inventory & Stock
| File | Use when user says… |
|---|---|
| [inventory/locations.md](skills/inventory/locations.md) | "add warehouse", "list locations", "set default location" |
| [inventory/transactions.md](skills/inventory/transactions.md) | "restock", "record damage", "adjust stock", "transfer stock" |
| [inventory/checks.md](skills/inventory/checks.md) | "check stock", "stock overview", "low stock", "availability" |

### Shop Administration
| File | Use when user says… |
|---|---|
| [admin/shop.md](skills/admin/shop.md) | "create shop", "rename shop", "activate", "set location", "customers", "notifications" |
| [admin/modules.md](skills/admin/modules.md) | "enable POS", "turn on stocking", "COD settings", "announcement", "business hours" |
| [admin/team.md](skills/admin/team.md) | "add staff", "create role", "remove member" |
| [admin/shipping.md](skills/admin/shipping.md) | "shipping fee", "delivery zones", "add delivery option" |
| [admin/discounts.md](skills/admin/discounts.md) | "create coupon", "discount rule", "sale on product" |

### Advanced Features
| File | Use when user says… |
|---|---|
| [advanced/membership.md](skills/advanced/membership.md) | "VIP tier", "membership card", "loyalty program", "add member" |
| [advanced/quotations.md](skills/advanced/quotations.md) | "create quote", "send quotation", "revise quote", "contract pricing" |
| [advanced/content.md](skills/advanced/content.md) | "write blog post", "create event", "publish article", "ticket event" |
| [advanced/dns.md](skills/advanced/dns.md) | "custom domain", "connect domain", "DNS settings" |
| [advanced/plugins.md](skills/advanced/plugins.md) | "install plugin", "list plugins", "shade finder", "plugin settings" |

---

## GraphQL Enums Reference

These exact string values are used throughout all mutations and queries.

### Order Enums
```
OrderStatus:   ALL | PENDING | CONFIRMED | COMPLETED | CANCELLED
PaymentStatus: ALL | PAID | UNPAID
DeliverySatus: ALL | PENDING | SHIPPED | DELIVERED    # Note: typo "Satus" is intentional - matches schema
DueType:       ALL | NOW | COD
FulfillmentMethod: DELIVERY | IN_STORE_PICKUP | ON_SITE | ONLINE_POS | DIGITAL
CodMethod:     CASH | ABA_TRANSFER
Platform:      TELEGRAM | OTHERS
OrderType:     USER | POS
```

### Inventory Enums
```
CreditTransactionType: PURCHASE | RETURN | CONSIGNMENT_RETURN | CONSIGNMENT_RECEIVED | MANUFACTURING | UNRESERVED | ADJUSTMENT | STOCK_COUNT | RESTOCK
DebitTransactionType:  SALE | REFUND | CANCELLATION | CONSIGNMENT_SENT | CONSIGNMENT_SOLD | RESERVED | TRANSFER | RAW_MATERIAL | DAMAGE | SHRINKAGE | EXPIRED | DONATION | DISPOSAL | PROMOTION | QUARANTINE | RECALL | SAMPLING | DEMO | ADJUSTMENT | STOCK_COUNT
InventoryAccountType:  RAW_MATERIALS | WORK_IN_PROGRESS | FINISHED_GOODS | MERCHANDISE_INVENTORY | RESERVED | QUARANTINE | DAMAGED | TRANSIT | CONSIGNMENT_IN | CONSIGNMENT_OUT | DEMO
InventoryLocationType: WAREHOUSE | STORE | DISPLAY
```

### Shop Enums
```
OperationalMode: OPEN | CLOSED | BUSY
ShippingModel:   DISTANCE_BASED | ZONE_BASED | FLAT_RATE | THIRD_PARTY_ONLY
CartMode:        DETAILS_ONLY | FULL_CHECKOUT
CardStyle:       MINIMAL | DETAILED
```

### Discount Enums
```
DiscountScope:   LINE_ITEM | TOTAL_CART | SHIPPING
DiscountType:    FIXED | PERCENTAGE | LOCKED_PRICE
UsageTarget:     EVERY_ONE | MEMBERSHIP | COUPON
```

---

## Global Agent Rules

1. **Resolve IDs first.** When a user refers to a product/category/brand by name, always run a search or list query to get the `id` before mutating.
2. **Price = String.** All monetary values (`price`, `amount`, `unitPrice`) are passed as `String` in GraphQL to preserve decimal precision.
3. **Confirm destructive ops.** Before deleting, cancelling, or completing an order, summarize the action and ask user to confirm.
4. **shopId is always required.** Every query/mutation requires `shopId`. Resolve it once at the start of a conversation via `ownedShops` or `joinedShops`.
5. **Pagination.** List queries return `{ data, limit, skip, totalPages, totalDocuments }`. Default to `limit: 20, page: 1`.
6. **Soft delete.** Products are archived (`productSetArchived`), not hard-deleted. Use `productDelete` only when the user explicitly says "permanently delete."

### Bootstrap Query (run first in every session)
```graphql
query Bootstrap {
  ownedShops {
    id
    name
    logo
    active
    productsCount
    modules { shopping stocking quoting codPayment posCodEnabled cartMode }
  }
  joinedShops {
    id
    name
    logo
    active
  }
}
```
