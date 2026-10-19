# Sections & Designs (Storefront Layout)

Sections are the building blocks of a Riverbase storefront page. Each section
belongs to one `page` and renders in `displayOrder` ascending. The admin builds
them through **Section Explorer** (`/sections/explorer?page=home`) — a gallery
grouped Hero / Products / Categories / Promo / Trust / Content — plus a
**Build custom** button for feed-driven product / post / event lists.

An agent can do everything the admin UI does — pick a kind, write its content,
upload images and video, set a background, order and publish — with
`createSection` / `updateSection`.

## Where sections render

| `page` value | Storefront route | Notes |
|---|---|---|
| `"home"` | `/` | The homepage **is** its sections, in order. No sections → shoppers see an empty page. |
| `"products"` | `/products` | The products page is also built only from sections. |
| `"posts"` | `/posts` | Usually a `POSTS` feed. |
| `"events"` | `/events` | Usually an `EVENTS` feed. |

`page` is a free string, but **only these four are rendered**. Sections cannot
be placed on custom pages (`/<slug>`, see `admin/pages.md`), on the About page,
or on category / product pages — a section saved with any other `page` value
is stored and never shown.

## The two section families

| Family | Kinds | Content comes from |
|---|---|---|
| **Explorer (designed) kinds** | `HERO_*`, `PRODUCTS_EDITORIAL_SPOTLIGHT`, `CATEGORIES_*`, `PROMO_*`, `TESTIMONIALS_*`, `FAQ_ACCORDION`, `BRAND_STRIP`, `IMAGE_TEXT`, `CTA_BANNER`, `COUNTDOWN`, `RICH_TEXT`, `NEWSLETTER`, `VIDEO_HERO`, `VIDEO_REELS_MARQUEE`, `VIDEO_REELS_DECK` | `kind` + `kindConfig` (JSON you write) |
| **Feed kinds** | `GRID`, `SLIDER`, `HORIZONTAL_SCROLL` | `layout` + `dataSource` + `filterCondition` (live catalog data). `kindConfig` must be `null`. |

Plus the legacy **static banner** (`dataType: STATIC` + `staticFilter`) — see the
end of this file.

---

## ⚠️ Rules that silently break sections

1. **`kindConfig` needs `kind` in the same call.** `createSection` /
   `updateSection` reject a `kindConfig` sent without `kind`
   (`"kindConfig requires kind to be provided alongside it"`). When editing
   content, always send both.
2. **`kindConfig` is replace-only.** It is stored whole; `updateSection`
   overwrites it. `getSection` → change the object in memory → send the
   **full** object back.
3. **Unknown fields are dropped without an error.** The server parses
   `kindConfig` into a typed struct per kind and re-serialises it; any key not
   listed in this file disappears on save. An unknown **enum value** (e.g.
   `"provider": "instagram"`) is the opposite: the whole save fails with
   `Invalid kindConfig payload`.
4. **Categories / brand strip render the INLINE shape only.** The backend
   accepts `categoryIds`, `featureCategoryId`…, `brandIds`, but the storefront
   on this version never reads them — a section written with ids only renders
   an empty grid. Write the inline `cards` / `feature`… / `logos` objects
   (recipes below show how to build them from the catalog).
5. **Product spotlight by id works on `page: "home"` only.** Only the homepage
   loads the products that `leftProductId` / `rightProductId` point at; on
   other pages the slots render empty.
6. **`filterBy: MULTI` does not filter a section feed.** The section product
   query treats `MULTI` as "no filter" and shows the whole catalog. Use one
   filter type per section (`CATEGORIES`, `ITEMS`, `BRANDS`, …).
7. **Feed kinds never take `kindConfig`.** `GRID` / `SLIDER` /
   `HORIZONTAL_SCROLL` have no config struct; sending one fails the save.
8. **Fields can't be cleared with `null`.** `updateSection` skips every null
   argument. To remove a background send `background: { color: null, image: null }`.
9. **Text is not translated.** Every string in `kindConfig` and the section
   `name` shows as written in every storefront language. Write in the language
   the shop's customers read (often Khmer), or keep bilingual copy short.

---

## Uploads (images and video)

Every image / video field takes a real CDN URL returned by the upload endpoint —
never a placeholder, a local path, or a hotlinked third-party image.

| Endpoint | Images | Video |
|---|---|---|
| `POST <API_URL>/uploads/s3` | Re-encoded to WebP, max **2400px** wide (q92). Logos, avatars, category tiles, posters. | Stored as-is, max 99 MB. |
| `POST <API_URL>/uploads/s3/hq` | Re-encoded to WebP, max **3840px** wide (q95). Heroes, banners, promo tiles, backgrounds. | Stored as-is, max 99 MB — **use this one for video** (the admin does). |

`<API_URL>` is the GraphQL URL without `/graphql` — `$RIVERBASE_API_URL` ends in
`/graphql`, so strip it. Multipart field name `file`, `Authorization: <token>`
header. The response body is the plain CDN URL string (`https://<cdn>/<uuid>.<ext>`).

```bash
API_BASE="${RIVERBASE_API_URL%/graphql}"

# Image — curl only sends image/jpeg|png|gif automatically; name the type for webp
curl -X POST -H "Authorization: $RIVERBASE_API_KEY" \
  -F "file=@./hero.jpg;type=image/jpeg" "$API_BASE/uploads/s3/hq"
# → "https://<cdn>/<uuid>.webp"

# Video — ALWAYS pass ;type=video/mp4, otherwise it is stored as a generic
# file with no size cap and may not play in Safari
curl -X POST -H "Authorization: $RIVERBASE_API_KEY" \
  -F "file=@./clip.mp4;type=video/mp4" "$API_BASE/uploads/s3/hq"
```

Video is never transcoded: what you upload is what every shopper downloads.
Export H.264 MP4 — 720×1280 for reels, ≤1080p for a hero. The admin caps a
reel clip at 99 MB / 5 min and a `VIDEO_HERO` clip at 63 MB / 5 min; stay well
under (a hero with `autoplay` downloads in full for every visitor). Prefer a
YouTube / TikTok / Vimeo / Facebook link when the video is already there.

### Image shapes the storefront crops to

| Slot | Crop |
|---|---|
| `HERO_IMMERSIVE`, `VIDEO_HERO`, static banner | wide: 4:3 on phones, 16:9 from 640px (static banner 16:9 → 21:9) |
| `HERO_EDITORIAL` image, `IMAGE_TEXT` image | portrait: 4:3 on phones, 4:5 from 640px |
| `CATEGORIES_GRID` card, product spotlight, `PROMO_MOSAIC` topA/topB | square 1:1 |
| `PROMO_MOSAIC` tall / wide, bento wide | 16:9 |
| Reel posters / clips | vertical 9:16 at every size |
| `BRAND_STRIP` logo | shown 28–32px tall, any width — use a transparent PNG |

Everything is `object-cover`: keep the subject centred so the phone crop keeps it.

---

## GraphQL Wire-Format Enums (SCREAMING_SNAKE_CASE)

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
              | VIDEO_REELS_MARQUEE | VIDEO_REELS_DECK
SectionStatus:  DRAFT | PUBLISHED | HIDDEN
SectionLayout:  GRID_1 | GRID_2 | GRID_3 | GRID_4 | GRID_5 | SLIDER | HORIZONTAL_SCROLL
SectionSpacing: NONE | SM | MD | LG | XL | XXL
TitleAlignment: LEFT | CENTER | RIGHT
DataType:       STATIC | DYNAMIC
DataSource:     PRODUCTS | EVENTS | POSTS
              | PRODUCT_CATEGORIES | EVENT_CATEGORIES | POST_CATEGORIES
StaticType:     IMAGE | DESIGN
SortOption:     NAME_ASC | NAME_DESC | PRICE_ASC | PRICE_DESC | DATE_ASC | DATE_DESC
              | MANUAL          # "As arranged": the order of filterCondition.items
FilterType:     PRICE_RANGE | CATEGORIES | SUB_CATEGORIES | BRANDS
              | ITEMS | PROMOTIONS | NONE | MULTI
```

Inside `kindConfig` keys are **camelCase** and enum values are **lowercase**:
```
CtaVariant:        primary | outline | link
BannerAlignment:   left | center | right          (PROMO_SINGLE_BANNER.alignment)
ImageTextPosition: left | right                   (IMAGE_TEXT.imagePosition)
RichTextWidth:     narrow | medium | wide         (RICH_TEXT.maxWidth)
VideoProvider:     youtube | vimeo | url          (VIDEO_HERO.provider)
VideoReelProvider: url | youtube | tiktok | vimeo | facebook
ReelSpeed:         slow | normal | fast           (VIDEO_REELS_MARQUEE.speed)
```

---

## Queries

### Sections on a page (in render order)
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
      sortBy filterBy categories subCategories brands items limit
    }
    staticFilter { designId images staticType }
  }
}
```
- Pass `statusFilter: [DRAFT, PUBLISHED, HIDDEN]` (or omit it) as shop staff to
  see everything. Anonymous callers — including the storefront itself, even for
  the owner — always get `PUBLISHED` only.

### Every section of the shop
```graphql
query AllSections($shopId: String!, $statusFilter: [SectionStatus!]) {
  getSections(shopId: $shopId, statusFilter: $statusFilter) {
    id name page displayOrder status kind
  }
}
```

### One section, with the data a feed resolves to
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
      sortBy filterBy categories subCategories brands items limit
    }
    staticFilter { designId images staticType }
    products(skip: $skip, locationId: $locationId) {
      data { id name price images } totalDocuments totalPages limit skip
    }
    events(skip: $skip)  { data { id name date venue price images } totalDocuments }
    posts(skip: $skip)   { data { id title images } totalDocuments }
    productCategories { data { id name logo } totalDocuments }
  }
}
```
Only the field matching `dataSource` returns data; the others are `null`.
`skip` is a **page number** (0-based), not an offset.

---

## Mutations

Permissions: `SectionCreate` / `SectionEdit` / `SectionDelete` (owner, or staff
whose role has them). Variable types below match the server schema — keep the
`!`s.

### Create → returns the new section id (`String`)
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
  $titleAlignment: TitleAlignment!
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
- `status` defaults to `PUBLISHED` when omitted.
- `kind` may be omitted for feed sections — it is then inferred from `layout`
  (`GRID_n` → `GRID`, `SLIDER` → `SLIDER`, `HORIZONTAL_SCROLL` → `HORIZONTAL_SCROLL`).

### Update → `Boolean`
Every argument except `shopId` + `sectionId` is optional; only non-null ones
are written (a sparse `$set`).
```graphql
mutation UpdateSection(
  $shopId: String!, $sectionId: String!,
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
    shopId: $shopId sectionId: $sectionId
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

### Publish (sets status to `PUBLISHED`)
```graphql
mutation PublishSection($shopId: String!, $sectionId: String!) {
  publishSection(shopId: $shopId, sectionId: $sectionId)
}
```
To hide a section without deleting it: `updateSection(..., status: HIDDEN)`.

### Reorder
There is no bulk reorder mutation. Send one `updateSection` per section with
**only** `displayOrder` (0, 1, 2, … in the new order) — this is exactly what the
admin's `/sections/reorder` screen does. Don't resend other fields.

### Delete
> ⚠️ **CONFIRMATION REQUIRED** — show the section's name, kind and page first.
```graphql
mutation DeleteSection($shopId: String!, $sectionId: String!) {
  deleteSection(shopId: $shopId, sectionId: $sectionId)
}
```

---

## Universal section fields

| Field | What it really does |
|---|---|
| `name` | Required. Admin label **and the visible heading when `showTitle: true`** (rendered as an `h2` above the section). |
| `showTitle` + `titleAlignment` | Heading bar above the section, built from `name`. Explorer kinds carry their own `heading` — keep `showTitle: false` for them. Feed sections usually want `showTitle: true` with a short `name` ("New arrivals"). |
| `padding` | **Horizontal** padding of the section (`NONE` = edge to edge … `XXL` = widest margins). Vertical spacing between sections is fixed. |
| `gap` | Space between items in grids, sliders and tile layouts (`MD` default). |
| `background` | `{ color: "#hex", image: { url, alt } }`. Image paints over the colour. On explorer kinds, setting a background turns the section into a rounded, padded panel and replaces the kind's default backdrop (e.g. the brand-colour fill of `COUNTDOWN`). `VIDEO_HERO` ignores it. |
| `status` | `DRAFT` and `HIDDEN` never reach the storefront; `PUBLISHED` does. |
| `displayOrder` | Render order, ascending. For a new section use `max(displayOrder) + 1` of that page (admin behaviour). |
| `dataType` | `STATIC` for explorer kinds and static banners; `DYNAMIC` for feeds. |

The admin's envelope for every explorer kind — copy it:
```json
{ "dataSource": "PRODUCTS", "dataType": "STATIC", "layout": "GRID_1",
  "showTitle": false, "titleAlignment": "LEFT", "padding": "MD", "gap": "MD",
  "filterCondition": null, "staticFilter": null, "background": null }
```

---

## Links (`href` / `cta.href` / `viewAllHref`)

Write **shop-relative paths**; the storefront prefixes the shop's locale /
tenant path itself. Use only routes that exist:

| Target | `href` |
|---|---|
| Home | `/` |
| All products | `/products` |
| One product | `/products/<productId>` |
| A category | `/search?source=PRODUCTS&category=<categoryId>` |
| A brand | `/search?brand=<brandId>` |
| A keyword search | `/search?keyword=<text>` |
| Products on sale | `/promotions` |
| Blog / events / about | `/posts`, `/events`, `/about` |
| A custom page | `/<page-slug>` (see `admin/pages.md`) |
| External | `https://…` (opens as-is) |

There is **no** `/shop`, `/sale`, `/categories`, `/collections`, `/c/…`, `/p/…`
or `/cart` route — those 404. A CTA with a `label` but no `href` links to `/`;
a CTA without a `label` is not rendered.

---

## Picking a kind

| Explorer group | Kind (style slug) | Use for |
|---|---|---|
| **Hero** | `HERO_EDITORIAL` (`editorial`) | Split hero: portrait photo + big 3-part headline, two CTAs, social proof, quick-pick thumbnails. Fashion / lifestyle. |
| Hero | `HERO_IMMERSIVE` (`immersive-banner`) | Full-width photo with centred copy over a dark scrim. Any shop with one strong image. |
| Hero | `HERO_CENTERED_MINIMAL` (`centered-minimal`) | Text-only hero. No photo yet, or a quiet premium brand. |
| **Products** | `GRID` (`grid-3col`, `grid-2col`) | Product / post / event / category grid, 1–5 columns. |
| Products | `SLIDER` (`slider`) | Auto-rotating carousel (3 s, loops). |
| Products | `HORIZONTAL_SCROLL` (`horizontal-scroll`) | Swipeable row; category sources become a chip row. |
| Products | `PRODUCTS_EDITORIAL_SPOTLIGHT` (`editorial-spotlight`) | Two big product cards + copy. Homepage only. |
| **Categories** | `CATEGORIES_GRID` (`standard`) | Square category tiles, 2 per row on phones, 3 or 4 on desktop. |
| Categories | `CATEGORIES_ASYMMETRIC_BENTO` (`asymmetric-bento`) | 4 category tiles in a bento (1 tall, 1 wide, 2 small). |
| **Promo** | `PROMO_MOSAIC` (`mosaic`) | 4 promo tiles, each with its own copy, link and colour. |
| Promo | `PROMO_SINGLE_BANNER` (`single-banner`) | One wide promo banner with a CTA. |
| **Trust** | `TESTIMONIALS_CARDS` (`cards`) | Grid of quotes with avatar and stars. |
| Trust | `TESTIMONIALS_CAROUSEL` (`carousel`) | One quote at a time, auto-rotating (5 s) on the brand colour. |
| Trust | `FAQ_ACCORDION` (`accordion`) | Questions that expand. |
| Trust | `BRAND_STRIP` (`logos`) | Row of brand / partner logos. |
| **Content** | `IMAGE_TEXT` (`image-left`) | Photo beside a story paragraph + CTA. |
| Content | `CTA_BANNER` (`default`) | Full-width call to action on a solid colour. |
| Content | `COUNTDOWN` (`default`) | Sale / launch timer on the brand colour. |
| Content | `RICH_TEXT` (`default`) | Long-form HTML (story, policy, founder note). |
| Content | `NEWSLETTER` (`default`) | Email form — only renders with a `submitHref` (see kind). |
| Content | `VIDEO_HERO` (`default`) | One landscape video, embedded or uploaded. |
| Content | `VIDEO_REELS_MARQUEE` (`marquee`) | Row of vertical clips that drifts sideways; tap opens a player. |
| Content | `VIDEO_REELS_DECK` (`stacked`) | The same clips as a swipeable card stack. |

---

## Per-kind `kindConfig` schemas

Every field is optional. Shared shapes:
```jsonc
// MediaRef (image)
{ "url": "https://<cdn>/x.webp", "alt": "Describe the photo" }
// CtaConfig — rendered only when label is set; variant defaults to primary
{ "label": "Shop now", "href": "/products", "variant": "primary" }   // primary | outline | link
```
`textColor` (hex) sets the copy colour. Where a kind lists a default, that is
what renders when you leave it out. Buttons always use the theme's primary
colour (see `admin/appearance.md`).

### `HERO_EDITORIAL`
```json
{
  "image": { "url": "https://<cdn>/hero.webp", "alt": "Spring collection" },
  "eyebrow": "NEW SEASON",
  "headingPre": "Defining the new",
  "headingAccent": "era",
  "headingPost": "of style.",
  "uppercaseHeading": true,
  "subhead": "Curated drops for the modern wardrobe.",
  "primaryCta":   { "label": "Shop the drop", "href": "/products", "variant": "primary" },
  "secondaryCta": { "label": "Lookbook", "href": "/posts", "variant": "outline" },
  "socialProof": {
    "avatars": [{ "url": "https://<cdn>/a1.webp", "alt": "Customer" }],
    "countLabel": "5,000+ happy customers",
    "quote": "The fit is unmatched."
  },
  "quickPicks": [
    { "image": { "url": "https://<cdn>/p1.webp" }, "href": "/products/<productId>" }
  ],
  "textColor": "#111111"
}
```
- `headingAccent` renders in the theme's primary colour; pre/accent/post read as one line.
- `uppercaseHeading` absent = **uppercase**. Set `false` for Khmer, accented
  names or deliberate mixed case.
- `quickPicks`: small round product thumbnails (3–5 is right) followed by a
  "view all" link to `/products`.
- `socialProof` floats over the photo from 640px up, and sits under the copy on phones.

### `HERO_IMMERSIVE`
```json
{
  "image": { "url": "https://<cdn>/hero.webp", "alt": "Hero photo" },
  "heading": "Engineered for everyday",
  "subhead": "Premium essentials, ethically made.",
  "cta": { "label": "Explore", "href": "/products", "variant": "primary" },
  "overlayOpacity": 0.35,
  "textColor": "#ffffff"
}
```
`overlayOpacity` 0–1 (black scrim). **Omitted = 0 (no scrim)** — set 0.3–0.5
for white text on a busy photo. `textColor` defaults to white.

### `HERO_CENTERED_MINIMAL`
```json
{
  "eyebrow": "WELCOME",
  "heading": "Quietly considered.",
  "subhead": "Pieces designed to last beyond the season.",
  "cta": { "label": "Shop now", "href": "/products" },
  "textColor": "#111111"
}
```
No image field. For colour, give the section a `background`.

### `PRODUCTS_EDITORIAL_SPOTLIGHT` (homepage only)
```json
{
  "heading": "This week's picks",
  "subhead": "Two staples we can't stop wearing.",
  "cta": { "label": "See all", "href": "/products" },
  "leftProductId":  "<productId>",
  "rightProductId": "<productId>",
  "leftTagOverride":  "Editor's pick",
  "rightTagOverride": null,
  "leftTextColor":  "#ffffff",
  "rightTextColor": "#ffffff"
}
```
Each card shows the product's first photo (square), name and price, linking to
the product. Text sits on the photo; defaults to white. A deleted product
empties its slot; both empty hides the section. Resolve product names → ids
first (`catalog/products.md`).

### `CATEGORIES_GRID`
```json
{
  "heading": "Shop by category",
  "columns": 4,
  "cards": [
    {
      "title": "Skincare",
      "image": { "url": "https://<cdn>/skincare.webp", "alt": "Skincare" },
      "href": "/search?source=PRODUCTS&category=<categoryId>",
      "textColor": "#ffffff"
    }
  ]
}
```
- `columns`: `3` or `4` on desktop (default 4); phones always show 2.
- Title sits at the bottom of the square image on a dark gradient, uppercase,
  max 2 lines. A card without `href` is not clickable.
- Build `cards` from `getCategoriesByShop(shopId) { id name logo }`; when a
  category has no `logo`, upload a photo for it. (`categoryIds` is accepted but
  **not rendered** — see the rules at the top.)

### `CATEGORIES_ASYMMETRIC_BENTO`
```json
{
  "heading": "Featured categories",
  "subhead": "Pick your aesthetic.",
  "viewAllHref": "/products",
  "feature": { "title": "Linen", "description": "Breathable staples", "image": { "url": "…" }, "href": "/search?source=PRODUCTS&category=<id>", "textColor": "#ffffff" },
  "wide":    { "title": "Knits",       "image": { "url": "…" }, "href": "/search?source=PRODUCTS&category=<id>" },
  "smallA":  { "title": "Outerwear",   "image": { "url": "…" }, "href": "/search?source=PRODUCTS&category=<id>" },
  "smallB":  { "title": "Accessories", "image": { "url": "…" }, "href": "/search?source=PRODUCTS&category=<id>" }
}
```
`feature` is the tall tile (put the strongest photo there); `wide` is 16:9 on
desktop; small tiles are square. Text defaults to white.
(`featureCategoryId` / `wideCategoryId` / `smallACategoryId` / `smallBCategoryId`
are accepted but not rendered.)

### `PROMO_MOSAIC`
```json
{
  "tall": {
    "image": { "url": "https://<cdn>/linen.webp" },
    "eyebrow": "Up to 40% off",
    "title": "Linen capsule",
    "subtitle": "Breathable for the hot season.",
    "cta": { "label": "Shop", "href": "/search?source=PRODUCTS&category=<id>" },
    "background": "#0f0f0f",
    "textColor": "#ffffff"
  },
  "topA": { "title": "New arrivals", "cta": { "label": "Browse", "href": "/products" }, "background": "#f7d4cb" },
  "topB": { "title": "Best sellers", "cta": { "label": "Browse", "href": "/products" }, "background": "#dde6d4" },
  "wide": { "title": "Gift sets", "subtitle": "Ready to give.", "cta": { "label": "Shop", "href": "/promotions" }, "background": "#f5b6b3" }
}
```
- The whole tile links to `cta.href`; the label shows as underlined text ("Shop »").
  No `href` → the tile is not clickable.
- `background` is any CSS colour, painted under the image (image at 90% opacity).
  Text defaults to white — set a dark `textColor` on pale tiles.
- `headerLabel` / `headerHref` are accepted but not rendered.

### `PROMO_SINGLE_BANNER`
```json
{
  "image": { "url": "https://<cdn>/banner.webp" },
  "eyebrow": "FRIENDS & FAMILY",
  "heading": "Take 20% off everything",
  "subhead": "Use code FAMILY at checkout.",
  "cta": { "label": "Shop the sale", "href": "/promotions", "variant": "primary" },
  "alignment": "left",
  "textColor": "#ffffff"
}
```
`alignment`: `left` (default) | `center` | `right`. With no image and no
section background it fills with the theme primary colour; an image gets a 30%
black scrim. Text defaults to white.

### `TESTIMONIALS_CARDS`
```json
{
  "heading": "Loved by our customers",
  "subhead": "What people are saying",
  "columns": 3,
  "items": [
    { "quote": "Best fit I've ever owned.", "author": "Alex P.", "role": "Verified buyer",
      "avatar": { "url": "https://<cdn>/alex.webp" }, "rating": 5 }
  ],
  "textColor": "#111111"
}
```
`columns`: `2` or `3` (default 3, one column on phones). `rating` 0–5 stars.

### `TESTIMONIALS_CAROUSEL`
```json
{
  "heading": "Customer love",
  "items": [{ "quote": "Beautiful packaging.", "author": "Maria L.", "role": "Returning customer" }],
  "textColor": "#ffffff"
}
```
One quote at a time, rotating every 5 s, on the brand colour unless the section
has a background. Items take only quote / author / role.

### `FAQ_ACCORDION`
```json
{
  "heading": "Frequently asked",
  "subhead": "Shipping, returns and sizing.",
  "items": [{ "question": "How long does delivery take?", "answer": "Same day in Phnom Penh." }],
  "visibleCount": 5,
  "textColor": "#111111"
}
```
`visibleCount`: questions shown before a "Show all" toggle; `0` / absent = all.

### `BRAND_STRIP`
```json
{
  "heading": "Brands we carry",
  "logos": [
    { "image": { "url": "https://<cdn>/brand-a.png" }, "label": "Brand A", "href": "/search?brand=<brandId>" },
    { "label": "Brand B" }
  ],
  "textColor": "#666666"
}
```
The heading renders as a small centred eyebrow. A logo without an image shows
its `label` as text. Build from `getBrandsByShop(shopId) { id name logo }`.
(`brandIds` is accepted but not rendered.)

### `IMAGE_TEXT`
```json
{
  "image": { "url": "https://<cdn>/workshop.webp" },
  "imagePosition": "left",
  "eyebrow": "OUR STORY",
  "heading": "Built by craftspeople",
  "body": "We work with small ateliers across Cambodia.",
  "cta": { "label": "Learn more", "href": "/about" },
  "textColor": "#111111"
}
```
`imagePosition`: `left` (default) | `right` — desktop only; phones stack image
first. Alternate left/right when stacking several.

### `CTA_BANNER`
```json
{
  "heading": "Join the inner circle",
  "subhead": "Early access to drops and members-only pricing.",
  "cta": { "label": "Become a member", "href": "/vip", "variant": "primary" },
  "background": "#1a1a1a",
  "textColor": "#ffffff"
}
```
`background` (any CSS colour) is used only when the section has no `background`.
Text defaults to white.

### `COUNTDOWN`
```json
{
  "heading": "Spring sale ends in",
  "subhead": "Final hours.",
  "targetAt": "2026-12-31T16:59:00Z",
  "cta": { "label": "Shop now", "href": "/promotions" },
  "textColor": "#ffffff"
}
```
`targetAt`: ISO-8601 UTC (Phnom Penh is UTC+7: midnight 1 Jan local =
`…-12-31T17:00:00Z`). Ticks days/hours/minutes/seconds on the brand colour;
once past, it shows an "ended" label — unpublish or update it after the sale.

### `RICH_TEXT`
```json
{
  "html": "<h2>Our promise</h2><p>Quality, transparency, and care.</p>",
  "maxWidth": "medium",
  "textColor": "#222222"
}
```
`maxWidth`: `narrow` | `medium` (default) | `wide`. Rendered with typographic
(`prose`) styles after DOMPurify sanitising — scripts, iframes and event
handlers are removed; headings, lists, links, images, tables survive.

### `NEWSLETTER`
```json
{
  "heading": "Get the drop",
  "subhead": "Weekly newsletter, no spam.",
  "placeholder": "your@email.com",
  "ctaLabel": "Subscribe",
  "submitHref": "https://forms.example.com/xyz",
  "textColor": "#111111"
}
```
Riverbase has **no subscribe endpoint**. The form only appears when
`submitHref` is set, and it does a plain HTML POST there — point it at an
external form service the merchant owns, or don't add this section. For
Telegram-first shops, a `CTA_BANNER` to the shop's Telegram is usually better.

### `VIDEO_HERO`
One landscape video, no copy (stack a text section above if needed).
```json
{
  "provider": "youtube",
  "sourceUrl": "https://www.youtube.com/watch?v=<11-char-id>",
  "posterImage": { "url": "https://<cdn>/poster.webp" },
  "autoplay": true,
  "loop": true,
  "muted": true,
  "showControls": false
}
```
- `provider`: `youtube` (`watch?v=`, `youtu.be/` or `/embed/` links — **not**
  `/shorts/`), `vimeo` (`vimeo.com/<id>`), or `url` (an uploaded / direct MP4).
- Browsers only autoplay muted video: pair `autoplay` with `muted`.
- `posterImage` shows before playback (`url` provider).

### `VIDEO_REELS_MARQUEE` and `VIDEO_REELS_DECK`
Short vertical clips (9:16). Both kinds share one item shape, so switching
style keeps the content.
```json
{
  "heading": "See it in action",
  "subhead": "Three steps, one minute.",
  "speed": "normal",
  "items": [
    {
      "provider": "tiktok",
      "sourceUrl": "https://www.tiktok.com/@shop/video/7300000000000000000",
      "posterImage": { "url": "https://<cdn>/cover1.webp", "alt": "Unboxing" },
      "badge": "1",
      "title": "Unbox",
      "description": "Open the carton from the top.",
      "cta": { "label": "Shop now", "href": "/products/<productId>", "variant": "primary" }
    }
  ]
}
```
- **Marquee** shows poster, badge, title and CTA; tapping a tile opens the
  player. `speed` `slow|normal|fast` makes the row drift right-to-left (pauses
  on hover / focus) — but only with **4 or more** items; with no `speed` it is
  a still, swipeable row.
- **Deck** (`VIDEO_REELS_DECK`) has no `speed`; it shows badge, title and
  `description` on the centre card and ignores `cta`.
- Up to **8** items (admin limit).
- `provider` + `sourceUrl` must be the canonical link that carries the video id:

  | provider | Accepted `sourceUrl` |
  |---|---|
  | `youtube` | `youtube.com/shorts/<id>`, `watch?v=<id>`, `youtu.be/<id>`, `/embed/<id>`, `/live/<id>` |
  | `tiktok` | `tiktok.com/@user/video/<numericId>` (also `/v/`, `/embed/v2/`, `/player/v1/`) |
  | `vimeo` | `vimeo.com/<id>` (plus `/<hash>` or `?h=` for unlisted) |
  | `facebook` | `facebook.com/…/videos/<id>`, `facebook.com/reel/<id>`, `watch?v=<id>` |
  | `url` | an uploaded clip (`/uploads/s3/hq`, `type=video/mp4`) or a direct `.mp4` / `.webm` |

  Short share links (`vm.tiktok.com/…`, `vt.tiktok.com/…`, `tiktok.com/t/…`,
  `fb.watch/…`, `facebook.com/share/v/…`) are resolved only by the admin
  editor in a browser — through the API they render a tile that cannot play.
  Open the short link, take the final URL, and store that.
- `posterImage`: required for `url` clips (otherwise the tile is black).
  YouTube covers are filled in automatically; TikTok / Vimeo / Facebook covers
  are fetched on view, but uploading your own cover is faster and stable.
- TikTok sometimes refuses a video on the first load in a fresh browser and
  plays it on the next; the shop retries once automatically. If a clip still
  won't play, re-upload it as a `url` clip.

---

## Feed kinds: `GRID` / `SLIDER` / `HORIZONTAL_SCROLL`

Live lists from the catalog. `kindConfig: null`, `dataType: DYNAMIC`.

```json
{
  "shopId": "...",
  "name": "New arrivals",
  "page": "home",
  "kind": "GRID",
  "kindConfig": null,
  "dataSource": "PRODUCTS",
  "dataType": "DYNAMIC",
  "layout": "GRID_4",
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
  },
  "background": null
}
```

**`layout`** decides the look (`kind` should match it):

| `layout` | Phones | Tablets (≥768px) | Desktop (≥1024px) |
|---|---|---|---|
| `GRID_1` | 1 | 1 | 1 |
| `GRID_2` | 2 | 2 | 2 |
| `GRID_3` | 2 | 3 | 3 |
| `GRID_4` | 2 | 3 | 4 |
| `GRID_5` | 2 | 3 | 5 |
| `SLIDER` | auto-advancing carousel, loops every 3 s | | |
| `HORIZONTAL_SCROLL` | 2½ cards, swipe | 2½ cards | 4 cards + a peek of the 5th, ‹ › arrows, mouse-wheel scrolls the row |

Grids load more as the shopper scrolls. Product cards use the shop's card style
from Appearance (`admin/appearance.md`). Category sources link each item to its
search page; in `HORIZONTAL_SCROLL` they render as chips.

**`filterCondition`** — all eight keys are required whenever you send it:

| `filterBy` | Shows | Fill in |
|---|---|---|
| `NONE` | everything from `dataSource` | — |
| `CATEGORIES` | products in any of these categories | `categories: [ids]` |
| `SUB_CATEGORIES` | products in these subcategories | `subCategories: [ids]` |
| `BRANDS` | products of these brands | `brands: [ids]` |
| `ITEMS` | exactly these products | `items: [productIds]` |
| `PRICE_RANGE` | `from` ≤ price ≤ `to` | `priceRange` — **both** bounds; `{0,0}` here means "free only" |
| `PROMOTIONS` | products with a live public or membership discount (not coupon-only); the header gets a "See More" link to `/promotions` | — |
| `MULTI` | ⚠️ not filtered in sections — avoid | — |

- `sortBy: MANUAL` ("As arranged") with `filterBy: ITEMS` shows the products in
  the order of `items` — use it for a hand-picked, hand-ordered row. With any
  other filter it falls back to the default order.
- `limit` is the page size (1–10 000; 8–12 is typical).
- Only on-sale, non-archived products appear.
- Filters apply to `PRODUCTS`. `PRODUCT_CATEGORIES` / `POST_CATEGORIES` /
  `EVENT_CATEGORIES` list all categories of that type; `POSTS` / `EVENTS`
  list the shop's posts / events.

---

## Static image banner & canvas designs (legacy)

Before the explorer kinds, banners were "static" sections. Still rendered;
prefer `PROMO_SINGLE_BANNER` / `HERO_IMMERSIVE` for new work (they carry text
and a CTA).

```json
{
  "kind": "GRID", "kindConfig": null,
  "dataSource": "PRODUCTS", "dataType": "STATIC", "layout": "GRID_1",
  "staticFilter": { "staticType": "IMAGE", "images": ["https://<cdn>/b1.webp", "https://<cdn>/b2.webp"] }
}
```
`IMAGE`: full-bleed slider (16:9, 21:9 from 768px), auto-advances every 5 s,
images not clickable. `DESIGN`: `{ "staticType": "DESIGN", "designId": "<id>" }`
renders a canvas design.

### Designs API (owner only)
```graphql
query DesignsByShop($shopId: String!) {
  getDesignsByShop(shopId: $shopId) { id name output layers canvasSize { width height } createdAt }
}
query DesignById($designId: String!) {
  getDesign(designId: $designId) { id name output layers canvasSize { width height } }
}
mutation CreateDesign($shopId: String!, $payload: NewDesign!) {
  createDesign(shopId: $shopId, payload: $payload) { id name output }
}
mutation UpdateDesign($shopId: String!, $designId: String!, $payload: NewDesign!) {
  updateDesign(shopId: $shopId, designId: $designId, payload: $payload) { id name output }
}
```
`NewDesign`: `{ name, layers (JSON), output (rendered image URL), canvasSize { width, height } }`.

> ⚠️ **CONFIRMATION REQUIRED** before delete — sections that reference the design go blank.
```graphql
mutation DeleteDesign($shopId: String!, $designId: String!) {
  deleteDesign(shopId: $shopId, designId: $designId)
}
```

---

## Composing a good page

Read the page first (`getPageSections`), then plan the whole stack before
writing anything. A homepage that sells:

1. **One hero** — `HERO_IMMERSIVE` (one great photo), `HERO_EDITORIAL`
   (fashion / lifestyle, portrait photo) or `HERO_CENTERED_MINIMAL` (no photo).
   Never two heroes in a row; the first section loads with priority.
2. **Categories** — `CATEGORIES_GRID` (4–8 cards) or the bento (exactly 4)
   so shoppers can jump in.
3. **Products** — a `GRID_4` "New arrivals" (`DATE_DESC`), a `HORIZONTAL_SCROLL`
   best-sellers row (`ITEMS` + `MANUAL`), or a `PROMOTIONS` feed when discounts run.
4. **Promo** — `PROMO_MOSAIC`, `PROMO_SINGLE_BANNER` or `COUNTDOWN` while a sale is live.
5. **Story / proof** — `IMAGE_TEXT`, reels, `TESTIMONIALS_*`, `BRAND_STRIP`.
6. **Close** — `FAQ_ACCORDION` and/or a `CTA_BANNER` (Telegram, VIP).

Six to ten sections is plenty. Then:

- **Colour.** Most kinds already follow the theme primary colour (countdown,
  carousel, banner fill, buttons, the hero accent word). Pick the brand colour
  in Appearance first (`admin/appearance.md`); use section `background` sparingly
  — one or two tinted panels per page, not every section.
- **Contrast.** Any text over a photo or colour needs `textColor` chosen for it:
  white on dark photos with `overlayOpacity` 0.3–0.5, near-black on pale tiles.
  Several kinds default to white text — check pale backgrounds.
- **Spacing.** Keep `padding` the same across the page (`MD` is the admin
  default); use `NONE` only for intentionally edge-to-edge media.
- **Images.** Upload to the right shape (table above), same light and colour
  grading across one page; never reuse the hero photo lower down.
- **Khmer copy.** Khmer runs longer than English and has no capitals: keep
  hero headings to one short line (≈ 4–6 words), set `uppercaseHeading: false`
  on a Khmer `HERO_EDITORIAL`, keep category and tile titles to 1–3 words
  (they clamp to 2 lines), and promo subtitles under ~28 characters.
- **Links.** Every CTA must point at a route from the Links table.
- **Draft first.** For a redesign, create the new sections as `DRAFT`, show the
  merchant the plan, then `publishSection` each and `HIDDEN` the ones they replace.

---

## End-to-end recipes

### Hero from a prompt: "a hero with my product photo, headline 'Spring drop', button to the sale"

1. Upload: `curl -X POST -H "Authorization: $RIVERBASE_API_KEY" -F "file=@spring.jpg;type=image/jpeg" "${RIVERBASE_API_URL%/graphql}/uploads/s3/hq"` → URL.
2. `getPageSections(shopId, page: "home", statusFilter: [DRAFT, PUBLISHED, HIDDEN])`
   → `displayOrder` = max + 1 (or 0 and shift the others to put it first).
3. `createSection`:
   ```json
   {
     "shopId": "...", "name": "Spring hero", "page": "home",
     "kind": "HERO_IMMERSIVE",
     "kindConfig": {
       "image": { "url": "https://<cdn>/<uuid>.webp", "alt": "Spring collection" },
       "heading": "Spring drop",
       "cta": { "label": "Shop the sale", "href": "/promotions", "variant": "primary" },
       "overlayOpacity": 0.35,
       "textColor": "#ffffff"
     },
     "dataSource": "PRODUCTS", "dataType": "STATIC", "layout": "GRID_1",
     "displayOrder": 0, "showTitle": false, "titleAlignment": "LEFT",
     "padding": "NONE", "gap": "MD", "status": "PUBLISHED", "background": null
   }
   ```

### Categories grid from the catalog: "show my categories"

1. `getCategoriesByShop(shopId) { id name logo }`; upload a square photo for any
   category without a `logo`.
2. `createSection` with `kind: CATEGORIES_GRID`:
   ```json
   {
     "kind": "CATEGORIES_GRID",
     "kindConfig": {
       "heading": "Shop by category",
       "columns": 4,
       "cards": [
         { "title": "Skincare", "image": { "url": "<logo url>", "alt": "Skincare" },
           "href": "/search?source=PRODUCTS&category=<skincare_id>" },
         { "title": "Makeup", "image": { "url": "<logo url>", "alt": "Makeup" },
           "href": "/search?source=PRODUCTS&category=<makeup_id>" }
       ]
     },
     "dataSource": "PRODUCTS", "dataType": "STATIC", "layout": "GRID_1",
     "displayOrder": 1, "showTitle": false, "titleAlignment": "LEFT",
     "padding": "MD", "gap": "MD", "status": "PUBLISHED"
   }
   ```

### Hand-picked, hand-ordered product row: "feature these 6 products in this order"

Resolve names → ids, then:
```json
{
  "name": "Best sellers", "page": "home",
  "kind": "HORIZONTAL_SCROLL", "kindConfig": null,
  "dataSource": "PRODUCTS", "dataType": "DYNAMIC", "layout": "HORIZONTAL_SCROLL",
  "displayOrder": 2, "showTitle": true, "titleAlignment": "LEFT",
  "padding": "MD", "gap": "MD", "status": "PUBLISHED",
  "filterCondition": {
    "priceRange": { "from": 0, "to": 0 },
    "sortBy": "MANUAL", "filterBy": "ITEMS",
    "categories": [], "subCategories": [], "brands": [],
    "items": ["<id1>", "<id2>", "<id3>", "<id4>", "<id5>", "<id6>"],
    "limit": 12
  }
}
```

### One category as a 4-column grid
Same as above with `"layout": "GRID_4"`, `"kind": "GRID"`,
`"filterBy": "CATEGORIES"`, `"categories": ["<categoryId>"]`, `"sortBy": "DATE_DESC"`.

### FAQ
```json
{
  "name": "FAQ", "page": "home",
  "kind": "FAQ_ACCORDION",
  "kindConfig": {
    "heading": "Frequently asked",
    "items": [
      { "question": "How long does delivery take?", "answer": "Same day in Phnom Penh, 1–3 days to provinces." },
      { "question": "Can I return an item?", "answer": "Within 7 days, unused, in its packaging." }
    ],
    "visibleCount": 5
  },
  "dataSource": "PRODUCTS", "dataType": "STATIC", "layout": "GRID_1",
  "displayOrder": 7, "showTitle": false, "titleAlignment": "LEFT",
  "padding": "MD", "gap": "MD", "status": "PUBLISHED"
}
```

### Replace a hero's image (or any one field)
1. Upload the new image to `/uploads/s3/hq`.
2. `getSection(sectionId)` → take `kind` and the whole `kindConfig`.
3. Change `kindConfig.image.url` in memory.
4. `updateSection(shopId, sectionId, kind: <same kind>, kindConfig: <full object>)`.

### Move a section to the top
Read the page's sections, build the new order, then send one
`updateSection(shopId, sectionId, displayOrder: i)` per section.

### Draft, review, publish
Create with `status: DRAFT` (invisible on the storefront, even to the owner) →
describe it to the merchant → `publishSection(shopId, sectionId)`.

---

## Agent Notes

- **Enum casing.** GraphQL enums are SCREAMING_SNAKE_CASE (`HERO_EDITORIAL`,
  `GRID_4`, `MANUAL`); `kindConfig` keys are camelCase and its enum values
  lowercase (`"variant": "outline"`, `"provider": "tiktok"`).
- **`kindConfig` is a JSON object in `variables`**, never a JSON-encoded string.
- **Always send `kind` with `kindConfig`, and always the full `kindConfig`.**
- **Only the fields in this file persist.** Anything else is silently dropped.
- **Categories and brand strip: inline content only** (`cards`, `feature`…,
  `logos`). Id-only configs render empty on this version.
- **Product spotlight:** homepage only.
- **Feed filters:** one `filterBy` per section; never `MULTI`.
- **Links:** only real routes (Links table). Never `/shop`, `/sale`,
  `/categories`, `/collections`.
- **Images and video:** upload first; never invent URLs. Video: `/uploads/s3/hq`
  with `;type=video/mp4`.
- **Sections only render on `home`, `products`, `posts` and `events`.** For a
  custom content page use `admin/pages.md`.
- **Visible title:** with `showTitle: true` the section `name` is shown to
  shoppers — write it as a real heading in the shop's language.
- **Look and colour** come from Appearance (`admin/appearance.md`) and the
  header / footer / Look from `admin/shop-layout.md`; sections inherit both.
