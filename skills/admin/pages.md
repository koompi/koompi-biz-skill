# Custom Pages

Create and manage custom content pages (Terms, Privacy, Contact, FAQ, Shipping, etc.) for the shop's storefront.

Pages support **two render modes**:
- **`"rich"` (default)** — the `content` field is a JSON array of themed
  blocks (hero / columns / features / callouts / heading / paragraph / etc.).
  Rendered inline in the storefront — sanitized, mobile-responsive, inherits
  the shop's theme. Use for almost every page.
- **`"html"` (static page)** — the `customHtml` field is a complete
  merchant-authored HTML document. Rendered ONLY inside a sandboxed
  `<iframe>` so its scripts cannot reach customer data. Use only when the
  merchant insists on pixel-perfect custom HTML.

> **Note**: The Pages RPC service must be running. If queries return "Failed to connect to RPC_PAGE", the service may be down.

## Reserved slugs (the backend rejects these)

The following slugs collide with literal storefront route folders — saving a
page with one of them would persist but never render, so the backend rejects
the create / update outright. Pick a different slug.

```
about           blog            checkout        events          frozen
login           notifications   orders          pages           posts
products        profile         promotions      quotes          receipts
search          sections        try-on          vip
```

> The `about` page is owned by Layout Settings (see `admin/shop-layout.md`
> → `updateShopAboutPage`), not by this skill. The backend returns a
> specific error pointing the agent there if you try to create a page
> with `slug: "about"`.

## List All Pages

```graphql
query ListPages($shopId: String!) {
  getPages(shopId: $shopId) {
    id slug title status showInHeader showInFooter displayOrder coverImage
    renderMode customHtml
    seo { metaTitle metaDescription }
    createdAt updatedAt
  }
}
```

## List Published Pages Only

```graphql
query PublishedPages($shopId: String!) {
  getPublishedPages(shopId: $shopId) {
    id slug title status showInHeader showInFooter displayOrder coverImage
    renderMode customHtml
    seo { metaTitle metaDescription }
  }
}
```

## Get Page by Slug

```graphql
query PageBySlug($shopId: String!, $slug: String!) {
  getPageBySlug(shopId: $shopId, slug: $slug) {
    id slug title content status coverImage showInHeader showInFooter displayOrder
    renderMode customHtml
    seo { metaTitle metaDescription }
  }
}
```

## Get Page by ID

```graphql
query PageById($pageId: String!) {
  getPage(pageId: $pageId) {
    id slug title content status coverImage showInHeader showInFooter displayOrder
    renderMode customHtml
    seo { metaTitle metaDescription }
  }
}
```

## Create Page

```graphql
mutation CreatePage(
  $shopId: String!
  $slug: String!
  $title: String!
  $content: String!
  $coverImage: String
  $status: String!
  $showInHeader: Boolean
  $showInFooter: Boolean
  $displayOrder: Int
  $seo: String
  $renderMode: String
  $customHtml: String
) {
  createPage(
    shopId: $shopId
    slug: $slug
    title: $title
    content: $content
    coverImage: $coverImage
    status: $status
    showInHeader: $showInHeader
    showInFooter: $showInFooter
    displayOrder: $displayOrder
    seo: $seo
    renderMode: $renderMode
    customHtml: $customHtml
  )
}
```

Returns `Boolean` (`true` on success).

### Parameters

| Field | Type | Required | Description |
|---|---|---|---|
| `slug` | String | ✅ | URL path: `"terms"`, `"contact"`, `"faq"` — must be unique, lowercase, no spaces. **Must not be a reserved slug** (see list above). |
| `title` | String | ✅ | Page heading: `"Terms & Conditions"`, `"Contact Us"` |
| `content` | String | ✅ | Page body — **JSON-string of a block array** for `renderMode: "rich"`. Pass `"[]"` when `renderMode: "html"` (the field is required but unused). |
| `status` | String | ✅ | `"published"` or `"draft"` |
| `coverImage` | String | ❌ | Hero/banner image (CDN URL from `/uploads/s3`) |
| `showInHeader` | Boolean | ❌ | Show in header navigation dropdown |
| `showInFooter` | Boolean | ❌ | Show in footer links |
| `displayOrder` | Int | ❌ | Sort order for header/footer (lower = first) |
| `seo` | String | ❌ | JSON string: `{"metaTitle":"...", "metaDescription":"..."}` |
| `renderMode` | String | ❌ | `"rich"` (default) or `"html"`. Selects which body field the storefront renders. |
| `customHtml` | String | ❌ | Full HTML document body. **Required when `renderMode: "html"`**; ignored otherwise. Rendered ONLY inside a sandboxed iframe — its scripts cannot reach customer data, localStorage, or the parent storefront's DOM. |

### Auth
Any shop **owner or staff member** can call `createPage` / `updatePage` /
`deletePage` — no special `Action::Page*` permission is required.

### Example: Create a Rich About Page (recommended)

Use JSON block array for rich layouts:

```json
{
  "shopId": "SHOP_ID",
  "slug": "about",
  "title": "About Us",
  "content": "[{\"type\":\"hero\",\"title\":\"Our Story\",\"subtitle\":\"Bringing you the finest since 2020\",\"imageUrl\":\"https://cdn.riverbase.org/about-hero.webp\",\"overlay\":true},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"columns\",\"columns\":[{\"title\":\"Our Mission\",\"text\":\"We believe quality should be accessible to everyone.\"},{\"title\":\"Our Values\",\"text\":\"Quality, transparency, and sustainability guide everything we do.\"}]},{\"type\":\"spacer\",\"size\":\"md\"},{\"type\":\"features\",\"features\":[{\"icon\":\"sparkles\",\"title\":\"Premium Quality\",\"description\":\"Hand-selected products from trusted suppliers\"},{\"icon\":\"truck\",\"title\":\"Fast Delivery\",\"description\":\"Same-day delivery within the city\"},{\"icon\":\"message-circle\",\"title\":\"24/7 Support\",\"description\":\"Always here to help via Telegram\"}],\"columns\":3},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"callout\",\"icon\":\"map-pin\",\"title\":\"Visit Us\",\"text\":\"123 Street, Phnom Penh, Cambodia. Open Mon-Sat 8am-6pm.\",\"variant\":\"info\"}]",
  "status": "published",
  "showInHeader": true,
  "showInFooter": true,
  "displayOrder": 1
}
```

### Example: Create a Contact Page

```json
{
  "shopId": "SHOP_ID",
  "slug": "contact",
  "title": "Contact Us",
  "content": "[{\"type\":\"hero\",\"title\":\"Get In Touch\",\"subtitle\":\"We'd love to hear from you\",\"bgColor\":\"bg-muted/50\"},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"columns\",\"gap\":\"lg\",\"columns\":[{\"title\":\"Location\",\"icon\":\"map-pin\",\"text\":\"123 Street, Phnom Penh, Cambodia\"},{\"title\":\"Phone\",\"icon\":\"phone\",\"text\":\"+855 12 345 678\"},{\"title\":\"Telegram\",\"icon\":\"send\",\"links\":[{\"label\":\"@yourshop\",\"href\":\"https://t.me/yourshop\"}]}]},{\"type\":\"spacer\",\"size\":\"md\"},{\"type\":\"heading\",\"text\":\"Business Hours\"},{\"type\":\"list\",\"items\":[\"Monday - Friday: 8:00 AM - 6:00 PM\",\"Saturday: 9:00 AM - 5:00 PM\",\"Sunday: Closed\"]},{\"type\":\"spacer\",\"size\":\"md\"},{\"type\":\"embed\",\"url\":\"https://www.google.com/maps/embed?pb=...\",\"title\":\"Our Location\",\"fullWidth\":true}]",
  "status": "published",
  "showInHeader": true,
  "showInFooter": true,
  "displayOrder": 2
}
```

### Example: Create FAQ Page

```json
{
  "shopId": "SHOP_ID",
  "slug": "faq",
  "title": "FAQ",
  "content": "[{\"type\":\"hero\",\"title\":\"Frequently Asked Questions\",\"bgColor\":\"bg-muted/50\"},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"heading\",\"level\":2,\"text\":\"Orders & Shipping\"},{\"type\":\"callout\",\"icon\":\"truck\",\"title\":\"How long does delivery take?\",\"text\":\"Same-day delivery within Phnom Penh. Provincial delivery takes 2-3 business days.\"},{\"type\":\"spacer\",\"size\":\"sm\"},{\"type\":\"callout\",\"icon\":\"wallet\",\"title\":\"What payment methods do you accept?\",\"text\":\"We accept cash on delivery, ABA Bank transfer, and major credit/debit cards.\"},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"heading\",\"level\":2,\"text\":\"Returns & Refunds\"},{\"type\":\"callout\",\"icon\":\"refresh-cw\",\"title\":\"Can I return a product?\",\"text\":\"Yes! We accept returns within 7 days of delivery. Items must be unused and in original packaging.\"},{\"type\":\"spacer\",\"size\":\"sm\"},{\"type\":\"callout\",\"icon\":\"wallet\",\"title\":\"How do I get a refund?\",\"text\":\"Refunds are processed within 3-5 business days to your original payment method.\"}]",
  "status": "published",
  "showInHeader": true,
  "displayOrder": 3
}
```

### Legacy Example: Simple HTML inside a rich block (still works)

You can still pass a single `html` rich-block when the merchant only has a
short snippet:

```json
{
  "shopId": "SHOP_ID",
  "slug": "terms",
  "title": "Terms & Conditions",
  "content": "[{\"type\":\"html\",\"html\":\"<h2>Terms & Conditions</h2><p>Last updated: April 2026</p><p>By using this website, you agree to these terms...</p>\"}]",
  "status": "published",
  "showInFooter": true,
  "displayOrder": 4
}
```

### Static-page (`renderMode: "html"`) — pixel-perfect custom HTML

When the merchant wants a fully hand-coded page (custom CSS / fonts /
animations that the rich block system doesn't expose), set
`renderMode: "html"` and put a complete HTML document in `customHtml`. The
storefront renders it inside a sandboxed iframe — the document is isolated
from the rest of the storefront and **cannot access** customer data,
localStorage, cookies, or the parent DOM.

```json
{
  "shopId": "SHOP_ID",
  "slug": "showcase",
  "title": "Showcase",
  "content": "[]",
  "renderMode": "html",
  "customHtml": "<!DOCTYPE html><html><head><meta charset=\"utf-8\"><style>body{margin:0;font-family:system-ui;background:#0f0f0f;color:#fff;}.hero{padding:80px 24px;text-align:center;}h1{font-size:48px;margin:0 0 12px}p{opacity:.7;max-width:540px;margin:0 auto}</style></head><body><section class=\"hero\"><h1>Spring drop</h1><p>A capsule of timeless essentials, hand-finished.</p></section></body></html>",
  "status": "published",
  "showInHeader": false,
  "showInFooter": true,
  "displayOrder": 9
}
```

Required: `content` is still required by the schema — pass `"[]"`. The
storefront ignores it when `renderMode` is `"html"`.

> Don't mix modes. A page is *either* `"rich"` *or* `"html"`. Switching modes
> on `updatePage` will toggle which body the storefront renders — the
> unused field is preserved on the document but not shown.

## Update Page

```graphql
mutation UpdatePage(
  $shopId: String!
  $pageId: String!
  $slug: String
  $title: String
  $content: String
  $coverImage: String
  $status: String
  $showInHeader: Boolean
  $showInFooter: Boolean
  $displayOrder: Int
  $seo: String
  $renderMode: String
  $customHtml: String
) {
  updatePage(
    shopId: $shopId
    pageId: $pageId
    slug: $slug
    title: $title
    content: $content
    coverImage: $coverImage
    status: $status
    showInHeader: $showInHeader
    showInFooter: $showInFooter
    displayOrder: $displayOrder
    seo: $seo
    renderMode: $renderMode
    customHtml: $customHtml
  )
}
```

Every field except `shopId` and `pageId` is **optional** — pass only the
fields you want to change.

### Empty-string clears `coverImage` and `seo`

- `coverImage: ""` → clears the cover image (sets it to null).
- `seo: ""` → clears the entire SEO object.
- Non-empty strings replace the stored value.

### Rename → reserved slug is rejected
If `slug` is being changed to one of the reserved slugs, the mutation fails
with the same error as `createPage`. Pick a different slug.

### Workflow: Update Existing Page

1. **Fetch** — Get current page via `getPage(pageId: $pageId)`.
2. **Modify** — Build a `variables` object with **only the fields you want to
   change**. Leave the rest off — they'll keep their stored values.
3. **Save** — Call `updatePage` with that partial variables object.
4. **Verify** — Re-fetch to confirm.

### Common update flows

| Goal | Send |
|---|---|
| Toggle from draft to published | `{ shopId, pageId, status: "published" }` |
| Change the cover image | upload via `/uploads/s3` → `{ shopId, pageId, coverImage: "<new url>" }` |
| Remove the cover image | `{ shopId, pageId, coverImage: "" }` |
| Update SEO only | `{ shopId, pageId, seo: "{\"metaTitle\":\"...\",\"metaDescription\":\"...\"}" }` |
| Clear SEO | `{ shopId, pageId, seo: "" }` |
| Switch a rich page to static HTML | `{ shopId, pageId, renderMode: "html", customHtml: "<!DOCTYPE html>…" }` |
| Switch back to rich | `{ shopId, pageId, renderMode: "rich" }` (the previously-saved `content` blocks reappear) |

## Delete Page

> ⚠️ **CONFIRMATION REQUIRED** — Show page title and slug before deleting. Warn that the URL will 404. Wait for explicit "Yes".

```graphql
mutation DeletePage($shopId: String!, $pageId: String!) {
  deletePage(shopId: $shopId, pageId: $pageId)
}
```

---

## Common Page Templates

AI can generate content for these standard pages. Reserved slugs (e.g.
`about`, `blog`, `products`) are NOT in this list — those are owned by
built-in storefront routes, not by this skill.

| Page | Slug | Suggested Settings |
|---|---|---|
| Contact | `contact` | `showInHeader: true`, `showInFooter: true` |
| Terms & Conditions | `terms` | `showInFooter: true` |
| Privacy Policy | `privacy` | `showInFooter: true` |
| FAQ | `faq` | `showInHeader: true` |
| Shipping Info | `shipping` | `showInFooter: true` |
| Returns & Refunds | `returns` | `showInFooter: true` |
| Size Guide | `size-guide` | `showInHeader: false` |
| Our Story | `our-story` | `showInHeader: true` |

> **About page lives elsewhere.** The About page slug is reserved because
> the storefront ships a dedicated About route powered by Layout Settings.
> Use `updateShopAboutPage` from `admin/shop-layout.md` instead of trying to
> create a custom page with `slug: "about"`.

---

## Rich Content Blocks

The `content` field accepts a **JSON array of block objects**. Each block has a `type` and type-specific properties. Mix and match blocks to create rich page layouts.

> **Important**: The `content` field is a JSON **string** (passed as a string to GraphQL). When constructing the mutation, stringify the block array.

### Full-Width Blocks (break out of text column)

#### `hero` — Full-width banner

```json
{ "type": "hero", "title": "Our Story", "subtitle": "Since 2020", "imageUrl": "https://cdn...", "buttonText": "Shop Now", "buttonHref": "/products", "overlay": true, "align": "center", "bgColor": "bg-muted/50" }
```

| Property | Type | Default | Description |
|---|---|---|---|
| `title` | string | — | Large heading text |
| `subtitle` | string | — | Subheading below title |
| `imageUrl` | string | — | Background image URL (CDN) |
| `buttonText` | string | — | CTA button label |
| `buttonHref` | string | — | CTA button link |
| `overlay` | boolean | false | Darken image for text readability |
| `align` | string | `"center"` | `"left"` / `"center"` / `"right"` |
| `bgColor` | string | — | Tailwind bg class (no image) |

#### `spacer` — Vertical spacing

```json
{ "type": "spacer", "size": "lg" }
```

`size`: `"sm"` (8px) | `"md"` (16px) | `"lg"` (32px) | `"xl"` (48px) | `"2xl"` (64px) | `"3xl"` (96px)

#### `divider` — Horizontal line

```json
{ "type": "divider", "style": "solid", "thickness": "border-t" }
```

`style`: `"solid"` | `"dashed"` | `"dotted"`

#### `columns` — Multi-column layout

```json
{ "type": "columns", "gap": "md", "columns": [
  { "title": "Address", "text": "123 Street, Phnom Penh", "imageUrl": "..." },
  { "title": "Hours", "text": "Mon-Sat 9-6pm", "links": [{ "label": "View map", "href": "/contact" }] }
]}
```

| Property | Type | Description |
|---|---|---|
| `columns` | array | 2–4 column items |
| `gap` | string | `"sm"` / `"md"` / `"lg"` |
| `align` | string | `"top"` / `"center"` / `"bottom"` |

Column item properties: `title`, `text`, `imageUrl`, `imageAlt`, `icon` (SVG icon name), `links[]` (`{label, href}`), `html` (raw HTML)

#### `features` — Icon + title + description grid

```json
{ "type": "features", "columns": 3, "style": "card", "features": [
  { "icon": "truck", "title": "Free Shipping", "description": "On orders over $50" },
  { "icon": "refresh-cw", "title": "Easy Returns", "description": "7-day return policy" },
  { "icon": "message-circle", "title": "24/7 Support", "description": "Via Telegram" }
]}
```

| Property | Type | Default | Description |
|---|---|---|---|
| `features` | array | — | Array of `{icon, title, description}` |
| `columns` | int | 3 | `2` / `3` / `4` |
| `style` | string | `"card"` | `"card"` (bordered, hover lift) / `"simple"` (inline list) / `"icon"` (inline list) |

#### `image_grid` — Gallery grid

```json
{ "type": "image_grid", "columns": 3, "gap": "md", "rounded": true, "images": [
  { "url": "https://cdn.../1.webp", "alt": "Store front", "caption": "Our new location" },
  { "url": "https://cdn.../2.webp", "alt": "Team" }
]}
```

#### `embed` — iframe embed (maps, videos)

```json
{ "type": "embed", "url": "https://www.google.com/maps/embed?pb=...", "title": "Location", "aspectRatio": "16:9", "fullWidth": true }
```

`aspectRatio`: `"16:9"` | `"4:3"` | `"1:1"` | `"21:9"`

### Inline Blocks (within text column, max-w-3xl)

#### `heading` — Section heading

```json
{ "type": "heading", "level": 2, "text": "Our Mission", "align": "left" }
```

`level`: 1–6, `align`: `"left"` / `"center"` / `"right"`

#### `paragraph` — Text paragraph

```json
{ "type": "paragraph", "text": "We are passionate about quality.", "align": "left" }
```

#### `image` — Standalone image

```json
{ "type": "image", "url": "https://cdn.../photo.webp", "alt": "Our team", "caption": "Team photo 2026", "width": "wide", "rounded": true }
```

`width`: `"full"` / `"wide"` (default) / `"contained"` (small)

#### `list` — Bullet or numbered list

```json
{ "type": "list", "ordered": false, "items": ["Monday - Friday: 8am-6pm", "Saturday: 9am-5pm", "Sunday: Closed"] }
```

#### `callout` — Highlighted info box

```json
{ "type": "callout", "icon": "truck", "title": "Shipping", "text": "Free delivery on orders over $50!", "variant": "info" }
```

| Property | Type | Description |
|---|---|---|
| `icon` | string | **Preferred** — SVG icon name (see icon list below) |
| `emoji` | string | **Fallback** — emoji string (rendered if no `icon`) |
| `title` | string | Bold heading |
| `text` | string | Body text |
| `variant` | string | `"info"` (blue) / `"success"` (green) / `"warning"` (amber) / `"error"` (red) |

> **Always use `icon` instead of `emoji`**. SVG icons look clean and professional. Only fall back to `emoji` if none of the available icons match the context.

#### `quote` — Blockquote with attribution

```json
{ "type": "quote", "text": "Best coffee in Phnom Penh!", "author": "Happy Customer", "authorRole": "Regular since 2022" }
```

#### `html` — Raw HTML passthrough

```json
{ "type": "html", "html": "<p>Custom <strong>HTML</strong> content</p>" }
```

### Tips for AI-Generated Pages

- **Use SVG icons, not emojis** — The `icon` field on features, callouts, and columns accepts icon names from the list below. Always prefer these over emoji.
- **Start with hero** — Always begin with a `hero` block for a strong first impression. Use `bgColor: "bg-gradient-to-br from-pink-50 via-rose-50/60 to-orange-50/40"` style gradients.
- **Use spacers generously** — Use `"lg"` or `"xl"` between sections. Never stack blocks directly.
- **Mix full-width + inline** — Hero, columns, and features are full-width; headings, paragraphs, callouts are inline (max-w-3xl on mobile, max-w-4xl on desktop).
- **Use callouts for FAQ** — Each FAQ item is a `callout` block with `icon`, title, and answer.
- **Contact pages** — Use `columns` for contact details + `embed` for Google Maps.
- **About pages** — `hero` + `columns` for story + `features` (card style) for selling points + `image_grid` for photos.
- **Careers pages** — `features` for benefits/perks, `callout` for each job listing, `columns` for apply links.
- **Images need upload** — All `imageUrl` values must be CDN URLs from `POST /uploads/s3`.
- **Content must be stringified** — The block array is passed as a JSON string in the `content` field.
- **Desktop-first thinking** — All layouts are responsive. Grid columns collapse on mobile automatically. Use `layout: "split"` on hero blocks for a side-by-side desktop layout (image right, text left). Use wider spacers (`"xl"`, `"2xl"`) on desktop.

### Responsive Visibility

Every block supports two optional fields to control visibility per viewport:

| Field | Type | Default | Description |
|---|---|---|---|
| `showOnMobile` | boolean | `true` | Show on screens < 768px |
| `showOnDesktop` | boolean | `true` | Show on screens ≥ 768px |

**Use cases:**
- Hide a mobile-only callout on desktop: `{ "type": "callout", "showOnDesktop": false, ... }`
- Hide a desktop-only hero on mobile and show a simplified mobile hero instead
- Hide decorative images on small screens: `{ "type": "image", "showOnMobile": false, ... }`

**Example — mobile and desktop hero variants:**
```json
[
  {"type": "hero", "title": "Sale! 50% Off", "subtitle": "Limited time only", "bgColor": "bg-gradient-to-r from-rose-500 to-pink-500", "showOnMobile": true, "showOnDesktop": false},
  {"type": "hero", "title": "Summer Collection", "subtitle": "Discover our curated selection", "imageUrl": "...", "layout": "split", "showOnMobile": false, "showOnDesktop": true}
]
```

### Hero Layout Options

The hero block supports a `layout` field:

| Layout | Description |
|---|---|
| `"stacked"` (default) | Full-width background image or gradient, text centered or left-aligned |
| `"split"` | Desktop: text on left, image on right (2-column grid). Mobile: stacked as usual. Requires `imageUrl`. |

### Available SVG Icons

Use these `icon` names in `features[].icon`, `callout.icon`, and `columns[].icon`:

| Icon Name | Use For | Icon Name | Use For |
|---|---|---|---|
| `briefcase` | Jobs, careers | `store` | Shop, retail |
| `camera` | Photos, content | `trending-up` | Growth, metrics |
| `gift` | Perks, rewards | `users` | Team, people |
| `target` | Goals, impact | `rocket` | Launch, startup |
| `heart` | Love, care | `star` | Reviews, ratings |
| `shield-check` | Trust, verified | `sparkles` | Premium, quality |
| `package` | Shipping, boxes | `send` | Messages, Telegram |
| `mail` | Email | `clock` | Hours, time |
| `map-pin` | Location, address | `phone` | Phone, contact |
| `globe` | Web, international | `instagram` | Social media |
| `link` | Links, URLs | `truck` | Delivery, shipping |
| `check-circle` | Done, confirmed | `zap` | Fast, instant |
| `award` | Achievement | `leaf` | Organic, eco |
| `palette` | Beauty, design | `smile` | Happy, friendly |
| `megaphone` | Announcement | `pen-tool` | Creator, editing |

If no icon matches the context, you may use an emoji string as fallback — the renderer will display it.

---

## Agent Notes

- **Slug rules**: Must be unique per shop. Use lowercase, hyphens for spaces, no special characters. E.g. `"terms-and-conditions"` not `"Terms & Conditions"`. Reserved slugs (`about`, `blog`, `checkout`, `products`, etc.) are rejected by the backend — see the list at the top.
- **About page is reserved** — generate it via `updateShopAboutPage` (`admin/shop-layout.md`), not via `createPage`. The backend returns a specific error message pointing the user there.
- **Pick the right render mode first.** `"rich"` for almost every page (themed, mobile-responsive, sanitized). `"html"` only when the merchant explicitly wants custom CSS / fonts / animations the rich blocks can't do — and accept that it renders inside a sandboxed iframe (no parent-DOM access).
- **`content` is always a JSON-string** of a block array, even when `renderMode: "html"` — pass `"[]"` in that case to satisfy the schema.
- **`customHtml` must be a complete HTML document** (`<!DOCTYPE html><html>…`). The iframe will not inherit the shop's theme stylesheet — include your own `<style>` block.
- **SEO as JSON string**: The `seo` field is a JSON **string** (not a JSON object). Serialize it: `'{"metaTitle":"...","metaDescription":"..."}'`. Pass `""` to clear.
- **AI content generation**: When a user asks "create a contact page", default to rich block layouts (hero + columns + features + spacers). Tailor content to the shop's business type. **Always use `icon` (SVG name) instead of `emoji`** for a professional look. Only generate `customHtml` if the user explicitly asks for a custom-coded page.
- **Don't duplicate**: Check if a page with the same slug already exists before creating. If it exists, offer to update it instead.
- **Images must be uploaded first**: Use `POST /uploads/s3` for all images. Never use placeholder URLs.
