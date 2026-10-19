# Storefront Design Playbook — start here for "redesign / customise my shop"

Load this file for any broad design request ("make my shop look premium",
"redesign my storefront", "customise everything", "make it look like <brand>").
It tells you what can be changed, in what order, and which file holds the exact
API shapes. For a single, narrow change ("change the footer links"), go straight
to the surface's file instead.

| Surface | What the merchant can control | File |
|---|---|---|
| Brand basics | Logo, footer logo, shop name shown or hidden next to the logo | `admin/shop.md`, `admin/seo.md` |
| Theme | 24 colour tokens (the brand colour is `primary`), corner radius, font (incl. Khmer-matched MiSans / Google Sans or an uploaded font), 8 colour presets | `admin/appearance.md` |
| Product display | Card style, quick add (cart mode), product page layout, storefront width, filter bar | `admin/appearance.md` |
| Chrome | Header, footer, announcement bar, desktop bar, phone bar, About page, blog page, 404 page | `admin/shop-layout.md` |
| Page content | Sections on the home / products / posts / events pages: heroes, category tiles, product grids and rows, promos, testimonials, FAQ, video reels… | `admin/sections.md` |
| Extra pages | Contact, FAQ, policies, designed landing pages (own HTML) | `admin/pages.md` |
| Pre-launch | Coming Soon teaser page | `admin/appearance.md` |
| Search / sharing | Meta title, description, favicon, share image | `admin/seo.md` |

**Not changeable through the API:** the items on the desktop/phone bars (only
their placement and look), the product page beyond its 4 layouts, and the
checkout. Sections cannot be placed on custom pages.

---

## 1. Take a snapshot (one read, and keep it)

```graphql
query DesignSnapshot($shopId: String!) {
  shopId(shopId: $shopId) {
    id name logo footerLogo showNameInNavbar showNameInFooter comingSoon
    modules { cardStyle cartMode productDetailThemeId appContainerSize showProductFilter
              blogging shopping ticketing stocking reselling quoting codPayment posCodEnabled }
  }
  themeByShop(shopId: $shopId) {
    radius
    color { background foreground card cardForeground popover popoverForeground
            primary primaryForeground secondary secondaryForeground muted mutedForeground
            accent accentForeground destructive destructiveForeground border input ring
            chart1 chart2 chart3 chart4 chart5 }
    font { source google { family weights } custom { familyName weights { weight fileUrl } } }
  }
  getShopLayout(shopId: $shopId) {
    header { enabled layout bgStyle navigation { label href } }
    footer { enabled layout bgStyle }
    announcement { enabled kind config }
    desktopBar { enabled position labels colour }
    mobileBar { style labels colour }
    notFoundTemplateId
    aboutPage { heroTitle }
  }
  getPageSections(shopId: $shopId, page: "home", statusFilter: [DRAFT, PUBLISHED, HIDDEN]) {
    id name kind status displayOrder
  }
  getPages(shopId: $shopId) { id slug title status renderMode showInHeader showInFooter }
}
```

- **Keep this result.** Theme and layout writes go live at once and have no
  undo (except Looks). The snapshot is how you restore the old design if the
  merchant doesn't like the new one — offer that.
- **Custom-coded shop?** Compare the shop id with the four ids in
  `admin/shop-layout.md`. For those shops header, footer and Look edits change
  nothing visible; theme, sections, pages, announcement and bars still work.
- `getShopLayout` and `themeByShop` create default documents on first read —
  that's expected, and `updateTheme` needs it.

---

## 2. Agree on a direction before writing

A redesign touches everything shoppers see, immediately. Before the first
write, get (or infer, then confirm) these five things in one short message:

1. **Brand colour(s)** — a hex, a logo to sample, or a reference brand.
2. **Mood** — minimal / luxury, friendly / everyday, bold / streetwear,
   editorial / magazine, catalogue / marketplace.
3. **Lead language** — Khmer-led or English-led (drives the font and copy length).
4. **What sells** — the 2–4 categories or products to put first.
5. **Photos** — which images they have for a hero and category tiles (you must
   upload real files; never invent URLs).

Then state the plan per surface (theme, chrome, product display, homepage,
pages) and apply it after a "yes". For a small request, skip the ceremony.

---

## 3. Order of operations

Work outward-in, so every later choice inherits the earlier ones:

1. **Brand** — logo (`shop.md`), then the theme: `primary` + `primaryForeground`,
   surfaces, radius, font (`appearance.md`). Everything else (buttons, announcement
   bars, "brand" nav bars, countdowns, highlighted links) follows `primary`.
2. **Chrome** — header + footer in one `updateShopLayout` call with
   `enabled: true` and registered templates only; announcement bar; desktop
   and phone bars (`shop-layout.md`).
3. **Product display** — card style, cart mode, product page layout, width via
   `shopUpdateModules`, echoing every current module flag (`appearance.md`).
4. **Homepage** — plan 6–10 sections, create them as `DRAFT`, then publish and
   hide the ones they replace (`sections.md`).
5. **Pages** — Contact, FAQ, policies in rich mode; a campaign page in html
   mode; flag them for header/footer (`pages.md`).
6. **SEO** — title, description, share image (`seo.md`).

---

## 4. Starting points by mood

Use these as a first draft and adapt to the brand. Every value below is one the
storefront renders well.

| Mood | Theme | Header / footer | Product display | Bars | Homepage spine |
|---|---|---|---|---|---|
| **Minimal / luxury** | `black` preset or ink `primary`, `0rem` radius, serif display font only if English-led (else MiSans) | `atelier` / `atelier_footer` | `MINIMAL` cards, `maison` product page, `MEDIUM` width | desktop `right` rail, `icons_and_labels`, `light`; phone `floating` | `HERO_CENTERED_MINIMAL` or `HERO_EDITORIAL` → `CATEGORIES_ASYMMETRIC_BENTO` → `GRID_4` new in → `IMAGE_TEXT` story |
| **Friendly / everyday** | saturated `primary` (e.g. `emerald`, `light`), `0.75–1rem` radius, MiSans or Google Sans | `classic` / `columns` | `BORDERED` cards, `BOTH` cart mode | desktop `bottom`, phone `floating` or `pill`, `brand` colour | `HERO_IMMERSIVE` → `CATEGORIES_GRID` → `HORIZONTAL_SCROLL` best sellers (`ITEMS` + `MANUAL`) → `PROMO_SINGLE_BANNER` → `TESTIMONIALS_CARDS` → `FAQ_ACCORDION` |
| **Bold / streetwear** | black or neon `primary` with strong `primaryForeground` contrast, `0–0.25rem` radius | `weyoung` / `weyoung_plus` | `OVERLAY` or `GLASS` cards | phone `cradle` or `split`, `dark` | `HERO_EDITORIAL` → `VIDEO_REELS_MARQUEE` → `GRID_3` drop → `COUNTDOWN` while a drop is live |
| **Editorial / magazine** | warm neutral surfaces (`claude`, `coffee`), `0.25–0.5rem` radius | `centered` / `newsletter_focus` | `MINIMAL` cards, `magazine` product page | desktop `bottom`, `light` | `HERO_EDITORIAL` → `IMAGE_TEXT` → `PRODUCTS_EDITORIAL_SPOTLIGHT` (home only) → `RICH_TEXT` note → `GRID_3` |
| **Catalogue / marketplace** | `business` or `koompi`, `0.5rem` radius | `marketplace` / `marketplace_footer_look` | `BORDERED` cards, `BOTH` cart mode, `LARGE` width, filter bar on | desktop `left` rail, `icons_and_labels` | `PROMO_MOSAIC` → `CATEGORIES_GRID` → several `GRID_5` / `HORIZONTAL_SCROLL` rows by category → `BRAND_STRIP` |

---

## 5. Quality bar — check before you say "done"

- **Contrast.** `primaryForeground` on `primary` ≥ 4.5:1. Every section with
  text over a photo or colour has a `textColor` chosen for it (and a scrim on
  `HERO_IMMERSIVE`).
- **Khmer.** Khmer-led shop → MiSans or Google Sans. Khmer headings short (one
  line on a phone), tile titles 1–3 words, no fixed heights on boxes holding
  Khmer, no all-caps styling on Khmer text.
- **Phones first.** Most shoppers arrive from Telegram on a phone: hero photos
  keep the subject centred (the phone crop is tighter), the phone bar style
  suits the brand, and nothing relies on hover.
- **Images.** All uploaded through the API, in the shape each slot crops to,
  with `alt` text; one hero photo per page, not reused lower down.
- **Links.** Header/footer and section links are `/path`
  (`/products`, `/promotions`, `/search?category=<id>`); links inside custom
  pages are relative without a slash (`products`). No `/shop`, `/sale`,
  `/collections` — they 404.
- **No fallback templates.** Header and footer use the registered layouts;
  Looks other than `classic`, `weyoung`, `atelier`, `marketplace` are avoided.
- **Re-read every surface you wrote** and compare with what you sent (dropped
  camelCase keys and reset defaults show up here).

---

## 6. Report back

Tell the merchant, per surface: what changed and why, what you left alone, and
anything that needs their input (photos, copy, Khmer text). Give them the link
to check on their phone:

- own domain: `https://<domain>/<locale>/` (e.g. `/km/`)
- platform: `https://riverbase.org/shops/<locale>/<shopId>`

Shopper-facing bar and layout changes show on the next page load. Say plainly
that you changed settings through the API and did not see the rendered page
yourself, unless you actually opened it in a browser. Offer to restore the
snapshot.
