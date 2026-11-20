# `sendoffer` v2 — full schema

`POST /api/v2/sendoffer` (v1 at `/api/v1/sendoffer` lacks dimensions, payer, prepayment % and comments). Source: api.merit.ee → Reference manual → Sales Offers → Create Sales Offer. Verified against a live company on 2026-10-15.

## Root fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `Customer` | CustomerObject | yes | `{ "Id": <guid> }` for an existing customer; otherwise a full object and Merit creates the customer |
| `OfferNo` | string(35) | **yes** | Unique. No "next number" endpoint; continue the company's series (UI default `P-1000NN`) |
| `DocDate` | YYYYMMDD | yes | Document date |
| `ExpireDate` | YYYYMMDD | yes | "Kehtiv kuni". For `DocType` 2 or 3 this is the due date |
| `DeliveryDate` | YYYYMMDD | yes | Planned delivery; shows as "Tarneaeg" |
| `DocType` | int | yes | 1 = quote (pakkumine), 2 = sales order (müügitellimus), 3 = prepayment invoice (ettemaksuarve) |
| `DocStatus` | int | yes | 1 created, 2 sent, 3 approved, 4 rejected, 5 comment received, 6 invoice created, 7 cancelled. Use 1 for a new document |
| `RefNo` | string(36) | no | Reference number; validate the 7-3-1 check digit yourself. Merit left it `null` on a quote |
| `CurrencyCode` | string(4) | no | Defaults to the customer's or the company's currency |
| `DepartmentCode` | string(20) | no | Must exist |
| `ProjectCode` | string(20) | no | v1 only; must exist |
| `Dimensions` | array of DimensionsObject | no | v2 only |
| `OfferRow` | array of OfferRowObject | yes | At least one row |
| `TaxAmount` | array of TaxObject | **yes** | One entry per distinct `TaxId`, summed from rounded rows |
| `TotalAmount` | decimal(18,2) | no | Net of VAT; server recomputes and compares |
| `RoundingAmount` | decimal(18,2) | no | Cash rounding; does not change `TotalAmount` |
| `Payment` | PaymentObject | no | Record a payment with the document (prepayment invoices) |
| `ContactInfo` | string(4K) | no | Contact person text printed on the document |
| `Hcomment` | string(4K) | no | Header comment (printed above the rows). Falls back to the customer card's default |
| `Fcomment` | string(4K) | no | Footer comment (terms, validity text). Falls back to the customer card's default |
| `ReserveItems` | bool | no | v2; reserve stock for stock items |
| `PrepaymPct` | decimal(18,2) | no | v2; prepayment percentage printed on the offer |
| `Payer` | PayerObject | no | v2; when the payer differs from the customer |
| `Comments` | array of `{ "Comment", "CommDate" }` | no | v2; internal comment log |
| `ProcCodes`, `PolDocType` | | no | Poland only |

## `OfferRowObject`

| Field | Type | Req | Notes |
|---|---|---|---|
| `Item` | ItemObject | yes | |
| `Quantity` | decimal(18,3) | yes | |
| `Price` | decimal(18,7) | yes | Net unit price |
| `DiscountPct` | decimal(18,2) | no | Whole-number percent |
| `DiscountAmount` | decimal(18,2) | no | `Quantity × Price × DiscountPct/100`, unrounded, subtracted before row rounding |
| `TaxId` | GUID | **yes** | From `gettaxes`. Required on every row, including 0 % |
| `LocationCode` | string(20) | no | Stock items in multi-warehouse companies |
| `DepartmentCode` | string(20) | no | Must exist |
| `GLAccountCode` | string(10) | no | Override the sales account for the row |
| `ItemCostAmount` | decimal(18,2) | cond. | Only for credit documents of stock items |
| `ProjectCode`, `CostCenterCode` | string(20) | no | v1 only |
| `Dimensions` | array | no | v2 only |
| `SalesAccCode`, `PurchaseAccCode`, `InventoryAccCode`, `CostAccCode` | string(10) | no | v2; `SalesAccCode` ignored when `GLAccountCode` is set |

## `ItemObject`

| Field | Type | Req | Notes |
|---|---|---|---|
| `Code` | string(20) | yes | Looked up; created if unknown and `Type` is present |
| `Description` | string(150) | yes | What prints on the row; truncated at 150 |
| `Type` | int | when adding | 1 stock, 2 service, 3 item (non-stock goods) |
| `UOMName` | string(64) | no | **Must match an existing unit's name** (`GET /api/v2/units`), else `Puudub mõõtühik nimega X` |
| `DefLocationCode` | string(20) | cond. | Stock items in multi-warehouse companies |
| `EANCode` | string(20) | no | |

## `CustomerObject`

Same as `sendcustomer` in `merit-aktiva-masters/references/customers.md`. Minimal new Estonian company:

```json
{ "Name": "Rada OÜ", "RegNo": "12112827", "VatRegNo": "EE101685146", "NotTDCustomer": false,
  "CountryCode": "EE", "Address": "Audru ringrada, Papsaare küla", "City": "Pärnu linn",
  "County": "Pärnu maakond", "PostalCode": "88317", "Email": "info@example.ee",
  "SalesInvLang": "ET", "PaymentDeadLine": 14 }
```

`NotTDCustomer` is `true` for private persons and foreign companies (they are left out of the KMD INF customer list).

## `TaxObject`, `PaymentObject`, `DimensionsObject`, `PayerObject`

Identical to `sendinvoice` (see `merit-aktiva-sales/references/sendinvoice-schema.md`). `PaymentObject.PaymDate` is `YYYYMMDDHHmm`.

## Response

```json
{ "CustomerId": "25014685-…", "InvoiceId": "9d0ca031-…", "InvoiceNo": "P-100011", "RefNo": null, "NewCustomer": null }
```

`InvoiceId` is the **offer's** Id. `NewCustomer` came back `null` even when the customer was created; check `CustomerId` against your cache instead.

## Rounding worked example

Eleven rows at 24 %: round each row's net to 2 dp (half-up), round each row's VAT to 2 dp, then sum. Row `6 × 193.60 = 1161.60`, VAT `278.78`; row `15 × 0.50 = 7.50`, VAT `1.80`. Totals net 4 280.70, VAT 1 027.37 (the unrounded 24 % of the net would be 1 027.368, so summing rounded rows matters once there are many small lines).

## Common errors

| Message | Cause | Fix |
|---|---|---|
| `Puudub mõõtühik nimega h.` | `UOMName` not in the unit list | `GET /api/v2/units`, map to an existing name or `senduom` |
| HTTP 500 `FormatException` on `getoffers` | A request field missing | Send all four: `DateType`, `PeriodStart`, `PeriodEnd`, `UnPaid` |
| 404 on `v1/getoffers` | Endpoint is v2 only | Use `/api/v2/getoffers` |
| Duplicate `OfferNo` | Number already used | List recent offers and pick the next free number |
