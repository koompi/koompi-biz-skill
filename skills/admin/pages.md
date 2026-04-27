# Custom Pages

Create and manage custom content pages (About, Terms, Privacy, Contact, FAQ, etc.) for the shop's storefront.

> **Note**: The Pages RPC service must be running. If queries return "Failed to connect to RPC_PAGE", the service may be down.

## List All Pages

```graphql
query ListPages($shopId: String!) {
  getPages(shopId: $shopId) {
    id slug title status showInHeader showInFooter displayOrder coverImage
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
    seo { metaTitle metaDescription }
  }
}
```

## Get Page by Slug

```graphql
query PageBySlug($shopId: String!, $slug: String!) {
  getPageBySlug(shopId: $shopId, slug: $slug) {
    id slug title content status coverImage showInHeader showInFooter displayOrder
    seo { metaTitle metaDescription }
  }
}
```

## Get Page by ID

```graphql
query PageById($pageId: String!) {
  getPage(pageId: $pageId) {
    id slug title content status coverImage showInHeader showInFooter displayOrder
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
  ) { id slug title status }
}
```

### Parameters

| Field | Type | Required | Description |
|---|---|---|---|
| `slug` | String | ✅ | URL path: `"about"`, `"terms"`, `"contact"` — must be unique, lowercase, no spaces |
| `title` | String | ✅ | Page heading: `"About Us"`, `"Terms & Conditions"` |
| `content` | String | ✅ | Page body content — **JSON array of blocks** (see Rich Content Blocks below) or raw HTML fallback |
| `status` | String | ✅ | `"published"` or `"draft"` |
| `coverImage` | String | ❌ | Hero/banner image (CDN URL from `/uploads/s3`) |
| `showInHeader` | Boolean | ❌ | Show in header navigation dropdown |
| `showInFooter` | Boolean | ❌ | Show in footer links |
| `displayOrder` | Int | ❌ | Sort order for header/footer (lower = first) |
| `seo` | String | ❌ | JSON string: `{"metaTitle":"...", "metaDescription":"..."}` |

### Example: Create a Rich About Page (recommended)

Use JSON block array for rich layouts:

```json
{
  "shopId": "SHOP_ID",
  "slug": "about",
  "title": "About Us",
  "content": "[{\"type\":\"hero\",\"title\":\"Our Story\",\"subtitle\":\"Bringing you the finest since 2020\",\"imageUrl\":\"https://cdn.riverbase.org/about-hero.webp\",\"overlay\":true},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"columns\",\"columns\":[{\"title\":\"Our Mission\",\"text\":\"We believe quality should be accessible to everyone.\"},{\"title\":\"Our Values\",\"text\":\"Quality, transparency, and sustainability guide everything we do.\"}]},{\"type\":\"spacer\",\"size\":\"md\"},{\"type\":\"features\",\"features\":[{\"icon\":\"☕\",\"title\":\"Premium Quality\",\"description\":\"Hand-selected products from trusted suppliers\"},{\"icon\":\"🚚\",\"title\":\"Fast Delivery\",\"description\":\"Same-day delivery within the city\"},{\"icon\":\"💬\",\"title\":\"24/7 Support\",\"description\":\"Always here to help via Telegram\"}],\"columns\":3},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"callout\",\"emoji\":\"📍\",\"title\":\"Visit Us\",\"text\":\"123 Street, Phnom Penh, Cambodia. Open Mon-Sat 8am-6pm.\",\"variant\":\"info\"}]",
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
  "content": "[{\"type\":\"hero\",\"title\":\"Get In Touch\",\"subtitle\":\"We'd love to hear from you\",\"bgColor\":\"bg-muted/50\"},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"columns\",\"gap\":\"lg\",\"columns\":[{\"title\":\"📍 Location\",\"text\":\"123 Street, Phnom Penh, Cambodia\"},{\"title\":\"📞 Phone\",\"text\":\"+855 12 345 678\"},{\"title\":\"💬 Telegram\",\"links\":[{\"label\":\"@yourshop\",\"href\":\"https://t.me/yourshop\"}]}]},{\"type\":\"spacer\",\"size\":\"md\"},{\"type\":\"heading\",\"text\":\"Business Hours\"},{\"type\":\"list\",\"items\":[\"Monday - Friday: 8:00 AM - 6:00 PM\",\"Saturday: 9:00 AM - 5:00 PM\",\"Sunday: Closed\"]},{\"type\":\"spacer\",\"size\":\"md\"},{\"type\":\"embed\",\"url\":\"https://www.google.com/maps/embed?pb=...\",\"title\":\"Our Location\",\"fullWidth\":true}]",
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
  "content": "[{\"type\":\"hero\",\"title\":\"Frequently Asked Questions\",\"bgColor\":\"bg-muted/50\"},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"heading\",\"level\":2,\"text\":\"Orders & Shipping\"},{\"type\":\"callout\",\"emoji\":\"📦\",\"title\":\"How long does delivery take?\",\"text\":\"Same-day delivery within Phnom Penh. Provincial delivery takes 2-3 business days.\"},{\"type\":\"spacer\",\"size\":\"sm\"},{\"type\":\"callout\",\"emoji\":\"💰\",\"title\":\"What payment methods do you accept?\",\"text\":\"We accept cash on delivery, ABA Bank transfer, and major credit/debit cards.\"},{\"type\":\"spacer\",\"size\":\"lg\"},{\"type\":\"heading\",\"level\":2,\"text\":\"Returns & Refunds\"},{\"type\":\"callout\",\"emoji\":\"🔄\",\"title\":\"Can I return a product?\",\"text\":\"Yes! We accept returns within 7 days of delivery. Items must be unused and in original packaging.\"},{\"type\":\"spacer\",\"size\":\"sm\"},{\"type\":\"callout\",\"emoji\":\"💳\",\"title\":\"How do I get a refund?\",\"text\":\"Refunds are processed within 3-5 business days to your original payment method.\"}]",
  "status": "published",
  "showInHeader": true,
  "displayOrder": 3
}
```

### Legacy Example: Simple HTML content (still works)

```json
{
  "shopId": "SHOP_ID",
  "slug": "terms",
  "title": "Terms & Conditions",
  "content": "<h2>Terms & Conditions</h2><p>Last updated: April 2026</p><p>By using this website, you agree to these terms...</p>",
  "status": "published",
  "showInFooter": true,
  "displayOrder": 4
}
```

## Update Page

```graphql
mutation UpdatePage(
  $shopId: String!
  $pageId: String!
  $slug: String!
  $title: String!
  $content: String!
  $coverImage: String
  $status: String
  $showInHeader: Boolean
  $showInFooter: Boolean
  $displayOrder: Int
  $seo: String
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
  ) { id slug title status }
}
```

> **Important**: All fields are required in the mutation, but you should fetch the current page first and preserve unchanged fields.

### Workflow: Update Existing Page

1. **Fetch** — Get current page via `getPage(pageId: $pageId)`
2. **Modify** — Change only the fields the user requested, keep the rest
3. **Save** — Call `updatePage` with all fields (modified + preserved)
4. **Verify** — Re-fetch to confirm

## Delete Page

> ⚠️ **CONFIRMATION REQUIRED** — Show page title and slug before deleting. Warn that the URL will 404. Wait for explicit "Yes".

```graphql
mutation DeletePage($shopId: String!, $pageId: String!) {
  deletePage(shopId: $shopId, pageId: $pageId)
}
```

---

## Common Page Templates

AI can generate content for these standard pages:

| Page | Slug | Suggested Settings |
|---|---|---|
| About Us | `about` | `showInHeader: true`, `showInFooter: true` |
| Contact | `contact` | `showInHeader: true`, `showInFooter: true` |
| Terms & Conditions | `terms` | `showInFooter: true` |
| Privacy Policy | `privacy` | `showInFooter: true` |
| FAQ | `faq` | `showInHeader: true` |
| Shipping Info | `shipping` | `showInFooter: true` |
| Returns & Refunds | `returns` | `showInFooter: true` |
| Size Guide | `size-guide` | `showInHeader: false` |
| Our Story | `our-story` | `showInHeader: true` |

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

Column item properties: `title`, `text`, `imageUrl`, `imageAlt`, `links[]` (`{label, href}`), `html` (raw HTML)

#### `features` — Icon + title + description grid

```json
{ "type": "features", "columns": 3, "style": "simple", "features": [
  { "icon": "🚚", "title": "Free Shipping", "description": "On orders over $50" },
  { "icon": "↩️", "title": "Easy Returns", "description": "7-day return policy" },
  { "icon": "💬", "title": "24/7 Support", "description": "Via Telegram" }
]}
```

| Property | Type | Default | Description |
|---|---|---|---|
| `features` | array | — | Array of `{icon, title, description}` |
| `columns` | int | 3 | `2` / `3` / `4` |
| `style` | string | `"simple"` | `"simple"` / `"card"` (bordered) / `"icon"` (show icon) |

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
{ "type": "callout", "emoji": "📦", "title": "Shipping", "text": "Free delivery on orders over $50!", "variant": "info" }
```

`variant`: `"info"` (blue) | `"success"` (green) | `"warning"` (amber) | `"error"` (red)

#### `quote` — Blockquote with attribution

```json
{ "type": "quote", "text": "Best coffee in Phnom Penh!", "author": "Happy Customer", "authorRole": "Regular since 2022" }
```

#### `html` — Raw HTML passthrough

```json
{ "type": "html", "html": "<p>Custom <strong>HTML</strong> content</p>" }
```

### Tips for AI-Generated Pages

- **Start with hero** — Always begin with a `hero` block for a strong first impression
- **Use spacers** — Add `spacer` blocks between sections for breathing room
- **Mix full-width + inline** — Hero, columns, and features are full-width; headings, paragraphs, callouts are inline
- **Use callouts for FAQ** — Each FAQ item is a `callout` block with emoji, title, and answer
- **Contact pages** — Use `columns` for contact details + `embed` for Google Maps
- **About pages** — Use `hero` + `columns` for story sections + `features` for selling points + `image_grid` for team photos
- **Images need upload** — All `imageUrl` values must be CDN URLs from `POST /uploads/s3`
- **Content must be stringified** — The block array is passed as a JSON string in the `content` field

---

## Agent Notes

- **Slug rules**: Must be unique per shop. Use lowercase, hyphens for spaces, no special characters. E.g. `"terms-and-conditions"` not `"Terms & Conditions"`.
- **Content format**: **Preferred** — JSON array of rich blocks (stringified). **Fallback** — raw HTML string. Both work.
- **SEO as JSON string**: The `seo` field is a JSON **string** (not a JSON object). Serialize it: `'{"metaTitle":"...","metaDescription":"..."}'`.
- **AI content generation**: When a user asks "write an about page" or "create a contact page", generate rich block layouts (hero + columns + features + spacers), not just raw HTML. Tailor content to the shop's business type.
- **Don't duplicate**: Check if a page with the same slug already exists before creating. If it exists, offer to update it instead.
- **Images must be uploaded first**: Use `POST /uploads/s3` for all images. Never use placeholder URLs.
