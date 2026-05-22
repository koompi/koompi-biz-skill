# Broadcasts, Direct Messages & Report Groups (Telegram)

Riverbase shops talk to customers and the owner/staff through Telegram. There
are three flavors of outbound message:

| Surface | What it is |
|---|---|
| **Broadcast** | One-to-many push to every customer of the shop. |
| **Direct Message** | One-to-one push to a specific customer. |
| **Report Group** | A Telegram group chat used for shop-side analytics + alerts (order received, low stock, etc.). |

All three require the shop to have a Telegram bot configured first (see
`admin/shop.md` → `botCreate`).

---

## Broadcasts (push to every customer)

A broadcast queues a single image + caption + optional CTA button. The server
fans it out to each customer over the shop's bot and tracks success/failure
per recipient.

### Create a broadcast
```graphql
mutation CreateBroadcast(
  $shopId: String!
  $imageUrl: String!
  $caption: String!
  $buttonLabel: String
  $buttonUrl: String
) {
  createBroadcast(
    shopId: $shopId
    imageUrl: $imageUrl
    caption: $caption
    buttonLabel: $buttonLabel
    buttonUrl: $buttonUrl
  )    # → returns the new broadcast id
}
```
- `imageUrl`: must be an uploaded CDN URL. Upload first via `POST /uploads/s3`
  (or `/hq` for full-res banner images).
- `caption`: text body. Telegram MarkdownV2 is supported by the bot.
- `buttonLabel` + `buttonUrl`: optional inline-button pair. Both must be set
  together or both `null`.

### List broadcasts
```graphql
query Broadcasts($shopId: String!) {
  broadcasts(shopId: $shopId) {
    id status imageUrl caption buttonLabel buttonUrl
    totalCount sentCount failedCount
    recipientCustomerId    # only set for direct messages — null for broadcasts
    createdAt startedAt completedAt createdBy
  }
}
```
`BroadcastStatus`: `PENDING | SENDING | DONE | PARTIAL_FAILURE`.

### Status semantics
- `PENDING` — queued, not yet started.
- `SENDING` — fan-out in progress, check `sentCount` / `failedCount` for live counters.
- `DONE` — all recipients delivered.
- `PARTIAL_FAILURE` — completed with some `failedCount > 0`. Common cause:
  customers who blocked the bot.

### Recipe — quick promo blast
```json
{
  "shopId": "...",
  "imageUrl": "https://cdn.riverbase.org/promo.webp",
  "caption": "🎉 Friends & Family weekend — take 20% off everything.\nUse code FAMILY at checkout.",
  "buttonLabel": "Shop the sale",
  "buttonUrl": "https://<shop>.riverbase.shop"
}
```

> ⚠️ **CONFIRMATION REQUIRED** — broadcasts go to **every** customer on the
> shop. Always show the recipient count first
> (`customers(shopId).length`) and the caption preview before sending. Wait
> for explicit "Yes".

---

## Direct messages (push to one customer)

Same shape as a broadcast, but targeted to a single customer:

```graphql
mutation SendDirectMessage(
  $shopId: String!
  $customerId: String!
  $imageUrl: String!
  $caption: String!
  $buttonLabel: String
  $buttonUrl: String
) {
  sendDirectMessage(
    shopId: $shopId
    customerId: $customerId
    imageUrl: $imageUrl
    caption: $caption
    buttonLabel: $buttonLabel
    buttonUrl: $buttonUrl
  )    # → returns the new broadcast-style id (DM uses the same pipeline)
}
```

DM records also surface through the `broadcasts(shopId)` query —
`recipientCustomerId` is the customer id when it's a DM, `null` for a true
broadcast.

### Resolving a customer id
```graphql
query Customers($shopId: String!) {
  customers(shopId: $shopId) {
    id userId
    user { id tgFirstname tgLastname tgUsername }
  }
}
```
Use `Customer.id` (not `user.id`) as `customerId` for `sendDirectMessage`.

### Recipe — abandoned-cart nudge
```json
{
  "shopId": "...",
  "customerId": "<customer_id>",
  "imageUrl": "https://cdn.riverbase.org/cart.webp",
  "caption": "Hey! Looks like you left something in your cart. Want to finish checking out?",
  "buttonLabel": "Resume checkout",
  "buttonUrl": "https://<shop>.riverbase.shop/cart"
}
```

---

## Report Groups (Telegram group chats for shop staff)

A report group is a Telegram group chat the shop's bot has been added to. The
backend pushes shop-side notifications (new orders, low stock, payment
events) into the group. Setup is two steps:

1. Server creates a **report group record** with an `activationKey`.
2. Owner adds the bot to a Telegram group and types `/activate <activationKey>`
   in that group. The bot writes back its `chatId` and flips the status to
   `ACTIVE`.

### Create the record (get an activation key)
```graphql
mutation CreateReportGroup($shopId: String!) {
  createReportGroup(shopId: $shopId) {
    id shopId botId name chatId status activationKey
  }
}
```
- `status` starts as `PENDING`.
- `chatId` / `name` are filled in once the owner activates inside the group.

### List groups
```graphql
query ReportGroups($shopId: String!) {
  getReportGroup(shopId: $shopId) {
    id name chatId status activationKey
  }
}
```
`ReportGroupStatus`: `PENDING | ACTIVE | PAUSE`.

### Pause / resume a group
```graphql
mutation ToggleReportGroup($shopId: String!, $groupId: String!, $status: Boolean!) {
  toggleReportGroup(shopId: $shopId, groupId: $groupId, status: $status)
}
```
- `status: true` → flip to `ACTIVE`.
- `status: false` → flip to `PAUSE` (the bot stops pushing notifications but
  keeps the activation).

### Delete a group
> ⚠️ **CONFIRMATION REQUIRED** — removes the bot's link to that group.
```graphql
mutation DeleteReportGroup($shopId: String!, $groupId: String!) {
  deleteReportGroup(shopId: $shopId, groupId: $groupId)
}
```

### Recipe — onboard a new shop's report group
1. `createReportGroup(shopId)` → grab `activationKey`.
2. Tell the owner: "Add @YourShopBot to your Telegram group and send
   `/activate <activationKey>`."
3. Poll `getReportGroup(shopId)` until `status: ACTIVE` (chatId populated).
4. Done — new-order notifications start flowing.

---

## Destructive operations (always confirm)

| Mutation | What it does |
|---|---|
| `createBroadcast` | Sends to **every customer** on the shop — confirm count + preview first. |
| `sendDirectMessage` | One-to-one Telegram push — confirm recipient + message preview. |
| `deleteReportGroup` | Removes the bot's link to a Telegram group. |

---

## Agent Notes

- **Image uploads**: every push needs a real CDN URL. Upload via `/uploads/s3`
  (standard) or `/uploads/s3/hq` (banner-quality) — see the section/products
  skills for the curl shape. Never invent a URL.
- **Bot required**: if the shop has no bot, `createBroadcast` and
  `sendDirectMessage` will fail. Check `shopId(shopId).activationStatus.telegramBot`
  first.
- **Caption length**: Telegram caps photo captions at 1024 chars. Trim long
  drafts before sending — the API will reject longer payloads.
- **Button URL**: Telegram requires `https://` URLs for inline buttons.
  Relative paths (`/shop`) will fail.
- **Audit trail**: every push (broadcast or DM) shows up in `broadcasts(shopId)`
  with `createdBy` and counters — use it to audit who sent what.
