# Example — Import a bank statement and reconcile

**Scenario**: It's the start of a new week. Download last week's camt.053 from Swedbank and import to Merit. 27 transactions; expect most to auto-match via viitenumber, then handle the rest interactively.

**Skills used**: `merit-aktiva-core`, `merit-aktiva-purchases-payments`, `merit-aktiva-masters`.

---

## Step 1 — Get the file

User exports from Swedbank Internetbank → Statements → Format: ISO XML (camt.053) for **2026-05-05 to 2026-05-11**. Save as `stmt-2026-05-w19.xml`.

## Step 2 — Base64-encode

```bash
CONTENT=$(base64 < stmt-2026-05-w19.xml | tr -d '\n')
```

Or in Python:

```python
import base64
with open("stmt-2026-05-w19.xml", "rb") as f:
    content_b64 = base64.b64encode(f.read()).decode("ascii")
```

## Step 3 — Show user the summary before posting

> About to import bank statement **stmt-2026-05-w19.xml** for account `EE382200221020145685`, period 2026-05-05 to 2026-05-11. File size: 87 KB. Confirm import?

Wait for "yes".

## Step 4 — POST

```
POST /api/v2/sendbankstatement
{
  "BankAccountIBAN": "EE382200221020145685",
  "FileContent": "<base64-content>",
  "FileName": "stmt-2026-05-w19.xml"
}
```

Response:

```json
{
  "ImportId": "<guid>",
  "LinesTotal": 27,
  "LinesMatched": 22,
  "LinesUnmatched": 5
}
```

## Step 5 — Inspect unmatched

```
POST /api/v2/getpaymentimports
{ "PeriodStart": "20260505", "PeriodEnd": "20260511" }
```

The response includes per-line details. Filter for the 5 unmatched.

For each unmatched line, common shapes:

| Date | Direction | Amount | Counterparty | RefNo | Description |
|---|---|---|---|---|---|
| 2026-05-06 | CRDT | 100.00 | "John Doe" | (empty) | "Refund Q1 work" |
| 2026-05-08 | CRDT | 992.00 | "Acme OÜ" | (empty) | "Inv 2026-0042" — forgot viitenumber |
| 2026-05-09 | DBIT | 75.50 | "OÜ Logistika Pluss" | (empty) | "Transport invoice" |
| 2026-05-10 | DBIT | 50.00 | (employee) | (empty) | Reimbursement |
| 2026-05-11 | CRDT | 12.50 | (unknown) | (empty) | Test transfer |

## Step 6 — Propose matches

For each unmatched line, search local data:

```
POST /api/v2/getinvoices         { "PaidStatus": 0 }        # unpaid sales invoices
POST /api/v2/getpurchaseinvoices { "PaidStatus": 0 }        # unpaid purchase invoices
```

Then for each unmatched line:

1. **Acme OÜ €992.00 CRDT** — search open sales invoices for Acme OÜ totaling €992.00 → finds invoice 2026-0042. High confidence.
2. **OÜ Logistika Pluss €75.50 DBIT** — search open purchase invoices for Logistika Pluss totaling €75.50 → finds purchase invoice LP-456. High confidence.
3. **John Doe €100.00 CRDT** — no open invoice match. Show user and ask.
4. **Employee €50.00 DBIT** — no open invoice match; this is likely a payroll-side reimbursement that should go to GL, not as a payment. Show user.
5. **€12.50 CRDT (unknown counterparty)** — likely test/junk; flag for user judgement.

## Step 7 — Show user a proposed reconciliation table

```
Line  | Date       | Amount  | Match                          | Action
------|------------|---------|--------------------------------|--------
  3   | 2026-05-08 | +992.00 | Inv 2026-0042 (Acme OÜ)        | sendpayment
  6   | 2026-05-09 | -75.50  | Inv LP-456 (Logistika Pluss)   | sendpurchasepayment
  9   | 2026-05-06 | +100.00 | (none — no open invoice)        | manual decision
 11   | 2026-05-10 | -50.00  | (none — likely reimbursement)   | manual decision
 19   | 2026-05-11 | +12.50  | (none — unknown counterparty)   | manual decision
```

Ask the user:
- Approve auto-matches for lines 3 and 6?
- For line 9: is this related to an unposted invoice or other receivable?
- For line 11: post to GL as expense, or to expense-claim subledger?
- For line 19: write off or hold?

## Step 8 — Execute approved matches

After user confirms lines 3 and 6:

```
POST /api/v2/sendpayment
{
  "Customer": { "Id": "<acme-id>" },
  "InvoiceNo": "2026-0042",
  "PaymentDate": "20260508",
  "Amount": 992.00,
  "IBAN": "EE382200221020145685",
  "CurrencyCode": "EUR",
  "RefNo": ""
}
```

```
POST /api/v2/sendpurchasepayment
{
  "Vendor": { "Id": "<logistika-id>" },
  "InvoiceNo": "LP-456",
  "PaymentDate": "20260509",
  "Amount": 75.50,
  "IBAN": "EE382200221020145685",
  "CurrencyCode": "EUR"
}
```

## Step 9 — Handle the leftovers

For line 9 (€100 from John Doe): if no open invoice and no clear assignment, two options:
- Record as prepayment via `sendprepayment` linked to John Doe (if a vendor/customer record exists), to be settled later.
- Post to a "to-investigate" GL account via `sendglbatch` and follow up.

For line 11 (€50 employee reimbursement): post a `sendpurchinvoice` with `ExpenseClaim: true` after gathering the receipts from the employee, then `sendpurchasepayment` to clear.

For line 19 (€12.50 test): if confirmed junk, post via `sendglbatch` to a suspense account (e.g. 1599 — Other receivables / suspense), then write off after investigation.

## Step 10 — Final state

After reconciliation:
- All 27 statement lines either auto-matched or manually handled.
- Bank account balance in Merit matches the closing balance on the statement.
- AR and AP balances updated for the matched invoices.

## Common pitfalls

- **Double-imports**: don't re-import a statement that's already been processed. Merit may create duplicate payments. Check `getpaymentimports` first.
- **Wrong IBAN**: the `BankAccountIBAN` in the request must match an account configured in Merit (Settings → Banks). Mismatched IBAN rejects the entire file.
- **Customer changed name**: auto-match by IBAN+amount might fail if a customer's stored IBAN changed. Surface as unmatched and update the customer record.

## Cross-references

- camt.053 structure & matching logic → `merit-aktiva-purchases-payments/references/bank-statement-import.md`.
- Payment endpoints → `merit-aktiva-purchases-payments/references/payments.md`.
- Customer/vendor lookup → `merit-aktiva-masters`.
