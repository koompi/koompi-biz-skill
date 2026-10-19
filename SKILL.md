# Riverbase AI Skill

> Complete GraphQL reference for AI agents managing shops on the Riverbase platform via Telegram.

## Endpoint & Auth

| Environment | GraphQL | Upload | Upload HQ | Font Upload |
|---|---|---|---|---|
| **Production** | `https://api.riverbase.org/graphql` | `https://api.riverbase.org/uploads/s3` | `https://api.riverbase.org/uploads/s3/hq` | `https://api.riverbase.org/uploads/font` |
| **Staging** | `https://staging-lite-api.riverbase.org/graphql` | `https://staging-lite-api.riverbase.org/uploads/s3` | `https://staging-lite-api.riverbase.org/uploads/s3/hq` | `https://staging-lite-api.riverbase.org/uploads/font` |

All upload endpoints accept `multipart/form-data` POST with field name `file`,
sent with the same `Authorization: <token>` header. The response body is the
plain URL string of the stored file. They sit **beside** `/graphql`, not under
it: build them from the API origin, `${RIVERBASE_API_URL%/graphql}/uploads/...`.
- `/uploads/s3` → images re-encoded to ≤2400px WebP (logos, cards, inline photos).
- `/uploads/s3/hq` → images re-encoded to ≤3840px WebP (hero / banner / promo / cover).
- Both also take video files, stored as-is (no transcoding), max 99 MB — see `admin/sections.md` for per-slot limits.
- `/uploads/font` → font files (`.woff2`, `.woff`, `.ttf`, `.otf`, max 2 MB).

| Key | Value |
|---|---|
| **Auth** | `Authorization: <USER_TOKEN>` header on all requests (no `Bearer` prefix) |
| **API Key Auth** | For server/agent access, set `RIVERBASE_API_KEY` and use as the auth token |
| **Method** | `POST` with `Content-Type: application/json` body `{"query":"...","variables":{...}}` |

### ⚡ Environment Variables — YOU MUST DO THIS FIRST

**Before doing anything else, you MUST execute the commands below to obtain `RIVERBASE_API_URL`, `RIVERBASE_API_KEY`, and `RIVERBASE_SHOP_ID`.**

These are almost never in your runtime environment. **Do not just check `env` and give up.** You have file access — use it.

#### Step 1: Read the .env file directly

Execute this command NOW:

```bash
grep -E 'RIVERBASE_API_URL|RIVERBASE_API_KEY|RIVERBASE_SHOP_ID' /zeroclaw-data/workspace/.env 2>/dev/null
```

If that file doesn't exist, try these alternatives (execute each one until you get results):

```bash
# Try openclaw workspace
grep -E 'RIVERBASE_API_URL|RIVERBASE_API_KEY|RIVERBASE_SHOP_ID' /openclaw-data/workspace/.env 2>/dev/null

# Try generic paths
grep -E 'RIVERBASE_API_URL|RIVERBASE_API_KEY|RIVERBASE_SHOP_ID' /data/workspace/.env /workspace/.env .env 2>/dev/null

# Last resort: find any .env containing RIVERBASE somewhere on disk
find / -maxdepth 4 -name '.env' -exec grep -l 'RIVERBASE' {} \; 2>/dev/null | head -5
# Then read the file it found
```

#### Step 2: Extract and use the values

Once you have output like:
```
RIVERBASE_API_URL=https://staging-lite-api.riverbase.org/graphql
RIVERBASE_API_KEY=<a real key from the .env file>
RIVERBASE_SHOP_ID=<a real shop id from the .env file>
```

**Use these values directly** in your GraphQL requests:
- `RIVERBASE_API_URL` → the GraphQL endpoint URL
- `RIVERBASE_API_KEY` → put in `Authorization` header (no `Bearer` prefix)
- `RIVERBASE_SHOP_ID` → the `shopId` variable in every query/mutation

#### What NOT to do
- ❌ Do NOT tell the user "the env vars aren't loaded" — you have file access, go read them
- ❌ Do NOT give up after checking `env` — the .env file is the primary source
- ❌ Do NOT ask the user to set environment variables manually
- ❌ Do NOT skip this step and try API calls without auth — they will fail

> **Never hardcode real API keys in the skill files.** They must only come from the runtime environment or `.env` file.

---

## Skill Router

Load **only** the file relevant to the user's request:

### Catalog (Products, Categories, Brands)
| File | Use when user says… |
|---|---|
| [catalog/products.md](skills/catalog/products.md) | "add product", "update price", "archive item", "search products", "combo", "bundle", "shade finder", "cosmetics try-on" |
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
| [admin/storefront-design.md](skills/admin/storefront-design.md) | **Start here for broad design work:** "redesign my shop", "customise my storefront", "make my shop look premium / modern / like <brand>", "change the whole look", "design my shop", "set up my shop's appearance". It sequences theme → chrome → product display → homepage → pages and routes to the files below. |
| [admin/shop.md](skills/admin/shop.md) | "create shop", "rename shop", "activate", "set location", "customers", "notifications", "set business category", "footer logo", "show shop name", "branding" |
| [admin/seo.md](skills/admin/seo.md) | "meta title", "meta description", "SEO", "keywords", "favicon", "og image", "change shop name", "update logo" |
| [admin/pages.md](skills/admin/pages.md) | "create page", "custom page", "contact page", "FAQ page", "terms", "privacy policy", "shipping page", "returns policy", "size guide", "landing page", "campaign page", "lookbook page", "static page", "custom HTML page", "upload my HTML", "add page to header/footer menu", "edit page", "delete page", "page SEO" |
| [admin/modules.md](skills/admin/modules.md) | "enable POS", "turn on stocking", "COD settings", "enable blog", "enable ticketing", "business hours", "stock debit", "unpaid order timeout" |
| [admin/team.md](skills/admin/team.md) | "add staff", "create role", "remove member" |
| [admin/shipping.md](skills/admin/shipping.md) | "shipping fee", "delivery zones", "add delivery option" |
| [admin/discounts.md](skills/admin/discounts.md) | "create coupon", "discount rule", "sale on product", "happy hour", "weekend discount", "first order discount", "win back", "scheduled discount", "recurring discount" |
| [admin/sections.md](skills/admin/sections.md) | "homepage", "home page layout", "add section", "hero", "banner", "promo", "categories grid", "bento", "product grid", "4 columns", "5 columns", "product row", "carousel", "slider", "best sellers", "hand-picked products", "reorder sections", "move section", "testimonials", "reviews section", "FAQ section", "brand logos", "countdown", "sale timer", "video", "reels", "TikTok videos", "YouTube shorts", "newsletter", "rich text", "image with text", "CTA", "products page layout" |
| [admin/shop-layout.md](skills/admin/shop-layout.md) | "header", "footer", "menu", "navigation links", "footer links", "social links", "announcement bar", "top bar", "promo bar", "countdown bar", "sale countdown", "bottom bar", "nav bar", "desktop bar", "side rail", "phone bar", "tab bar", "floating bar", "look", "atelier", "weyoung", "marketplace", "about page", "about us", "team", "timeline", "blog layout", "404 page", "not found page", "reset layout" |
| [admin/appearance.md](skills/admin/appearance.md) | "change theme", "brand colour", "custom colors", "colour palette", "preset theme", "dark shop", "change font", "upload font", "Khmer font", "MiSans", "Google Sans", "border radius", "reset theme", "card style", "product card look", "quick add / cart mode", "product page layout", "magazine / maison layout", "storefront width", "full width", "coming soon", "pre-launch", "launch countdown" |
| [admin/customer-auth.md](skills/admin/customer-auth.md) | "customer login", "telegram auth", "user token", "initData", "connect account", "register user", "customer JWT" |
| [admin/customer-service-agent.md](skills/admin/customer-service-agent.md) | "build CS bot", "customer service agent", "shop bot", "telegram bot for customers", "AI assistant for shop", "OpenClaw agent" |
| [admin/broadcasts.md](skills/admin/broadcasts.md) | "broadcast", "blast", "send to all customers", "direct message", "DM customer", "abandoned cart message", "report group", "telegram group", "owner notifications" |
| [admin/printing.md](skills/admin/printing.md) | "printer", "receipt printer", "auto print", "print job", "thermal printer", "configure printer", "test print", "provision printing", "RabbitMQ printer" |

### Advanced Features
| File | Use when user says… |
|---|---|
| [advanced/membership.md](skills/advanced/membership.md) | "VIP tier", "membership card", "loyalty program", "add member" |
| [advanced/quotations.md](skills/advanced/quotations.md) | "create quote", "send quotation", "revise quote", "contract pricing" |
| [advanced/content.md](skills/advanced/content.md) | "write blog post", "create event", "publish article", "ticket event", "blog layout", "cms" |
| [advanced/dns.md](skills/advanced/dns.md) | "custom domain", "connect domain", "DNS settings" |
| [advanced/plugins.md](skills/advanced/plugins.md) | "install plugin", "list plugins", "plugin settings" |

### Superadmin (Platform-Level — requires `role: "admin"`)
| File | Use when user says… |
|---|---|
| [superadmin/dashboard.md](skills/superadmin/dashboard.md) | "platform stats", "total revenue", "top shops", "platform dashboard" |
| [superadmin/shops.md](skills/superadmin/shops.md) | "list all shops", "activate shop", "deactivate shop", "all shops" |
| [superadmin/users.md](skills/superadmin/users.md) | "list all users", "freeze user", "activate user", "all users" |
| [superadmin/business-categories.md](skills/superadmin/business-categories.md) | "business categories", "add biz category", "shop types" |

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
OperationalMode:  OPEN | CLOSED | BUSY
ShippingModel:    DISTANCE_BASED | ZONE_BASED | FLAT_RATE | THIRD_PARTY_ONLY
CartMode:         DETAILS_ONLY | QUICK_CART_ONLY | BOTH
CardStyle:        MINIMAL | BORDERED | OVERLAY | GLASS
AppContainerSize: STANDARD | MEDIUM | LARGE | FULLSCREEN
```
`productDetailThemeId` (a String, not an enum): `classic | magazine | maison | camprotec`.
Theme JSON `font.source` is written `"Google"` / `"Custom"` but reads back as `google` / `custom`.

### Section Enums (Section Explorer) — SCREAMING_SNAKE_CASE on the wire
```
SectionKind:     HERO_EDITORIAL | HERO_IMMERSIVE | HERO_CENTERED_MINIMAL
                 | GRID | SLIDER | HORIZONTAL_SCROLL
                 | PRODUCTS_EDITORIAL_SPOTLIGHT
                 | CATEGORIES_GRID | CATEGORIES_ASYMMETRIC_BENTO
                 | PROMO_MOSAIC | PROMO_SINGLE_BANNER
                 | TESTIMONIALS_CARDS | TESTIMONIALS_CAROUSEL
                 | FAQ_ACCORDION | BRAND_STRIP
                 | IMAGE_TEXT | CTA_BANNER | COUNTDOWN
                 | RICH_TEXT | NEWSLETTER | VIDEO_HERO
                 | VIDEO_REELS_MARQUEE | VIDEO_REELS_DECK
SectionStatus:   DRAFT | PUBLISHED | HIDDEN
SectionLayout:   GRID_1 | GRID_2 | GRID_3 | GRID_4 | GRID_5 | SLIDER | HORIZONTAL_SCROLL
SectionSpacing:  NONE | SM | MD | LG | XL | XXL
DataSource:      PRODUCTS | EVENTS | POSTS
                 | PRODUCT_CATEGORIES | EVENT_CATEGORIES | POST_CATEGORIES
DataType:        STATIC | DYNAMIC
StaticType:      IMAGE | DESIGN
TitleAlignment:  LEFT | CENTER | RIGHT
SortOption:      NAME_ASC | NAME_DESC | PRICE_ASC | PRICE_DESC | DATE_ASC | DATE_DESC
                 | MANUAL            # "as arranged": the order of filterCondition.items
FilterType:      PRICE_RANGE | CATEGORIES | SUB_CATEGORIES | BRANDS | ITEMS
                 | PROMOTIONS | NONE | MULTI   # MULTI does not filter a section feed
```
> **Note:** the JSON inside `kindConfig` uses **camelCase** field names
> (`headingAccent`, `leftProductId`, `textColor`, etc.) and lowercase enum
> values (`"variant": "outline"`, `"provider": "tiktok"`, `"speed": "slow"`).
> Only the top-level GraphQL enums are SCREAMING_SNAKE_CASE. Sections render
> only on the `home`, `products`, `posts` and `events` pages.

### Discount Enums
```
DiscountScope:        LINE_ITEM | TOTAL_CART | SHIPPING | BUY_X_GET_Y
DiscountType:         FIXED | PERCENTAGE | LOCKED_PRICE
ShippingDiscountType: FIXED | PERCENTAGE | FREE
BxgyGetDiscountType:  FREE | PERCENTAGE | FIXED_AMOUNT_OFF
UsageTarget:          EVERY_ONE | MEMBERSHIP | COUPON
Day (for schedules): MON | TUE | WED | THU | FRI | SAT | SUN
```

### Shop Layout Enums (Looks, Announcement, Navigation bars)
> ⚠️ **These use snake_case on the wire** (not SCREAMING_SNAKE_CASE) —
> matches the storage tag.
```
LookId:                   classic | weyoung | atelier | editorial
                        | marketplace | maison | flagship | studio
FontPair:                 sans | serif_sans | slab_sans | mono_sans | serif_serif
Density:                  compact | comfortable | spacious
AnnouncementTemplateKind: classic_promo_socials | classic_centered_minimal
                        | classic_editorial_dark | weyoung
DesktopBarPosition:       bottom | left | right
BarLabels:                icons | icons_and_labels
BarColour:                light | dark | brand
MobileBarStyle:           classic | floating | pill | split | cradle
```
Header / footer JSON string values (snake_case). Only these render as designed
templates; the API accepts more, which fall back to a plain generic bar:
```
header.layout:  classic | minimal | centered | atelier | marketplace | weyoung
footer.layout:  columns | simple | newsletter_focus | atelier_footer
              | marketplace_footer_look | weyoung_plus | weyoung
header.bg_style: transparent | solid | blurred | custom
footer.bg_style: default | dark | light | primary | custom
404 templateId:  simple | illustrated | bold | lost | minimal | animated
```
> A Look's `brandColor` / `fontPair` / `density` are stored but **not applied**
> by the storefront. The brand colour and font come from the theme
> (`admin/appearance.md`).

---

## Global Agent Rules

1. **Resolve IDs first.** When a user refers to a product/category/brand/customer/variant by name, run a search or list query to get the `id` before mutating.
2. **Price = String.** All monetary values (`price`, `amount`, `unitPrice`) are passed as `String` in GraphQL to preserve decimal precision.
3. **shopId is always required.** Every query/mutation requires `shopId`. Resolve it once at the start of a conversation via `ownedShops` or `joinedShops`.
4. **Pagination.** List queries return `{ data, limit, skip, totalPages, totalDocuments }`. Default to `limit: 20, page: 1`.
5. **Soft delete.** Products are archived (`productSetArchived`), not hard-deleted. Use `productDelete` only when the user explicitly says "permanently delete."

## Critical Conventions — read before any mutation

These bite agents that skip the per-skill files. Internalize all of them.

### Image / video / font inputs are ALWAYS uploaded CDN URLs

Any field shaped like `image: { url, alt }`, `images: [String!]`, `logo: String`,
`imageUrl: String`, `coverImage: String`, `output: String` (designs), or
`sourceUrl: String` (video hero) takes a **real CDN URL from the upload
endpoints above** — never a literal path, placeholder, or external URL the
storefront doesn't control. Upload first, then write the returned string into
the mutation.

```bash
# canonical recipe — RIVERBASE_API_URL ends in /graphql, so strip it first
curl -X POST -H "Authorization: $RIVERBASE_API_KEY" \
  -F "file=@./local.jpg" "${RIVERBASE_API_URL%/graphql}/uploads/s3/hq"
# → "https://<storage-gateway>/<uuid>.webp"   (use the returned string as-is)
```

### JSON-scalar inputs take parsed JSON, not strings

`updateTheme(theme: JSON!)`, `updateShopHeader(header: JSON!)`,
`updateShopFooter`, `updateShopAboutPage`, `updateShopBlogLayout`,
`updateShopLayout`, `updateShopAnnouncement`'s `config`, and
`createSection`/`updateSection`'s `kindConfig: JSON` all accept the JSON value
directly (an object in `variables`), **not a JSON-encoded string**.
Stringifying first will fail server-side validation.

The opposite holds for pages: `createPage`/`updatePage` take `content` and
`seo` as **String** arguments holding JSON text (see `admin/pages.md`).

Key casing inside those JSON values is **not uniform** — check the file before
writing:
- Theme JSON: `shop_id`, camelCase colour keys, snake_case custom-font keys (`admin/appearance.md`).
- Header / footer / about / blog JSON: **snake_case** keys; camelCase keys are silently ignored (`admin/shop-layout.md`).
- Announcement `config` and section `kindConfig`: camelCase keys.
- Page `seo` JSON text: snake_case keys (`meta_title`).

Most of these writes **replace the whole object** (theme, each layout section,
`kindConfig`, the announcement surface, the module set): read the current
value, change it in memory, and send all of it back.

```jsonc
// ✅ correct — kindConfig is an object
"variables": { "kindConfig": { "heading": "...", "image": { "url": "..." } } }

// ❌ wrong — never stringify
"variables": { "kindConfig": "{\"heading\":\"...\"}" }
```

### `kindConfig` is REPLACE-only

`kindConfig` is stored as a single blob; `updateSection` overwrites it
wholesale. To change one field: `getSection` → mutate the object in memory →
send the **full** updated `kindConfig` back. Never send a partial object.

### Order creation is asynchronous (QueueTicket)

`orderCreate`, `orderCreateGuest`, and `posCreateOrder` return a
`QueueTicket { ticketId, status, message }`, NOT the created `Order`. The
actual order is produced asynchronously by the queue worker. Either listen
on Socket.IO for the ticket result or poll `orderById` once the ticket
finishes. Tell the user "Order submitted, processing…" instead of pretending
it's done.

### Telegram-bot prerequisite

`createBroadcast`, `sendDirectMessage`, `createReportGroup` (and the
notifications they fan out) require the shop to have an active Telegram bot
already configured. Check `shopId(shopId).activationStatus.telegramBot`
first — if `false`, prompt the user to wire the bot via `botCreate` before
attempting any push.

### Discount engine — best-saving wins by default

Multiple eligible discount rules in the same scope (`LINE_ITEM` / `TOTAL_CART` /
`SHIPPING`) compete and the largest saving wins. Merchant overrides:
- `forceApply: true` — this rule always wins in its scope (not allowed on `BUY_X_GET_Y`).
- `exclusionGroupId: "<string>"` — at most one rule per `(shopId, exclusionGroupId)` fires per cart, resolved by `priority` ascending.
- `exclusiveWithBxgy: true` on a non-BXGY rule — suppress this rule whenever any BXGY rule fires on the same order.

See `admin/discounts.md` for the full semantics before authoring any rule.

### Wire-format casing

- **GraphQL enums on the wire are SCREAMING_SNAKE_CASE** (`PUBLISHED`,
  `GRID_2`, `HERO_EDITORIAL`, `DATE_DESC`, `PRODUCTS`).
- **`kindConfig` JSON fields are camelCase** (`headingAccent`, `leftProductId`, `textColor`).
- **Sub-input objects in regular GraphQL inputs are camelCase** (`priceRange.from`, `socialProof.countLabel`, `comboComponents[].productId`).

### ⚠️ Destructive Operations — ALWAYS Require Confirmation

**NEVER execute any of these mutations without explicit user confirmation first.**
Show a summary of what will be affected, then ask "Are you sure? Reply Yes to confirm" or send a Telegram inline button.

| Mutation | What it does | File |
|---|---|---|
| `productDelete` | Permanently deletes a product | catalog/products.md |
| `productSetArchived(archived: true)` | Archives (soft-deletes) a product | catalog/products.md |
| `deleteCategory` | Deletes a category (products become uncategorized) | catalog/categories.md |
| `deleteSubcategory` | Deletes a subcategory | catalog/categories.md |
| `deletePage` | Deletes a custom page (URL will 404) | admin/pages.md |
| `deleteBrand` | Deletes a brand (products lose brand) | catalog/brands.md |
| `setOrderStatus(CANCELLED)` | Cancels an order (triggers stock reversal) | orders/lifecycle.md |
| `cancelOrder` | Rich cancellation (CancelledBy + CancelReason + note); triggers stock reversal | orders/lifecycle.md |
| `createBroadcast` | Pushes a Telegram message to EVERY customer of the shop | admin/broadcasts.md |
| `sendDirectMessage` | One-to-one Telegram push to a customer | admin/broadcasts.md |
| `deleteReportGroup` | Removes the bot's link to a Telegram report group | admin/broadcasts.md |
| `botRemove` | Disconnects the shop's Telegram bot | admin/shop.md |
| `printingDeprovisionTenant` | Wipes the entire printing setup (providers + printers) | admin/printing.md |
| `printingRemoveProvider` | Deletes a provider and cascades to its printers | admin/printing.md |
| `printingRemovePrinter` | Removes a printer | admin/printing.md |
| `printingClearJobHistory` | Bulk-deletes completed/failed/cancelled print jobs | admin/printing.md |
| `printingDeleteAutoPrintConfig` | Disables auto-print (back to manual jobs) | admin/printing.md |
| `setEntityType` | Flips the shop's legal entity type (changes receipt formatting) | admin/shop.md |
| `deleteEvent` | Deletes a ticketed event | advanced/content.md |
| `deletePost` | Deletes a blog post | advanced/content.md |
| `deleteInventoryLocation` | Deletes a warehouse/store location | inventory/locations.md |
| `shopRoleDelete` | Deletes a staff role | admin/team.md |
| `shopMemberRemove` | Removes a team member from the shop | admin/team.md |
| `deleteDeliveryOption` | Removes a delivery option | admin/shipping.md |
| `deleteCoupon` | Deletes a coupon code | admin/discounts.md |
| `deleteDiscountRule` | Deletes a discount rule | admin/discounts.md |
| `deleteSection` | Deletes a storefront section | admin/sections.md |
| `resetShopLayout` | Resets ALL chrome: header + footer (both disabled), about, blog, announcement, Look and its undo history, 404 choice, both navigation bars | admin/shop-layout.md |
| `resetTheme` | Resets theme (colors, radius, font) to defaults — owner only | admin/appearance.md |
| `shopSetComingSoon(comingSoon: true)` | Hides the whole storefront from shoppers behind a teaser | admin/appearance.md |
| `deleteDesign` | Deletes a canvas design | admin/sections.md |
| `deleteTeir` | Deletes a membership tier | advanced/membership.md |
| `deleteMembership` | Revokes a user's membership | advanced/membership.md |
| `deleteDns` | Removes custom domain | advanced/dns.md |
| `pluginUninstall` | Uninstalls a plugin | advanced/plugins.md |
| `terminateAgreement` | Terminates a consignment agreement | advanced (not yet documented) |
| `shopTransfer` | Hands shop ownership to another user | admin/shop.md |
| `shopActivate(active: false)` | Deactivates a shop (storefront goes offline) | superadmin/shops.md |
| `userSetActive(active: false)` | Freezes a user (loses all access) | superadmin/users.md |
| `deleteBusinessCategory` | Deletes a platform business category | superadmin/business-categories.md |

### Bootstrap Query (run first in every session)

**Before running any query**, if you have a local clone of the skill repo,
pull the latest skill files (the repo path varies by deployment):

```bash
# If you have a local clone at $KOOMPI_BIZ_SKILL, refresh it:
git -C "${KOOMPI_BIZ_SKILL:-/path/to/koompi-biz-skill}" pull origin master 2>/dev/null || true
```

Then run the bootstrap query — gives you the caller identity, the shops they
own, and the shops they're a staff member of in one round-trip:

```graphql
query Bootstrap {
  me {
    id role tgUsername tgFirstname tgLastname active
  }
  ownedShops {
    id name logo active productsCount
    activationStatus { shopLocation paymentConnection telegramBot }
    modules { shopping stocking quoting codPayment posCodEnabled cartMode }
  }
  joinedShops {
    id name logo active
  }
}
```

Why this shape:
- `me.role` tells you whether `superadmin/*` files are usable (`role: "admin"`).
- `activationStatus.telegramBot` gates the broadcasts / DMs / report-groups skills.
- `modules` flags which feature areas the shop has turned on (POS, stocking, quoting, COD).
