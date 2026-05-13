# Purchase invoices — full reference

## `POST /api/v2/sendpurchinvoice`

### Root fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `Vendor` | VendorObject | yes | By `Id` (preferred) or fully-specified new vendor |
| `BillNo` | string(35) | no | The **vendor's** invoice number |
| `DocDate` | YYYYMMDD | no | Issue date on the vendor's invoice |
| `TransactionDate` | YYYYMMDD | no | Date the supply was performed; defaults to `DocDate` |
| `DueDate` | YYYYMMDD | no | Defaults to `DocDate + vendor.PaymentDeadLine` |
| `CurrencyCode` | string(4) | no | ISO; defaults to vendor's currency or company default |
| `CurrencyRate` | decimal(18,7) | no | v2: ECB rate of `DocDate` if omitted |
| `RefNo` | string(36) | no | Vendor's reference number (e.g. their payment reference) |
| `BankAccount` | string(50) | no | Vendor bank account |
| `InvoiceRow` | array of InvoiceRowObject | no | Line items |
| `TaxAmount` | array of TaxObject | yes | Grouped by `TaxId`, summed |
| `TotalAmount` | decimal(18,2) | no | Net of VAT |
| `RoundingAmount` | decimal(18,2) | no | |
| `Hcomment`, `Fcomment` | strings | no | |
| `ExpenseClaim` | bool | no | `true` → expense claim semantics |
| `Attachment` | object | no | `{ FileName, FileContent: <base64> }` — e.g. scanned receipt |
| `DepartmentCode` | string(20) | no | Must exist in database |
| `Dimensions` | array of DimensionsObjects | no | v2; must exist in database |
| `Payment` | PaymentObject | no | Inline payment at time of entry |
| `Receiver` | ReceiverObject | no | v2 |

### `InvoiceRowObject` (purchase side)

Note: the row-level `Description` field is **not** a top-level InvoiceRowObject field in the Merit API. Description lives inside the `Item` sub-object. For expense lines without a stored item, omit `Item` and use `GLAccountCode` directly; the line description is taken from `Item.Description` if an item is supplied.

| Field | Type | Req | Notes |
|---|---|---|---|
| `Item` | ItemObject | no | Required for stock-item purchases; contains `Description` |
| `Quantity` | decimal(18,3) | no | |
| `Price` | decimal(18,7) | no | |
| `TaxId` | GUID | yes | From `gettaxes` cache |
| `GLAccountCode` | string(10) | no | Expense account; without it, default routing applies |
| `LocationCode` | string(20) | no | Required for stock items |
| `DepartmentCode` | string(20) | no | Must exist in database |
| `Dimensions` | array of DimensionsObjects | no | v2 |
| `VatDate` | string(20) | no | v2 |
| `SalesAccCode` | string(10) | no | v2 |
| `PurchaseAccCode` | string(10) | no | v2; ignored if `GLAccountCode` supplied |
| `InventoryAccCode` | string(10) | no | v2 |
| `CostAccCode` | string(10) | no | v2 |

### Response

```json
{ "VendorId": "...", "BillId": "...", "BillNo": "...", "RefNo": "...", "BatcInfo": {} }
```

## Reverse-charge purchase invoice — worked example

Buying €100 of consulting from a German supplier (intra-EU service, Art. 44 reverse charge). The vendor charges 0% VAT; you self-account for 24% Estonian VAT on both sides (input and output) so net VAT cash is zero, but both KMD lines are populated.

```json
POST /api/v2/sendpurchinvoice
{
  "Vendor": { "Id": "<DE-vendor-guid>" },
  "BillNo": "DE-2026-INV-7788",
  "DocDate": "20260508",
  "DueDate": "20260522",
  "CurrencyCode": "EUR",
  "InvoiceRow": [
    {
      "Item": { "Code": "CONSULT", "Description": "Consulting (Berlin team) — May", "Type": 2 },
      "Quantity": 1,
      "Price": 100.00,
      "GLAccountCode": "55300",
      "TaxId": "<EU-services-reverse-charge-tax-guid>"
    }
  ],
  "TaxAmount": [
    { "TaxId": "<EU-services-reverse-charge-tax-guid>", "Amount": 24.00 }
  ],
  "TotalAmount": 100.00
}
```

Merit auto-books:

- Dr 55300 Expense €100
- Dr 1230 Input VAT €24
- Cr 21xx AP €100
- Cr 2120 Output VAT €24 (the self-charge)

Net cash payable to vendor: €100. Net VAT cash position: 0 (input and output offset). KMD impact: line 7 (other reverse-charge acquisitions) + line 4 (VAT due 24%) + line 5 (deductible input VAT).

For intra-EU goods, use the `EL kaup pöördkm` tax code instead — same mechanics, hits KMD lines 6 and 6.1 (intra-EU acquisition of goods).

## Approval queue — `sendpurchorder`

Same payload shape. The invoice lands in the approval queue rather than posting directly to the active ledger. Approvers see it in the Merit UI and approve/reject; on approval it posts normally.

Use cases:
- 4-eyes principle (finance must approve invoices over €X).
- Bookkeepers approving purchase orders and expense claims before they hit the ledger.

The API does not currently expose programmatic approval; approval happens in the UI or via Merit's mobile app.

> **Note:** The endpoint is `sendpurchorder`, not `sendpurchinvoiceforapproval`. Using the wrong name will result in a 404.

## Listing purchase invoices — `getpurchorders`

```json
POST /api/v2/getpurchorders
{ "PeriodStart": "20260101", "PeriodEnd": "20260512", "DateType": 1, "UnPaid": false }
```

`DateType`: 0 = DocumentDate, 1 = ChangedDate. For incremental sync, use `DateType: 1` with a ChangedDate high-water-mark.

Response fields include: `PIHId`, `BillNo`, `DocumentDate`, `VendorId`, `VendorName`, `DueDate`, `CurrencyCode`, `TotalAmount`, `PaidAmount`, `Paid`, `ChangedDate`, and more.

## Get single purchase invoice with rows and attachments — `getpurchorder`

```json
POST /api/v2/getpurchorder
{ "Id": "<purch-invoice-guid>", "SkipAttachment": false }
```

Response includes header (with `BillNo`, `VendorName`, `TotalSum`, `PaidAmount`, etc.), line items (with `ArticleCode`, `Quantity`, `Price`, `AccountCode`, `Description`), payments allocated, and attachment (base64) if `SkipAttachment: false`.

## Delete

```json
POST /api/v2/deletepurchinvoice
{ "Id": "<purch-invoice-guid>" }
```

Constraints: not allowed if there are payments allocated against it. Reverse the payment first via `deletepayment` (where permitted) or post a reversing payment.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/purchase-invoices/create-purchase-invoice/
