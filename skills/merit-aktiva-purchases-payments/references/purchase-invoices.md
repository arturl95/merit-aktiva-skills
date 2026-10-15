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

### Verified posting behaviour (live tests, 2026-10)

- **Item with a unit of measure:** if the item has a unit (e.g. `tk`), you must send `Item.UOMName` with it, otherwise 400 `"Item with code mat has unit tk in the program."`
- **Company not VAT-registered** (Ettevõtte andmed → KM kohustuslane ≠ Jah): Merit silently adds the VAT into the row cost (price 10 + 24% booked as 12.40 with tax "-"). Check the company setting before blaming the payload.
- **EU vendor (CountryCode in EU + VatRegNo) with a normal "24%" TaxId:** Merit applies the reverse charge automatically (D 2314 / C 2310). No special tax code needed.
- **Foreign currency:** `CurrencyRate` omitted gives the ECB rate of `DocDate` (verified against the ECB API). GL batch date = `TransactionDate`.
- **Import VAT** (forwarder/customs invoice): the "toll" TaxId is ignored via the API (the base went to the row account). What works is a row with `GLAccountCode` 2313 (Impordilt tasutud km), the VAT amount as price, and tax "-". It lands on KMD line 5.1.
- **The invoice `BankAccount` is not written to the vendor card and not returned by `getpurchorder`**, but Merit's payment-order screen uses it as the payee IBAN. If you need an IBAN-change check, keep your own history.
- Duplicate vendor + `BillNo` → 400 "Duplicate invoice number."; the same `BillNo` for a different vendor is accepted.


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

Same payload shape as `sendpurchinvoice`, v1 and v2. The invoice lands in **Ost → Kinnitamata ostuarved** (unconfirmed purchase invoices) instead of posting. Approvers confirm it in the Merit UI or mobile app; on confirmation it posts normally. The API has no approve call.

Use cases:
- 4-eyes principle (finance must approve invoices over €X).
- AI/OCR pipelines: the machine prepares, the bookkeeper confirms against the original.

> **Note:** The endpoint is `sendpurchorder`, not `sendpurchinvoiceforapproval`. Using the wrong name will result in a 404.

### Verified behaviour (live tests, 2026-10)

| Field / behaviour | Result |
|---|---|
| `DocDate`, `DueDate`, `TransactionDate` as `YYYYMMDD` | Kept (Kuupäev, Maksetähtpäev, Kande kuupäev filled) |
| Same dates as ISO `YYYY-MM-DD` | **Silently dropped.** 200 OK, but all three dates are empty in the UI and the rate is 0 |
| `CurrencyCode` (e.g. `USD`) | Kept |
| `CurrencyRate` | **Ignored** in v1 and v2. Merit sets its own rate for the document date (sent 0.9, UI showed 1.1269000 for USD). If no valid date arrives, the rate stays 0.0000000 |
| `BankAccount` | Kept (Pangakonto) |
| Per-row `GLAccountCode` | Kept, so different accounts per row work (e.g. materials row → 1310, other row → 4050) |
| `Vendor` given as new vendor (Name/RegNo) | Vendor card is **not** created; the response `VendorId` is `00000000-0000-0000-0000-000000000000`. The UI shows the name as text, and the vendor is created or matched on approval |
| Same `BillNo` + same vendor sent twice | **Rejected**: 400 `{"Message":"Duplicate invoice number."}`, already at queue level |

| `TransactionDate` (booking/"Kande kuupäev") ≠ `DocDate` | **v2 keeps it; v1 ignores it** (booking date = DocDate). Rate is still ECB of `DocDate`. Use v2 when posting date must differ (e.g. non-EU goods booked at receipt date) |
| Existing vendor (matched by name/RegNo) | `VendorId` returned |

### Approving a queued invoice (UI) — what the approver must do

- **Every row needs a Merit item (Artikkel) before "Salvesta ostuarvena"**, otherwise you get "Artikkel on kohustuslik". The API's `Item.Code` lands only in "Algdok. artikkel" (source item); the Artikkel column starts empty, even when the code exists in Merit.
- **Choosing an Artikkel overwrites the row account with that item's purchase account** (e.g. 1310 → 4002). Design items as one per target account (Merit's own rule) and send those codes, so the mapped item and your `GLAccountCode` agree.
- After the first mapping, **later invoices from the same vendor with the same source codes are auto-mapped**, with accounts intact. Mapping is **per vendor**: a new vendor with the same codes is not auto-mapped.
- With kinnitusring on (Ettevõtte andmed → "Ettevõttes on kasutusel kinnitusring"), "Salvesta ostuarvena" stays disabled until an approver confirms: Lisa kinnitaja → Saada kinnitamisele (emails the approver) → Kinnita.
- `sendpurchinvoice` posts directly even when kinnitusring is on. Only the queue goes through approval.

**Queued invoices are invisible to the API.** `getpurchorder` with the returned `BillId` gives 400 `"Pole sellist arvet."`. `getpurchorders` does not list them, and `GetPOrders` (that's purchase *orders*) doesn't either. `deletepurchinvoice/{BillId}` gives 404 `api-wrongguid`. Consequences:
- Keep your own log of what you sent (BillId, BillNo, vendor, amounts).
- Any check on the booked values (rate, accounts) is possible only after approval, via `getpurchorders` / `getpurchorder`.
- Test invoices must be deleted by hand in the UI.

### E-invoice variant — `sendpurchorderxml`

```
POST /api/v2/sendpurchorderxml
Content-Type: application/xml
<raw Estonian e-invoice 1.2 XML (E_Invoice root)>
```

The body is the raw XML file, signed like any other body. The response is XML (`ArrayOfSendPurchInvoiceResponse` with `BillId`, `BillNo`, `VendorId`). Verified:
- Kept: `InvoiceDate`, `DueDate`, currency, and the seller's IBAN (`AccountInfo`/`PayToAccount`). Merit sets the rate itself.
- **Rows arrive without item and without account.** The row description becomes the item text, and the VAT rate is mapped from `VATRate`. The approver has to set the account per row.
- So for machine-coded invoices (account per row), prefer the JSON `sendpurchorder`.

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
POST /api/v2/deletepurchinvoice/{invoiceId}
{}
```

The ID goes in the URL path (per the official reference page); the body is `{}`. Only posted invoices can be deleted this way. Queued (`sendpurchorder`) invoices return 404 `api-wrongguid`.

Constraints: not allowed if there are payments allocated against it. Reverse the payment first via `deletepayment` (where permitted) or post a reversing payment.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/purchase-invoices/create-purchase-invoice/
- https://api.merit.ee/connecting-robots/reference-manual/purchase-invoices/create-purchase-order/
- https://api.merit.ee/connecting-robots/reference-manual/purchase-invoices/create-purchase-invoice-waiting-approval-of-estonian-e-invoice-standard-1-2/
- https://api.merit.ee/connecting-robots/reference-manual/purchase-invoices/delete-purchase-invoice/
- Live tests against a Merit test company, 2026-10 (see "Verified behaviour")
