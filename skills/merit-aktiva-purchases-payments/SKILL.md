---
name: merit-aktiva-purchases-payments
description: Post purchase invoices and expense claims, record payments, and import bank statements (camt.053) in Merit Aktiva. Use when entering supplier bills, paying invoices, or reconciling bank transactions.
last-verified: 2026-05-12
---

# merit-aktiva-purchases-payments

The AP side of Merit Aktiva — purchase invoices, expense claims, payments, and bank reconciliation.

## When to use

- "Enter this supplier invoice", "post a bill from X".
- "Submit an expense claim", "reimburse employee".
- "Pay invoice X", "record this bank transfer".
- "Import last week's bank statement", "reconcile the bank".

## Pre-flight (every call)

1. Confirm `MERIT_API_ID` / `MERIT_API_KEY` are loaded — see `merit-aktiva-core`.
2. Initialize the per-session caches via `merit-aktiva-masters` (`gettaxes`, `getaccounts`).
3. Resolve the vendor's `Id` via `getvendors` (or create via `sendvendor`).
4. Resolve item codes via `getitems` if line items reference inventory; for pure expense lines, an item record is not required — use `GLAccountCode` directly on the row.
5. Pick the right `TaxId` — especially carefully for **reverse-charge** cases (intra-EU goods, intra-EU services, services from outside EU). See `estonian-bookkeeping/references/vat-rates-2026.md` for the mapping.
6. **Show the payload, wait for user confirmation, then POST.**

## `sendpurchinvoice` happy path

```json
POST /api/v2/sendpurchinvoice
{
  "Vendor": { "Id": "<vendor-guid>" },
  "DocNo": "VEN-INV-2025-A4321",
  "DocDate": "20260508",
  "TransactionDate": "20260508",
  "DueDate": "20260522",
  "CurrencyCode": "EUR",
  "InvoiceRow": [
    {
      "Description": "AWS hosting May 2026",
      "Quantity": 1,
      "Price": 420.50,
      "GLAccountCode": "55100",
      "TaxId": "<EU-services-reverse-charge-tax-guid>"
    }
  ],
  "TaxAmount": [{ "TaxId": "<EU-services-reverse-charge-tax-guid>", "Amount": 100.92 }],
  "TotalAmount": 420.50
}
```

Notes:

- `DocNo` is the **vendor's** invoice number (the number they printed on it), not your own counter.
- `Description` on the row substitutes for `Item.Description` if you're not referencing a stored item.
- `GLAccountCode` on the row routes the expense to the right account; without it, Merit uses the item's default purchase account or a tenant-wide default.
- For reverse-charge VAT cases, the `TaxId` is the dedicated reverse-VAT code from `gettaxes` (typically `EL24` prefix). Merit auto-generates the offsetting input/output VAT journal entries.

## Expense claim

Add `ExpenseClaim: true` and supply the employee (using a vendor record of `VendorType: 1` representing the employee, or per Merit configuration):

```json
{
  "Vendor": { "Id": "<employee-vendor-guid>" },
  "DocNo": "EXP-2026-05-Mari",
  "DocDate": "20260512",
  "ExpenseClaim": true,
  "InvoiceRow": [
    { "Description": "Taxi to client meeting", "Quantity": 1, "Price": 12.00,
      "GLAccountCode": "55200", "TaxId": "<24pct-tax-guid>" },
    { "Description": "Coffee with client",     "Quantity": 1, "Price":  6.50,
      "GLAccountCode": "60800", "TaxId": "<24pct-tax-guid>" }
  ],
  "TaxAmount": [{ "TaxId": "<24pct-tax-guid>", "Amount": 4.44 }],
  "TotalAmount": 18.50
}
```

The second line falls under representation costs (account 6080) — note that representation is tax-free up to €50/month + 2% of payroll (see `estonian-bookkeeping/references/corporate-tax.md`).

## Approval queue — `sendpurchinvoiceforapproval`

Same payload, lands in the company's approval queue rather than posting directly to the ledger. Use for workflows where a manager must approve before the invoice books:

```
POST /api/v2/sendpurchinvoiceforapproval
{ ...same payload as sendpurchinvoice... }
```

Once approved in the Merit UI (or via API approval endpoints), the invoice posts automatically. Until approved it doesn't affect P&L or AP balances.

## Payment rules

### `sendpayment` (for sales invoices) and `sendpurchasepayment` (for purchase invoices)

```json
POST /api/v2/sendpurchasepayment
{
  "Vendor": { "Id": "<vendor-guid>" },
  "InvoiceNo": "VEN-INV-2025-A4321",
  "PaymentDate": "20260512",
  "Amount": 420.50,
  "IBAN": "EE382200221020145685",
  "CurrencyCode": "EUR"
}
```

**Cast-iron rules:**

1. **`Amount` must equal the invoice total.** No partials via `sendpayment` / `sendpurchasepayment`. For partials, use `sendprepayment` (prepayment booking, later settled) or `sendsettlement` (link existing payment to existing invoice with a specific amount).
2. **`IBAN` must match a payment method configured in Merit.** Configure payment methods (bank accounts, cash) via UI before first call. Mismatched IBAN is rejected.
3. **Mismatched customer/vendor name is rejected.** Always supply the right `Id`.
4. **No free deletion.** `deletepayment` works only for payments without GL postings or downstream relations. For most cases, post a reversing payment instead.

### Partial payments

```json
POST /api/v2/sendprepayment
{
  "Vendor": { "Id": "<vendor-guid>" },
  "PaymentDate": "20260512",
  "Amount": 200.00,
  "IBAN": "EE...",
  "CurrencyCode": "EUR"
}
```

Later, link the prepayment to specific invoices:

```json
POST /api/v2/sendsettlement
{
  "Vendor": { "Id": "<vendor-guid>" },
  "Allocations": [
    { "PrepaymentId": "<prepayment-guid>", "InvoiceId": "<invoice-guid>", "Amount": 200.00 }
  ]
}
```

## Bank statement import

```json
POST /api/v2/sendbankstatement
{
  "BankAccountIBAN": "EE382200221020145685",
  "FileContent": "<base64-of-camt.053-xml>",
  "FileName": "stmt-2026-05-12.xml"
}
```

The XML must be **ISO 20022 camt.053** — the format that all Estonian banks (Swedbank, SEB, LHV, Luminor) have used since 2018. Download it from the bank's online business banking under "Statements / Pangaväljavõtted".

Merit auto-matches statement lines to outstanding invoices by:
- **`RefNo`** (Estonian viitenumber / reference number) — primary match.
- Customer/vendor IBAN — secondary.
- Amount + date proximity — last resort.

Unmatched lines surface as "to-be-reconciled" entries in the Merit UI. The API doesn't currently expose a callback for unmatched lines; poll `getpaymentimports` to inspect status:

```json
POST /api/v2/getpaymentimports
{ "PeriodStart": "20260501", "PeriodEnd": "20260512" }
```

## Confirmation guardrail

Show the payload, summarise:

> Will post purchase invoice **VEN-INV-2025-A4321** from **Hetzner Online GmbH** dated **2026-05-08**, due **2026-05-22**. AWS hosting €420.50 net, EU reverse-charge VAT 24% €100.92. The reverse-charge entries net to zero VAT cash; both input and output VAT post.

Wait for explicit confirmation before POST.

For bank statement imports, show the count of lines and the date range, then confirm. Per-line reconciliation requires its own confirmation pass before any `sendpayment` / `sendpurchasepayment` for unmatched lines.

## When to read each reference

- [purchase-invoices.md](references/purchase-invoices.md) — full `sendpurchinvoice` schema, reverse-charge bookings, approval queue mechanics.
- [payments.md](references/payments.md) — `sendpayment` / `sendpurchasepayment` / `sendprepayment` / `sendsettlement` with worked examples.
- [bank-statement-import.md](references/bank-statement-import.md) — camt.053 structure, auto-match logic, reconciliation workflow.

## Cross-references

- For auth and conventions: `merit-aktiva-core`.
- For resolving vendor / item / TaxId: `merit-aktiva-masters`.
- For which VAT code applies (reverse charge, etc.): `estonian-bookkeeping`.
- For pulling reports of AP balances and aging: `merit-aktiva-ledger-reports`.
