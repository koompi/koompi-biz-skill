# Sections & Designs (Storefront Layout)

Sections are the building blocks of a Riverbase storefront. They render on a
page (`"home"`, `"shop"`, etc.) in `displayOrder` order. The admin builds them
through **Section Explorer** (`/sections/explorer?page=home`) — a gallery
grouped by `kind` (Hero / Products / Categories / Promo / Trust / Content)
plus a **Build custom** option for legacy feed-driven layouts.

An agent can do everything the admin UI does — pick a kind, set its content,
upload images, choose a background, and publish — by calling `createSection` /
`updateSection` with the right `kind`, `kindConfig`, and uploaded image URLs.

## What the admin UI actually does

When the admin clicks a tile in Section Explorer:

| Tile group | What clicking it does |
|---|---|
| Hero / Products spotlight / Categories / Promo / Trust / Content | Routes to `/sections/new?kind=<KIND>&style=<slug>` which seeds `kindConfig` from the registry's `defaultConfig`, then opens the per-kind Editor. On Publish it calls `createSection` with `status: PUBLISHED` (or `DRAFT` if saved as draft). |
| Grid 3-col / Grid 2-col / Slider / Horizontal Scroll | "Retrofit" tiles — route to the legacy `/sections/create?layout=GRID_3` builder. No `kindConfig`; instead `dataSource` + `filterCondition` drive what shows up. |
| **Build custom** | Same legacy builder, blank — admin picks dataSource + filter from scratch. |

Agents follow the same two flows:

- **Editorial / explorer kinds** → set `kind`, fill `kindConfig`, leave `dataSource: PRODUCTS` and `dataType: STATIC` (or whatever the storefront ignores for that kind).
- **Feed kinds (GRID / SLIDER / HORIZONTAL_SCROLL)** → set `kind` to one of those + `layout` (`GRID_1` / `GRID_2` / `GRID_3` / `SLIDER` / `HORIZONTAL_SCROLL`) + `dataSource` + `filterCondition`. `kindConfig: null`.

---

## Image upload (REQUIRED for any kind that takes an image)

There is no "image URL" field that agents can fake. Real CDN URLs come from the
shop's upload pipeline:

| Endpoint | Use for |
|---|---|
| `POST <API_URL>/uploads/s3` | Standard quality (logos, thumbnails, small decorative images). Re-encoded to ≤800px wide WebP. |
| `POST <API_URL>/uploads/s3/hq` | **Use for hero / banner / promo / immersive imagery**. Re-encoded to ≤1920px wide WebP. |
| `POST <API_URL>/uploads/font` | Custom font files (TTF/OTF/WOFF). Only used by the Appearance skill. |

Both image endpoints accept a `multipart/form-data` POST with field name
`file`, authenticated via the same `Authorization: <token>` header. Response
body is the full CDN URL string to write back into `kindConfig`.

```bash
# Example: upload a hero image and capture the returned URL
curl -X POST \
  -H "Authorization: $RIVERBASE_API_KEY" \
  -F "file=@./hero.jpg" \
  "$RIVERBASE_API_URL/uploads/s3/hq"
# → "https://cdn.riverbase.org/...hero.webp"
```

Aspect ratio hints the storefront uses (set the image you upload to roughly
match — the storefront object-covers either way, but right-aspect avoids
ugly cropping):

| Kind / slot | Aspect |
|---|---|
| Hero, video, banner, promo wide tile, full-bleed | **16:9 wide** (use `/hq`) |
| Categories grid tile, brand logo, product tile | **1:1 square** |
| Editorial spotlight, image-text, hero-editorial side image | **4:5 portrait** |

---

## GraphQL Wire-Format Enums (SCREAMING_SNAKE_CASE)

These are the literal string values you pass over GraphQL. **Not** the
PascalCase you might see in Rust source. If you pass `"hero_editorial"` or
`"HeroEditorial"` it will fail enum coercion.

```
SectionKind:    HERO_EDITORIAL | HERO_IMMERSIVE | HERO_CENTERED_MINIMAL
              | GRID | SLIDER | HORIZONTAL_SCROLL
              | PRODUCTS_EDITORIAL_SPOTLIGHT
              | CATEGORIES_GRID | CATEGORIES_ASYMMETRIC_BENTO
              | PROMO_MOSAIC | PROMO_SINGLE_BANNER
              | TESTIMONIALS_CARDS | TESTIMONIALS_CAROUSEL
              | FAQ_ACCORDION | BRAND_STRIP
              | IMAGE_TEXT | CTA_BANNER | COUNTDOWN
              | RICH_TEXT | NEWSLETTER | VIDEO_HERO

SectionStatus:   DRAFT | PUBLISHED | HIDDEN
SectionLayout:   GRID_1 | GRID_2 | GRID_3 | SLIDER | HORIZONTAL_SCROLL
SectionSpacing:  NONE | SM | MD | LG | XL | XXL
TitleAlignment:  LEFT | CENTER | RIGHT
DataType:        STATIC | DYNAMIC
DataSource:      PRODUCTS | EVENTS | POSTS
               | PRODUCT_CATEGORIES | EVENT_CATEGORIES | POST_CATEGORIES
StaticType:      IMAGE | DESIGN
SortOption:      NAME_ASC | NAME_DESC | PRICE_ASC | PRICE_DESC | DATE_ASC | DATE_DESC
FilterType:      PRICE_RANGE | CATEGORIES | SUB_CATEGORIES | BRANDS
               | ITEMS | PROMOTIONS | NONE | MULTI
```

> **Note on `kindConfig` JSON:** the JSON *inside* `kindConfig` uses
> `camelCase` field names (e.g. `headingAccent`, `leftProductId`,
> `textColor`). Only the top-level GraphQL enums are SCREAMING_SNAKE_CASE.

---

## Queries

### List sections for a page
```graphql
query PageSections($shopId: String!, $page: String!, $statusFilter: [SectionStatus!]) {
  getPageSections(shopId: $shopId, page: $page, statusFilter: $statusFilter) {
    id name page displayOrder status
    kind kindConfig
    dataSource dataType layout
    showTitle titleAlignment padding gap
    background { color image { url alt } }
    filterCondition {
      priceRange { from to }
      sortBy filterBy categories items limit subCategories brands
    }
    staticFilter { designId images staticType }
  }
}
```
- `page`: `"home"`, `"shop"`, etc.
- `statusFilter`: optional, e.g. `[DRAFT, PUBLISHED]` for admin views.
  Anonymous storefront callers always get only `PUBLISHED` regardless.

### List all sections for a shop
```graphql
query AllSections($shopId: String!, $statusFilter: [SectionStatus!]) {
  getSections(shopId: $shopId, statusFilter: $statusFilter) {
    id name page displayOrder status kind
  }
}
```

### Get one section (with nested products / posts / events / categories)
```graphql
query SectionById($sectionId: String!, $skip: Int, $locationId: String) {
  getSection(sectionId: $sectionId) {
    id shopId name page displayOrder status
    kind kindConfig
    dataSource dataType layout
    showTitle titleAlignment padding gap
    background { color image { url alt } }
    filterCondition {
      priceRange { from to }
      sortBy filterBy categories items limit subCategories brands
    }
    staticFilter { designId images staticType }
    products(skip: $skip, locationId: $locationId) {
      data { id name price images } totalDocuments totalPages limit skip
    }
    events(skip: $skip) {
      data { id name date venue price images } totalDocuments totalPages
    }
    posts(skip: $skip) {
      data { id title images } totalDocuments totalPages
    }
    productCategories { data { id name } totalDocuments }
    eventCategories   { data { id name } totalDocuments }
    postCategories    { data { id name } totalDocuments }
  }
}
```

---

## Mutations

### Create
```graphql
mutation CreateSection(
  $shopId: String!
  $name: String!
  $page: String!
  $dataSource: DataSource!
  $filterCondition: FilterStateInput
  $layout: SectionLayout!
  $displayOrder: Int!
  $showTitle: Boolean!
  $titleAlignment: TitleAlignment
  $dataType: DataType!
  $padding: SectionSpacing!
  $gap: SectionSpacing!
  $staticFilter: StaticFilterInput
  $kind: SectionKind
  $kindConfig: JSON
  $status: SectionStatus
  $background: SectionBackgroundInput
) {
  createSection(
    shopId: $shopId name: $name page: $page
    dataSource: $dataSource filterCondition: $filterCondition
    layout: $layout displayOrder: $displayOrder
    showTitle: $showTitle titleAlignment: $titleAlignment
    dataType: $dataType padding: $padding gap: $gap
    staticFilter: $staticFilter
    kind: $kind kindConfig: $kindConfig
    status: $status background: $background
  )
}
```
Returns the new section's `id` (String).

### Update
Every field except `shopId` + `sectionId` is optional; only what you pass is touched.
```graphql
mutation UpdateSection(
  $sectionId: String!, $shopId: String!,
  $name: String, $page: String,
  $dataSource: DataSource, $filterCondition: FilterStateInput,
  $layout: SectionLayout, $displayOrder: Int,
  $showTitle: Boolean, $titleAlignment: TitleAlignment,
  $dataType: DataType, $padding: SectionSpacing, $gap: SectionSpacing,
  $staticFilter: StaticFilterInput,
  $kind: SectionKind, $kindConfig: JSON,
  $status: SectionStatus, $background: SectionBackgroundInput
) {
  updateSection(
    sectionId: $sectionId shopId: $shopId
    name: $name page: $page
    dataSource: $dataSource filterCondition: $filterCondition
    layout: $layout displayOrder: $displayOrder
    showTitle: $showTitle titleAlignment: $titleAlignment
    dataType: $dataType padding: $padding gap: $gap
    staticFilter: $staticFilter
    kind: $kind kindConfig: $kindConfig
    status: $status background: $background
  )
}
```

### Publish (draft → published)
```graphql
mutation PublishSection($shopId: String!, $sectionId: String!) {
  publishSection(shopId: $shopId, sectionId: $sectionId)
}
```

### Delete
> ⚠️ **CONFIRMATION REQUIRED** — Show section name and page before deleting.
```graphql
mutation DeleteSection($sectionId: String!, $shopId: String!) {
  deleteSection(sectionId: $sectionId, shopId: $shopId)
}
```

---

## Universal section fields (apply to every kind)

The admin UI surfaces these four groups for *every* kind:

### 1. Content
- `name` — internal label (storefront does not render this; just for the admin list).
- `showTitle` + `titleAlignment` — when true, render a heading bar above the section.

### 2. Appearance — `background`
Solid color, image, or both. Image wins when both are set.
```json
{ "background": {
    "color": "#f5e6d8",
    "image": { "url": "https://cdn.riverbase.org/bg.webp", "alt": "Sand pattern" }
} }
```

### 3. Spacing
- `padding` (`NONE`/`SM`/`MD`/`LG`/`XL`/`XXL`) — inner padding of the section block.
- `gap` (same enum) — gap between items inside grid / slider kinds.

### 4. Lifecycle
- `status: DRAFT` while authoring, `PUBLISHED` when ready.
- `displayOrder` — sections render ascending. New section? Query existing
  sections first and use `max(displayOrder) + 1` (the admin UI appends to
  the bottom; matching that is good etiquette).

---

## Picking a kind: registry table

This mirrors what the admin sees in Section Explorer. Use it to translate a
user request ("hero with a big image", "FAQ", "promo grid") into the right
`kind` + `kindConfig` shape.

| Group | Kind | When to use |
|---|---|---|
| **Hero** | `HERO_EDITORIAL` | Split layout — image left, tri-segment heading right. Editorial feel. |
| Hero | `HERO_IMMERSIVE` | Full-bleed photo with overlay copy + dark overlay. |
| Hero | `HERO_CENTERED_MINIMAL` | Headline-only hero on a neutral background. No image. |
| **Products** | `GRID` + `layout: GRID_2`/`GRID_3` | Standard product grid. Configure data + filter. |
| Products | `SLIDER` | Auto-rotating product carousel. |
| Products | `HORIZONTAL_SCROLL` | Swipeable product row. |
| Products | `PRODUCTS_EDITORIAL_SPOTLIGHT` | Two large product cards side-by-side; pick the products. |
| **Categories** | `CATEGORIES_GRID` | Uniform category cards in a 3/4-column grid. Reference categories by id. |
| Categories | `CATEGORIES_ASYMMETRIC_BENTO` | 1 feature + 1 wide + 2 small bento layout. 4 category refs. |
| **Promo** | `PROMO_MOSAIC` | 4-tile asymmetric mosaic with per-tile copy + CTA. |
| Promo | `PROMO_SINGLE_BANNER` | One wide banner with CTA. |
| **Trust** | `TESTIMONIALS_CARDS` | Grid of testimonial cards (avatar + quote + rating). |
| Trust | `TESTIMONIALS_CAROUSEL` | Auto-rotating single-quote carousel. |
| Trust | `FAQ_ACCORDION` | Q&A accordion. |
| Trust | `BRAND_STRIP` | Logo strip referencing the shop's brand catalog. |
| **Content** | `IMAGE_TEXT` | Side-by-side image and copy with CTA. |
| Content | `CTA_BANNER` | Full-width CTA block on a colored background. |
| Content | `COUNTDOWN` | Sale/launch countdown to a target ISO timestamp. |
| Content | `RICH_TEXT` | Long-form HTML body. |
| Content | `NEWSLETTER` | Email signup with submit form. |
| Content | `VIDEO_HERO` | Embedded YouTube/Vimeo or self-hosted video. |

---

## Per-kind `kindConfig` schemas

> All fields are optional unless noted. Empty / null fields render with the
> kind's defaults. JSON keys are **camelCase**.

> Every kind that renders text supports an optional `textColor` (hex string)
> so copy stays legible over branded backgrounds or images. `PROMO_MOSAIC`
> and `PRODUCTS_EDITORIAL_SPOTLIGHT` carry per-card text colors.

### `HERO_EDITORIAL`
```json
{
  "image": { "url": "https://cdn.riverbase.org/hero.webp", "alt": "Spring collection" },
  "eyebrow": "NEW SEASON",
  "headingPre": "DEFINING THE NEW",
  "headingAccent": "ERA",
  "headingPost": "OF STYLE.",
  "subhead": "Curated drops for the modern wardrobe.",
  "primaryCta":   { "label": "Shop now", "href": "/shop", "variant": "primary" },
  "secondaryCta": { "label": "Lookbook", "href": "/posts", "variant": "outline" },
  "socialProof": {
    "avatars": [{ "url": "...", "alt": "Customer" }],
    "countLabel": "5,000+ happy customers",
    "quote": "The fit is unmatched."
  },
  "quickPicks": [
    { "image": { "url": "..." }, "href": "/p/sku-1" }
  ],
  "textColor": "#111111"
}
```
`CtaConfig.variant` ∈ `"primary" | "outline" | "link"`.

### `HERO_IMMERSIVE`
```json
{
  "image": { "url": "...", "alt": "Hero photo" },
  "heading": "Engineered for everyday",
  "subhead": "Premium essentials, ethically made.",
  "cta": { "label": "Explore", "href": "/shop", "variant": "primary" },
  "overlayOpacity": 0.35,
  "textColor": "#ffffff"
}
```
`overlayOpacity` ∈ `0..1` (dark scrim over the photo). Default 0.35.

### `HERO_CENTERED_MINIMAL`
```json
{
  "eyebrow": "WELCOME",
  "heading": "Quietly considered.",
  "subhead": "Pieces designed to last beyond the season.",
  "cta": { "label": "Shop now", "href": "/shop" },
  "textColor": "#111111"
}
```

### `PRODUCTS_EDITORIAL_SPOTLIGHT`
Prefer product-ID references. Legacy inline `leftFeatured`/`rightFeatured`
still render for older documents.
```json
{
  "heading": "This week's picks",
  "subhead": "Two staples we can't stop wearing.",
  "cta": { "label": "See all", "href": "/shop" },
  "leftProductId":  "<product_id>",
  "rightProductId": "<product_id>",
  "leftTagOverride":  "Editor's pick",
  "rightTagOverride": null,
  "leftTextColor":  "#ffffff",
  "rightTextColor": "#111111"
}
```
Always resolve product names → ids first (use the `Products` skill).

### `CATEGORIES_GRID`
```json
{
  "heading": "Shop by category",
  "columns": 4,
  "categoryIds": ["<cat_id_1>", "<cat_id_2>", "<cat_id_3>", "<cat_id_4>"]
}
```
Legacy inline `cards: [{ image, title, href, textColor }]` still works for old docs.

### `CATEGORIES_ASYMMETRIC_BENTO`
```json
{
  "heading": "Featured categories",
  "subhead": "Pick your aesthetic.",
  "viewAllHref": "/categories",
  "featureCategoryId": "<cat_id>",
  "wideCategoryId":    "<cat_id>",
  "smallACategoryId":  "<cat_id>",
  "smallBCategoryId":  "<cat_id>"
}
```
Legacy inline `feature` / `wide` / `smallA` / `smallB` (each `{ image, title,
description, href, textColor }`) still works for old docs.

### `PROMO_MOSAIC` — 4 tiles, per-tile CTA + textColor
```json
{
  "headerLabel": "View all promos",
  "headerHref": "/promo",
  "tall": {
    "image": { "url": "..." },
    "eyebrow": "LIMITED",
    "title": "Linen capsule",
    "subtitle": "Up to 40% off",
    "cta": { "label": "Shop", "href": "/c/linen" },
    "background": "#0f0f0f",
    "textColor": "#ffffff"
  },
  "topA":  { "image": { "url": "..." }, "title": "New arrivals", "cta": { "label": "Browse" }, "textColor": "#111111" },
  "topB":  { "image": { "url": "..." }, "title": "Best sellers", "cta": { "label": "Browse" }, "textColor": "#111111" },
  "wide":  { "image": { "url": "..." }, "title": "Gift sets",    "cta": { "label": "Shop" },    "textColor": "#ffffff" }
}
```
Tile `background` is a free-form CSS color string (hex / named / `var(--…)`).

### `PROMO_SINGLE_BANNER`
```json
{
  "image": { "url": "..." },
  "eyebrow": "FRIENDS & FAMILY",
  "heading": "Take 20% off everything",
  "subhead": "Use code FAMILY at checkout.",
  "cta": { "label": "Shop the sale", "href": "/shop", "variant": "primary" },
  "alignment": "left",
  "textColor": "#ffffff"
}
```
`alignment` ∈ `"left" | "center" | "right"`.

### `TESTIMONIALS_CARDS`
```json
{
  "heading": "Loved by our customers",
  "subhead": "What people are saying",
  "columns": 3,
  "items": [
    {
      "quote": "Best fit I've ever owned.",
      "author": "Alex P.",
      "role": "Verified buyer",
      "avatar": { "url": "..." },
      "rating": 5.0
    }
  ],
  "textColor": "#111111"
}
```

### `TESTIMONIALS_CAROUSEL`
```json
{
  "heading": "Customer love",
  "items": [
    { "quote": "Beautiful packaging.", "author": "Maria L.", "role": "Returning customer" }
  ],
  "textColor": "#111111"
}
```

### `FAQ_ACCORDION`
```json
{
  "heading": "Frequently asked",
  "subhead": "Everything about shipping, returns, and sizing.",
  "items": [
    { "question": "How long does shipping take?", "answer": "Most orders ship within 2 business days." }
  ],
  "textColor": "#111111"
}
```

### `BRAND_STRIP`
```json
{
  "heading": "As featured in",
  "brandIds": ["<brand_id_1>", "<brand_id_2>", "<brand_id_3>"],
  "textColor": "#666666"
}
```
Legacy inline `logos: [{ image, label, href }]` still renders for old documents.

### `IMAGE_TEXT`
```json
{
  "image": { "url": "..." },
  "imagePosition": "left",
  "eyebrow": "OUR STORY",
  "heading": "Built by craftspeople",
  "body": "We work with small ateliers across Cambodia.",
  "cta": { "label": "Learn more", "href": "/about" },
  "textColor": "#111111"
}
```
`imagePosition` ∈ `"left" | "right"`.

### `CTA_BANNER`
```json
{
  "heading": "Join the inner circle",
  "subhead": "Early access to drops, members-only pricing, and free shipping over $50.",
  "cta": { "label": "Become a member", "href": "/membership", "variant": "primary" },
  "background": "#1a1a1a",
  "textColor": "#ffffff"
}
```

### `COUNTDOWN`
```json
{
  "heading": "Spring sale ends in",
  "subhead": "Final hours.",
  "targetAt": "2026-06-01T00:00:00Z",
  "cta": { "label": "Shop now", "href": "/sale" },
  "textColor": "#ffffff"
}
```
`targetAt` is an ISO-8601 UTC string. Past timestamps render `0d 0h 0m 0s`.

### `RICH_TEXT`
```json
{
  "html": "<h2>Our promise</h2><p>Quality, transparency, and care.</p>",
  "maxWidth": "narrow",
  "textColor": "#222222"
}
```
`maxWidth` ∈ `"narrow" | "medium" | "wide"`. HTML is sanitized at render time.

### `NEWSLETTER`
```json
{
  "heading": "Get the drop",
  "subhead": "Weekly newsletter, no spam.",
  "placeholder": "your@email.com",
  "ctaLabel": "Subscribe",
  "submitHref": "/api/subscribe",
  "textColor": "#111111"
}
```

### `VIDEO_HERO`
Header-less — stack a separate text section above if you need a title.
```json
{
  "provider": "youtube",
  "sourceUrl": "https://youtu.be/<id>",
  "posterImage": { "url": "https://cdn.riverbase.org/poster.webp" },
  "autoplay": true,
  "loop": true,
  "muted": true,
  "showControls": false
}
```
`provider` ∈ `"youtube" | "vimeo" | "url"`. For `"url"`, upload the file via
`/uploads/s3/hq` (or any direct media URL).

---

## Feed kinds: `GRID` / `SLIDER` / `HORIZONTAL_SCROLL`

These read from `dataSource` + `filterCondition` and ignore `kindConfig`.

```json
{
  "shopId": "...",
  "name": "All Products",
  "page": "home",
  "kind": "GRID",
  "kindConfig": null,
  "dataSource": "PRODUCTS",
  "dataType": "DYNAMIC",
  "layout": "GRID_2",
  "displayOrder": 1,
  "showTitle": true,
  "titleAlignment": "LEFT",
  "padding": "MD",
  "gap": "MD",
  "status": "PUBLISHED",
  "filterCondition": {
    "priceRange": { "from": 0, "to": 0 },
    "sortBy": "DATE_DESC",
    "filterBy": "NONE",
    "categories": [],
    "subCategories": [],
    "brands": [],
    "items": [],
    "limit": 20
  },
  "background": null
}
```

`filterCondition` cheat sheet:
- Use `filterBy: "NONE"` to ignore filters entirely.
- `filterBy: "MULTI"` applies every non-empty field at once.
- `filterBy: "CATEGORIES"` / `"SUB_CATEGORIES"` / `"BRANDS"` / `"ITEMS"` /
  `"PROMOTIONS"` applies just that one filter.
- `priceRange: { from: 0, to: 0 }` is the "no range" sentinel.

Set `dataSource` to `EVENTS`, `POSTS`, `PRODUCT_CATEGORIES`,
`POST_CATEGORIES`, or `EVENT_CATEGORIES` to feed those catalogs instead.

---

## Static-image / Design banner (legacy)

Pre-Explorer way to render a single image or canvas design. Still supported.
Newer kinds (`PROMO_SINGLE_BANNER`, `HERO_IMMERSIVE`) are preferred for new work
because they give the admin more controls.

```json
{
  "kind": "GRID",
  "kindConfig": null,
  "dataSource": "PRODUCTS",
  "dataType": "STATIC",
  "layout": "GRID_1",
  "staticFilter": {
    "staticType": "IMAGE",
    "images": ["https://cdn.riverbase.org/your-uploaded-image.webp"]
  }
}
```
For a canvas design instead of an image:
```json
{ "staticFilter": { "staticType": "DESIGN", "designId": "<design_id>" } }
```

---

# Designs (Canvas Editor)

Designs are visual layouts created in the canvas editor. They can be referenced
by static-flow sections via `staticFilter.designId`.

### List
```graphql
query DesignsByShop($shopId: String!) {
  getDesignsByShop(shopId: $shopId) {
    id shopId name output layers
    canvasSize { width height } createdAt
  }
}
```
### Get
```graphql
query DesignById($designId: String!) {
  getDesign(designId: $designId) {
    id shopId name output layers
    canvasSize { width height } createdAt
  }
}
```
### Create
```graphql
mutation CreateDesign($shopId: String!, $payload: NewDesignInput!) {
  createDesign(shopId: $shopId, payload: $payload) { id name output }
}
```
```json
{
  "name": "Promo Banner",
  "layers": {},
  "output": "https://cdn.riverbase.org/rendered-design.webp",
  "canvasSize": { "width": 360, "height": 450 }
}
```
- `layers`: canvas editor layer data (JSON object).
- `output`: pre-rendered image URL of the design (upload via `/uploads/s3`).
- `canvasSize`: pixel dimensions (default 360x450).

### Update
```graphql
mutation UpdateDesign($shopId: String!, $designId: String!, $payload: NewDesignInput!) {
  updateDesign(shopId: $shopId, designId: $designId, payload: $payload) { id name output }
}
```

### Delete
> ⚠️ **CONFIRMATION REQUIRED** — show name + warn that sections referencing it will break.
```graphql
mutation DeleteDesign($designId: String!, $shopId: String!) {
  deleteDesign(shopId: $shopId, designId: $designId)
}
```

---

## End-to-end agent recipes

### Build a hero from a user prompt: "add a hero with an image of my product, headline 'Spring drop', and a button to /spring"

1. **Resolve shopId** via the bootstrap query.
2. **Upload the hero image**:
   ```bash
   curl -X POST -H "Authorization: $RIVERBASE_API_KEY" \
     -F "file=@spring.jpg" "$RIVERBASE_API_URL/uploads/s3/hq"
   # → "https://cdn.riverbase.org/<uuid>.webp"
   ```
3. **Find the next displayOrder**: query `getPageSections(shopId, page: "home")` and use `max(displayOrder) + 1`.
4. **Create the section**:
   ```json
   {
     "shopId": "...",
     "name": "Spring hero",
     "page": "home",
     "kind": "HERO_IMMERSIVE",
     "kindConfig": {
       "image": { "url": "https://cdn.riverbase.org/<uuid>.webp", "alt": "Spring collection" },
       "heading": "Spring drop",
       "subhead": null,
       "cta": { "label": "Shop the drop", "href": "/spring", "variant": "primary" },
       "overlayOpacity": 0.35,
       "textColor": "#ffffff"
     },
     "dataSource": "PRODUCTS",
     "dataType": "STATIC",
     "layout": "GRID_1",
     "displayOrder": 1,
     "showTitle": false,
     "titleAlignment": "LEFT",
     "padding": "NONE",
     "gap": "NONE",
     "status": "PUBLISHED",
     "background": null
   }
   ```

### Build a 4-category bento: "make a categories bento with Linen, Knits, Outerwear, Accessories"

1. List categories via the Catalog → categories skill to resolve names → ids.
2. Create with `kind: CATEGORIES_ASYMMETRIC_BENTO`:
   ```json
   {
     "kind": "CATEGORIES_ASYMMETRIC_BENTO",
     "kindConfig": {
       "heading": "Featured categories",
       "viewAllHref": "/categories",
       "featureCategoryId": "<linen_id>",
       "wideCategoryId":    "<knits_id>",
       "smallACategoryId":  "<outerwear_id>",
       "smallBCategoryId":  "<accessories_id>"
     },
     "dataSource": "PRODUCT_CATEGORIES",
     "dataType": "STATIC",
     "layout": "GRID_1",
     "displayOrder": 2,
     "showTitle": false,
     "titleAlignment": "LEFT",
     "padding": "MD",
     "gap": "MD",
     "status": "PUBLISHED"
   }
   ```

### Quick FAQ section
```json
{
  "kind": "FAQ_ACCORDION",
  "kindConfig": {
    "heading": "Frequently asked",
    "items": [
      { "question": "How long does shipping take?", "answer": "1–2 business days." },
      { "question": "Returns?", "answer": "Within 30 days, unworn." }
    ]
  },
  "dataSource": "PRODUCTS",
  "dataType": "STATIC",
  "layout": "GRID_1",
  "displayOrder": 4,
  "showTitle": false,
  "titleAlignment": "LEFT",
  "padding": "MD",
  "gap": "MD",
  "status": "PUBLISHED"
}
```

### Plain product grid (the "Build custom" path)
```json
{
  "name": "New arrivals",
  "kind": "GRID",
  "kindConfig": null,
  "dataSource": "PRODUCTS",
  "dataType": "DYNAMIC",
  "layout": "GRID_3",
  "displayOrder": 3,
  "showTitle": true,
  "titleAlignment": "LEFT",
  "padding": "MD",
  "gap": "MD",
  "status": "PUBLISHED",
  "filterCondition": {
    "priceRange": { "from": 0, "to": 0 },
    "sortBy": "DATE_DESC",
    "filterBy": "NONE",
    "categories": [], "subCategories": [], "brands": [], "items": [],
    "limit": 12
  }
}
```

### Make a draft, then publish later
1. Same `createSection` call as above but with `status: DRAFT`.
2. When the merchant approves: `publishSection(shopId, sectionId)`.

### Replace an existing hero's image
1. Upload the new image to `/uploads/s3/hq`.
2. Read the current section with `getSection` to keep the rest of `kindConfig`.
3. Call `updateSection` with `kindConfig` containing the new `image.url` (you
   must send the **entire** kindConfig object — it's stored whole, not patched).

---

## Agent Notes

- **Enum casing matters.** GraphQL enums on the wire are SCREAMING_SNAKE_CASE
  (`HERO_EDITORIAL`, `PUBLISHED`, `GRID_2`, `DATE_DESC`). The JSON inside
  `kindConfig` uses **camelCase** (`headingAccent`, `leftProductId`).
- **`kindConfig` is replace-only.** It's stored as a single JSON blob; updates
  overwrite the whole thing. Always re-read the current value, modify in
  memory, then send the full object back.
- **Editorial kinds need `kind` + `kindConfig`.** Feed kinds (`GRID` /
  `SLIDER` / `HORIZONTAL_SCROLL`) need `dataSource` + `filterCondition`.
  Static legacy banners need `dataType: STATIC` + `staticFilter`. Don't mix.
- **Display order:** sections render in `displayOrder` ascending. New section?
  Query existing sections first and use `max(displayOrder) + 1`. The admin UI
  appends to the bottom; matching that is good etiquette.
- **Lifecycle:** create as `DRAFT` → review → `publishSection`. Or pass
  `status: PUBLISHED` directly if there's no review step. `HIDDEN` lets you
  temporarily pull a section without deleting.
- **Images:** must be uploaded via `POST /uploads/s3` (or `/hq` for hero /
  banner). Never invent placeholder URLs — the storefront will render a
  broken image and the admin UI's editor will not let the user save it.
- **References > inline.** Category + brand kinds prefer `*Ids` arrays;
  product spotlight prefers `*ProductId`. The legacy inline fields
  (`cards`, `feature`, `wide`, `logos`, `leftFeatured`) still render for
  back-compat but new writes should reference catalog records by id.
- **`textColor` + backgrounds:** use `textColor` to keep copy legible over
  colorful backgrounds. `background.color` paints the section backdrop;
  `background.image` overrides color when its `url` is set.
- **Page values:** `"home"` for the homepage, `"products"` for the products
  page; other page names depend on shop configuration.
