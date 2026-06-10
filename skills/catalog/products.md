# Products

## List All Products (paginated)
```graphql
query PaginatedProducts($shopId: String!, $limit: Int!, $page: Int!, $locationId: String) {
  paginatedProducts(shopId: $shopId, limit: $limit, page: $page) {
    data {
      id
      name
      price
      images
      onSale
      archived
      categoryId
      brandId
      stockBalance(locationId: $locationId)
      variants { id sku price active stockBalance(locationId: $locationId) variantAttributes { attributeName value } }
    }
    totalDocuments
    totalPages
  }
}
```

## Search Products
```graphql
query ProductSearch($shopId: String!, $keyword: String, $filter: FilterStateInput, $locationId: String) {
  productSearch(shopId: $shopId, keyword: $keyword, filter: $filter) {
    id
    name
    price
    description
    images
    onSale
    archived
    categoryId
    brandId
    tags
    stockBalance(locationId: $locationId)
    variants { id sku price active stockBalance(locationId: $locationId) variantAttributes { attributeName value } }
  }
}
```
`FilterStateInput` is optional. Fields: `{ filterBy: FilterType, categories: [String], subCategories: [String], brands: [String], sortBy: SortOption, limit: Int, items: [String] }`.

## Get Single Product
```graphql
query ProductById($productId: String!, $locationId: String) {
  productId(productId: $productId) {
    id name description price images shopId tags onSale archived isPreorder preorderDeposit
    categoryId category { id name } brandId subCategories detail
    lowStockThreshold maxQuantityPerOrder
    options { id name values { id name extraAmount image } }
    variants { id sku price images active lowStockThreshold stockBalance(locationId: $locationId) variantAttributes { attributeName value displayOrder } }
    stockBalance(locationId: $locationId)
  }
}
```

## Create Product
All prices are **String** type (e.g. `"12.50"`).
```graphql
mutation CreateProduct(
  $shopId: String!
  $name: String!
  $price: String!
  $description: String
  $images: [String!]
  $options: [ProductOptionsInput!]!  # pass [] if none
  $tags: [String!]
  $category: String       # categoryId
  $isPreorder: Boolean!   # false for normal products
  $preorderDeposit: String
  $brandId: String
  $subCategories: [String!]
  $detail: String
  $weightKg: Float        # measurement fields, all optional
  $lengthCm: Float
  $widthCm: Float
  $heightCm: Float
  $unit: String           # free-form selling unit: "kg", "ml", "L", "piece", "pack of 6", …
) {
  productCreate(
    shopId: $shopId name: $name price: $price description: $description
    images: $images options: $options tags: $tags category: $category
    isPreorder: $isPreorder preorderDeposit: $preorderDeposit
    brandId: $brandId subCategories: $subCategories detail: $detail
    weightKg: $weightKg lengthCm: $lengthCm widthCm: $widthCm heightCm: $heightCm
    unit: $unit
  ) {
    id name price unit
  }
}
```

### Measurement & selling unit
- `weightKg`, `lengthCm`, `widthCm`, `heightCm` — physical dimensions used
  by `DISTANCE_BASED` shipping calculation and the receipt template.
- `unit` — free-form string for how the product is sold (`"kg"`, `"ml"`,
  `"L"`, `"piece"`, `"pack of 6"`, `"bottle"`). Surfaces on the product card
  and receipt next to the price. Null/empty means "per item" (no suffix).

### ProductOptionsInput (non-stock options like "Sugar Level")
```json
{ "name": "Ice Level", "values": [{ "name": "No Ice" }, { "name": "Less Ice" }, { "name": "Normal Ice" }] }
```
Each value can optionally have `extraAmount: "0.50"` and `image: "/path"`.

## Update Product
Same fields as Create, plus `$productId: String!`:
```graphql
mutation ProductUpdate(
  $shopId: String! $productId: String!
  $name: String! $price: String! $description: String $images: [String!]
  $options: [ProductOptionsUpdateInput!]!
  $tags: [String!] $category: String $isPreorder: Boolean!
  $preorderDeposit: String $brandId: String $subCategories: [String!] $detail: String
  $weightKg: Float $lengthCm: Float $widthCm: Float $heightCm: Float
  $unit: String
) {
  productUpdate(
    shopId: $shopId productId: $productId name: $name price: $price
    description: $description images: $images options: $options tags: $tags
    category: $category isPreorder: $isPreorder preorderDeposit: $preorderDeposit
    brandId: $brandId subCategories: $subCategories detail: $detail
    weightKg: $weightKg lengthCm: $lengthCm widthCm: $widthCm heightCm: $heightCm
    unit: $unit
  ) { id name price unit }
}
```
**ProductOptionsUpdateInput** includes `_id` for existing options: `{ "_id": "abc", "name": "Size", "values": [{ "_id": "v1", "name": "S" }] }`.

## Toggle On Sale
```graphql
mutation SetOnSale($shopId: String!, $productId: String!, $onSale: Boolean!) {
  productSetOnSale(shopId: $shopId, productId: $productId, onSale: $onSale)
}
```

## Archive / Unarchive
> ⚠️ **CONFIRMATION REQUIRED** — When archiving, summarize the product name and ask user to confirm.
```graphql
mutation SetArchived($shopId: String!, $productId: String!, $archived: Boolean!) {
  productSetArchived(shopId: $shopId, productId: $productId, archived: $archived)
}
```

## Permanently Delete
> ⚠️ **CONFIRMATION REQUIRED** — Show product name/price and warn this is irreversible. Wait for explicit "Yes".
```graphql
mutation DeleteProduct($productId: String!, $shopId: String!) {
  productDelete(productId: $productId, shopId: $shopId)
}
```

## Set Max Order Quantity
```graphql
mutation SetMaxQty($shopId: String!, $productId: String!, $maxQuantity: Int) {
  setMaxOrderQuantity(shopId: $shopId, productId: $productId, maxQuantity: $maxQuantity) { id }
}
```
Pass `null` to remove the limit.

## Get Archived Products
```graphql
query ArchivedProducts($shopId: String!) {
  getArchivedProducts(shopId: $shopId) { id name price images stockBalance }
}
```

---

## Combo / Bundle Products

Turn a product into a **combo** — a bundle that auto-adds component products
when it's added to cart. Useful for set menus, gift bundles, kits, etc.

```graphql
mutation SetComboConfig(
  $shopId: String!
  $productId: String!
  $isCombo: Boolean!
  $comboComponents: [ComboComponentInput!]
  $comboAllowDiscounts: Boolean!
) {
  setComboConfig(
    shopId: $shopId
    productId: $productId
    isCombo: $isCombo
    comboComponents: $comboComponents
    comboAllowDiscounts: $comboAllowDiscounts
  ) {
    id name
  }
}
```

### `ComboComponentInput`
```json
{ "productId": "<id>", "variantId": "<variant_id_or_null>", "quantity": 1 }
```
`variantId` is optional — set it when the combo locks the customer to a
specific variant (e.g. Size M of a t-shirt). Leave `null` to use the product's
default selection at checkout.

### Recipes

**Build a "Burger Set" combo (burger + fries + drink):**
```json
{
  "shopId": "...",
  "productId": "<burger_set_product_id>",
  "isCombo": true,
  "comboComponents": [
    { "productId": "<burger_id>",  "quantity": 1 },
    { "productId": "<fries_id>",   "quantity": 1 },
    { "productId": "<drink_id>",   "quantity": 1 }
  ],
  "comboAllowDiscounts": false
}
```

**Turn a product back into a normal SKU:** pass `isCombo: false` and `null` /
empty for `comboComponents`. The components are forgotten on the next save.

### `comboAllowDiscounts`
- `false` (typical): the combo's own price is the final price; the discount
  engine does NOT separately discount the components when this combo is in
  the cart.
- `true`: each component is exposed to discount rules independently, so a
  line-item discount on a component will reduce the combo's effective total.
  Use sparingly — most merchants want fixed combo pricing.

> Combo components must be **active, non-archived products** in the same shop.
> Resolve names → ids via `productSearch` before calling `setComboConfig`.

---

## Shade Finder (cosmetics try-on)

For cosmetics shops — turn a product into a try-on swatch palette mapped onto a
face zone (lips, eyes, lashes, cheeks, foundation). Each variant becomes a
selectable shade in the storefront's shade-finder widget.

```graphql
mutation SetShadeFinder(
  $productId: String!
  $shopId: String!
  $enabled: Boolean!
  $zone: ApplicationZone!
  $shades: [ShadeConfigInput!]
  $singleShadeColor: String
  $singleShadeOpacity: Float
  $smoothing: Float
  $toneEvening: Float
) {
  productSetShadeFinder(
    productId: $productId
    shopId: $shopId
    enabled: $enabled
    zone: $zone
    shades: $shades
    singleShadeColor: $singleShadeColor
    singleShadeOpacity: $singleShadeOpacity
    smoothing: $smoothing
    toneEvening: $toneEvening
  )
}
```

### Enums + input shape
```
ApplicationZone: LIPS | EYES | LASHES | CHEEKS | FOUNDATION
```
```json
// ShadeConfigInput — one entry per shade/variant
{
  "variantId": "<existing ProductOptionValue._id>",
  "opacity": 0.85,                 // 0.0..1.0 overlay opacity
  "colorHex": "#b32f3a"            // optional; if empty the variant's image/color is used
}
```
- `smoothing`: skin-smoothing factor, 0.0..1.0 (default ~0.4).
- `toneEvening`: tone-evening factor, 0.0..1.0 (default ~0.3).
- `singleShadeColor` / `singleShadeOpacity`: fallback for products without
  per-variant shade overrides (single-shade lipstick, for example).

### Recipes

**Enable a 6-shade lipstick palette:**
```json
{
  "productId": "<lipstick_id>",
  "shopId": "...",
  "enabled": true,
  "zone": "LIPS",
  "shades": [
    { "variantId": "<v_rose>",   "opacity": 0.85, "colorHex": "#c2486d" },
    { "variantId": "<v_red>",    "opacity": 0.90, "colorHex": "#b32f3a" },
    { "variantId": "<v_nude>",   "opacity": 0.70, "colorHex": "#d39080" },
    { "variantId": "<v_berry>",  "opacity": 0.88, "colorHex": "#6a1f3c" },
    { "variantId": "<v_coral>",  "opacity": 0.82, "colorHex": "#e07a5f" },
    { "variantId": "<v_mauve>",  "opacity": 0.80, "colorHex": "#9a6c8a" }
  ],
  "smoothing": 0.4,
  "toneEvening": 0.3
}
```

**Disable Shade Finder for a product:** pass `enabled: false` (other fields
ignored — the stored config is cleared).

> Shade Finder requires the product to have **variants** (each shade is mapped
> to a variant). Add variants via `productUpdate` with `options` before
> enabling Shade Finder.

---

## Agent Notes
- **Images**: Upload files via `POST https://api.riverbase.org/uploads/s3` (multipart, field name `file`, header `authorization: <token>`). Returns a full CDN URL like `https://cdn.riverbase.org/uuid.webp` — pass that URL in the `images` array. Images are auto-converted to WebP (800px max width).
- **Variants vs Options**: `variants` are stock-tracked (Size L = separate SKU). `options` are non-stock (Sugar Level = no inventory impact). Use variants when the shop needs per-attribute stock tracking.
- **Filtering tip**: Use `keyword` for text search. Use `filter.categories` to narrow by category ID array.
- **Combo + discounts**: default to `comboAllowDiscounts: false` unless the merchant explicitly wants per-component discounting.
- **Shade Finder + variants**: each shade must map to an existing variant — resolve variant ids from `productId(productId).variants[].id` before saving the config.
