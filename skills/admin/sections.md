# Sections & Designs (Storefront Layout)

Sections are the building blocks of a Riverbase storefront. They render on a
page (`"home"`, `"shop"`, etc.) in `displayOrder` order. Each section has:

- **A kind** (`SectionKind`) — the visual style / component to render. Examples:
  `hero_editorial`, `categories_grid`, `cta_banner`, `slider`. See the full list
  in the *Section Kinds* section below.
- **A status** (`draft` | `published` | `hidden`) — only `published` sections
  appear on the public storefront.
- **A data source** (`products` | `events` | `posts` | `productCategories` |
  `postCategories` | `eventCategories`) — the catalog feed for product/category
  kinds (`grid`, `slider`, `horizontal_scroll`, `categories_grid`, etc.).
- **A filter condition** — sort + filter for dynamic feeds (only used by
  data-source-driven kinds; static editorial kinds ignore this).
- **A `kindConfig`** — kind-specific content blob (heading, image, CTA, etc.).
  Shape depends on `kind`. See the *Kind Configs* section below.
- **Optional `background`** — solid color or image painted behind the section.
- **Container styling** — `padding`, `gap`, `titleAlignment`, `showTitle`,
  `layout` (`Grid1`/`Grid2`/`Grid3`/`Slider`/`HorizontalScroll`).

> **Two flows in one system.** "Section Explorer" kinds (hero, promo, FAQ, etc.)
> carry their content in `kindConfig`. The classic feed kinds (`grid`, `slider`,
> `horizontal_scroll`) read from `dataSource` + `filterCondition`. Both are
> stored as `Section` documents; the storefront component registry picks the
> right renderer per `kind`.

---

## List Sections for a Page

```graphql
query PageSections($shopId: String!, $page: String!, $statusFilter: [SectionStatus!]) {
  getPageSections(shopId: $shopId, page: $page, statusFilter: $statusFilter) {
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
  }
}
```

- `page`: `"home"`, `"shop"`, etc.
- `statusFilter`: optional list — e.g. `[draft, published]` to see drafts in the
  admin UI. Anonymous storefront callers always get only `published` regardless.

## List All Sections for a Shop

```graphql
query AllSections($shopId: String!, $statusFilter: [SectionStatus!]) {
  getSections(shopId: $shopId, statusFilter: $statusFilter) {
    id name page displayOrder status kind
  }
}
```

## Get Single Section (with nested data)

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
      data { id name price images }
      totalDocuments totalPages limit skip
    }
    events(skip: $skip) {
      data { id name date venue price images }
      totalDocuments totalPages
    }
    posts(skip: $skip) {
      data { id title images }
      totalDocuments totalPages
    }
    productCategories { data { id name } totalDocuments }
    eventCategories { data { id name } totalDocuments }
    postCategories { data { id name } totalDocuments }
  }
}
```

`kindConfig` is returned as a `JSON` scalar — its shape matches the per-kind
schema in *Kind Configs* below.

---

## Create Section

```graphql
mutation CreateSection(
  $shopId: String!
  $name: String!
  $page: String!
  $dataSource: DataSource!
  $layout: SectionLayout!
  $displayOrder: Int!
  $showTitle: Boolean!
  $dataType: DataType!
  $padding: SectionSpacing!
  $gap: SectionSpacing!
  $titleAlignment: TitleAlignment!
  $filterCondition: FilterStateInput
  $staticFilter: StaticFilterInput
  $kind: SectionKind
  $kindConfig: JSON
  $status: SectionStatus
  $background: SectionBackgroundInput
) {
  createSection(
    shopId: $shopId name: $name page: $page
    dataSource: $dataSource layout: $layout displayOrder: $displayOrder
    showTitle: $showTitle dataType: $dataType
    padding: $padding gap: $gap titleAlignment: $titleAlignment
    filterCondition: $filterCondition staticFilter: $staticFilter
    kind: $kind kindConfig: $kindConfig
    status: $status background: $background
  )
}
```

Returns the new section's `id` (String).

### Key rules

- **`kind` + `kindConfig` go together.** If you set `kindConfig`, you MUST also
  set `kind`. The server validates the config shape against the kind.
- **`kindConfig` is just the inner config object** (e.g. `{ heading: "...",
  image: {...} }`), NOT wrapped in `{ kind, config }`. The server adds the
  envelope.
- **Feed kinds (`grid`, `slider`, `horizontal_scroll`)** use `dataSource` +
  `filterCondition` (and ignore `kindConfig`). Pass `kindConfig: null`.
- **Editorial kinds** (`hero_editorial`, `promo_mosaic`, `cta_banner`, etc.)
  carry their content in `kindConfig`. `dataSource` is still required by the
  schema — pass `products` as the default for editorial kinds.
- **`status`** defaults to `published` for backward compatibility, but new
  Explorer-flow saves should pass `draft` until the admin publishes.

---

## Update Section

```graphql
mutation UpdateSection(
  $shopId: String!
  $sectionId: String!
  $name: String
  $page: String
  $dataSource: DataSource
  $layout: SectionLayout
  $displayOrder: Int
  $showTitle: Boolean
  $titleAlignment: TitleAlignment
  $dataType: DataType
  $padding: SectionSpacing
  $gap: SectionSpacing
  $filterCondition: FilterStateInput
  $staticFilter: StaticFilterInput
  $kind: SectionKind
  $kindConfig: JSON
  $status: SectionStatus
  $background: SectionBackgroundInput
) {
  updateSection(
    shopId: $shopId sectionId: $sectionId
    name: $name page: $page
    dataSource: $dataSource layout: $layout displayOrder: $displayOrder
    showTitle: $showTitle titleAlignment: $titleAlignment
    dataType: $dataType padding: $padding gap: $gap
    filterCondition: $filterCondition staticFilter: $staticFilter
    kind: $kind kindConfig: $kindConfig
    status: $status background: $background
  )
}
```

Every field except `shopId` + `sectionId` is optional; only the fields you pass
are updated.

## Publish Section

Promotes a draft to published.

```graphql
mutation PublishSection($shopId: String!, $sectionId: String!) {
  publishSection(shopId: $shopId, sectionId: $sectionId)
}
```

## Delete Section

> ⚠️ **CONFIRMATION REQUIRED** — Show section name and page before deleting.

```graphql
mutation DeleteSection($sectionId: String!, $shopId: String!) {
  deleteSection(sectionId: $sectionId, shopId: $shopId)
}
```

---

## Section Enums

```
SectionStatus:   draft | published | hidden
SectionKind:     hero_editorial | hero_immersive | hero_centered_minimal
                 | grid | slider | horizontal_scroll
                 | products_editorial_spotlight
                 | categories_grid | categories_asymmetric_bento
                 | promo_mosaic | promo_single_banner
                 | testimonials_cards | testimonials_carousel
                 | faq_accordion | brand_strip
                 | image_text | cta_banner | countdown
                 | rich_text | newsletter | video_hero
DataSource:      products | events | posts
                 | productCategories | eventCategories | postCategories
DataType:        Static | Dynamic
SectionLayout:   Grid1 | Grid2 | Grid3 | Slider | HorizontalScroll
SectionSpacing:  None | Sm | Md | Lg | Xl | Xxl
TitleAlignment:  Left | Center | Right
StaticType:      Design | Image
SortOption:      NameAsc | NameDesc | PriceAsc | PriceDesc | DateAsc | DateDesc
FilterType:      PriceRange | Categories | SubCategories | Brands | Items
                 | Promotions | None | Multi
```

---

## Background (Universal)

Every section may set an optional background — solid color, image, or both.
Render precedence: image (if `image.url` non-empty) wins over color.

```json
{
  "background": {
    "color": "#f5e6d8",
    "image": { "url": "https://cdn.riverbase.org/bg.webp", "alt": "Sand pattern" }
  }
}
```

`background` is `null` on legacy sections that pre-date this field — they render
with no backdrop.

---

## Kind Configs

> `text_color` (camelCase: `textColor`) is a hex string (e.g. `"#ffffff"`) used
> to override heading/body text color so copy stays legible over branded
> backgrounds or images. Many kinds support it on the parent, some support
> per-card overrides.

### `hero_editorial` — Split image + tri-segment headline

```json
{
  "image": { "url": "https://cdn.riverbase.org/hero.webp", "alt": "Spring collection" },
  "eyebrow": "NEW SEASON",
  "headingPre": "DEFINING THE NEW",
  "headingAccent": "ERA",
  "headingPost": "OF STYLE.",
  "subhead": "Curated drops for the modern wardrobe.",
  "primaryCta": { "label": "Shop now",   "href": "/shop", "variant": "primary" },
  "secondaryCta": { "label": "Lookbook", "href": "/posts", "variant": "outline" },
  "socialProof": {
    "avatars": [{ "url": "https://cdn.riverbase.org/a1.webp" }],
    "countLabel": "5,000+ happy customers",
    "quote": "The fit is unmatched."
  },
  "quickPicks": [
    { "image": { "url": "https://cdn.riverbase.org/p1.webp" }, "href": "/p/sku-1" }
  ],
  "textColor": "#111111"
}
```

### `hero_immersive` — Full-bleed photo + overlay copy

```json
{
  "image": { "url": "https://cdn.riverbase.org/immersive.webp" },
  "heading": "Engineered for everyday",
  "subhead": "Premium essentials, ethically made.",
  "cta": { "label": "Explore", "href": "/shop", "variant": "primary" },
  "overlayOpacity": 0.35,
  "textColor": "#ffffff"
}
```

### `hero_centered_minimal` — Headline-only hero

```json
{
  "eyebrow": "WELCOME",
  "heading": "Style starts here",
  "subhead": "Hand-picked drops every week.",
  "cta": { "label": "Browse", "href": "/shop" },
  "textColor": "#111111"
}
```

### `products_editorial_spotlight` — Two large product cards

Prefers product-ID references; legacy inline `leftFeatured` / `rightFeatured`
still render for old documents.

```json
{
  "heading": "This week's picks",
  "subhead": "Two staples we can't stop wearing.",
  "cta": { "label": "See all", "href": "/shop" },
  "leftProductId": "<product_id>",
  "rightProductId": "<product_id>",
  "leftTagOverride": "Editor's pick",
  "rightTagOverride": null,
  "leftTextColor": "#ffffff",
  "rightTextColor": "#111111"
}
```

### `categories_grid` — Uniform category cards

```json
{
  "heading": "Shop by category",
  "columns": 4,
  "categoryIds": ["<cat_id_1>", "<cat_id_2>", "<cat_id_3>", "<cat_id_4>"]
}
```

> Legacy inline `cards` (with `image`, `title`, `href`, `textColor`) still
> renders — but new writes should populate `categoryIds`.

### `categories_asymmetric_bento` — 1 feature + 1 wide + 2 small

```json
{
  "heading": "Featured categories",
  "subhead": "Pick your aesthetic.",
  "viewAllHref": "/categories",
  "featureCategoryId": "<cat_id>",
  "wideCategoryId":   "<cat_id>",
  "smallACategoryId": "<cat_id>",
  "smallBCategoryId": "<cat_id>"
}
```

Legacy inline `feature` / `wide` / `smallA` / `smallB` tiles still render.

### `promo_mosaic` — 4 irregular promo tiles

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

### `promo_single_banner` — Single wide banner

```json
{
  "image": { "url": "https://cdn.riverbase.org/banner.webp" },
  "eyebrow": "FRIENDS & FAMILY",
  "heading": "Take 20% off everything",
  "subhead": "Use code FAMILY at checkout.",
  "cta": { "label": "Shop the sale", "href": "/shop", "variant": "primary" },
  "alignment": "left",
  "textColor": "#ffffff"
}
```

### `testimonials_cards` — Grid of testimonial cards

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

### `testimonials_carousel` — Auto-rotating single quote

```json
{
  "heading": "Customer love",
  "items": [
    { "quote": "Beautiful packaging.", "author": "Maria L.", "role": "Returning customer" }
  ],
  "textColor": "#111111"
}
```

### `faq_accordion`

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

### `brand_strip` — Logo strip referencing the shop's brand catalog

```json
{
  "heading": "As featured in",
  "brandIds": ["<brand_id_1>", "<brand_id_2>", "<brand_id_3>"],
  "textColor": "#666666"
}
```

Legacy `logos` (inline `{ image, label, href }`) still renders for old documents.

### `image_text` — Side-by-side image + copy

```json
{
  "image": { "url": "https://cdn.riverbase.org/story.webp" },
  "imagePosition": "left",
  "eyebrow": "OUR STORY",
  "heading": "Built by craftspeople",
  "body": "We work with small ateliers across Cambodia.",
  "cta": { "label": "Learn more", "href": "/about" },
  "textColor": "#111111"
}
```

### `cta_banner` — Full-width banner with single CTA

```json
{
  "heading": "Join the community",
  "subhead": "Get 10% off your first order.",
  "cta": { "label": "Sign up", "href": "/signup", "variant": "primary" },
  "background": "#1f1f1f",
  "textColor": "#ffffff"
}
```

### `countdown` — Timer ticking to a target date

```json
{
  "heading": "Spring sale ends in",
  "subhead": "Final hours.",
  "targetAt": "2026-06-01T00:00:00Z",
  "cta": { "label": "Shop now", "href": "/sale" },
  "textColor": "#111111"
}
```

`targetAt` is an ISO-8601 UTC timestamp string.

### `rich_text` — Long-form HTML body

```json
{
  "html": "<h2>Our promise</h2><p>Quality, transparency, and care.</p>",
  "maxWidth": "narrow",
  "textColor": "#222222"
}
```

`maxWidth`: `"narrow"` | `"medium"` | `"wide"`.

### `newsletter`

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

### `video_hero` — Embedded or self-hosted video

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

`provider`: `"youtube"` | `"vimeo"` | `"url"`. (`video_hero` is headerless —
stack a separate text section above it if you need a title.)

---

## Feed kinds: `grid`, `slider`, `horizontal_scroll`

These render dynamic feeds. Their content comes from `dataSource` +
`filterCondition` — `kindConfig` is unused.

```json
{
  "shopId": "xxx",
  "name": "All Products",
  "page": "home",
  "kind": "grid",
  "kindConfig": null,
  "dataSource": "products",
  "dataType": "Dynamic",
  "layout": "Grid2",
  "displayOrder": 0,
  "showTitle": true,
  "titleAlignment": "Left",
  "padding": "Sm",
  "gap": "Sm",
  "status": "published",
  "filterCondition": {
    "priceRange": { "from": 0, "to": 0 },
    "sortBy": "DateDesc",
    "filterBy": "None",
    "categories": [],
    "subCategories": [],
    "brands": [],
    "items": [],
    "limit": 20
  }
}
```

Set `dataSource` to `events`, `posts`, `productCategories`, `postCategories`, or
`eventCategories` to feed other catalogs.

---

## Static-image / Design banner (legacy)

Pre-Explorer-flow way to render a single image or canvas design. Still
supported — pass `dataType: "Static"` and a `staticFilter`. Newer kinds like
`promo_single_banner` / `hero_immersive` are preferred for new work.

```json
{
  "shopId": "xxx",
  "name": "Hero Banner",
  "page": "home",
  "kind": "grid",
  "kindConfig": null,
  "dataSource": "products",
  "dataType": "Static",
  "layout": "Grid1",
  "displayOrder": 0,
  "showTitle": false,
  "titleAlignment": "Left",
  "padding": "None",
  "gap": "None",
  "status": "published",
  "staticFilter": {
    "staticType": "Image",
    "images": ["https://cdn.riverbase.org/your-uploaded-image.webp"]
  }
}
```

For a canvas design:

```json
{
  "staticFilter": {
    "staticType": "Design",
    "designId": "<design_id>"
  }
}
```

---

# Designs (Canvas Editor)

Designs are visual layouts created in the canvas editor. They can be referenced
by static-flow sections via `staticFilter.designId`.

## List Designs

```graphql
query DesignsByShop($shopId: String!) {
  getDesignsByShop(shopId: $shopId) {
    id shopId name output layers
    canvasSize { width height }
    createdAt
  }
}
```

## Get Design

```graphql
query DesignById($designId: String!) {
  getDesign(designId: $designId) {
    id shopId name output layers
    canvasSize { width height }
    createdAt
  }
}
```

## Create Design

```graphql
mutation CreateDesign($shopId: String!, $payload: NewDesignInput!) {
  createDesign(shopId: $shopId, payload: $payload) { id name output }
}
```

### NewDesignInput

```json
{
  "name": "Promo Banner",
  "layers": {},
  "output": "https://cdn.riverbase.org/rendered-design.webp",
  "canvasSize": { "width": 360, "height": 450 }
}
```

- `layers`: JSON object representing canvas editor layer data
- `output`: Pre-rendered image URL (upload via `/uploads/s3`)
- `canvasSize`: Canvas dimensions in pixels (default 360x450)

## Update Design

```graphql
mutation UpdateDesign($shopId: String!, $designId: String!, $payload: NewDesignInput!) {
  updateDesign(shopId: $shopId, designId: $designId, payload: $payload) { id name output }
}
```

## Delete Design

> ⚠️ **CONFIRMATION REQUIRED** — Show design name and warn that sections
> referencing it will break. Wait for explicit "Yes".

```graphql
mutation DeleteDesign($designId: String!, $shopId: String!) {
  deleteDesign(shopId: $shopId, designId: $designId)
}
```

---

## Agent Notes

- **Pick the right kind first.** Match the user's intent to a `SectionKind`
  before writing the mutation. "Hero with image" → `hero_immersive`. "Categories
  in a row" → `categories_grid`. "Customer quotes" → `testimonials_cards`.
- **Editorial kinds need `kind` + `kindConfig`.** Feed kinds (`grid`, `slider`,
  `horizontal_scroll`) need `dataSource` + `filterCondition`. Static legacy
  banners need `dataType: Static` + `staticFilter`. Don't mix.
- **Display order**: Sections render in `displayOrder` ascending. New section?
  Query existing sections first and use `max(displayOrder) + 1`.
- **Workflow**: Create as `draft` → review → `publishSection` (or pass
  `status: published` directly if no review step). `hidden` lets you
  temporarily pull a section without deleting.
- **Images** must be uploaded CDN URLs via `POST /uploads/s3` (or `/hq` for
  higher-res). Never use placeholder URLs.
- **References > inline.** Category and brand kinds prefer `*Ids` arrays;
  product spotlight prefers `*ProductId`. The legacy inline fields still
  render for back-compat but new writes should reference catalog records.
- **textColor / backgrounds.** Use `textColor` to keep copy legible over
  branded backgrounds. `background.color` paints the section backdrop;
  `background.image` overrides color when its url is set.
- **`page` values**: usually `"home"` for the homepage. Other pages exist
  depending on shop configuration.
