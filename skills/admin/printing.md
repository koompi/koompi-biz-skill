# Printing (Receipts, Auto-Print, Printer Setup)

Riverbase ships its own receipt-printing pipeline backed by RabbitMQ. Each
shop is a multi-tenant printing **tenant** with its own vhost + credentials.
Inside a tenant, the merchant registers **providers** (consumer servers
running at physical stock locations) and adds **printers** to each provider.
A print **job** publishes an order to a printer's queue; the consumer renders
the receipt and reports back.

### Building blocks

| Concept | What it is |
|---|---|
| `PrintingTenant` | Per-shop RabbitMQ vhost + user. One per shop. |
| `PrintingProvider` | A consumer server at a physical location (e.g. "Riverside POS"). |
| `PrintingPrinter` | A printer attached to a provider, with its own paper size + template. |
| `PrintingJob` | One queued print attempt for a given order on a given printer. |
| `AutoPrintConfig` | Per-shop rule list — auto-publish jobs when a payment transitions. |

---

## 1. Provision the tenant (one time per shop)

```graphql
mutation ProvisionTenant($shopId: String!) {
  printingProvisionTenant(shopId: $shopId) {
    id shopId vhost username rabbitmqUri createdAt active
  }
}
```
Creates the RabbitMQ vhost + user + permissions. `rabbitmqUri` is the
credential the consumer server needs to connect.

> Run this **before** any provider/printer setup. The tenant is idempotent —
> calling it twice returns the existing tenant.

### Read current tenant
```graphql
query Tenant($shopId: String!) {
  printingTenant(shopId: $shopId) {
    id vhost username rabbitmqUri active
  }
}
```

### Deprovision (⚠️ removes ALL providers + printers)
> ⚠️ **CONFIRMATION REQUIRED** — wipes the entire printing setup for the shop.
```graphql
mutation Deprovision($shopId: String!) {
  printingDeprovisionTenant(shopId: $shopId)
}
```

---

## 2. Register a provider (consumer server)

```graphql
mutation RegisterProvider($shopId: String!, $name: String!, $location: String!) {
  printingRegisterProvider(shopId: $shopId, name: $name, location: $location) {
    id shopId name location mgmtQueue active createdAt
  }
}
```
- `name`: human label (e.g. `"Riverside POS"`).
- `location`: stock-location identifier (often the warehouse / store name).
- `mgmtQueue`: the management-queue name the consumer subscribes to for
  AddPrinter / RemovePrinter commands.

### List providers
```graphql
query Providers($shopId: String!) {
  printingProviders(shopId: $shopId) {
    id name location mgmtQueue active
  }
}
```

### Remove a provider (cascade-deletes its printers)
> ⚠️ **CONFIRMATION REQUIRED**
```graphql
mutation RemoveProvider($providerId: String!) {
  printingRemoveProvider(providerId: $providerId)
}
```

---

## 3. Add a printer to a provider

```graphql
mutation AddPrinter($providerId: String!, $printerName: String!, $systemName: String) {
  printingAddPrinter(providerId: $providerId, printerName: $printerName, systemName: $systemName) {
    id printerName systemName queueName active createdAt
  }
}
```
- `printerName`: human label (`"Kitchen 80mm"`).
- `systemName`: OS-level printer name (`"EPSON_TM_T20"`); when omitted, the
  consumer picks the OS default.

Sends an `AddPrinter` command over the provider's management queue — the
consumer registers the printer with the OS and reports back. The returned
`queueName` is the printer's job queue (used by `printingPublishJob`).

### List printers
```graphql
query Printers($shopId: String!) {
  printingPrinters(shopId: $shopId) {
    id providerId printerName systemName queueName active
    paperWidthMm paperHeightMm marginTopMm marginBottomMm marginLeftMm marginRightMm
    orientation fontSizePt dpi tracking qrModulePixels templateJson
  }
}
```
Or scoped to one provider: `printingPrintersByProvider(providerId: String!)`.

### Configure a printer (paper, margins, font, template)

```graphql
mutation ConfigurePrinter(
  $printerId: String!
  $paperWidthMm: Float!
  $paperHeightMm: Float
  $marginTopMm: Float!
  $marginBottomMm: Float!
  $marginLeftMm: Float!
  $marginRightMm: Float!
  $orientation: String! = "portrait"
  $fontSizePt: Float! = 8
  $dpi: Float! = 203
  $tracking: Float! = 0.82
  $qrModulePixels: Int! = 4
  $templateJson: String
) {
  printingConfigurePrinter(
    printerId: $printerId
    paperWidthMm: $paperWidthMm paperHeightMm: $paperHeightMm
    marginTopMm: $marginTopMm marginBottomMm: $marginBottomMm
    marginLeftMm: $marginLeftMm marginRightMm: $marginRightMm
    orientation: $orientation
    fontSizePt: $fontSizePt dpi: $dpi
    tracking: $tracking qrModulePixels: $qrModulePixels
    templateJson: $templateJson
  ) { id paperWidthMm fontSizePt }
}
```

Typical 80mm thermal printer (continuous roll):
```json
{
  "paperWidthMm": 80,
  "paperHeightMm": null,
  "marginTopMm": 3, "marginBottomMm": 3,
  "marginLeftMm": 2, "marginRightMm": 2,
  "orientation": "portrait",
  "fontSizePt": 9, "dpi": 203,
  "tracking": 0.82, "qrModulePixels": 4
}
```

`templateJson` (optional): a JSON string with merchant-defined receipt
template overrides (header text, logo position, footer note). When `null` the
consumer uses its built-in default receipt layout.

### Test print
Sends a test page through the printer:
```graphql
mutation TestPrint($printerId: String!) {
  printingTestPrint(printerId: $printerId)
}
```

### Remove a printer
> ⚠️ **CONFIRMATION REQUIRED**
```graphql
mutation RemovePrinter($printerId: String!) {
  printingRemovePrinter(printerId: $printerId)
}
```

---

## 4. Publish a print job (print an order)

```graphql
mutation PublishJob($shopId: String!, $printerId: String!, $orderId: String!, $priority: Priority!) {
  printingPublishJob(shopId: $shopId, printerId: $printerId, orderId: $orderId, priority: $priority) {
    id orderId printerId queueName status priority createdAt
  }
}
```
`Priority`: `LOW | NORMAL | HIGH`. The consumer fetches the order itself —
only the `orderId` is sent on the wire.

### List jobs

```graphql
query Jobs($shopId: String!, $status: PrintJobStatus, $limit: Int!, $offset: Int!) {
  printingJobs(shopId: $shopId, status: $status, limit: $limit, offset: $offset) {
    id orderNumber printerName status priority retryCount
    createdAt startedAt completedAt errorMessage
  }
}
```
Filter by `status` (or pass `null` for all). Also:
- `printingJobsByOrder(orderId: String!): [PrintingJob!]!`
- `printingJobsByPrinter(printerId: String!): [PrintingJob!]!`

### Job status enum
```
PrintJobStatus: QUEUED | PRINTING | COMPLETED | FAILED | CANCELLED | DELIVERED
```
`DELIVERED` is a legacy alias kept for back-compat — treat as `COMPLETED`.

### Retry / cancel / reprint
```graphql
mutation Retry($jobId: String!)   { printingRetryJob(jobId: $jobId)   { id status retryCount } }
mutation Cancel($jobId: String!)  { printingCancelJob(jobId: $jobId)  { id status } }
mutation Reprint($jobId: String!) { printingReprintJob(jobId: $jobId) { id status } }
```
`printingReprintJob` creates a **new** job for the same order + printer
(the original is unchanged).

### Clear completed/failed/cancelled history
```graphql
mutation ClearHistory($shopId: String!) {
  printingClearJobHistory(shopId: $shopId)   # → number of jobs cleared
}
```
> ⚠️ **CONFIRMATION REQUIRED** — show the count first.

---

## 5. Auto-print rules

When the merchant doesn't want to publish jobs manually, save an
`AutoPrintConfig`. The server watches order payment transitions and fires
matching rules.

```graphql
mutation SaveAutoPrint($shopId: String!, $enabled: Boolean!, $rules: [AutoPrintRule!]!) {
  printingSaveAutoPrintConfig(shopId: $shopId, enabled: $enabled, rules: $rules) {
    id enabled rules { paymentStatus dueType fulfillmentType copies enabled }
    createdAt updatedAt
  }
}
```

### `AutoPrintRule` input
```json
{
  "paymentStatus": "PAID",            // required: PAID | UNPAID
  "dueType": null,                    // null = match all; or NOW | COD
  "fulfillmentType": null,            // null = match all; or DELIVERY | IN_STORE_PICKUP | ON_SITE | ONLINE_POS | DIGITAL
  "copies": 1,
  "enabled": true
}
```

### Recipes

**Auto-print one copy on every paid order:**
```json
{
  "enabled": true,
  "rules": [
    { "paymentStatus": "PAID", "dueType": null, "fulfillmentType": null, "copies": 1, "enabled": true }
  ]
}
```

**Auto-print two copies on PAID + in-store-pickup, one on PAID + delivery:**
```json
{
  "enabled": true,
  "rules": [
    { "paymentStatus": "PAID", "dueType": null, "fulfillmentType": "IN_STORE_PICKUP", "copies": 2, "enabled": true },
    { "paymentStatus": "PAID", "dueType": null, "fulfillmentType": "DELIVERY",        "copies": 1, "enabled": true }
  ]
}
```

**Auto-print on COD order creation (UNPAID + COD):**
```json
{
  "enabled": true,
  "rules": [
    { "paymentStatus": "UNPAID", "dueType": "COD", "fulfillmentType": null, "copies": 1, "enabled": true }
  ]
}
```

### Delete the auto-print config
> ⚠️ **CONFIRMATION REQUIRED**
```graphql
mutation DeleteAutoPrint($shopId: String!) {
  printingDeleteAutoPrintConfig(shopId: $shopId)
}
```

---

## End-to-end agent recipe: "set up receipt printing for my shop"

1. **Provision the tenant** — `printingProvisionTenant(shopId)` → returns `rabbitmqUri`. Share the URI with the merchant; they paste it into their consumer-server config.
2. **Register a provider** — `printingRegisterProvider(shopId, name, location)` → returns the `mgmtQueue` the consumer listens on.
3. **Add a printer** — `printingAddPrinter(providerId, printerName, systemName)` over the management queue. The consumer reports back when it succeeds.
4. **Configure the printer** — `printingConfigurePrinter(printerId, paperWidthMm: 80, marginTopMm: 3, …)` to match the merchant's paper.
5. **Test print** — `printingTestPrint(printerId)` to verify the chain end-to-end.
6. **(optional) Auto-print** — `printingSaveAutoPrintConfig(enabled: true, rules: [...])` if the merchant wants hands-off receipts.
7. **Manual print** — for one-off receipts: `printingPublishJob(shopId, printerId, orderId, priority: NORMAL)`.

---

## Destructive operations (always confirm)

| Mutation | What it does |
|---|---|
| `printingDeprovisionTenant` | Wipes the whole printing setup for a shop. |
| `printingRemoveProvider` | Deletes a provider and ALL its printers. |
| `printingRemovePrinter` | Removes a printer (sends RemovePrinter to the consumer). |
| `printingCancelJob` | Cancels a queued job. |
| `printingClearJobHistory` | Bulk-deletes completed/failed/cancelled jobs. |
| `printingDeleteAutoPrintConfig` | Removes the auto-print config (printing returns to manual). |
