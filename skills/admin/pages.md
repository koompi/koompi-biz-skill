# Custom Pages

Create and manage the shop's own content pages (Contact, FAQ, Terms, Privacy,
Shipping, a campaign landing page…). A page lives at its slug:

- Shop's own domain: `https://<domain>/<locale>/<slug>` (e.g. `/en/contact`, `/km/contact`)
- Riverbase platform: `https://riverbase.org/shops/<locale>/<shopId>/<slug>`

Merchants manage the same pages in **Appearance → Pages** (`/profile/appearance?tab=pages`).

Pages have **two render modes**. Pick one per page:

| Mode | Body field | Renders as | Use for |
|---|---|---|---|
| `"rich"` (default) | `content` — JSON string of blocks (use one `html` block) | Sanitised HTML inside the shop's own page: shop header/footer, theme colours and fonts, Khmer repair | Contact, FAQ, policies, size guides, "our story" — almost every page |
| `"html"` | `customHtml` — a complete HTML document | A sandboxed `<iframe>`, full width, auto-height. Nothing from the shop theme reaches it | A designed landing / campaign page with its own CSS, layout and scripts |

> **The old rich-block catalogue is gone.** Block types `hero`, `columns`,
> `features`, `callout`, `spacer`, `divider`, `list`, `quote`, `embed` and
> `image_grid` do **not** exist on the current platform: the backend rejects a
> `content` that contains them (`Invalid content JSON: unknown variant …`).
> Build the same layouts with HTML + inline styles (rich mode) or a full
> document (html mode) — recipes below.

> **Note**: The Pages RPC service must be running. If queries return "Failed to connect to RPC_PAGE", the service may be down.

---

## What actually renders (read before designing)

### Which page wins for a slug
The storefront resolves `/<slug>` in this order:
1. `about` only — the About page from Layout Settings (`updateShopAboutPage`, see `admin/shop-layout.md`).
2. A hand-built page shipped in code for that shop (see "Taken slugs" below).
3. The page you create here (`getPageBySlug`).
4. Otherwise 404 (`about` falls back to a generic placeholder).

### Rich mode
- Layout: the shop's content container (its width follows the shop's container-size
  setting, up to 1280px or wider), `coverImage` first as a 192px-tall full-width
  banner, then `title`, then your content.
- The `title` prints as a **plain line of body text**, not a big heading — so open
  your content with a lead paragraph or an `<h2>`, and don't repeat the title.
- Base text is 14px, line-height ~1.6, in the shop's font (Latin + Khmer fallback).
  Styled for you: `h1`–`h6` (sized, bold), `p`, `ul`/`ol` (bullets/numbers),
  `blockquote` (left border in the brand colour), `hr`, `code`/`pre`, `mark`,
  `img` (rounded, max-width 100%), `a` (brand colour, underlined).
  **Not styled**: `table`, `details`, `figure`, `div` — style them inline.
- Khmer copied from legacy converters (vowel stored before the coeng) is reordered
  automatically so it doesn't render as dotted circles.

**The sanitiser (DOMPurify, default allowlist + embeds) keeps / drops:**

| Kept | Dropped |
|---|---|
| Normal HTML tags: `div section h1-h6 p span strong em a ul ol li table thead tbody tr th td details summary figure figcaption img video blockquote hr br` | `<script>`, every `on*=` handler (`onclick`…) |
| `style="…"` attributes — **this is how you design** | `<style>` blocks (stripped) |
| Inline `<svg>` icons | `target="_blank"` (links open in the same tab) |
| `<iframe>` **only** from `www.youtube.com`, `youtube.com`, `www.youtube-nocookie.com`, `player.vimeo.com`, `www.google.com` (Maps embed), `maps.google.com`, `www.openstreetmap.org` | `<iframe>` from any other host |
| `class="…"` (kept, but only classes the app already compiled exist — don't rely on Tailwind classes) | |

The theme's colours are CSS variables on the page, so inline styles can follow
the brand: `var(--primary)`, `var(--primary-foreground)`, `var(--foreground)`,
`var(--muted)`, `var(--muted-foreground)`, `var(--border)`, `var(--card)`,
`var(--accent)`. Use literal sizes for radius/spacing (`12px`).

### HTML mode (static page)
- Rendered with `srcdoc` in a sandboxed iframe:
  `allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-top-navigation-by-user-activation`.
  No `allow-same-origin` → your scripts run but `localStorage`, `document.cookie`
  and the parent page are unreachable (accessing storage throws — wrap it in
  `try`). No `allow-modals` → `alert()` / `confirm()` do nothing.
- The iframe is full width of the content area (no container, no padding), and
  its height follows your document (min 320px). The shop header, footer and nav
  bars still frame it. `title` and `coverImage` are **not shown** in this mode.
- **Never size anything by the viewport** (`100vh`, `dvh`, `svh`, `height:100%` on
  `html/body`): the frame grows to fit the content, so a viewport-tall section
  makes it grow again — an endless page. Avoid `position: fixed/sticky` too
  (they pin to the frame, not the screen).
- Links: a shopper's click on a link with no `target` (or `_self`/`_parent`)
  navigates the **whole shop tab**; `target="_blank"` opens a normal new tab;
  `#anchor` links stay inside the frame. Scripts cannot redirect the shop.
- Nothing is inherited: bring your own fonts, colours, `body{margin:0}` and
  Khmer-capable font stack. Media queries see the frame width.
- Search engines don't index iframe content as the page's text.

### Links that work on every domain
Shop URLs carry a locale (and on the platform, `/shops/<locale>/<shopId>`), so
**`href="/products"` is wrong** (it leaves the shop or hits an unknown locale).
Write shop links **relative, without a leading slash** — they resolve against the
page's own URL on both the custom domain and the platform, in both modes:

| Target | `href` |
|---|---|
| Home | `./` |
| All products | `products` |
| One product | `products/<productId>` |
| Promotions | `promotions` |
| Search | `search?keyword=linen` (also `search?category=<categoryId>`, `search?brand=<brandId>`) |
| Another custom page | `faq` |
| Blog / a post | `blog` / `posts/<postId>` |
| About | `about` |

External links (`https://t.me/…`, `tel:+855…`, `mailto:`) are fine as absolute URLs.

### Navigation, sitemap, tab title
- `showInHeader: true` appends the page to the header nav, `showInFooter: true` adds
  it to an extra footer column titled "Information" / "ព័ត៌មាន", both sorted by
  `displayOrder` (then creation date). Labels are the page `title`.
  They appear **only when the shop has a saved, enabled header/footer** in Layout
  Settings (`admin/shop-layout.md`), and never on the four custom-coded storefronts
  (ids listed in `admin/shop-layout.md`), which hardcode their own nav.
- Published pages are added to the shop's sitemap (`/en/<slug>`, `/km/<slug>`).
- The browser tab title is the **slug title-cased** (`shipping-policy` → "Shipping
  Policy"), not `title` or `seo.meta_title` — choose slugs that read well.
- `status: "draft"` hides the page from nav and the sitemap, but **its URL still
  opens for anyone** (`getPageBySlug` ignores status). Don't park private content in drafts.

---

## Reserved and taken slugs

**Rejected by the backend** (they collide with built-in routes):
```
about           blog            checkout        events          frozen
login           notifications   orders          pages           posts
products        profile         promotions      quotes          receipts
search          sections        try-on          vip
```
`about` is rejected with a message pointing to the About editor (Layout Settings) —
use `updateShopAboutPage` in `admin/shop-layout.md`.

**Accepted but never shown** (a built-in route shadows them) — don't use:
`b2b`, `games`, `offers`, `api`.

**Already taken by hand-built pages** on these shops (a page you create with the
same slug saves but never renders):

| Shop id | Slugs |
|---|---|
| `6891aafc524b59e2b1b79177` | `contact`, `faq`, `terms-and-conditions` |
| `6891a59c524b59e2b1b79176` | `about-us`, `services`, `warranty`, `terms-and-conditions`, `repair-and-maintenance`, `e-commerce-platform`, `web-and-mobile-app-development`, `stock-and-accountant-management-system` |
| `692524eaf9741887fb63bbfc` | `returns` |
| `6a1d0e4198857f09ca7a897e` | `contact-us`, `shipping-and-returns`, `terms-and-conditions` |

Slug format is not validated server-side beyond reserved + unique-per-shop: always
send lowercase words joined by hyphens (`size-guide`), no spaces, no `/`.

---

## Queries

All page queries are public (no auth). Field set:

```graphql
fragment PageFields on Page {
  id shopId slug title content coverImage status
  showInHeader showInFooter displayOrder
  renderMode customHtml
  seo { metaTitle metaDescription ogImage keywords }
  createdAt updatedAt
}
```

```graphql
# Every page, drafts included, sorted by displayOrder then createdAt
query ListPages($shopId: String!) { getPages(shopId: $shopId) { ...PageFields } }

# Published only (what the storefront nav uses)
query PublishedPages($shopId: String!) { getPublishedPages(shopId: $shopId) { ...PageFields } }

# One page — returns null if missing; does NOT filter by status
query PageBySlug($shopId: String!, $slug: String!) { getPageBySlug(shopId: $shopId, slug: $slug) { ...PageFields } }
query PageById($pageId: String!) { getPage(pageId: $pageId) { ...PageFields } }
```

`content` comes back as a JSON **string** of blocks, e.g. `"[{\"type\":\"html\",\"html\":\"<p>…</p>\"}]"`.

---

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
    shopId: $shopId slug: $slug title: $title content: $content
    coverImage: $coverImage status: $status
    showInHeader: $showInHeader showInFooter: $showInFooter displayOrder: $displayOrder
    seo: $seo renderMode: $renderMode customHtml: $customHtml
  )
}
```

Returns `Boolean` (`true`). It does **not** return the page — fetch it with
`getPageBySlug(shopId, slug)` if you need its `id`.

| Arg | Type | Required | Notes |
|---|---|---|---|
| `slug` | String | ✅ | Unique per shop; not reserved/taken (above). |
| `title` | String | ✅ | Nav label and the line above rich content. Keep it short ("Contact", "FAQ"). |
| `content` | String | ✅ | JSON **string** of a block array (see Content format). Send `"[]"` for html mode. Invalid JSON or an unknown block `type` fails the create. |
| `status` | String | ✅ | Exactly `"published"` or `"draft"` (lowercase; anything else errors). |
| `coverImage` | String | ❌ | Rich mode only. URL from `${RIVERBASE_API_URL%/graphql}/uploads/s3/hq` (wide image). |
| `showInHeader` / `showInFooter` | Boolean | ❌ | Default `false`. See Navigation above. |
| `displayOrder` | Int ≥ 0 | ❌ | Default `0`; lower comes first in nav. |
| `seo` | String | ❌ | JSON **string** with **snake_case** keys — see SEO below. |
| `renderMode` | String | ❌ | `"rich"` (default) or `"html"`. Not validated: anything other than `"html"` renders as rich. |
| `customHtml` | String | ❌ | Full HTML document. Only rendered when `renderMode: "html"` **and** it is non-empty. |

### SEO field
```json
"{\"meta_title\":\"Contact us\",\"meta_description\":\"Reach us on Telegram or visit our Phnom Penh store.\",\"og_image\":\"https://…/og.webp\",\"keywords\":[\"contact\",\"phnom penh\"]}"
```
- Keys are **snake_case**. camelCase keys (`metaTitle`) are silently ignored and the page is saved with empty SEO.
- Read back as `seo { metaTitle metaDescription ogImage keywords }`.
- Stored only: the storefront does not use it for `<title>`/meta today (see "tab title" above). Fill it anyway, it costs nothing.

### Auth
Any shop **owner or staff member** can create / update / delete (`is_owner_or_member`).
Always take `pageId` from `getPages(shopId)` for **this** shop, never from user text.

---

## Update Page

```graphql
mutation UpdatePage(
  $shopId: String!, $pageId: String!,
  $slug: String, $title: String, $content: String, $coverImage: String,
  $status: String, $showInHeader: Boolean, $showInFooter: Boolean,
  $displayOrder: Int, $seo: String, $renderMode: String, $customHtml: String
) {
  updatePage(
    shopId: $shopId pageId: $pageId slug: $slug title: $title content: $content
    coverImage: $coverImage status: $status showInHeader: $showInHeader
    showInFooter: $showInFooter displayOrder: $displayOrder seo: $seo
    renderMode: $renderMode customHtml: $customHtml
  )
}
```

Returns `Boolean`. `shopId` **and** `pageId` are required; every other arg is
optional and only what you send changes.

| Goal | Send |
|---|---|
| Publish a draft | `{ shopId, pageId, status: "published" }` |
| Replace the body (rich) | `{ shopId, pageId, content: "<JSON string>" }` — replaces all blocks |
| Change / remove the cover | `coverImage: "<url>"` / `coverImage: ""` (empty clears) |
| Set / clear SEO | `seo: "{\"meta_title\":\"…\"}"` / `seo: ""` (empty clears) |
| Switch to a designed page | `{ renderMode: "html", customHtml: "<!DOCTYPE html>…" }` |
| Switch back to rich | `{ renderMode: "rich" }` — the stored `content` shows again |
| Rename the URL | `slug: "new-slug"` — same reserved/unique checks; old links 404 |

`customHtml: ""` stores an empty document; an html-mode page with empty
`customHtml` falls back to rendering its rich `content`.

## Delete Page

> ⚠️ **CONFIRMATION REQUIRED** — Show page title and slug before deleting. Warn that the URL will 404 and any nav link disappears. Wait for explicit "Yes".

```graphql
mutation DeletePage($shopId: String!, $pageId: String!) {
  deletePage(shopId: $shopId, pageId: $pageId)
}
```

---

## Content format (rich mode)

`content` is a JSON string of an array of blocks. Only four block types exist:

| `type` | Fields | Renders |
|---|---|---|
| `html` | `html` (string, required) | The HTML, sanitised (see table above) |
| `heading` | `text` (required), `level` (1–6, default 2) | `<hN>text</hN>` |
| `paragraph` | `text` (required) | `<p>text</p>` |
| `image` | `url` (required), `alt`, `caption` | `<figure><img></figure>` — `caption` is **not** shown |

**Use exactly one `html` block** holding the whole page:
`[{"type":"html","html":"<…whole page…>"}]`. That's what the admin editor writes,
and the editor only loads `html` and `paragraph` blocks back — a page built from
`heading`/`image` blocks loses them the first time the merchant edits it in the app.

Because `content` is a string *inside* the GraphQL variables, it is double-encoded.
Build it with a tool rather than by hand:

```bash
# contact.html holds the page body; jq's tojson does the inner encoding
jq -n --arg shopId "$RIVERBASE_SHOP_ID" --rawfile html contact.html '{
  query: "mutation($shopId:String!,$slug:String!,$title:String!,$content:String!,$status:String!,$showInHeader:Boolean,$showInFooter:Boolean,$displayOrder:Int,$seo:String){createPage(shopId:$shopId,slug:$slug,title:$title,content:$content,status:$status,showInHeader:$showInHeader,showInFooter:$showInFooter,displayOrder:$displayOrder,seo:$seo)}",
  variables: {
    shopId: $shopId, slug: "contact", title: "Contact", status: "published",
    showInHeader: true, showInFooter: true, displayOrder: 1,
    content: ([{type: "html", html: $html}] | tojson),
    seo: ({meta_title: "Contact", meta_description: "Telegram, phone and store hours."} | tojson)
  }
}' | curl -s -X POST "$RIVERBASE_API_URL" \
  -H "Authorization: $RIVERBASE_API_KEY" -H "Content-Type: application/json" -d @-
```

### Designing a rich page
- **Readable width**: the content spans the whole container; wrap everything in
  `<div style="max-width:720px;margin:0 auto">` (or 960px for card grids).
- **Structure**: lead paragraph → `<h2>` sections → short paragraphs / lists. One idea per section.
- **Cards and grids**: `div` with `display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:12px`
  collapses to one column on phones by itself — no media queries needed (you can't add any).
- **Brand colour**: `var(--primary)` for accents, `var(--muted)` for soft panels,
  `var(--border)` for hairlines, `var(--muted-foreground)` for secondary text.
- **Buttons**: an `<a>` with `display:inline-block;padding:12px 20px;border-radius:999px;background:var(--primary);color:var(--primary-foreground);text-decoration:none;font-weight:600`.
- **FAQ**: `<details><summary>Question</summary><p>Answer</p></details>`, one per question — native, accessible accordions.
- **Icons**: small inline `<svg>` (24×24, `stroke="currentColor"`, `fill="none"`) — not emoji.
- **Tables** (hours, sizes, fees): add `border-collapse:collapse;width:100%` on `<table>` and `padding:8px 12px;border-bottom:1px solid var(--border)` on cells.
- **Images**: upload first and use the returned URL; always set `alt`. The upload
  endpoints sit beside `/graphql`, not under it, and answer with the plain URL string:
  ```bash
  curl -s -X POST -H "Authorization: $RIVERBASE_API_KEY" -F "file=@./store.jpg" \
    "${RIVERBASE_API_URL%/graphql}/uploads/s3"      # ≤2400px WebP (cards, inline photos)
  # "${RIVERBASE_API_URL%/graphql}/uploads/s3/hq"   # ≤3840px WebP (cover, full-bleed hero)
  ```
- **Khmer**: write the Khmer content properly (not a translation note), keep line-height ≥ 1.6, and never fix a height on a box that holds Khmer text (coeng and vowels stack vertically).

### Example — Contact page (rich)

`contact.html` (the value of the single `html` block):

```html
<div style="max-width:760px;margin:0 auto">
  <p style="font-size:1.125rem;line-height:1.7;color:var(--muted-foreground);margin:0 0 24px">
    We answer on Telegram within the hour, 8:00–18:00 every day. Prefer to see the pieces first? Visit our Toul Kork store.
  </p>

  <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:12px;margin:0 0 32px">
    <a href="https://t.me/yourshop" style="display:block;padding:16px;border:1px solid var(--border);border-radius:12px;color:inherit;text-decoration:none">
      <strong style="display:block;color:var(--primary)">Telegram</strong>
      <span style="color:var(--muted-foreground)">@yourshop — fastest reply</span>
    </a>
    <a href="tel:+85512345678" style="display:block;padding:16px;border:1px solid var(--border);border-radius:12px;color:inherit;text-decoration:none">
      <strong style="display:block;color:var(--primary)">Phone</strong>
      <span style="color:var(--muted-foreground)">012 345 678</span>
    </a>
    <a href="products" style="display:block;padding:16px;border:1px solid var(--border);border-radius:12px;color:inherit;text-decoration:none">
      <strong style="display:block;color:var(--primary)">Order online</strong>
      <span style="color:var(--muted-foreground)">Browse the full catalogue</span>
    </a>
  </div>

  <h2>Visit the store</h2>
  <p>#123, Street 315, Toul Kork, Phnom Penh. Parking in front.</p>
  <iframe src="https://www.google.com/maps/embed?pb=YOUR_EMBED_CODE" loading="lazy" referrerpolicy="no-referrer-when-downgrade" allowfullscreen
    style="width:100%;aspect-ratio:16/9;border:0;border-radius:12px;margin:8px 0 24px"></iframe>

  <h2>Opening hours</h2>
  <table style="border-collapse:collapse;width:100%;margin:0 0 24px">
    <tr><td style="padding:8px 0;border-bottom:1px solid var(--border)">Monday – Saturday</td><td style="padding:8px 0;border-bottom:1px solid var(--border);text-align:right">8:00 – 18:00</td></tr>
    <tr><td style="padding:8px 0;border-bottom:1px solid var(--border)">Sunday</td><td style="padding:8px 0;border-bottom:1px solid var(--border);text-align:right">9:00 – 17:00</td></tr>
  </table>

  <h2>Quick answers</h2>
  <details style="border-bottom:1px solid var(--border);padding:12px 0"><summary style="cursor:pointer;font-weight:600">Do you deliver outside Phnom Penh?</summary><p>Yes — to every province in 2–3 days.</p></details>
  <details style="border-bottom:1px solid var(--border);padding:12px 0"><summary style="cursor:pointer;font-weight:600">How do I pay?</summary><p>ABA transfer or cash on delivery.</p></details>
  <p style="margin-top:16px">More in our <a href="faq">FAQ</a>.</p>
</div>
```

Create it with the jq recipe above (`slug: "contact"`, `showInHeader: true`, `showInFooter: true`).

### Example — Policy page (rich, short)
```json
[{"type":"html","html":"<div style=\"max-width:720px;margin:0 auto\"><p style=\"color:var(--muted-foreground)\">Last updated 1 October 2026</p><h2>Returns</h2><p>Unworn items can be returned within 7 days of delivery, with tags attached.</p><h2>Refunds</h2><ul><li>Refunds go back to the original payment method within 3–5 business days.</li><li>Delivery fees are not refunded.</li></ul><h2>Exchanges</h2><p>Message us on <a href=\"https://t.me/yourshop\">Telegram</a> with your order number.</p></div>"}]
```
(`slug: "returns"`, `showInFooter: true` — check the "taken slugs" table first.)

---

## Designed landing page (html mode)

Use html mode when the merchant wants a page that looks like a campaign site:
full-bleed hero, custom type, product cards, animation. Rules:

1. **Complete document**: `<!DOCTYPE html>`, `<meta charset="utf-8">`, `<meta name="viewport" content="width=device-width, initial-scale=1">`, `body{margin:0}`.
2. **Match the shop**: read `themeByShop` (see `admin/appearance.md`) and copy its
   `primary`, `primaryForeground`, `background`, `foreground`, `border` colours and font
   family into `:root` variables — the frame inherits nothing.
3. **Fonts with Khmer**: load the Latin font plus **Kantumruy Pro** from Google Fonts and
   stack them `font-family:"<Latin font>","Kantumruy Pro",system-ui,sans-serif`; give
   Khmer runs `lang="km"` and `:lang(km){line-height:1.9}`.
4. **Responsive without the viewport**: `max-width` + `margin:0 auto` wrappers,
   `clamp()` type, `repeat(auto-fit,minmax(…))` grids. **No `vh`/`dvh`, no `height:100%`, no `position:fixed`.**
5. **Links**: relative shop paths (`products`, `products/<id>`) with no `target`, so a click
   moves the whole tab into the shop. Look up product ids first (`catalog/products.md`).
6. **Images**: upload (`${RIVERBASE_API_URL%/graphql}/uploads/s3/hq` for the hero,
   `…/uploads/s3` for cards — see the curl above), `loading="lazy"` below the fold. Never base64-embed.
7. **Scripts** are allowed but can't use storage, cookies or `alert()`. Keep them optional — the page must read fine without them.

### Example — "Linen Edit" landing page

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&family=Kantumruy+Pro:wght@400;600;700&display=swap" rel="stylesheet">
<style>
  :root{--bg:#ffffff;--fg:#111111;--muted:#6b6b6b;--primary:#1f9d63;--on-primary:#ffffff;--line:#e7e7e7}
  *{box-sizing:border-box}
  body{margin:0;background:var(--bg);color:var(--fg);font-family:"Inter","Kantumruy Pro",system-ui,sans-serif;line-height:1.6}
  :lang(km){line-height:1.9}
  .wrap{max-width:1120px;margin:0 auto;padding:0 20px}
  .hero{position:relative;color:#fff;background:#222 url("https://gateway.lite.riverbase.org/HERO_UPLOAD.webp") center/cover;padding:clamp(72px,12vw,160px) 0}
  .hero::before{content:"";position:absolute;inset:0;background:linear-gradient(90deg,rgba(0,0,0,.55),rgba(0,0,0,.1))}
  .hero .wrap{position:relative}
  .eyebrow{font-size:13px;letter-spacing:.14em;text-transform:uppercase;font-weight:600;opacity:.9}
  h1{font-size:clamp(34px,6vw,64px);line-height:1.1;margin:12px 0 16px;max-width:12ch}
  .lead{font-size:clamp(16px,2vw,19px);max-width:46ch;margin:0 0 28px;opacity:.92}
  .btn{display:inline-block;padding:14px 26px;border-radius:999px;background:var(--primary);color:var(--on-primary);text-decoration:none;font-weight:600}
  h2{font-size:clamp(24px,3.5vw,34px);margin:56px 0 20px}
  .grid{display:grid;gap:16px;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));padding-bottom:64px}
  .card{display:block;border:1px solid var(--line);border-radius:14px;overflow:hidden;color:inherit;text-decoration:none;transition:transform .2s}
  .card:hover{transform:translateY(-3px)}
  .card img{display:block;width:100%;aspect-ratio:4/5;object-fit:cover}
  .card div{padding:12px 14px}
  .price{color:var(--primary);font-weight:700}
  .band{background:#f4f1ec;padding:40px 0;text-align:center}
</style>
</head>
<body>
  <section class="hero">
    <div class="wrap">
      <div class="eyebrow">New season · <span lang="km">សូមស្វាគមន៍</span></div>
      <h1>The Linen Edit</h1>
      <p class="lead">Breathable pieces cut for Phnom Penh heat. Free delivery in the city this week.</p>
      <a class="btn" href="products">Shop the collection</a>
    </div>
  </section>

  <div class="wrap">
    <h2>Most loved</h2>
    <div class="grid">
      <a class="card" href="products/PRODUCT_ID_1">
        <img src="https://gateway.lite.riverbase.org/CARD_1.webp" alt="Relaxed linen shirt in sand" loading="lazy">
        <div><strong>Relaxed linen shirt</strong><br><span class="price">$24</span></div>
      </a>
      <a class="card" href="products/PRODUCT_ID_2">
        <img src="https://gateway.lite.riverbase.org/CARD_2.webp" alt="Wide linen trousers in olive" loading="lazy">
        <div><strong>Wide linen trousers</strong><br><span class="price">$29</span></div>
      </a>
      <a class="card" href="products/PRODUCT_ID_3">
        <img src="https://gateway.lite.riverbase.org/CARD_3.webp" alt="Linen midi dress in white" loading="lazy">
        <div><strong>Linen midi dress</strong><br><span class="price">$35</span></div>
      </a>
    </div>
  </div>

  <section class="band">
    <div class="wrap">
      <p style="margin:0 0 16px;font-size:18px">Questions about sizing? We reply on Telegram within the hour.</p>
      <a class="btn" href="https://t.me/yourshop" target="_blank" rel="noopener">Message us</a>
    </div>
  </section>
</body>
</html>
```

Create it (`content` must still be sent, as `"[]"`):

```bash
jq -n --arg shopId "$RIVERBASE_SHOP_ID" --rawfile html linen-edit.html '{
  query: "mutation($shopId:String!,$slug:String!,$title:String!,$content:String!,$status:String!,$renderMode:String,$customHtml:String,$showInHeader:Boolean){createPage(shopId:$shopId,slug:$slug,title:$title,content:$content,status:$status,renderMode:$renderMode,customHtml:$customHtml,showInHeader:$showInHeader)}",
  variables: {shopId: $shopId, slug: "linen-edit", title: "Linen Edit", content: "[]",
              status: "published", renderMode: "html", customHtml: $html, showInHeader: true}
}' | curl -s -X POST "$RIVERBASE_API_URL" \
  -H "Authorization: $RIVERBASE_API_KEY" -H "Content-Type: application/json" -d @-
```

---

## Common pages

| Page | Slug | Mode | Nav |
|---|---|---|---|
| Contact | `contact` | rich | header + footer |
| FAQ | `faq` | rich (`<details>`) | header or footer |
| Terms & Conditions | `terms` | rich | footer |
| Privacy Policy | `privacy` | rich | footer |
| Shipping Info | `shipping` | rich (table of zones/fees) | footer |
| Returns & Refunds | `returns` | rich | footer |
| Size Guide | `size-guide` | rich (table) | none — link it from products |
| Our Story | `our-story` | rich, or the About page via `updateShopAboutPage` | header |
| Campaign / lookbook | `linen-edit`, `new-year-sale` | html | header while it runs |

Check the "taken slugs" table for the shop before using one of these.

---

## Workflow

1. `getPages(shopId)` — does a page with this slug exist? If so, offer to update it.
2. Check the slug against the reserved / taken lists.
3. For a designed page, read the theme (`themeByShop`) and the products you will link.
4. Upload every image; write the HTML; build the payload with `jq` (`tojson`).
5. `createPage` (or `updatePage` with only the changed fields).
6. Verify: `getPageBySlug(shopId, slug)` — check `status`, `renderMode`, and that
   `content`/`customHtml` came back as sent. Give the merchant the URL
   (`/<locale>/<slug>` on their domain).
7. If they wanted it in the nav, confirm the shop has an enabled header/footer in
   Layout Settings; otherwise the flags have nothing to attach to.

## Agent Notes

- **Rich first.** It inherits the theme, fonts, Khmer repair and the shop's chrome,
  and the merchant can keep editing it in the app. Use html mode only for a designed
  landing page or when the merchant hands you a finished HTML file.
- **One `html` block** per rich page. Never send `hero`/`features`/`callout`/… blocks — the backend rejects them.
- **Inline styles, not `<style>` or Tailwind classes**, in rich mode.
- **Relative links without a leading slash** (`products`, `faq`) in both modes.
- **SEO JSON is snake_case** (`meta_title`), passed as a string; `""` clears it.
- **`content` is always a JSON string**, `"[]"` in html mode.
- **Drafts are reachable by URL** — they're only hidden from nav and the sitemap.
- **Don't duplicate**: check `getPages` for the slug first.
- **Images must be uploaded first** (`${RIVERBASE_API_URL%/graphql}/uploads/s3` or `…/uploads/s3/hq`); never placeholder or hotlinked URLs.
- **About page** is not a custom page: use `updateShopAboutPage` (`admin/shop-layout.md`).
