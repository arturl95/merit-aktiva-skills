# `sendinvoice` v2 — full schema

`POST /api/v2/sendinvoice`. Use v2 unless you specifically need v1 behaviour; v2 adds dimensions, currency rate auto-fill, payer object, file attachments, and Polish KSeF fields.

## Root fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `Customer` | CustomerObject | yes | By `Id` (recommended) or fully-specified new customer |
| `InvoiceNo` | string(35) | yes | Unique. No server-side locking. |
| `AccountingDoc` | enum | no | 1=invoice, 2=rachunek (PL), 3=paragon (PL), 4=no doc, **5=credit invoice**, 6=prepayment invoice, 7=finance charge, 8=delivery order, 9=group invoice. Default 1. |
| `DocDate` | YYYYMMDD | no | Defaults to today |
| `DueDate` | YYYYMMDD | no | Defaults to `DocDate + customer.PaymentDeadLine` |
| `TransactionDate` | YYYYMMDD | no | Date the supply was performed; defaults to `DocDate` |
| `RefNo` | string(36) | no | Auto-generated viitenumber if omitted |
| `CurrencyCode` | string(4) | no | ISO; defaults to customer's `CurrencyCode` or company default |
| `CurrencyRate` | decimal(18,7) | no | v2: ECB rate of `DocDate` if omitted |
| `DepartmentCode` | string(20) | no | Must exist |
| `ProjectCode` | string(20) | no | Must exist |
| `InvoiceRow` | array of InvoiceRowObject | yes | At least one |
| `TaxAmount` | array of TaxObject | yes | Grouped by `TaxId`, summed across rows |
| `TotalAmount` | decimal(18,2) | no | Net of VAT (server recomputes; passing it explicitly is a sanity check) |
| `RoundingAmount` | decimal(18,2) | no | Cash rounding adjustment |
| `Payment` | PaymentObject | no | Marks invoice paid in one go |
| `Hcomment` | string(4K) | no | Header comment (printed top) |
| `Fcomment` | string(4K) | no | Footer comment (printed bottom) |
| `ContractNo` | string(35) | no | Reference to a contract |
| `PDF` | string | no | Base64 of a custom PDF to attach |
| `FileName` | string(100) | no | v2 — file name for the PDF |
| `Payer` | PayerObject | no | v2 — when payer differs from invoice customer |
| `ReserveItems` | bool | no | v2 — reserve stock immediately |
| `Dimensions` | array of `{ DimCode, DimValueId }` | no | v2 |
| `ProcCodes` | array of strings | no | PL only: SW, EE, TP, TT_WNT, TT_D, MR_T, MR_UZ, I_42, I_63, B_SPV, B_SPV_DOSTAWA, B_MRV_PROWIZJA, MPP, WSTO_EE, IED |
| `PolDocType` | enum | no | PL only: 1=RO, 2=WEW, 3=FP, 4=OJPK |
| `KsefNumber` | string(36) | no | PL only |

## `InvoiceRowObject`

| Field | Type | Req | Notes |
|---|---|---|---|
| `Item` | ItemObject | yes | See below |
| `Quantity` | decimal(18,3) | yes | |
| `Price` | decimal(18,7) | no | If omitted, looked up from the item's price table |
| `DiscountPct` | decimal(18,2) | no | Whole-number percent (5 = 5%) |
| `DiscountAmount` | decimal(18,2) | no | Computed by server if `DiscountPct` set |
| `TaxId` | GUID | yes | From `gettaxes` cache. Required on every row, even 0%. |
| `LocationCode` | string(20) | no | For stock items |
| `DepartmentCode` | string(20) | no | |
| `GLAccountCode` | string(10) | no | Override default sales account |
| `ProjectCode` | string(20) | no | |
| `ItemCostAmount` | decimal(18,2) | conditional | **Required for credit-invoice rows of stock items.** |
| `VatDate` | YYYYMMDD | no | Tax point if different from invoice date |
| `Dimensions` | array | no | v2 |

## `ItemObject` (within row)

| Field | Type | Req | Notes |
|---|---|---|---|
| `Code` | string(20) | yes | Looked up by code; if not found and other fields supplied, created on the fly |
| `Description` | string(150) | yes | Truncated if longer |
| `Type` | enum | yes when adding | 1=stock, 2=service, 3=item |
| `UOMName` | string(64) | no | Unit of measure |
| `DefLocationCode` | string(20) | conditional | Required for stock items in multi-stock companies |
| `GTUCode` | enum | no | PL-only (1–13) |
| `SalesAccCode` | string(10) | no | v2; override |
| `PurchaseAccCode` | string(10) | no | v2 |
| `InventoryAccCode` | string(10) | no | v2 |
| `CostAccCode` | string(10) | no | v2 |

## `CustomerObject` (within invoice)

If `Id` is present, all other fields are ignored. Otherwise, the same shape as `merit-aktiva-masters/references/customers.md` `sendcustomer` request.

## `TaxObject` (within `TaxAmount`)

| Field | Type | Req |
|---|---|---|
| `TaxId` | GUID | yes |
| `Amount` | decimal(18,2) | yes |

**Important**: group rows by `TaxId`, sum the per-row VAT (each row's `Quantity * Price * (1 - DiscountPct/100) * (TaxPct/100)`, rounded to 2dp), then build one `TaxAmount` entry per distinct `TaxId`.

## `PaymentObject` (optional, marks invoice paid)

| Field | Type | Req | Notes |
|---|---|---|---|
| `PaymentMethod` | string(150) | yes | Must match a method configured in Merit (e.g. `Bank`, `Cash`) |
| `PaidAmount` | decimal(18,2) | yes | Including VAT |
| `PaymDate` | YYYYMMDD or YYYYMMDDHHmm | yes | |

## Response

```json
{
  "CustomerId": "...",
  "InvoiceId":  "...",
  "InvoiceNo":  "2026-0042",
  "RefNo":      "20260042",
  "NewCustomer": false
}
```

## Example 1 — minimal viable

```json
{
  "Customer": { "Name": "Acme OÜ", "RegNo": "12345678", "NotTDCustomer": false, "CountryCode": "EE" },
  "InvoiceNo": "2026-0001",
  "DocDate": "20260512",
  "DueDate": "20260612",
  "InvoiceRow": [
    {
      "Item": { "Code": "CONSULT-HR", "Description": "Consulting", "Type": 2 },
      "Quantity": 1,
      "Price": 100.00,
      "TaxId": "<24pct-tax-guid>"
    }
  ],
  "TaxAmount": [{ "TaxId": "<24pct-tax-guid>", "Amount": 24.00 }],
  "TotalAmount": 100.00
}
```

## Example 2 — paid invoice with header/footer comments

```json
{
  "Customer": { "Id": "8a3b...-..." },
  "InvoiceNo": "2026-0042",
  "DocDate": "20260512",
  "DueDate": "20260526",
  "InvoiceRow": [
    {
      "Item": { "Code": "CONSULT-HR", "Description": "Consulting (May)", "Type": 2 },
      "Quantity": 8,
      "Price": 100.00,
      "TaxId": "<24pct-tax-guid>"
    }
  ],
  "TaxAmount": [{ "TaxId": "<24pct-tax-guid>", "Amount": 192.00 }],
  "TotalAmount": 800.00,
  "Payment": { "PaymentMethod": "Bank", "PaidAmount": 992.00, "PaymDate": "20260512" },
  "Hcomment": "Thank you for your business.",
  "Fcomment": "Reference: contract 2025-04. Payment within 14 days."
}
```

## Example 3 — rich: multi-row, multi-VAT, dimensions, custom PDF

```json
{
  "Customer": { "Id": "8a3b...-..." },
  "InvoiceNo": "2026-0123",
  "DocDate": "20260512",
  "DueDate": "20260526",
  "TransactionDate": "20260512",
  "CurrencyCode": "EUR",
  "CurrencyRate": 1.0,
  "DepartmentCode": "MAIN",
  "ProjectCode": "P-100",
  "InvoiceRow": [
    {
      "Item": { "Code": "CONSULT-HR", "Description": "Consulting (hourly)", "Type": 2 },
      "Quantity": 12, "Price": 150.0, "DiscountPct": 10,
      "TaxId": "<24pct-tax-guid>",
      "GLAccountCode": "30100",
      "Dimensions": [{ "DimCode": "REGION", "DimValueId": "<region-value-guid>" }]
    },
    {
      "Item": { "Code": "BOOK", "Description": "Book sale (reduced VAT)", "Type": 3 },
      "Quantity": 5, "Price": 30.0,
      "TaxId": "<9pct-tax-guid>"
    }
  ],
  "TaxAmount": [
    { "TaxId": "<24pct-tax-guid>", "Amount": 388.80 },
    { "TaxId": "<9pct-tax-guid>",  "Amount":  13.50 }
  ],
  "TotalAmount": 1770.00,
  "Hcomment": "Q2 services + book stock",
  "PDF": "<base64-encoded-pdf>",
  "FileName": "INV-2026-0123.pdf"
}
```

VAT math for the rich example:
- Row 1: 12 × 150 × 0.9 = 1,620.00 net × 24% = 388.80 VAT.
- Row 2: 5 × 30 = 150.00 net × 9% = 13.50 VAT.
- TotalAmount (net) = 1,620 + 150 = 1,770.00. Grand total incl VAT = 2,172.30.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/sales-invoices/create-sales-invoice/
- https://api.merit.ee/connecting-robots/reference-manual/sales-invoices/get-invoice-details/
