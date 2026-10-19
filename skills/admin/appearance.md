# Shop Appearance: Theme, Fonts, Product Display, Width, Coming Soon

The shop's look is split across a few APIs. This file covers the **theme**
(colours, radius, font), the **product display** settings (card style, cart
mode, product page layout), **storefront width**, and the **Coming Soon**
pre-launch page. Everything else in the admin's Appearance console lives in a
sibling file.

## Appearance console → API map

| Admin: Appearance tab → item | API | Documented in |
|---|---|---|
| Design → Brand (logo, footer logo, shop name, show name) | `shopUpdateLogo`, `shopUpdateBranding`, `shopUpdateName` | `admin/shop.md` |
| Design → Colors | `updateTheme` (`color`) | **here** |
| Design → Typography | `updateTheme` (`font`) + `POST /uploads/font` | **here** |
| Layout → Header / Footer / Announcement bar | `setShopLook`, `updateShopHeader`, `updateShopFooter`, `updateShopAnnouncement` | `admin/shop-layout.md` |
| Layout → Navigation (desktop bar, phone bar) | `updateShopDesktopBar`, `updateShopMobileBar` | `admin/shop-layout.md` |
| Layout → Storefront width | `shopUpdateModules` (`appContainerSize`) | **here** |
| Pages | `createPage` / `updatePage` | `admin/pages.md` |
| Product Display → Product cards | `shopUpdateModules` (`cardStyle`, `cartMode`) | **here** |
| Product Display → Product detail | `shopUpdateModules` (`productDetailThemeId`) | **here** |
| 404 | `updateShopNotFoundTemplate` | `admin/shop-layout.md` |
| Coming Soon | `shopSetComingSoon`, `shopUpdateComingSoonContent` | **here** |
| Homepage blocks (hero, grids, banners…) | `createSection` / `updateSection` | `admin/sections.md` |

Colour/size swatches on product cards are set **per product** (variant
"Show on product card"), not here — see `catalog/products.md`.

---

## Read the current look (do this first)

```graphql
query AppearanceState($shopId: String!) {
  themeByShop(shopId: $shopId) {
    shopId radius
    color {
      background foreground card cardForeground popover popoverForeground
      primary primaryForeground secondary secondaryForeground
      muted mutedForeground accent accentForeground
      destructive destructiveForeground border input ring
      chart1 chart2 chart3 chart4 chart5
    }
    font {
      source
      google { family weights }
      custom { familyName weights { weight fileUrl } }
    }
  }
  shopId(shopId: $shopId) {
    modules {
      blogging shopping ticketing stocking reselling quoting
      codPayment posCodEnabled cartMode cardStyle showProductFilter
      productDetailThemeId appContainerSize
    }
    comingSoon comingSoonImage comingSoonHeadline comingSoonMessage comingSoonLaunchAt
  }
}
```

`themeByShop` is public and also **creates the shop's theme document on first
read**. `updateTheme` fails with `Theme not found` on a shop whose theme was
never read, so always read before the first write.

---

## Update Theme

```graphql
mutation UpdateTheme($shopId: String!, $theme: JSON!) {
  updateTheme(shopId: $shopId, theme: $theme) {
    shopId radius
    color { background foreground primary primaryForeground ring }
    font { source google { family weights } custom { familyName weights { weight fileUrl } } }
  }
}
```

**Permission:** shop owner, or a staff role granted `ThemeEdit`.

`$theme` is a JSON value (an object in `variables`, not a string). It
**replaces the whole theme** — there is no partial update:

```json
{
  "shop_id": "<SHOP_ID>",
  "radius": "0.5rem",
  "color": {
    "background": "hsl(0, 0%, 100%)",
    "foreground": "hsl(0, 0%, 3.9%)",
    "card": "hsl(0, 0%, 100%)",
    "cardForeground": "hsl(0, 0%, 3.9%)",
    "popover": "hsl(0, 0%, 100%)",
    "popoverForeground": "hsl(0, 0%, 3.9%)",
    "primary": "hsl(152, 62%, 39%)",
    "primaryForeground": "hsl(0, 0%, 98%)",
    "secondary": "hsl(0, 0%, 96.1%)",
    "secondaryForeground": "hsl(0, 0%, 9%)",
    "muted": "hsl(0, 0%, 96.1%)",
    "mutedForeground": "hsl(0, 0%, 45.1%)",
    "accent": "hsl(0, 0%, 96.1%)",
    "accentForeground": "hsl(0, 0%, 9%)",
    "destructive": "hsl(0, 84.2%, 60.2%)",
    "destructiveForeground": "hsl(0, 0%, 98%)",
    "border": "hsl(0, 0%, 89.8%)",
    "input": "hsl(0, 0%, 89.8%)",
    "ring": "hsl(152, 62%, 39%)",
    "chart1": "hsl(12 76% 61%)",
    "chart2": "hsl(173 58% 39%)",
    "chart3": "hsl(197 37% 24%)",
    "chart4": "hsl(43 74% 66%)",
    "chart5": "hsl(27 87% 67%)"
  },
  "font": {
    "source": "Google",
    "google": { "family": "Inter", "weights": [400, 600, 700] },
    "custom": null
  }
}
```

### Rules the server enforces (or silently applies)

- **`shop_id` (snake_case) is required and must equal the `shopId` argument.**
  Permission is checked on `shopId`, but the write goes to the theme named by
  `shop_id`.
- **`radius` and all 24 `color` keys are required.** Colour keys are camelCase.
  A missing key fails with `missing field ...`.
- **`font` is optional, but omitting it (or `null`) erases the font** — the
  storefront falls back to Inter. Always send the current font back.
- **The read shape and the write shape differ.** Never echo `themeByShop`'s
  `font` back verbatim:

| | `themeByShop` returns | `updateTheme` needs |
|---|---|---|
| font source | `"google"` / `"custom"` (lowercase) | `"Google"` / `"Custom"` (capitalised — lowercase is rejected) |
| custom family | `custom.familyName` | `custom.family_name` |
| custom weight file | `weights[].fileUrl` | `weights[].file_url` |

- **Colours** are written straight into CSS variables (`primary` →
  `--primary`, `cardForeground` → `--card-foreground`, `chart1` → `--chart-1`),
  so any CSS colour works. Use `hsl(h, s%, l%)` or `#rrggbb` (the admin's
  palette editor saves hex). Avoid `oklch()`/`lab()` — the Telegram mini-app
  header colour is computed from `background` with a parser that doesn't read them.
- **`radius` must be in `rem`.** The storefront reads the number and appends
  `rem`, so `"8px"` becomes 8rem. Send `"0rem"`, `"0.5rem"`, `"1rem"`.

---

## Reset Theme to Default

```graphql
mutation ResetTheme($shopId: String!) {
  resetTheme(shopId: $shopId)
}
```

**Owner only** (staff get `Unauthorized`). Returns `true`. Resets to radius
`"1rem"`, font Google / Inter / `[400, 600, 700]`, and the platform colours
(white background, green primary `hsl(152, 62%, 39%)`, near-black ring).

> ⚠️ **CONFIRMATION REQUIRED** — this discards every custom colour, the font
> and the radius. Wait for an explicit "Yes".

---

## Colour tokens

| Token | Where shoppers see it |
|---|---|
| `background` / `foreground` | Page background and body text. `background` also tints the Telegram mini-app header. |
| `card` / `cardForeground` | Product cards, panels, dialogs |
| `popover` / `popoverForeground` | Dropdowns, menus, sheets |
| `primary` / `primaryForeground` | **The brand colour**: primary buttons, links, active states, the "Brand" colour of the nav bars, the announcement bars (text on them is `primaryForeground`) |
| `secondary` / `secondaryForeground` | Secondary buttons, tags |
| `muted` / `mutedForeground` | Subtle fills, captions, placeholders, helper text |
| `accent` / `accentForeground` | Hover / selected highlights |
| `destructive` / `destructiveForeground` | Errors, delete, sold-out |
| `border` / `input` | Dividers and card borders / form field borders |
| `ring` | Keyboard focus ring |
| `chart1`–`chart5` | Charts (mainly the admin dashboard) |

The storefront's brand colour and font come from **this theme**. A Look's
`brandColor` / `fontPair` (`admin/shop-layout.md`) are saved on the layout, but
the storefront does not apply them — to rebrand, change `primary` (and the
font) here.

---

## Fonts

The admin's font list (`source: "Google"`, saved by family name):

| Family | Khmer | Notes |
|---|---|---|
| **MiSans** | ✅ own Khmer design | Self-hosted (not fetched from Google). Weights 400/500/600/700. Credit "MiSans by Xiaomi" (licence). |
| **Google Sans** | ✅ own Khmer design | Google Fonts, Khmer subset |
| Inter, Roboto, Open Sans, Lato, Montserrat, Poppins, Comfortaa, Raleway, Nunito, Playfair Display, Merriweather, Oswald, Source Sans Pro, PT Sans, Ubuntu, Noto Sans, Work Sans, Fira Sans, Quicksand, Bebas Neue | Latin only | Khmer text falls back to **Kantumruy Pro** |

Any other Google Fonts family name also works (it is requested from Google
Fonts by name), but only the families above are offered in the admin.

**Khmer is real content on Riverbase, not an edge case.** Every font chain ends
in Kantumruy Pro, so Khmer always renders — but with a Latin-only font the two
scripts are two different designs side by side. When the shop's text is
largely Khmer, or the merchant wants Khmer and English to match, pick
**MiSans** or **Google Sans**. Display faces (Bebas Neue, Oswald, Playfair
Display) set every Latin heading in that face while Khmer stays Kantumruy Pro —
fine for an English-led brand, jarring for a Khmer-led one.

### Option A — Library / Google font

```json
{ "source": "Google", "google": { "family": "MiSans", "weights": [400, 600, 700] }, "custom": null }
```

`weights` are what is requested from Google (100–900). The admin always saves
`[400, 600, 700]`; include `500` if the design leans on medium weight.

### Option B — Custom (uploaded) font

1. Upload one file per weight:
   ```bash
   curl -X POST -H "Authorization: $RIVERBASE_API_KEY" \
     -F "file=@./Brand-Regular.woff2" "<API_BASE>/uploads/font"
   # → full URL string, e.g. ".../fonts/<uuid>.woff2"
   ```
   `.woff2` / `.woff` / `.ttf` / `.otf`, max 2 MB per file. `<API_BASE>` is the
   GraphQL URL without `/graphql`.
2. Save it — note the **snake_case** keys:
   ```json
   {
     "source": "Custom",
     "google": null,
     "custom": {
       "family_name": "BrandSans",
       "weights": [
         { "weight": 400, "file_url": "https://.../fonts/abc.woff2" },
         { "weight": 700, "file_url": "https://.../fonts/def.woff2" }
       ]
     }
   }
   ```
   Custom fonts load with `font-display: optional`: on a slow first visit the
   browser may show the fallback until the next page. Khmer glyphs come from
   the uploaded file only if it contains them; otherwise Kantumruy Pro.

---

## Radius

`radius` sets `--radius`; small/medium/large corners derive from it.

| Value | Style |
|---|---|
| `"0rem"` | Sharp (editorial, luxury, "black" preset) |
| `"0.25rem"` | Slightly rounded (corporate) |
| `"0.5rem"` | Standard rounded |
| `"0.75rem"` | Soft |
| `"1rem"` | Platform default, friendly |

---

## Colour presets

The Colors tab offers 8 presets. Picking one replaces `color` only (radius and
font stay). To apply one, put its object in `theme.color`:

```json
{
"default": {"background":"hsl(0 0% 100%)","foreground":"hsl(222 12% 15%)","card":"hsl(0 0% 100%)","cardForeground":"hsl(222 12% 15%)","popover":"hsl(0 0% 100%)","popoverForeground":"hsl(222 12% 15%)","primary":"hsl(0 0% 0%)","primaryForeground":"hsl(0 0% 100%)","secondary":"hsl(220 8% 46%)","secondaryForeground":"hsl(0 0% 100%)","muted":"hsl(220 10% 96%)","mutedForeground":"hsl(220 8% 46%)","accent":"hsl(220 14% 93%)","accentForeground":"hsl(222 12% 15%)","destructive":"hsl(0 72% 51%)","destructiveForeground":"hsl(0 0% 100%)","border":"hsl(220 10% 91%)","input":"hsl(220 10% 91%)","ring":"hsl(0 0% 0%)","chart1":"hsl(0 0% 0%)","chart2":"hsl(220 8% 46%)","chart3":"hsl(200 98% 39%)","chart4":"hsl(263 70% 50%)","chart5":"hsl(142 71% 45%)"},
"koompi": {"background":"hsl(0 0% 99%)","foreground":"hsl(210 65% 9%)","card":"hsl(0 0% 100%)","cardForeground":"hsl(210 65% 9%)","popover":"hsl(0 0% 100%)","popoverForeground":"hsl(210 65% 9%)","primary":"hsl(210 65% 9%)","primaryForeground":"hsl(0 0% 100%)","secondary":"hsl(210 20% 85%)","secondaryForeground":"hsl(210 65% 9%)","muted":"hsl(210 30% 96%)","mutedForeground":"hsl(210 20% 45%)","accent":"hsl(210 25% 92%)","accentForeground":"hsl(210 65% 9%)","destructive":"hsl(345 75% 53%)","destructiveForeground":"hsl(0 0% 100%)","border":"hsl(210 20% 88%)","input":"hsl(210 20% 88%)","ring":"hsl(210 65% 9%)","chart1":"hsl(210 65% 9%)","chart2":"hsl(210 40% 20%)","chart3":"hsl(345 75% 53%)","chart4":"hsl(200 60% 40%)","chart5":"hsl(210 20% 45%)"},
"business": {"background":"hsl(0 0% 100%)","foreground":"hsl(222 12% 15%)","card":"hsl(0 0% 100%)","cardForeground":"hsl(222 12% 15%)","popover":"hsl(0 0% 100%)","popoverForeground":"hsl(222 12% 15%)","primary":"hsl(213 94% 68%)","primaryForeground":"hsl(0 0% 100%)","secondary":"hsl(210 40% 15%)","secondaryForeground":"hsl(0 0% 100%)","accent":"hsl(214 95% 93%)","accentForeground":"hsl(210 40% 15%)","muted":"hsl(214 32% 95%)","mutedForeground":"hsl(215 16% 47%)","destructive":"hsl(0 72% 51%)","destructiveForeground":"hsl(0 0% 100%)","border":"hsl(214 32% 91%)","input":"hsl(214 32% 91%)","ring":"hsl(213 94% 68%)","chart1":"hsl(213 94% 68%)","chart2":"hsl(210 40% 15%)","chart3":"hsl(197 71% 52%)","chart4":"hsl(221 83% 53%)","chart5":"hsl(212 95% 85%)"},
"emerald": {"background":"hsl(0 0% 100%)","foreground":"hsl(222 12% 15%)","card":"hsl(0 0% 100%)","cardForeground":"hsl(222 12% 15%)","popover":"hsl(0 0% 100%)","popoverForeground":"hsl(222 12% 15%)","primary":"hsl(142 71% 45%)","primaryForeground":"hsl(0 0% 100%)","secondary":"hsl(160 84% 39%)","secondaryForeground":"hsl(0 0% 100%)","accent":"hsl(138 76% 92%)","accentForeground":"hsl(160 84% 39%)","muted":"hsl(138 76% 97%)","mutedForeground":"hsl(215 16% 47%)","destructive":"hsl(0 72% 51%)","destructiveForeground":"hsl(0 0% 100%)","border":"hsl(138 76% 88%)","input":"hsl(138 76% 88%)","ring":"hsl(142 71% 45%)","chart1":"hsl(142 71% 45%)","chart2":"hsl(160 84% 39%)","chart3":"hsl(158 64% 52%)","chart4":"hsl(173 58% 39%)","chart5":"hsl(138 76% 75%)"},
"coffee": {"background":"hsl(0 0% 100%)","foreground":"hsl(20 14% 20%)","card":"hsl(0 0% 100%)","cardForeground":"hsl(20 14% 20%)","popover":"hsl(0 0% 100%)","popoverForeground":"hsl(20 14% 20%)","primary":"hsl(25 35% 35%)","primaryForeground":"hsl(0 0% 100%)","secondary":"hsl(30 25% 25%)","secondaryForeground":"hsl(0 0% 100%)","accent":"hsl(30 40% 92%)","accentForeground":"hsl(25 35% 35%)","muted":"hsl(30 40% 96%)","mutedForeground":"hsl(30 10% 50%)","destructive":"hsl(0 72% 51%)","destructiveForeground":"hsl(0 0% 100%)","border":"hsl(30 40% 88%)","input":"hsl(30 40% 88%)","ring":"hsl(25 35% 35%)","chart1":"hsl(25 35% 35%)","chart2":"hsl(30 25% 25%)","chart3":"hsl(20 50% 55%)","chart4":"hsl(35 60% 65%)","chart5":"hsl(25 45% 75%)"},
"black": {"background":"hsl(0 0% 100%)","foreground":"hsl(0 0% 0%)","card":"hsl(0 0% 100%)","cardForeground":"hsl(0 0% 0%)","popover":"hsl(0 0% 100%)","popoverForeground":"hsl(0 0% 0%)","primary":"hsl(0 0% 0%)","primaryForeground":"hsl(0 0% 100%)","secondary":"hsl(0 0% 20%)","secondaryForeground":"hsl(0 0% 100%)","muted":"hsl(0 0% 96%)","mutedForeground":"hsl(0 0% 45%)","accent":"hsl(45 93% 47%)","accentForeground":"hsl(0 0% 0%)","destructive":"hsl(0 72% 51%)","destructiveForeground":"hsl(0 0% 100%)","border":"hsl(0 0% 89%)","input":"hsl(0 0% 89%)","ring":"hsl(0 0% 0%)","chart1":"hsl(0 0% 0%)","chart2":"hsl(0 0% 20%)","chart3":"hsl(45 93% 47%)","chart4":"hsl(0 0% 60%)","chart5":"hsl(0 0% 80%)"},
"light": {"background":"hsl(0 0% 100%)","foreground":"hsl(215 25% 27%)","card":"hsl(0 0% 100%)","cardForeground":"hsl(215 25% 27%)","popover":"hsl(0 0% 100%)","popoverForeground":"hsl(215 25% 27%)","primary":"hsl(200 98% 39%)","primaryForeground":"hsl(0 0% 100%)","secondary":"hsl(210 17% 82%)","secondaryForeground":"hsl(215 25% 27%)","accent":"hsl(210 17% 95%)","accentForeground":"hsl(200 98% 39%)","muted":"hsl(210 17% 98%)","mutedForeground":"hsl(215 16% 47%)","destructive":"hsl(0 72% 51%)","destructiveForeground":"hsl(0 0% 100%)","border":"hsl(210 17% 92%)","input":"hsl(210 17% 92%)","ring":"hsl(200 98% 39%)","chart1":"hsl(200 98% 39%)","chart2":"hsl(210 17% 82%)","chart3":"hsl(142 71% 45%)","chart4":"hsl(263 70% 50%)","chart5":"hsl(47 96% 53%)"},
"claude": {"background":"hsl(38 20% 92%)","foreground":"hsl(35 19% 17%)","card":"hsl(38 20% 92%)","cardForeground":"hsl(35 19% 17%)","popover":"hsl(38 20% 92%)","popoverForeground":"hsl(35 19% 17%)","primary":"hsl(20 53% 46%)","primaryForeground":"hsl(0 0% 100%)","secondary":"hsl(20 49% 42%)","secondaryForeground":"hsl(0 0% 100%)","muted":"hsl(38 13% 85%)","mutedForeground":"hsl(35 19% 17%)","accent":"hsl(20 53% 46%)","accentForeground":"hsl(35 19% 17%)","destructive":"hsl(0 72% 51%)","destructiveForeground":"hsl(0 0% 100%)","border":"hsl(38 13% 82%)","input":"hsl(38 13% 82%)","ring":"hsl(20 53% 46%)","chart1":"hsl(35 19% 17%)","chart2":"hsl(20 53% 46%)","chart3":"hsl(200 60% 55%)","chart4":"hsl(150 45% 55%)","chart5":"hsl(340 60% 55%)"}
}
```

| Preset | Vibe |
|---|---|
| `default` | Monochrome: black primary on white |
| `koompi` | Deep navy on near-white, crisp tech |
| `business` | Light blue primary, corporate |
| `emerald` | Fresh green, eco / health |
| `coffee` | Warm brown, café / bakery |
| `black` | Pure black and white with a gold accent; pair with `0rem` radius |
| `light` | Bright sky-blue, clean |
| `claude` | Warm sand background, terracotta primary |

There is no dark-mode switch: a "dark shop" is a palette whose surfaces
(`background`, `card`, `popover`, `muted`) are dark and whose foregrounds are light.

---

## Design guidance (building a custom palette)

Most requests are "make it look like my brand": start from the closest preset
and change a few tokens rather than inventing 24 values.

1. **Brand colour → `primary`, and `ring` = `primary`.** Every brand-coloured
   surface (buttons, nav bar "Brand" style, announcement bars) puts
   `primaryForeground` text on `primary`. Make that pair at least 4.5:1
   contrast: white on a dark or saturated primary, near-black on a light one
   (yellow, pastel, lime). A light primary with white text is the most common
   way a storefront becomes unreadable.
2. **Surfaces stay quiet.** `background`, `card`, `popover` are the same or
   near-identical colour; `muted` is `background` shifted a few % darker
   (`accent` a little more); `border` and `input` a bit darker again. Keep them
   in the brand's hue at low saturation (2–20%) so the page feels branded
   without competing with photos.
3. **Text pairs.** `foreground`, `cardForeground`, `popoverForeground` share one
   dark (or, on a dark palette, light) value. `mutedForeground` must still read
   as text: ~45% lightness on white, not 70%.
4. **Leave `destructive` red** (`hsl(0 72% 51%)` + white) unless the brand
   itself is red; then shift it slightly so errors stay distinguishable.
5. **Radius signals tone:** `0rem` editorial/luxury, `0.25–0.5rem`
   professional, `0.75–1rem` friendly/consumer.
6. **Font:** see "Fonts" above. Default to MiSans or Google Sans for Khmer-led
   shops; pair a Latin display font only when the brand is English-led.
7. Change one thing per save when the merchant is watching, and describe the
   change in words (and Khmer if the merchant writes in Khmer).

---

## Product display and storefront width — `shopUpdateModules`

Card style, cart mode, product page layout and storefront width are fields of
the shop's `modules`, written by `shopUpdateModules`.

```graphql
mutation ShopUpdateModules(
  $shopId: String!
  $blogging: Boolean!, $shopping: Boolean!, $ticketing: Boolean!
  $stocking: Boolean!, $reselling: Boolean!, $quoting: Boolean!
  $codPayment: Boolean!, $posCodEnabled: Boolean!
  $cartMode: CartMode!
  $cardStyle: CardStyle
  $showProductFilter: Boolean
  $productDetailThemeId: String
  $appContainerSize: AppContainerSize
) {
  shopUpdateModules(
    shopId: $shopId
    blogging: $blogging, shopping: $shopping, ticketing: $ticketing
    stocking: $stocking, reselling: $reselling, quoting: $quoting
    codPayment: $codPayment, posCodEnabled: $posCodEnabled
    cartMode: $cartMode, cardStyle: $cardStyle
    showProductFilter: $showProductFilter
    productDetailThemeId: $productDetailThemeId
    appContainerSize: $appContainerSize
  ) {
    id
    modules { cartMode cardStyle productDetailThemeId appContainerSize showProductFilter }
  }
}
```

> ⚠️ **This mutation replaces the module set.** Read `shopId(shopId) { modules {…} }`
> first and send **every** flag back with only your field changed. In particular
> an omitted `cardStyle` resets to `MINIMAL` and an omitted `showProductFilter`
> resets to `false`. Omitted `productDetailThemeId` / `appContainerSize` are
> left unchanged. `reselling` is ignored unless the caller is a platform admin.
> Permission: owner, or a role granted `ShopUpdateModules`.

| Field | Values | Effect |
|---|---|---|
| `cardStyle` | `MINIMAL` (default) · `BORDERED` · `OVERLAY` · `GLASS` | Product card look in every product grid / slider. `OVERLAY` and `GLASS` put name and price over the photo (they compact on narrow phone cards). |
| `cartMode` | `DETAILS_ONLY` (default) · `QUICK_CART_ONLY` · `BOTH` | Tapping a card opens the product · cards get an add button (quick add, no product page) · both. Only meaningful when `shopping` is on. |
| `productDetailThemeId` | `classic` (default) · `magazine` · `maison` · `camprotec` | Product page layout. Classic: image and details side by side. Magazine: masonry image grid, editorial type, sticky info rail. Maison: vertical scrolling gallery, sticky details, uppercase type. Camprotec: dark hero, spec/service tiles, sticky cart+buy bar. List the live set with `query { productDetailThemes { id name description isActive } }`; an unknown id renders as `classic`. Cannot be cleared once set. |
| `appContainerSize` | `STANDARD` (default, ~1280px) · `MEDIUM` (~1440px) · `LARGE` (~1680px) · `FULLSCREEN` (no cap) | Max content width on desktop. Phones are unaffected. |

---

## Coming Soon (pre-launch page)

While `comingSoon` is on, shoppers see a branded teaser (in the shop's theme
colours); the owner and staff still see the real storefront. Any owner or staff
member may call these.

```graphql
mutation ShopSetComingSoon($shopId: String!, $comingSoon: Boolean!) {
  shopSetComingSoon(shopId: $shopId, comingSoon: $comingSoon) { id comingSoon }
}

mutation ShopUpdateComingSoonContent(
  $shopId: String!, $image: String, $headline: String, $message: String, $launchAt: String
) {
  shopUpdateComingSoonContent(
    shopId: $shopId, image: $image, headline: $headline, message: $message, launchAt: $launchAt
  ) { id comingSoonImage comingSoonHeadline comingSoonMessage comingSoonLaunchAt }
}
```

- Partial update: omit an argument to keep it, send `""` to clear it.
- `image`: upload to `/uploads/s3/hq` first.
- `launchAt`: RFC 3339 (`"2026-12-01T09:00:00+07:00"`). It only shows a
  countdown — **the shop does not open by itself**; call
  `shopSetComingSoon(comingSoon: false)` at launch.
- Turning Coming Soon **on** hides the storefront from every shopper — confirm
  with the merchant first.

---

## Workflow: rebrand a shop

1. **Read** with `AppearanceState` above.
2. **Theme** — build the full theme JSON from what you read: copy `radius` and
   all 24 colours, change what the merchant asked for, and re-shape the font
   (`"google"` → `"Google"`, `familyName` → `family_name`, `fileUrl` → `file_url`).
3. **Write** with `updateTheme`, then re-read to confirm.
4. **Product display / width** — `shopUpdateModules` with every flag echoed.
5. **Chrome** (header, footer, announcement, nav bars) — `admin/shop-layout.md`.
   **Homepage content** — `admin/sections.md`.

### Example: blue brand, Poppins headings, slightly rounded

```json
{
  "query": "mutation UpdateTheme($shopId: String!, $theme: JSON!) { updateTheme(shopId: $shopId, theme: $theme) { shopId radius color { primary primaryForeground ring } font { source google { family weights } } } }",
  "variables": {
    "shopId": "<SHOP_ID>",
    "theme": {
      "shop_id": "<SHOP_ID>",
      "radius": "0.5rem",
      "color": {
        "background": "hsl(0 0% 100%)",
        "foreground": "hsl(222 47% 11%)",
        "card": "hsl(0 0% 100%)",
        "cardForeground": "hsl(222 47% 11%)",
        "popover": "hsl(0 0% 100%)",
        "popoverForeground": "hsl(222 47% 11%)",
        "primary": "hsl(221 83% 53%)",
        "primaryForeground": "hsl(0 0% 100%)",
        "secondary": "hsl(214 32% 91%)",
        "secondaryForeground": "hsl(222 47% 11%)",
        "muted": "hsl(214 32% 96%)",
        "mutedForeground": "hsl(215 16% 47%)",
        "accent": "hsl(214 95% 93%)",
        "accentForeground": "hsl(221 83% 40%)",
        "destructive": "hsl(0 72% 51%)",
        "destructiveForeground": "hsl(0 0% 100%)",
        "border": "hsl(214 32% 91%)",
        "input": "hsl(214 32% 91%)",
        "ring": "hsl(221 83% 53%)",
        "chart1": "hsl(221 83% 53%)",
        "chart2": "hsl(173 58% 39%)",
        "chart3": "hsl(197 37% 24%)",
        "chart4": "hsl(43 74% 66%)",
        "chart5": "hsl(27 87% 67%)"
      },
      "font": {
        "source": "Google",
        "google": { "family": "Poppins", "weights": [400, 600, 700] },
        "custom": null
      }
    }
  }
}
```

(Poppins is Latin-only: Khmer text in this shop renders in Kantumruy Pro.)

---

## Agent Notes

- **Always read before writing.** Both `updateTheme` and `shopUpdateModules`
  replace whole objects; a field you leave out is erased or reset.
- **Re-shape the font on every theme write** — capitalised `source`,
  snake_case `family_name` / `file_url`. Echoing the read shape fails.
- **`shop_id` inside the theme must be the same shop as `shopId`.**
- **Radius in `rem` only.** Colours as `hsl()` or hex.
- **Brand colour = `primary`.** Check `primaryForeground` contrast every time
  you change it; it is the text colour on every brand-coloured bar and button.
- **Khmer:** recommend MiSans or Google Sans when the shop writes in Khmer.
- **Destructive / outward-facing:** `resetTheme` (owner only) and turning
  Coming Soon on both need explicit confirmation.
- **Uploads:** fonts via `/uploads/font` (2 MB), images via `/uploads/s3/hq`.
  Never invent URLs.
