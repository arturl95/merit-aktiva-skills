# Reports

All report endpoints are `POST` and return JSON. Request payloads filter by date range or as-of date; responses are pre-aggregated.

## Available reports

| Endpoint | Returns | Typical use |
|---|---|---|
| `getprofitloss` | P&L | Monthly close, board reporting |
| `getfinpos` | Balance sheet | As-of snapshot for board / audit |
| `getcustdebtrep` | Customer debts (AR aging) | Collections workflow |
| `getcustpaymrep` | Customer payments | Cash receipts review |
| `getsalesreport` | Sales summary (group by Customer/Item/Department/Project) | Revenue analytics |
| `getpurchasereport` | Purchase summary | Spend analytics |
| `getinventoryreport` | Stock balances + movements | Inventory close |

There is no documented general-purpose "trial balance" endpoint; build it client-side from `getglbatchesfull` if needed, or pull `getfinpos` + `getprofitloss` for the period.

## `getprofitloss`

```json
POST /api/v2/getprofitloss
{ "PeriodStart": "20260101", "PeriodEnd": "20260512" }
```

Response (abridged):

```json
{
  "PeriodStart": "20260101",
  "PeriodEnd":   "20260512",
  "Revenue":     [
    { "AccountCode": "3000", "Name": "Müügitulu", "Amount": 50000.00 },
    { "AccountCode": "3050", "Name": "EL müük",  "Amount":  5000.00 }
  ],
  "CostOfSales": [...],
  "Expenses":    [...],
  "GrossProfit":     45000.00,
  "OperatingProfit": 32000.00,
  "NetIncome":       30000.00
}
```

## `getfinpos`

```json
POST /api/v2/getfinpos
{ "AsOfDate": "20260512" }
```

Response groups accounts under Assets, Liabilities, Equity. Each group lists `{ AccountCode, Name, Amount }` and a group total. The fundamental identity `Assets = Liabilities + Equity` should hold to ±€0.01.

## `getcustdebtrep`

```json
POST /api/v1/getcustdebtrep
{ "AsOfDate": "20260512" }
```

Response per customer:

```json
{
  "CustomerId": "...",
  "Name":       "Acme OÜ",
  "TotalDebt":  3500.00,
  "Aging": {
    "0to30":   1500.00,
    "31to60":  1000.00,
    "61to90":   500.00,
    "Over90":   500.00
  },
  "Invoices": [
    { "InvoiceNo": "2026-0042", "DocDate": "20260412", "DueDate": "20260426", "Outstanding": 500.00 }
  ]
}
```

Drives standard collections workflows — sort by `Over90`, follow up.

## `getcustpaymrep`

```json
POST /api/v2/getcustpaymrep
{ "PeriodStart": "20260501", "PeriodEnd": "20260512" }
```

Lists customer payments received in the period. Useful for daily cash review.

## `getsalesreport`

```json
POST /api/v2/getsalesreport
{
  "PeriodStart": "20260101",
  "PeriodEnd":   "20260512",
  "GroupBy":     "Customer"
}
```

`GroupBy` values: `Customer`, `Item`, `Department`, `Project`.

Response: array of `{ Group, Name, Quantity, NetAmount, VatAmount, GrossAmount }`.

## `getpurchasereport`

Same shape as `getsalesreport` but for purchase-side activity. `GroupBy`: `Vendor`, `Item`, `Department`, `Project`.

## `getinventoryreport`

```json
POST /api/v2/getinventoryreport
{ "AsOfDate": "20260512", "LocationCode": "TLN" }
```

Response per item:

```json
{
  "ItemId":   "...",
  "Code":     "WIDGET-A",
  "Name":     "Widget A",
  "LocationCode": "TLN",
  "QtyOnHand":      125.0,
  "QtyReserved":      8.0,
  "QtyAvailable":   117.0,
  "AverageCost":     28.50,
  "TotalValue":    3562.50
}
```

## Building a VAT reconciliation client-side

The most useful month-end report is "did my KMD agree with the GL"? Build it from:

1. `getinvoices` for the period — sum net and VAT per `TaxId`.
2. `getpurchaseinvoices` for the period — same.
3. `getfinpos` as-of period-end — read account 2120 (output VAT) and 1230 (input VAT).
4. Cross-check: `output_VAT - input_VAT` from steps 1+2 should equal the net of GL accounts in step 3 movement for the period.

Discrepancies usually indicate:
- A miscoded VAT line.
- Reverse-charge bookings that didn't net correctly.
- A GL adjustment without a corresponding VAT entry.

See `examples/month-end-vat-reconciliation.md` for the full walk-through.

## Rate-limit tips

Each report can be heavy. The big ones (`getfinpos`, `getsalesreport`, `getinventoryreport` over long periods) can each consume several seconds of compute and several KB of bandwidth. Don't loop them — pull once per period, cache locally.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/reports/ (per-endpoint pages)
