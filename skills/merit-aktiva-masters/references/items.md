# Items — full reference

Items represent both physical stock and services in Merit Aktiva. The `Type` and `Usage` enums determine how the item behaves on invoices, in inventory, and in VAT declarations.

## `POST /api/v2/senditems` (batch)

The endpoint name is plural — it accepts a batch in `{ "Items": [ ... ] }`.

### Item object fields

| Field | Type | Req (create) | Notes |
|---|---|---|---|
| `Code` | string(20) | yes | Unique per company. **Different VAT rates require different codes** so KMD INF reconciles. |
| `Description` | string(100) | yes | Primary description. Truncated if too long. |
| `Type` | enum | yes | 1 = stock, 2 = service, 3 = item (non-stock physical) |
| `Usage` | enum | yes | 1 = sales, 2 = purchases, 3 = both |
| `UOMName` | string(64) | yes for stock | Unit of measure (`tk`, `h`, `kg`, …). Must exist or be added via `senduom`. |
| `EANCode` | string | no | Barcode |
| `DefLocationCode` | string(20) | required if multi-stock company | Default warehouse |
| `DescriptionEN`, `DescriptionRU`, `DescriptionFI` | string(100) | no | Translations |
| `TaxId` | GUID | recommended | Default VAT code from `gettaxes` cache |
| `ItemGrCode` / `ItemGrId` | string / GUID | no | Item group |
| `GTUCode` | enum | no | PL-only sales tag (1–13) |
| `SalesAccCode` | string(10) | no | GL account override for sales (default from chart of accounts) |
| `PurchaseAccCode` | string(10) | no | GL account override for purchases |
| `InventoryAccCode` | string(10) | no | Stock asset account |
| `CostAccCode` | string(10) | no | Cost-of-goods-sold account |

### Response

```json
[
  { "ItemId": "guid", "Code": "CONSULT-HR" },
  { "ItemId": "guid", "Code": "WIDGET-A" }
]
```

## `POST /api/v1/getitems`

Filter fields:

| Field | Type | Notes |
|---|---|---|
| `Id` | GUID | Exclusive |
| `Code` | string | Broad case-insensitive substring |
| `Description` | string | Broad |
| `LocationCode` | string | When set, returns current on-hand qty for that location |
| `Usage` | int | 1/2/3 |
| `Type` | int | 1/2/3 |

Response object fields: `ItemId`, `Code`, `Name`, `NameEN`, `NameFI`, `NameRU`, `UnitofMeasureName`, `Type0` (numeric), `Type` (text), `SalesPrice`, `InventoryQty`, `ReservedQty`, `VatTaxName`, `Usage0`, `Usage`, `SalesAccountCode`, `PurchaseAccountCode`, `InventoryAccountCode`, `ItemCostAccountCode`, `DiscountPct`, `LastPurchasePrice`, `ItemUnitCost`, `InventoryCost`, `ItemGroupName`, `DefLoc_Name`, `EANCode`, `GTUCodes`.

## When to use each `Type`

- **`Type: 1` (stock)** — physical goods with inventory tracking. Requires `UOMName`, `InventoryAccCode`, and (in multi-stock companies) `DefLocationCode`. The system tracks quantity and weighted-average / FIFO cost.
- **`Type: 2` (service)** — labour, consulting, subscriptions. No inventory. Use this for the vast majority of B2B SaaS line items.
- **`Type: 3` (item)** — non-stock physical (e.g. one-off resale where you don't want stock tracking).

## When to use each `Usage`

- **`Usage: 1` (sales only)** — appears in sales-invoice item pickers, hidden from purchase pickers.
- **`Usage: 2` (purchases only)** — purchase pickers only. Useful for cost-side items (e.g. specific expense categories).
- **`Usage: 3` (both)** — appears everywhere. Default for most services.

## Item code naming for VAT

Because **different VAT rates require different item codes**, name items so the VAT rate is implicit, e.g.:

- `CONSULT-HR` — consulting hourly @ 24%
- `BOOK` — books @ 9%
- `HOTEL` — accommodation @ 13%
- `EXPORT-CONSULT-HR` — consulting hourly @ 0% (export of services)
- `EU-CONSULT-HR` — consulting hourly @ 0% (intra-EU B2B)

This keeps the KMD INF reconciliation per item code straightforward.

## Multi-location stock

If the company has multiple warehouses (`Settings → Locations`), every stock item (`Type: 1`) must declare a `DefLocationCode`. Fetch the list via `POST /api/v1/getlocations` (returns `Code`, `Name`).

## Rich example

```json
POST /api/v2/senditems
{
  "Items": [
    {
      "Code": "CONSULT-HR",
      "Description": "Consulting (hourly)",
      "DescriptionEN": "Consulting (hourly)",
      "DescriptionRU": "Консультации (почасово)",
      "Type": 2,
      "Usage": 1,
      "UOMName": "h",
      "TaxId": "24pct-tax-guid",
      "ItemGrCode": "SERVICES",
      "SalesAccCode": "30000"
    },
    {
      "Code": "WIDGET-A",
      "Description": "Widget A",
      "EANCode": "4751111111110",
      "Type": 1,
      "Usage": 3,
      "UOMName": "tk",
      "DefLocationCode": "TLN",
      "TaxId": "24pct-tax-guid",
      "ItemGrCode": "GOODS",
      "SalesAccCode": "30100",
      "PurchaseAccCode": "52100",
      "InventoryAccCode": "13700",
      "CostAccCode": "52200"
    }
  ]
}
```

## Source

- https://api.merit.ee/connecting-robots/reference-manual/items/add-items/
- https://api.merit.ee/connecting-robots/reference-manual/items/items-list/
