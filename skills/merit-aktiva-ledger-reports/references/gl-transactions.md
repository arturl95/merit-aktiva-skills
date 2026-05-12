# `sendglbatch` — full reference

## Root fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `DocNo` | string(35) | yes | Your journal entry number; must be unique per period |
| `BatchDate` | YYYYMMDD | yes | Journal entry date |
| `EntryRow` | array of EntryRowObject | yes | At least two rows (one debit, one credit) |
| `Attachment` | object | no | `{ FileName, FileContent: <base64> }` — supporting document |
| `CurrencyCode` | string(4) | no | ISO; defaults to company default |
| `CurrencyRate` | decimal(18,7) | no | ECB if omitted (v2) |

## `EntryRowObject`

| Field | Type | Req | Notes |
|---|---|---|---|
| `AccountCode` | string(8) | yes | Must exist in chart of accounts |
| `Debit` | decimal(12,2) | conditional | Either `Debit` or `Credit` per row, not both |
| `Credit` | decimal(12,2) | conditional | |
| `Memo` | string(150) | no | Row description; printed on the journal |
| `DepartmentCode` | string(16) | no | Must exist if supplied |
| `ProjectCode` | string(20) | no | |
| `CostCenterCode` | string(20) | no | |
| `TaxId` | GUID | no | From `gettaxes` cache, for VAT-tagged rows |
| `VatAmount` | decimal(18,2) | **mandatory if `TaxId` set** | Forgetting it returns a 500 |
| `Dimensions` | array | no | v2 |

## Rules

1. **Balance**: `sum(row.Debit for row) == sum(row.Credit for row)`. Validated server-side.
2. **`TaxId` ↔ `VatAmount`**: paired. Either both or neither.
3. **AccountCode/DepartmentCode/ProjectCode** must exist.
4. **500 rows max** per batch.
5. **No update / no row delete** — to reverse, post a new batch.

## Minimum example

```json
POST /api/v2/sendglbatch
{
  "DocNo": "JV-2026-001",
  "BatchDate": "20260512",
  "EntryRow": [
    { "AccountCode": "1010", "Debit": 100.00, "Memo": "Cash in" },
    { "AccountCode": "3000", "Credit": 100.00, "Memo": "Revenue" }
  ]
}
```

## Rich example — purchase with VAT

```json
POST /api/v2/sendglbatch
{
  "DocNo": "JV-2026-002",
  "BatchDate": "20260512",
  "CurrencyCode": "EUR",
  "CurrencyRate": 1.0,
  "EntryRow": [
    {
      "AccountCode": "55100",
      "Debit": 100.00,
      "Memo": "Office supplies",
      "TaxId": "<24pct-tax-guid>",
      "VatAmount": 24.00,
      "DepartmentCode": "MAIN",
      "ProjectCode": "P-1"
    },
    { "AccountCode": "1230", "Debit": 24.00,   "Memo": "Input VAT" },
    { "AccountCode": "21100", "Credit": 124.00, "Memo": "AP - Vendor X" }
  ],
  "Attachment": {
    "FileName": "receipt.pdf",
    "FileContent": "<base64>"
  }
}
```

Note: the `TaxId` + `VatAmount` pair sit on the expense row, **not** on the VAT row. The VAT row is just a normal debit to the input-VAT account.

## Currency

For a non-EUR batch, set `CurrencyCode` and (optionally) `CurrencyRate`. The system computes the EUR equivalent on each row for reporting. The rate defaults to ECB rate on `BatchDate` if omitted (v2).

## Attachments

`Attachment.FileContent` is base64 of the source document (a PDF receipt, scanned form, etc.). Maximum size is undocumented; keep under a few MB.

## Listing GL batches

```json
POST /api/v2/getglbatches
{ "PeriodStart": "20260501", "PeriodEnd": "20260512" }
```

Returns headers only. For full details (rows):

```json
POST /api/v2/getglbatch
{ "Id": "<batch-guid>" }
```

Or:

```json
POST /api/v2/getglbatchesfull
{ "PeriodStart": "20260501", "PeriodEnd": "20260512" }
```

Returns headers + rows in one call (heavier response; rate-limit accordingly).

## No delete — how to reverse

Post a new batch that mirrors the original with `Debit` / `Credit` swapped:

```json
POST /api/v2/sendglbatch
{
  "DocNo": "JV-2026-002-REV",
  "BatchDate": "20260513",
  "EntryRow": [
    { "AccountCode": "55100", "Credit": 100.00, "Memo": "Reversal of JV-2026-002 - wrong account" },
    { "AccountCode": "1230",  "Credit": 24.00,  "Memo": "Reversal" },
    { "AccountCode": "21100", "Debit": 124.00,  "Memo": "Reversal" }
  ]
}
```

This is best practice for audit anyway — never edit history.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/general-ledger-transactions/creating-general-ledger-transactions/
