# Credit invoices

Estonian VAT law requires that corrections to an issued sales invoice happen via a **credit invoice** (kreeditarve) rather than an update. Merit Aktiva enforces this — there is no `updateinvoice` endpoint.

## The pattern

Use `sendinvoice` (v2) with `AccountingDoc: 5`. The rows mirror the original invoice but with **negative quantities** (or negative prices — pick one and be consistent) so the net effect reverses the original posting.

```json
POST /api/v2/sendinvoice
{
  "Customer": { "Id": "<same-customer-guid-as-original>" },
  "InvoiceNo": "2026-0042-C1",
  "AccountingDoc": 5,
  "DocDate": "20260518",
  "DueDate": "20260518",
  "TransactionDate": "20260512",
  "InvoiceRow": [
    {
      "Item": { "Code": "CONSULT-HR", "Description": "Consulting (May) - credited", "Type": 2 },
      "Quantity": -8,
      "Price": 100.00,
      "TaxId": "<24pct-tax-guid>"
    }
  ],
  "TaxAmount": [{ "TaxId": "<24pct-tax-guid>", "Amount": -192.00 }],
  "TotalAmount": -800.00,
  "Hcomment": "Credit invoice for 2026-0042 — service was not delivered."
}
```

Notes:

- **`AccountingDoc: 5`** is what marks this as a credit invoice. Without it, you'd be creating a regular invoice with negative amounts (which Merit may accept but will book differently and confuse KMD INF).
- **`InvoiceNo`** must still be unique. Convention: append `-C1`, `-C2`, etc. to the original invoice number, OR use a separate credit-invoice number series.
- **`TransactionDate`** points back to the original supply date (for VAT period attribution).
- **`Hcomment`** should reference the original invoice number. This is not enforced by the API but is best practice for audit trail.

## The stock-item gotcha

For credit invoices of **stock items (`Type: 1`)**, you **must** supply `ItemCostAmount` on each row. Without it, the API returns a misleading "internal error" 500.

```json
{
  "Item": { "Code": "WIDGET-A", "Description": "Widget A", "Type": 1 },
  "Quantity": -3,
  "Price": 50.00,
  "TaxId": "<24pct-tax-guid>",
  "ItemCostAmount": 28.00,
  "LocationCode": "TLN"
}
```

`ItemCostAmount` is the unit cost the item was carried at when the original sale debited cost of goods sold. Look it up:

- From the original invoice's row data via `POST /api/v2/getinvoice` (pass `{ "Id": "<invoice-guid>" }`).
- Or via `getitems` with `LocationCode` filter (returns `ItemUnitCost`).

`LocationCode` is required to indicate where the credited stock returns to.

## Partial credits

Credit only part of the original invoice by reducing the quantity/price on the credit. Example: original 8h consulting, credit 2h:

```json
{
  "Item": { "Code": "CONSULT-HR", "Description": "Consulting (May) - partial credit", "Type": 2 },
  "Quantity": -2,
  "Price": 100.00,
  "TaxId": "<24pct-tax-guid>"
}
```

## Customer payment already made?

If the original invoice was paid before the credit was issued, the credit invoice creates a **negative receivable**. This appears as a customer prepayment / overpayment. Either:

- Apply it to a future invoice (link via `RefNo` or via `sendsettlement`).
- Refund via `sendpayment` of `Type: 2` (outgoing) — see `merit-aktiva-purchases-payments/references/payments.md`.

## KMD impact

A credit invoice reverses the original VAT period's lines:

- KMD line 1 (taxable supply 24% base) goes down by the credit's net.
- KMD line 4 (VAT due at 24%) goes down by the credit's VAT.
- The reversal lands in the **VAT period containing the credit's `TransactionDate`** — not the period of the original invoice unless you set `TransactionDate` to fall in that period.

For corrections affecting a closed VAT period, file a corrected KMD for that period. EMTA permits corrections up to 3 years back.

## When NOT to use a credit invoice

- Pure typo correction with no financial change (e.g. fixing the customer's address): contact Merit support; some metadata can be edited via the UI without a credit.
- Wrong `InvoiceNo`: `deleteinvoice` + `sendinvoice` if the invoice is the only one in the period. Otherwise credit + reissue.
- Date correction within the same VAT period and before KMD filing: `deleteinvoice` + `sendinvoice`.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/sales-invoices/create-sales-invoice/
- https://api.merit.ee/connecting-robots/reference-manual/sales-invoices/get-sales-invoice-details/
- https://www.emta.ee/en/business-client/taxes-and-payment/value-added-tax/calculation-and-refund-vat
