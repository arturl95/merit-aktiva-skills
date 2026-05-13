---
name: merit-aktiva-ledger-reports
description: Create general-ledger journal entries (sendglbatch) and pull P&L (getprofitrep), balance sheet (getbalancerep), customer-debt (getcustdebtrep), sales (getsalesrep), and inventory reports from Merit Aktiva. Use for month-end adjustments and reporting.
last-verified: 2026-05-12
---

# merit-aktiva-ledger-reports

The general-ledger and reporting layer. Direct journal entry for adjustments that don't fit an invoice/payment pattern, plus the read-side endpoints that drive monthly close.

## When to use

- Manual journal entry — depreciation, accruals, reclassifications, FX adjustments, payroll postings, year-end true-ups.
- Pulling P&L, balance sheet, customer-debt aging, sales report, purchase report, inventory report.
- Reconciling the VAT control account at month-end.

## GL rules (cast-iron)

1. **EntryRow must balance.** Sum of `Debit` across rows must equal sum of `Credit`. Server validates; unbalanced batch is rejected.
2. **`AccountCode` and `DepartmentCode` are both required on every EntryRow.** Both must exist in the company database. Pull valid codes from `merit-aktiva-masters` (`getaccounts`, `getdepartments`).
3. **`VatAmount` is mandatory whenever `TaxId` is set on a row.** Forgetting it returns a misleading "internal error" 500.
4. **500-row cap per `sendglbatch`.** Split large batches.
5. **No GL row deletion.** Reverse with a new batch (swap Debit/Credit on the rows that need to be undone).
6. **All GL and report endpoints are v1 or v2 as specified.** `sendglbatch`, `getglbatches`, `getglbatch`, `getglbatchesfull` are all **v1**. Report endpoints vary (see references/reports.md).

## `sendglbatch` happy path

```json
POST /api/v1/sendglbatch
{
  "DocNo": "JV-2026-05-001",
  "BatchDate": "20260512",
  "EntryRow": [
    { "AccountCode": "1010", "DepartmentCode": "MAIN", "Debit": 100.00,  "Memo": "Cash in" },
    { "AccountCode": "3000", "DepartmentCode": "MAIN", "Credit": 100.00, "Memo": "Revenue" }
  ]
}
```

Response: `{ "BatchId": "...", "DocNo": "JV-2026-05-001" }`.

> Note: `DepartmentCode` is **required** on every EntryRow (API v1 only for all GL endpoints).

## Confirmation guardrail

Before any `sendglbatch`, show:

1. The full payload in a fenced block.
2. A summary line per row (`Dr/Cr | Account name (code) | Amount | Memo`).
3. The balance check: sum of debits = sum of credits.
4. The user-facing effect: "Posts €100 from cash to revenue, dated 2026-05-12."

Wait for explicit "yes" before POST. For batch journals (e.g. monthly depreciation across 30 fixed assets), show a row-count summary and require batch confirmation.

## The four most-used reports

### `getprofitrep` — P&L (income statement)

```json
POST /api/v1/getprofitrep
{ "EndDate": "20260512", "PerCount": 5 }
```

Returns hierarchical report lines with `RowType` (1=label, 3=account turnover, 4=formula), `Balance` array (one entry per month, descending from `EndDate`), and `Details` per account.

### `getbalancerep` — Financial position (balance sheet)

```json
POST /api/v1/getbalancerep
{ "EndDate": "20260512", "PerCount": 1 }
```

Returns assets / liabilities / equity at a snapshot date. Same structure as `getprofitrep` with `RowType` 2=account balance.

### `getcustdebtrep` — Customer debts (AR aging)

```json
POST /api/v1/getcustdebtrep
{ "CustName": "", "DebtDate": "20260512" }
```

Pass `CustName: ""` to get all customers. Returns one row per outstanding document with `PartnerName`, `DocType`, `DueDate`, `TotalAmount`, `PaidAmount`, `UnPaidAmount`. Age buckets must be computed client-side from `DueDate`.

### `getsalesrep` — Sales report

```json
POST /api/v2/getsalesrep
{ "StartDate": "20260101", "EndDate": "20260512", "ReportType": 2 }
```

`ReportType` values: 1=by invoices, 2=by customers, 3=by articles, 4=by fixed assets. Response shape varies by type.

See [reports.md](references/reports.md) for the full list and response shapes.

## Pre-flight

1. Init caches — `merit-aktiva-masters` `gettaxes`, `getaccounts`.
2. For VAT-affecting GL rows, decide the TaxId. If unsure, consult `estonian-bookkeeping`.
3. Build the batch, balance-check locally, then **show + confirm** before POSTing.

## Common GL patterns

### Monthly depreciation

For each fixed asset, debit depreciation expense and credit accumulated depreciation:

```json
POST /api/v1/sendglbatch
{
  "DocNo": "DEP-2026-05",
  "BatchDate": "20260531",
  "EntryRow": [
    { "AccountCode": "6210", "DepartmentCode": "MAIN", "Debit": 333.33, "Memo": "Depreciation - Office equipment May" },
    { "AccountCode": "1825", "DepartmentCode": "MAIN", "Credit": 333.33, "Memo": "Accum. depreciation - Office equipment" },
    { "AccountCode": "6210", "DepartmentCode": "MAIN", "Debit": 250.00, "Memo": "Depreciation - Vehicle May" },
    { "AccountCode": "1825", "DepartmentCode": "MAIN", "Credit": 250.00, "Memo": "Accum. depreciation - Vehicle" }
  ]
}
```

For automated depreciation across many assets, use the fixed-asset endpoints (`sendfixedasset`, `getfixedassets`) and let Merit compute monthly depreciation — usually a better fit than manual GL.

### Accrued expenses (month-end)

```json
POST /api/v1/sendglbatch
{
  "DocNo": "ACR-2026-05",
  "BatchDate": "20260531",
  "EntryRow": [
    { "AccountCode": "5300", "DepartmentCode": "MAIN", "Debit": 1500.00,  "Memo": "Accrued legal fees - May (invoice expected June)" },
    { "AccountCode": "2150", "DepartmentCode": "MAIN", "Credit": 1500.00, "Memo": "Accrued liabilities" }
  ]
}
```

Reverse in the following month when the actual invoice posts.

### VAT control reconciliation

At month-end, the input-VAT account (1230) and output-VAT account (2120) balances should sum to the net VAT payable on the KMD. If they don't reconcile (e.g. due to reverse-charge bookings not netting correctly), post an adjusting entry — but first investigate; reconciliation failures usually indicate a miscoded transaction.

## When to read each reference

- [gl-transactions.md](references/gl-transactions.md) — full `sendglbatch` schema, VatAmount coupling, attachments, dimensions.
- [chart-of-accounts.md](references/chart-of-accounts.md) — `getaccounts` deep dive, RTJ mapping, common Estonian codes.
- [reports.md](references/reports.md) — full list of report endpoints with request payloads and response highlights.

## Cross-references

- For auth and conventions: `merit-aktiva-core`.
- For account/dimension lookup: `merit-aktiva-masters`.
- For VAT codes and Estonian tax math: `estonian-bookkeeping`.
- For sales / purchase / payment side: respective `merit-aktiva-*` skills.
