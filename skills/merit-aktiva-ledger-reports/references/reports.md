# Reports

All report endpoints are `POST` and return JSON. Request payloads filter by date range or as-of date; responses are pre-aggregated.

## Available reports

| Endpoint | Version | Returns | Typical use |
|---|---|---|---|
| `getprofitrep` | v1 | P&L (income statement) | Monthly close, board reporting |
| `getbalancerep` | v1 | Balance sheet (financial position) | As-of snapshot for board / audit |
| `getcustdebtrep` | v1 | Customer debts (AR aging) | Collections workflow |
| `getcustpaymrep` | v2 | Customer payments | Cash receipts review |
| `getsalesrep` | v2 | Sales report (multiple report types) | Revenue analytics |
| `getpurchrep` | v2 | Purchase report (multiple report types) | Spend analytics |
| `getinventoryreport` | v2 | Stock balances + movements | Inventory close |

There is no documented general-purpose "trial balance" endpoint; build it client-side from `getglbatchesfull` if needed, or pull `getbalancerep` + `getprofitrep` for the period.

## `getprofitrep` — P&L (income statement)

```json
POST /api/v1/getprofitrep
{ "EndDate": "20260512", "PerCount": 1 }
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `EndDate` | YYYYMMDD | yes | Period end date |
| `PerCount` | int | yes | Number of months to include |
| `DepFilter` | string | no | Department filter |

Response structure: array of report lines, each with `RDid`, `Description`, `RowType` (1=description, 3=account turnover, 4=formula), `Balance` (array of period totals descending from EndDate), and `Details` (per-account breakdown with `AccountCode`, `AccountName`, `TypeId` [3=revenue, 4=expenses], `Balance`).

## `getbalancerep` — Balance sheet (financial position)

```json
POST /api/v1/getbalancerep
{ "EndDate": "20260512", "PerCount": 1 }
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `EndDate` | YYYYMMDD | yes | Balance date |
| `PerCount` | int | yes | Number of comparison periods |

Response structure: array of report lines, each with `RDid`, `Description`, `RowType` (1=description, 2=account balance, 4=formula), `Balance` (period totals), and `Details` (per-account with `TypeId` [1=assets, 2=liabilities]). The fundamental identity `Assets = Liabilities + Equity` should hold to ±€0.01.

## `getcustdebtrep`

```json
POST /api/v1/getcustdebtrep
{ "CustName": "", "DebtDate": "20260512" }
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `CustName` | string(150) | conditional | Required if `CustId` not used; `""` selects all customers |
| `CustId` | GUID | conditional | Required if `CustName` not used |
| `OverDueDays` | int | no | Filter for amounts overdue more than N days |
| `DebtDate` | YYYYMMDD | no | Defaults to current date if omitted |

Response per document (not per customer — one row per outstanding invoice/document):

| Field | Notes |
|---|---|
| `PartnerName` | Customer name |
| `PartnerId` | Customer GUID |
| `DocType` | SO, MA, SBx, PR, BA |
| `DocDate` | Document date |
| `DocNo` | Document number |
| `RefNo` | Reference number |
| `DueDate` | Payment due date |
| `TotalAmount` | Invoice total |
| `PaidAmount` | Amount paid |
| `UnPaidAmount` | Outstanding balance |
| `CurrencyCode` | Currency |
| `CurrencyRate` | Exchange rate |

To get all customers, pass `CustName: ""`. To sort by most overdue, filter client-side on `DueDate` vs today.

## `getcustpaymrep`

```json
POST /api/v2/getcustpaymrep
{ "PeriodStart": "20260501", "PeriodEnd": "20260512" }
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `CustName` | string(150) | no | Filter by customer name |
| `CustId` | GUID | no | Filter by customer GUID |
| `PeriodStart` | YYYYMMDD | no | Start of payment date range |
| `PeriodEnd` | YYYYMMDD | no | End of payment date range |
| `CurrncyCode` | string(4) | no | Filter by currency (note: field name typo in API — `CurrncyCode`) |

Lists customer payments received in the period. Useful for daily cash review.

Response includes `Data` (array of payment objects with `CustName`, `CustId`, `DocDate`, `DocNo`, `TotalAmount`, `DueDate`, `UnPaidAmount`, `OverDue`), `HasMore` (boolean), and `Id4More` (string for pagination). When `HasMore` is true, repeat the request with `Id4More` value until `HasMore` is false.

## `getsalesrep`

```json
POST /api/v2/getsalesrep
{
  "StartDate":  "20260101",
  "EndDate":    "20260512",
  "ReportType": 2
}
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `StartDate` | date | no | |
| `EndDate` | date | no | |
| `ReportType` | int | no | 1=by invoices, 2=by customers, 3=by articles, 4=by fixed assets, 5=by invoices (alt), 6=by invoices (alt) |
| `UserFilter` | string | no | |
| `CustGrpFilter` | string | no | Customer group filter |
| `CustFilter` | string | no | Customer filter |
| `ItemGrFilter` | string | no | Item group filter |
| `ItemFilter` | string | no | Item filter |
| `DepartFilter` | string | no | Department filter |
| `FixAssetFilter` | string | no | Fixed asset filter |

Response fields vary by `ReportType`. Type 2 (by customer): `CustomerId`, `CustomerName`, `RegNo`, `VatRegNo`, `Amount`, `VatAmount`, `TotalAmount`, etc. Type 3 (by article): `ItemId`, `ItemCode`, `ItemName`, `Quantity`, `Price`, `Amount`.

## `getpurchrep`

```json
POST /api/v2/getpurchrep
{
  "StartDate":  "20260101",
  "EndDate":    "20260512",
  "ReportType": 2
}
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `StartDate` | date | no | |
| `EndDate` | date | no | |
| `ReportType` | int | no | 1=by invoices, 2=by vendors, 3=by articles, 4=by fixed assets |
| `VendChoice` | int | no | |
| `VendGrpFilter` | string | no | Vendor group filter |
| `VendFilter` | string | no | Vendor filter |
| `ItemGrFilter` | string | no | Item group filter |
| `ItemFilter` | string/array | no | Item filter |
| `DepartFilter` | string/array | no | Department filter |
| `FixAssetFilter` | string/array | no | Fixed asset filter |
| `ByEntryNo` | boolean | no | |

Response fields vary by `ReportType`; include `VendorName`, `Amount`, `VatAmount`, `TotalAmount`, and others.

## `getinventoryreport`

```json
POST /api/v2/getinventoryreport
{ "RepDate": "20260512", "Location": "TLN" }
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `ArticleGroups` | array of strings | no | Filter by article group codes |
| `Location` | string | no | Location filter (note: field is `Location`, not `LocationCode`) |
| `RepDate` | YYYYMMDD | no | Report as-of date (note: field is `RepDate`, not `AsOfDate`) |
| `ShowZero` | boolean | no | Include items with zero quantity |
| `WithReservations` | boolean | no | Include reserved quantities |

Response per item: `ItemCode`, `EANCode`, `ItemName`, `LocName`, `Quantity`, `ReservedQuantity`, `UnitCode`, `Amount`, `Price`.

## Building a VAT reconciliation client-side

The most useful month-end report is "did my KMD agree with the GL"? Build it from:

1. `getinvoices` for the period — sum net and VAT per `TaxId`.
2. `getpurchaseinvoices` for the period — same.
3. `getbalancerep` as-of period-end — read account 2120 (output VAT) and 1230 (input VAT).
4. Cross-check: `output_VAT - input_VAT` from steps 1+2 should equal the net of GL accounts in step 3 movement for the period.

Discrepancies usually indicate:
- A miscoded VAT line.
- Reverse-charge bookings that didn't net correctly.
- A GL adjustment without a corresponding VAT entry.

See `examples/month-end-vat-reconciliation.md` for the full walk-through.

## Rate-limit tips

Each report can be heavy. The big ones (`getbalancerep`, `getsalesrep`, `getinventoryreport` over long periods) can each consume several seconds of compute and several KB of bandwidth. Don't loop them — pull once per period, cache locally.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/reports/income-statement/
- https://api.merit.ee/connecting-robots/reference-manual/reports/balance-sheet/
- https://api.merit.ee/connecting-robots/reference-manual/reports/customer-debts-report/
- https://api.merit.ee/connecting-robots/reference-manual/reports/customer-payment-report/
- https://api.merit.ee/connecting-robots/reference-manual/reports/sales-report/
- https://api.merit.ee/connecting-robots/reference-manual/reports/purchase-report/
- https://api.merit.ee/connecting-robots/reference-manual/reports/get-inventory-report/
