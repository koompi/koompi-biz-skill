# Shop Layout: Header, Footer, Announcement, Navigation Bars, Looks, About, Blog & 404

The storefront "chrome" — everything around the page content — lives in one
`ShopLayout` document per shop. This file covers every surface on it and how
to change each one through GraphQL.

| Surface | Read | Write | Shown on the storefront when… |
|---|---|---|---|
| Header | `header` | `updateShopHeader` | `header.enabled: true` (else the default Navbar: logo, name, bell) |
| Footer | `footer` | `updateShopFooter` | `footer.enabled: true` (else **no footer at all**) |
| Header + footer at once | — | `updateShopLayout` | — |
| Announcement bar | `announcement` | `updateShopAnnouncement` | `enabled: true` **and** a `kind` is set |
| Desktop bar (≥768px) | `desktopBar` | `updateShopDesktopBar` | always (defaults = today's bar) |
| Phone bar (<768px) | `mobileBar` | `updateShopMobileBar` | always (defaults = today's bar) |
| Look (header+footer+bar template trio) | `look`, `publishedVersions` | `setShopLook`, `revertShopLook` | see "Looks" — read the warnings |
| About page (`/about`) | `aboutPage` | `updateShopAboutPage` | once set (else a placeholder) |
| Blog page (`/blog`) | `blogLayout` | `updateShopBlogLayout` | always |
| 404 page | `notFoundTemplateId` | `updateShopNotFoundTemplate` | always (default `simple`) |
| Everything above | — | `resetShopLayout` ⚠️ | — |

**Permissions:** every mutation here requires the `ThemeEdit` permission on the
shop (owner, or staff whose role grants it). `getShopLayout` needs no auth.

**Colours come from the theme.** "Brand" / "primary" everywhere in this file
means the shop theme's `primary` / `primaryForeground` tokens (set with
`updateTheme`, see `admin/appearance.md`). Change the brand colour there, not
here.

**Images:** upload first, then put the returned CDN URL in the JSON. Build the
upload URL from the API **origin** — `RIVERBASE_API_URL` ends in `/graphql`:
```bash
curl -X POST -H "Authorization: $RIVERBASE_API_KEY" -F "file=@./hero.jpg" \
  "${RIVERBASE_API_URL%/graphql}/uploads/s3/hq"   # /uploads/s3 for logos & photos
```

---

## Custom-coded shops (header/footer settings don't apply)

Four shops render a hand-built header and footer from code. If the shop
you are working on has one of these ids, it is custom-coded:

```
6891a59c524b59e2b1b79176
6891aafc524b59e2b1b79177
692524eaf9741887fb63bbfc
69b17a54efb9ace104f14789
```

For these: `updateShopHeader` / `updateShopFooter` / `setShopLook` save but
**change nothing visible**, and pages' `showInHeader` / `showInFooter` add no
links. What does work: the **announcement bar** (drawn inside their header, in
place of the shop's own bar, while it is on), the **desktop and phone bars**,
About, Blog and 404. Tell the merchant this before editing their header/footer.

---

## Get Shop Layout

```graphql
query GetShopLayout($shopId: String!) {
  getShopLayout(shopId: $shopId) {
    id shopId createdAt updatedAt
    header {
      enabled layout logoPosition sticky
      showSearch showCart
      showAnnouncement announcementText
      bgStyle bgColor textColor
      navigation { label href visibleTo highlight }
      blocks { blockType order props visible }
    }
    footer {
      enabled layout description
      columns { title links { label href visibleTo highlight } }
      showSocial socialLinks { platform url icon }
      showNewsletter copyrightText
      bgStyle bgColor textColor linkStyle separatorStyle
      blocks { blockType order props visible }
    }
    announcement { enabled kind config }     # config = inner JSON object (camelCase)
    desktopBar { enabled position labels colour }
    mobileBar { style labels colour }
    look { lookId brandColor fontPair density }
    publishedVersions { lookId brandColor fontPair density savedAt }
    notFoundTemplateId
    aboutPage {
      template heroTitle heroSubtitle heroImage
      storyText missionText visionText
      teamMembers { name role photo bio }
      showTimeline timelineItems { date title description }
      showStats stats { value label icon }
      blocks { blockType order props visible }
    }
    blogLayout {
      template showFeatured postsPerPage showSidebar showCategories
      showSearch showAuthor showDate showExcerpt heroTitle heroSubtitle
      blocks { blockType order props visible }
    }
  }
}
```

- If the shop has no layout yet, this query **creates the default one**
  (header and footer both `enabled: false`).
- `aboutPage`, `blogLayout`, `announcement`, `look` are `null` until first set.
  `desktopBar` / `mobileBar` are never null (unset = defaults).

---

## Wire rules (read before any write)

1. **JSON inputs are snake_case.** `updateShopHeader(header: JSON!)`,
   `updateShopFooter`, `updateShopLayout`, `updateShopAboutPage`,
   `updateShopBlogLayout` take a JSON object whose keys are **snake_case**
   (`logo_position`, `show_search`, `bg_style`, `copyright_text`,
   `visible_to`, `hero_title`…). Responses come back camelCase.
   **camelCase keys are silently ignored** (no error) and the field falls back
   to its default — e.g. sending `showSearch: false` leaves search ON.
2. **Send the JSON as an object**, not a JSON-encoded string.
3. **Each write replaces the whole section.** `updateShopHeader` overwrites the
   entire header; any field you omit takes its default, not its previous value.
   Always `getShopLayout` → modify in memory → send the full section back
   (converting the camelCase you read to snake_case).
4. **Required keys:** `layout` (header, footer) and `template` (about, blog);
   inside lists `label` + `href` (links), `title` + `links` (footer columns),
   `platform` + `url` (social links), `name` (team member), `date` + `title`
   (timeline), `value` + `label` (stats). Missing one → `Invalid header JSON:
   missing field …`.
5. **`enabled` defaults to `false`.** Omit it and the header/footer disappears.
   Always send `"enabled": true` unless you mean to turn the surface off.
6. **Enum values are snake_case strings on the wire** — in the JSON, in the
   GraphQL enum arguments (`LookId`, `FontPair`, `Density`,
   `AnnouncementTemplateKind`, `DesktopBarPosition`, `BarLabels`, `BarColour`,
   `MobileBarStyle`) and in responses (`"classic"`, `"icons_and_labels"`).
   Exception: `blocks[].blockType` reads back SCREAMING (`"HERO"`) but is
   written snake_case (`"block_type": "hero"`).
7. **Links are relative shop paths.** `href` in header navigation and footer
   columns must be a path like `/products`, `/promotions`, `/blog`, `/about`,
   `/<page-slug>`. The template prefixes the locale/shop path itself. A full
   `https://…` URL gets prefixed too and **breaks** — put external links in
   `social_links` (or announcement `extras`) instead.

---

## Header

```graphql
mutation UpdateShopHeader($shopId: String!, $header: JSON!) {
  updateShopHeader(shopId: $shopId, header: $header) {
    id
    header { enabled layout logoPosition sticky showSearch showCart bgStyle bgColor textColor
             navigation { label href visibleTo highlight } }
  }
}
```

### Header JSON

```json
{
  "enabled": true,
  "layout": "classic",
  "logo_position": "left",
  "sticky": true,
  "show_search": true,
  "show_cart": true,
  "show_announcement": false,
  "announcement_text": null,
  "bg_style": "blurred",
  "bg_color": null,
  "text_color": null,
  "navigation": [
    { "label": "Home", "href": "/", "visible_to": "all", "highlight": false },
    { "label": "Shop", "href": "/products", "visible_to": "all", "highlight": false },
    { "label": "Sale", "href": "/promotions", "visible_to": "all", "highlight": true }
  ],
  "blocks": []
}
```

| Field | Values | Default if omitted |
|---|---|---|
| `enabled` | bool | `false` ⚠️ |
| `layout` | see catalog below | **required** |
| `logo_position` | `left` `center` `right` | `left` |
| `sticky` | bool | `true` |
| `show_search`, `show_cart` | bool | `true` |
| `bg_style` | `transparent` `solid` `blurred` `custom` | `transparent` |
| `bg_color`, `text_color` | hex, used only with `bg_style: "custom"` | `null` |
| `navigation` | `[{label, href, visible_to, highlight}]` | `[]` → template shows Home, Products, About, Contact |
| `show_announcement`, `announcement_text` | legacy one-line bar inside the header (Classic & Atelier only). Prefer the Announcement bar surface. | `false` |
| `blocks` | see "Advanced: blocks" | `[]` |

`visible_to`: send `"all"`. (`"members"` / `"guests"` are honoured only by the
`weyoung` header.) `highlight: true` paints the link in the brand colour.

### Header template catalog (`layout`)

Only these render as designed templates on the storefront:

| `layout` | Look & feel | Honours |
|---|---|---|
| `classic` | Logo + nav + actions, frosted bar. Most flexible. | everything in the table above |
| `minimal` | Slim bar, logo + hamburger feel | `bg_*`, `sticky`, `show_search`, `show_cart`, `navigation` |
| `centered` | Logo centred, nav below | `bg_*`, `sticky`, `show_search`, `show_cart`, `navigation` (logo always centre) |
| `atelier` | Luxury: serif logo centre, hairline rule, airy | `sticky`, `show_search`, `show_cart`, `navigation`, legacy announcement |
| `marketplace` | Dense sticky bar, big search, brand-colour category band | `sticky`, `show_search`, `show_cart`, `navigation` |
| `weyoung` | weYOUNG style: blurred sticky bar, pill nav, profile card | `navigation` (+ `visible_to`) only |

Other values the API accepts — `classic_split`, `centered_all`,
`sidebar_docked`, `transparent_overlay`, `ecommerce_elite`, `maison`,
`editorial`, `flagship`, `studio`, `custom` — are **not** registered templates:
they render through a generic fallback header (logo + nav + actions, plus any
`blocks`). Don't pick them for a polished result.

---

## Footer

```graphql
mutation UpdateShopFooter($shopId: String!, $footer: JSON!) {
  updateShopFooter(shopId: $shopId, footer: $footer) {
    id
    footer { enabled layout description copyrightText bgStyle bgColor textColor linkStyle
             columns { title links { label href } } socialLinks { platform url } }
  }
}
```

### Footer JSON

```json
{
  "enabled": true,
  "layout": "columns",
  "description": "Handmade ceramics from Phnom Penh.",
  "columns": [
    { "title": "Shop", "links": [
      { "label": "All products", "href": "/products", "visible_to": "all", "highlight": false },
      { "label": "On sale", "href": "/promotions", "visible_to": "all", "highlight": false }
    ]},
    { "title": "Help", "links": [
      { "label": "Shipping", "href": "/shipping", "visible_to": "all", "highlight": false },
      { "label": "Contact", "href": "/contact", "visible_to": "all", "highlight": false }
    ]}
  ],
  "show_social": true,
  "social_links": [
    { "platform": "facebook", "url": "https://facebook.com/myshop", "icon": "" },
    { "platform": "telegram", "url": "https://t.me/myshop", "icon": "" }
  ],
  "show_newsletter": false,
  "copyright_text": "© {year} My Shop. All rights reserved.",
  "bg_style": "dark",
  "bg_color": null,
  "text_color": null,
  "link_style": "plain",
  "separator_style": "none",
  "blocks": []
}
```

| Field | Values | Default if omitted |
|---|---|---|
| `enabled` | bool | `false` ⚠️ (no footer at all) |
| `layout` | see catalog below | **required** |
| `description` | string — or a JSON blob for some templates (below) | `null` |
| `columns` | `[{title, links:[NavLink]}]` | `[]` → template's own sample columns |
| `social_links` | `[{platform, url, icon}]`; `platform` ∈ `facebook` `telegram` `tiktok` `instagram` `twitter` `youtube`; empty `url` is hidden | `[]` |
| `copyright_text` | string; `{year}` is replaced with the current year | `null` |
| `bg_style` | API accepts `default` `dark` `light` `primary` `custom` | `default` |
| `bg_color`, `text_color` | hex, with `bg_style: "custom"` | `null` |
| `link_style` | `plain` `underline` `pill` (also `dot_prefix` `dash_prefix`, legacy layouts only) | `plain` |
| `separator_style` | `none` `line` `dashed` `dotted` `gradient` (legacy layouts only) | `none` |
| `show_social`, `show_newsletter` | bool (registered templates decide from the data instead) | `true`, `false` |

**Background on the registered templates:** `dark` (near-black), `primary`
(brand colour), `custom` (your hex pair). `default` and `light` both render the
template's normal light surface.

### Footer template catalog (`layout`)

| `layout` | Look & feel | Notes |
|---|---|---|
| `columns` | Brand block left, link columns right | honours `link_style` (`plain`/`underline`/`pill`) |
| `simple` | One row of links + socials | uses only the **first** column's links (send one column with `"title": ""`) |
| `newsletter_focus` | Big signup block (optional cover image) + columns | extra fields in the `description` blob ↓ |
| `atelier_footer` | Luxury, minimal, legal-links row + badge | extra fields in the `description` blob ↓ |
| `marketplace_footer_look` | Dense multi-column sitemap | columns + socials + copyright |
| `weyoung_plus` | Dark footer with a flickering-grid backdrop | description + columns + socials |
| `weyoung` | Brand block + columns + contact strip | extra fields in the `description` blob ↓ |

Other accepted values (`columns_showcase`, `centered`, `centered_minimal`,
`product_catalog`, `support_hub`, `marketplace_footer`, `maison_footer`,
`editorial_footer`, `flagship_footer`, `studio_footer`, `custom`) render
through a generic fallback; `maison_footer`, `editorial_footer`,
`flagship_footer` and `studio_footer` come out **almost empty** (copyright
only). Avoid them.

### Templates that pack extra fields into `description`

These templates store their extra settings as a **JSON-encoded string** in
`description` (keys camelCase inside the blob):

```jsonc
// newsletter_focus
"description": "{\"headline\":\"Join the list\",\"subtext\":\"New drops first.\",\"coverImage\":\"https://cdn…/cover.webp\",\"bgStyle\":\"dark\"}"

// atelier_footer
"description": "{\"description\":\"Curated finds, shipped fast.\",\"badgeText\":\"NEW\",\"bgStyle\":\"solid\",\"legalLinks\":[{\"label\":\"Terms\",\"href\":\"/terms\",\"highlight\":false,\"visibleTo\":\"all\"}]}"

// weyoung
"description": "{\"description\":\"Since 2020.\",\"contact\":{\"phones\":[\"+855 12 345 678\"],\"address\":\"St 51, Phnom Penh\",\"mapUrl\":\"https://maps.app.goo.gl/…\"}}"
```

For `newsletter_focus` and `atelier_footer` the blob's `bgStyle`
(`solid` `transparent` `blurred` `dark` `primary` `custom`) wins over the
top-level `bg_style`; keep `bg_color`/`text_color` at top level.

### Custom pages join the header and footer automatically

Published pages with `showInHeader: true` are appended to the header
navigation; `showInFooter: true` pages go into an extra footer column titled
"Info" — both sorted by the page's `displayOrder`, linked as `/<slug>`. This
happens **only while that header/footer is `enabled`**, and not on the
custom-coded shops. So don't also add those pages to `navigation`/`columns`
by hand, or they appear twice. (See `admin/pages.md`.)

---

## Update header + footer together

```graphql
mutation UpdateShopLayout($shopId: String!, $header: JSON!, $footer: JSON!) {
  updateShopLayout(shopId: $shopId, header: $header, footer: $footer) {
    id header { enabled layout } footer { enabled layout }
  }
}
```
Same JSON rules as the two single mutations; both sections are replaced.

---

## Announcement bar

A bar above the header for promos, a countdown or social links. It shows on
the main browsing pages (not on checkout, product detail, orders, profile or
search). Its background follows the brand colour unless the template takes
custom colours.

```graphql
mutation UpdateShopAnnouncement(
  $shopId: String!, $enabled: Boolean!,
  $kind: AnnouncementTemplateKind, $config: JSON
) {
  updateShopAnnouncement(shopId: $shopId, enabled: $enabled, kind: $kind, config: $config) {
    announcement { enabled kind config }
  }
}
```

- `config` is the **inner** object (camelCase keys) — the server wraps it.
- Sending `config` without `kind` is an error.
- ⚠️ **The call replaces the whole surface.** Omitting `kind`/`config` clears
  them. To turn the bar off but keep its content, resend the current `kind`
  and `config` with `enabled: false`.
- Always fill `text`: an enabled bar with empty text shows a placeholder
  ("Free shipping on orders over $50").

### `kind` and its `config`

**`classic_promo_socials`** — one line of text plus a row of icon links.
```json
{
  "text": "Free delivery in Phnom Penh this week",
  "fallbackText": "Fresh flowers, delivered daily",
  "theme": "primary",
  "bgColor": null,
  "textColor": null,
  "extras": [
    { "label": "Telegram", "href": "https://t.me/myshop", "icon": "telegram" },
    { "label": "Facebook", "href": "https://facebook.com/myshop", "icon": "facebook" }
  ]
}
```
`theme`: `primary` (brand) · `dark` · `custom` (uses `bgColor`/`textColor`).
`fallbackText` shows when `text` is empty. `extras[].icon` ∈ `facebook`
`telegram` `tiktok` `instagram` `x` `youtube` `linkedin` `globe` (else the
label shows as text); entries without `href` are dropped. The row is labelled
"Media". (`alignment` is still accepted but no longer changes the bar.)

**`classic_centered_minimal`** — centred message + optional pill button; the
shopper can dismiss it (it comes back when you change the text or button).
```json
{ "text": "New: gift wrapping on every order", "ctaLabel": "Learn more", "ctaHref": "/gift-wrap",
  "bgColor": null, "textColor": null }
```
`null` colours = brand colour; set a hex in either to override that one.
The button needs both `ctaLabel` and `ctaHref`.

**`classic_editorial_dark`** — "Sale countdown": headline, subtitle, live
timer, buy button, on the brand colour.
```json
{ "text": "Khmer New Year sale", "subtitle": "Up to 40% off — ends soon",
  "endDate": "2027-04-16T17:00:00Z", "ctaLabel": "Shop now", "ctaHref": "/promotions",
  "showDays": true }
```
`endDate` is ISO-8601 (UTC). Empty = no timer. **After `endDate` passes the
bar hides itself** and counts as off (custom-coded shops get their own bar
back). Button needs both `ctaLabel` and `ctaHref`. Colours aren't adjustable.

**`weyoung`** — one short centred line on the brand colour.
```json
{ "text": "Just in — the rainy-season collection", "bgColor": null, "textColor": null }
```

---

## Navigation bars (desktop & phone)

Every storefront has a floating **desktop bar** (screens ≥768px) and a
**phone bar** (<768px) with fixed items: Home, Products, Search, Cart, You —
plus Orders, Edit page and **Create product** for the owner/staff. The items
can't be changed; their placement and look can.

### Read
```graphql
query NavBars($shopId: String!) {
  getShopLayout(shopId: $shopId) {
    desktopBar { enabled position labels colour }
    mobileBar { style labels colour }
  }
}
```

### Desktop bar — every argument required (send the full set)
```graphql
mutation UpdateShopDesktopBar(
  $shopId: String!, $enabled: Boolean!,
  $position: DesktopBarPosition!, $labels: BarLabels!, $colour: BarColour!
) {
  updateShopDesktopBar(shopId: $shopId, enabled: $enabled, position: $position,
                       labels: $labels, colour: $colour) {
    desktopBar { enabled position labels colour }
  }
}
```

| Arg | Values | Default | Effect |
|---|---|---|---|
| `enabled` | bool | `true` | `false` hides the bar; **the header takes over** its links, search, cart and account on desktop (and the owner's Create / Edit), and the header then also shows on `/profile`, `/vip`, `/notifications`, `/quotes`. With the header disabled, the default Navbar does this. |
| `position` | `bottom` `left` `right` | `bottom` | `left`/`right` = vertical rail; the page shifts over to make room. |
| `labels` | `icons` `icons_and_labels` | `icons` | Names show only on screens ≥1280px; narrower screens get icons. |
| `colour` | `light` `dark` `brand` | `light` | Light = frosted background colour; Dark = fixed near-black; Brand = theme primary (auto-contrast). |

### Phone bar
```graphql
mutation UpdateShopMobileBar(
  $shopId: String!, $style: MobileBarStyle!, $labels: BarLabels!, $colour: BarColour!
) {
  updateShopMobileBar(shopId: $shopId, style: $style, labels: $labels, colour: $colour) {
    mobileBar { style labels colour }
  }
}
```

| Arg | Values | Default |
|---|---|---|
| `style` | `classic` (full width — today's) · `floating` (rounded, lifted, raised Create) · `pill` (open tab in a solid pill) · `split` (capsule + separate round Create) · `cradle` (notch holding Create) | `classic` |
| `labels` | `icons` `icons_and_labels` | `icons_and_labels` |
| `colour` | `light` `dark` `brand` | `light` |

Create is merchant-only, so shoppers see `split` as just the capsule and
`cradle` like `floating`. Long tab sets drop names rather than truncate them.

A saved bar setting reaches shoppers on their **next page load**.

---

## Looks

A Look is a preset trio — header `layout`, footer `layout`, announcement
`kind` — plus stored tokens (`brandColor`, `fontPair`, `density`). Applying
one keeps an undo history (newest first, max 5).

```graphql
mutation SetShopLook($shopId: String!, $lookId: LookId!,
                     $brandColor: String, $fontPair: FontPair, $density: Density) {
  setShopLook(shopId: $shopId, lookId: $lookId, brandColor: $brandColor,
              fontPair: $fontPair, density: $density) {
    look { lookId brandColor fontPair density }
    header { enabled layout } footer { enabled layout } announcement { enabled kind }
  }
}

mutation RevertShopLook($shopId: String!, $versionIndex: Int!) {
  revertShopLook(shopId: $shopId, versionIndex: $versionIndex) {
    look { lookId } header { layout } footer { layout }
  }
}
```

| `lookId` | header `layout` | footer `layout` | announcement `kind` | Renders well? |
|---|---|---|---|---|
| `classic` | `classic` | `columns` | `classic_promo_socials` | ✅ |
| `weyoung` | `weyoung` | `weyoung` | `weyoung` | ✅ |
| `atelier` | `atelier` | `atelier_footer` | `classic_promo_socials` | ✅ |
| `marketplace` | `marketplace` | `marketplace_footer_look` | `classic_promo_socials` | ✅ |
| `maison` | `maison` | `maison_footer` | `classic_promo_socials` | ❌ generic header, near-empty footer |
| `editorial` | `editorial` | `editorial_footer` | `classic_promo_socials` | ❌ same |
| `flagship` | `flagship` | `flagship_footer` | `classic_promo_socials` | ❌ same |
| `studio` | `studio` | `studio_footer` | `classic_promo_socials` | ❌ same |

What a Look does **not** do — know this before using it:
- It only swaps the three template ids. It does **not** set `enabled` on the
  header/footer/announcement (a disabled header stays the default Navbar),
  and keeps the old announcement `config` (re-send one for the new `kind`).
- `brandColor`, `fontPair`, `density` are **stored but not applied** by the
  storefront today. For a brand colour use the theme (`admin/appearance.md`).
  Omitted tokens reset to `null` / `sans` / `comfortable`.
- The merchant's Appearance screen has no Look picker; it edits header,
  footer and announcement templates directly. Setting `updateShopHeader` /
  `updateShopFooter` yourself gives the same result with more control.
- `revertShopLook(versionIndex)` re-applies `publishedVersions[versionIndex]`
  (0 = the Look before the last apply) and snapshots the current Look first,
  so a revert can itself be reverted. Out-of-range index → error.

Enums: `LookId` `classic` `weyoung` `atelier` `editorial` `marketplace`
`maison` `flagship` `studio` · `FontPair` `sans` `serif_sans` `slab_sans`
`mono_sans` `serif_serif` · `Density` `compact` `comfortable` `spacious`.

---

## About page (`/about`)

```graphql
mutation UpdateShopAboutPage($shopId: String!, $aboutPage: JSON!) {
  updateShopAboutPage(shopId: $shopId, aboutPage: $aboutPage) {
    aboutPage { template heroTitle heroSubtitle heroImage storyText missionText visionText
                teamMembers { name role photo bio } showTimeline timelineItems { date title description }
                showStats stats { value label } }
  }
}
```

```json
{
  "template": "company_story",
  "hero_title": "Our story",
  "hero_subtitle": "Small-batch coffee, roasted in Siem Reap since 2019",
  "hero_image": "https://gateway.lite.riverbase.org/<uuid>.webp",
  "story_text": "It started with one roaster and a market stall…",
  "mission_text": "Fair prices for Mondulkiri farmers.",
  "vision_text": "A Cambodian coffee on every table.",
  "team_members": [ { "name": "Dara", "role": "Head roaster", "photo": "https://cdn…", "bio": "12 years at the roaster." } ],
  "show_timeline": true,
  "timeline_items": [ { "date": "2019", "title": "First stall", "description": "Psar Chas market" } ],
  "show_stats": true,
  "stats": [ { "value": "40+", "label": "Partner farms", "icon": "" } ],
  "blocks": []
}
```

How it renders (one design, top to bottom): dark hero (shown when
`hero_title` or `hero_image` is set; image at 30% under the text) → story →
mission & vision cards → stats band on the brand colour (if `show_stats`) →
team grid → timeline (if `show_timeline`) → a fixed "Shop all products" call
to action. Text is plain (no HTML).
- `template` must be one of `company_story` `team_showcase` `photo_timeline`
  `simple_bio` `mission_values` `custom`, but it **does not change the
  layout** today. `stats[].icon` and `blocks` aren't rendered.
- Once saved, this replaces any built-in or static About page. Don't create a
  custom page with slug `about` (reserved — see `admin/pages.md`).

---

## Blog page (`/blog`)

```graphql
mutation UpdateShopBlogLayout($shopId: String!, $blogLayout: JSON!) {
  updateShopBlogLayout(shopId: $shopId, blogLayout: $blogLayout) {
    blogLayout { template heroTitle heroSubtitle showDate showExcerpt }
  }
}
```

```json
{ "template": "card_grid", "hero_title": "Journal", "hero_subtitle": "Recipes and roasting notes",
  "show_date": true, "show_excerpt": true,
  "show_featured": true, "posts_per_page": 12, "show_sidebar": true,
  "show_categories": true, "show_search": true, "show_author": true, "blocks": [] }
```

What the storefront honours: `template` (`magazine_grid` / `card_grid` = a
grid; `classic_list` / `minimal_list` / `custom` = a list), `hero_title`
(default "Blog"), `hero_subtitle`, `show_date`, `show_excerpt`. The other
flags are stored but not used yet.

---

## 404 page

```graphql
mutation UpdateShopNotFoundTemplate($shopId: String!, $templateId: String) {
  updateShopNotFoundTemplate(shopId: $shopId, templateId: $templateId) { notFoundTemplateId }
}
```
`templateId`: `simple` (default) · `illustrated` · `bold` (huge brand-colour
numerals) · `lost` · `minimal` · `animated`. `null` clears it. The API accepts
any string; an unknown id renders `simple`.

---

## Reset

> ⚠️ **CONFIRMATION REQUIRED** — `resetShopLayout` restores **everything** on
> this page to defaults: header and footer (both become disabled), about,
> blog, announcement, Look **and its undo history**, 404 choice, and both
> navigation bars. It cannot be undone. List what will be lost and wait for an
> explicit "Yes".

```graphql
mutation ResetShopLayout($shopId: String!) {
  resetShopLayout(shopId: $shopId) { id header { enabled layout } footer { enabled layout } }
}
```

---

## Advanced: blocks (`layout: "custom"`)

`blocks` render only on the **fallback** surfaces: in the header for any
`layout` not in the header catalog above (e.g. `custom`), and in the footer
only when `layout: "custom"`. Registered templates ignore them. Use them only
when no template fits.

```json
{ "block_type": "banner", "order": 0, "visible": true,
  "props": "{\"text\":\"Visit our Toul Kork store\",\"href\":\"/contact\",\"backgroundColor\":\"#111\"}" }
```
`props` is a **JSON-encoded string**. Block types that render, with the props
they read: `heading` (text, level, align) · `paragraph` (text, align) ·
`image` (src, alt, width, height) · `html` / `rich_text` (html) · `hero`
(title, subtitle, backgroundImage, ctaText, ctaHref, align) · `banner` (text,
href, backgroundImage, backgroundColor) · `cta` (text, href, variant) ·
`divider` (color, thickness) · `spacer` (height) · `gallery` (images, columns)
· `link_list` (title, links, direction) · `nav_links` (links, direction) ·
`social_links` (links, size) · `logo` (src, alt, href, width, height) ·
`search_bar` (placeholder) · `cart_icon` · `newsletter` (title, placeholder).
Other types the API accepts (`contact_form`, `map`, `testimonials`, `faq`,
`pricing_table`, `team_grid`, `timeline`, `stats`) render nothing.

---

## Recipes

### "Give my shop a premium, minimal look" (template shop)

1. `getShopLayout` — confirm the shop isn't one of the custom-coded four, and
   keep the current navigation/columns/socials.
2. Brand colour + font via `updateTheme` (`admin/appearance.md`) — e.g. a deep
   ink primary, a serif display font.
3. Header and footer in one call:
   ```json
   {
     "shopId": "<SHOP_ID>",
     "header": {
       "enabled": true, "layout": "atelier", "sticky": true,
       "show_search": true, "show_cart": true,
       "show_announcement": false, "bg_style": "solid",
       "navigation": [
         { "label": "New in", "href": "/products", "visible_to": "all", "highlight": false },
         { "label": "Journal", "href": "/blog", "visible_to": "all", "highlight": false },
         { "label": "About", "href": "/about", "visible_to": "all", "highlight": false }
       ],
       "blocks": []
     },
     "footer": {
       "enabled": true, "layout": "atelier_footer",
       "description": "{\"description\":\"Objects made slowly, in Phnom Penh.\",\"badgeText\":\"\",\"bgStyle\":\"solid\",\"legalLinks\":[{\"label\":\"Terms\",\"href\":\"/terms\",\"highlight\":false,\"visibleTo\":\"all\"},{\"label\":\"Privacy\",\"href\":\"/privacy\",\"highlight\":false,\"visibleTo\":\"all\"}]}",
       "columns": [
         { "title": "Shop", "links": [ { "label": "All products", "href": "/products", "visible_to": "all", "highlight": false } ] },
         { "title": "Help", "links": [ { "label": "Shipping", "href": "/shipping", "visible_to": "all", "highlight": false } ] }
       ],
       "social_links": [ { "platform": "instagram", "url": "https://instagram.com/myshop", "icon": "" } ],
       "copyright_text": "© {year} My Shop",
       "bg_style": "default", "link_style": "plain", "separator_style": "none", "blocks": []
     }
   }
   ```
4. Quiet announcement:
   `updateShopAnnouncement(enabled: true, kind: classic_centered_minimal, config: {"text":"Free gift wrapping this month","ctaLabel":"","ctaHref":"","bgColor":null,"textColor":null})`.
5. Bars to match: `updateShopDesktopBar(enabled: true, position: right, labels: icons_and_labels, colour: light)`
   and `updateShopMobileBar(style: floating, labels: icons_and_labels, colour: light)`.
6. Re-read `getShopLayout` and summarise what changed.

### "Run a sale banner until Sunday night and make it loud"

1. `getShopLayout` → keep the current `announcement` in case they want it back.
2. `updateShopAnnouncement(enabled: true, kind: classic_editorial_dark, config: {"text":"Weekend sale","subtitle":"20% off everything","endDate":"<Sunday 23:59 Phnom Penh, as UTC: Sunday 16:59Z>","ctaLabel":"Shop the sale","ctaHref":"/promotions","showDays":false})`.
3. Optional: `updateShopMobileBar(style: pill, labels: icons_and_labels, colour: brand)` for the weekend.
4. The bar hides itself after `endDate`; offer to restore the previous
   announcement afterwards.

---

## Agent Notes

- **Pull the latest skill first** if you have a local clone:
  `git -C "${KOOMPI_BIZ_SKILL:-/path/to/koompi-biz-skill}" pull origin master 2>/dev/null || true`.
- **Read → modify → write the full section.** Every JSON write replaces the
  section; omitted fields reset to defaults (rule 3).
- **snake_case keys in, camelCase out.** camelCase input keys are dropped
  silently — the most common way an edit "does nothing".
- **Always send `"enabled": true`** for a header/footer you want visible.
- **Use the registered templates** (header: classic, minimal, centered,
  atelier, marketplace, weyoung; footer: columns, simple, newsletter_focus,
  atelier_footer, marketplace_footer_look, weyoung_plus, weyoung). Everything
  else is a plain fallback.
- **Prefer direct header/footer edits over Looks**; if you do apply a Look,
  only `classic`, `weyoung`, `atelier`, `marketplace` render well, and you
  still need to enable header/footer and re-send the announcement config.
- **Brand colour = theme primary.** Announcement bars, "brand" nav bars, the
  footer's `primary` background and highlighted links all follow it.
- **Relative hrefs only** in navigation/columns; external URLs go in
  `social_links` or announcement `extras`.
- **Custom pages** flagged `showInHeader`/`showInFooter` are added for you —
  don't duplicate them.
- **Custom-coded shops** (4 ids above): only the announcement, bars, About,
  Blog and 404 settings have a visible effect.
- **Khmer copy:** keep nav labels short — they don't truncate on the phone bar;
  the bar drops names instead when tabs crowd.
- **Confirm before `resetShopLayout`** — it wipes all chrome settings and the
  Look history.
