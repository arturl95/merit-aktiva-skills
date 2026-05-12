# Purchase invoices — full reference

## `POST /api/v2/sendpurchinvoice`

### Root fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `Vendor` | VendorObject | yes | By `Id` (preferred) or fully-specified new vendor |
| `DocNo` | string(35) | yes | The **vendor's** invoice number |
| `DocDate` | YYYYMMDD | yes | Issue date on the vendor's invoice |
| `TransactionDate` | YYYYMMDD | no | Date the supply was performed; defaults to `DocDate` |
| `DueDate` | YYYYMMDD | no | Defaults to `DocDate + vendor.PaymentDeadLine` |
| `CurrencyCode` | string(4) | no | ISO; defaults to vendor's currency or company default |
| `CurrencyRate` | decimal(18,7) | no | v2: ECB rate of `DocDate` if omitted |
| `RefNo` | string(36) | no | Vendor's reference number (e.g. their payment reference) |
| `ContractNo` | string(35) | no | |
| `InvoiceRow` | array of InvoiceRowObject | yes | At least one |
| `TaxAmount` | array of TaxObject | yes | Grouped by `TaxId`, summed |
| `TotalAmount` | decimal(18,2) | no | Net of VAT |
| `RoundingAmount` | decimal(18,2) | no | |
| `Hcomment`, `Fcomment` | strings | no | |
| `ExpenseClaim` | bool | no | `true` → expense claim semantics |
| `Attachment` | object | no | `{ FileName, FileContent: <base64> }` — e.g. scanned receipt |
| `DepartmentCode`, `ProjectCode` | string(20) | no | |
| `Dimensions` | array | no | v2 |

### `InvoiceRowObject` (purchase side)

| Field | Type | Req | Notes |
|---|---|---|---|
| `Item` | ItemObject | conditional | Required for stock-item purchases |
| `Description` | string(150) | yes when no `Item` | The row label that prints |
| `Quantity` | decimal(18,3) | yes | |
| `Price` | decimal(18,7) | yes | |
| `DiscountPct` | decimal(18,2) | no | |
| `TaxId` | GUID | yes | From `gettaxes` cache |
| `GLAccountCode` | string(10) | recommended | Expense account; without it, default routing applies |
| `LocationCode` | string(20) | conditional | Required for stock items |
| `DepartmentCode`, `ProjectCode` | strings | no | |
| `Dimensions` | array | no | v2 |

### Response

```json
{ "VendorId": "...", "PurchInvoiceId": "...", "DocNo": "...", "NewVendor": false }
```

## Reverse-charge purchase invoice — worked example

Buying €100 of consulting from a German supplier (intra-EU service, Art. 44 reverse charge). The vendor charges 0% VAT; you self-account for 24% Estonian VAT on both sides (input and output) so net VAT cash is zero, but both KMD lines are populated.

```json
POST /api/v2/sendpurchinvoice
{
  "Vendor": { "Id": "<DE-vendor-guid>" },
  "DocNo": "DE-2026-INV-7788",
  "DocDate": "20260508",
  "DueDate": "20260522",
  "CurrencyCode": "EUR",
  "InvoiceRow": [
    {
      "Description": "Consulting (Berlin team) — May",
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

## Approval queue — `sendpurchinvoiceforapproval`

Same payload shape. The invoice lands in `Purchases → Approval queue` rather than the active ledger. Approvers see it in the Merit UI and approve/reject; on approval it posts normally.

Use cases:
- 4-eyes principle (finance must approve invoices over €X).
- E-invoice inbound flow — incoming e-invoices automatically land here (per support docs); your AP team triage and approves.

The API does not currently expose programmatic approval; approval happens in the UI or via Merit's mobile app.

## Listing purchase invoices

```json
POST /api/v2/getpurchaseinvoices
{ "PeriodStart": "20260101", "PeriodEnd": "20260512", "UpdatedDate": null }
```

For incremental sync, use the `UpdatedDate` high-water-mark cursor pattern.

## Get single purchase invoice with rows and attachments

```json
POST /api/v2/getpurchaseinvoice
{ "Id": "<purch-invoice-guid>", "AddAttachment": true }
```

Response includes header, rows (with TaxId/account/quantity/price), payments allocated against it, and (if `AddAttachment: true`) the base64-encoded scanned receipt.

## Delete

```json
POST /api/v2/deletepurchinvoice
{ "Id": "<purch-invoice-guid>" }
```

Constraints: not allowed if there are payments allocated against it. Reverse the payment first via `deletepayment` (where permitted) or post a reversing payment.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/purchase-invoices/create-purchase-invoice/
