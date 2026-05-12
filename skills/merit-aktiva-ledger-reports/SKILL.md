---
name: merit-aktiva-ledger-reports
description: Create general-ledger journal entries (sendglbatch) and pull P&L, balance, customer-debt, sales, and inventory reports from Merit Aktiva. Use for month-end adjustments and reporting.
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
2. **`AccountCode` and `DepartmentCode` must exist.** Pull from `merit-aktiva-masters` (`getaccounts`, `getdepartments`).
3. **`VatAmount` is mandatory whenever `TaxId` is set on a row.** Forgetting it returns a misleading "internal error" 500.
4. **500-row cap per `sendglbatch`.** Split large batches.
5. **No GL row deletion.** Reverse with a new batch (swap Debit/Credit on the rows that need to be undone).

## `sendglbatch` happy path

```json
POST /api/v2/sendglbatch
{
  "DocNo": "JV-2026-05-001",
  "BatchDate": "20260512",
  "EntryRow": [
    { "AccountCode": "1010", "Debit": 100.00,  "Memo": "Cash in" },
    { "AccountCode": "3000", "Credit": 100.00, "Memo": "Revenue" }
  ]
}
```

Response: `{ "BatchId": "...", "DocNo": "JV-2026-05-001" }`.

## Confirmation guardrail

Before any `sendglbatch`, show:

1. The full payload in a fenced block.
2. A summary line per row (`Dr/Cr | Account name (code) | Amount | Memo`).
3. The balance check: sum of debits = sum of credits.
4. The user-facing effect: "Posts €100 from cash to revenue, dated 2026-05-12."

Wait for explicit "yes" before POST. For batch journals (e.g. monthly depreciation across 30 fixed assets), show a row-count summary and require batch confirmation.

## The four most-used reports

### `getprofitloss` — P&L

```json
POST /api/v2/getprofitloss
{ "PeriodStart": "20260101", "PeriodEnd": "20260512" }
```

Returns revenue, COGS, expenses, operating income, net income, grouped by account.

### `getfinpos` — Financial position (balance sheet)

```json
POST /api/v2/getfinpos
{ "AsOfDate": "20260512" }
```

Returns assets / liabilities / equity at a snapshot date.

### `getcustdebtrep` — Customer debts (AR aging)

```json
POST /api/v2/getcustdebtrep
{ "AsOfDate": "20260512" }
```

Returns each customer's outstanding receivables aged into buckets (0–30, 31–60, 61–90, 90+).

### `getsalesreport` — Sales summary

```json
POST /api/v2/getsalesreport
{ "PeriodStart": "20260101", "PeriodEnd": "20260512", "GroupBy": "Customer" }
```

`GroupBy` values: `Customer`, `Item`, `Department`, `Project`. Returns aggregated sales per group.

See [reports.md](references/reports.md) for the full list and response shapes.

## Pre-flight

1. Init caches — `merit-aktiva-masters` `gettaxes`, `getaccounts`.
2. For VAT-affecting GL rows, decide the TaxId. If unsure, consult `estonian-bookkeeping`.
3. Build the batch, balance-check locally, then **show + confirm** before POSTing.

## Common GL patterns

### Monthly depreciation

For each fixed asset, debit depreciation expense and credit accumulated depreciation:

```json
{
  "DocNo": "DEP-2026-05",
  "BatchDate": "20260531",
  "EntryRow": [
    { "AccountCode": "6210", "Debit": 333.33, "Memo": "Depreciation - Office equipment May" },
    { "AccountCode": "1825", "Credit": 333.33, "Memo": "Accum. depreciation - Office equipment" },
    { "AccountCode": "6210", "Debit": 250.00, "Memo": "Depreciation - Vehicle May" },
    { "AccountCode": "1825", "Credit": 250.00, "Memo": "Accum. depreciation - Vehicle" }
  ]
}
```

For automated depreciation across many assets, use the fixed-asset endpoints (`sendfixedasset`, `getfixedassets`) and let Merit compute monthly depreciation — usually a better fit than manual GL.

### Accrued expenses (month-end)

```json
{
  "DocNo": "ACR-2026-05",
  "BatchDate": "20260531",
  "EntryRow": [
    { "AccountCode": "5300", "Debit": 1500.00,  "Memo": "Accrued legal fees - May (invoice expected June)" },
    { "AccountCode": "2150", "Credit": 1500.00, "Memo": "Accrued liabilities" }
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
