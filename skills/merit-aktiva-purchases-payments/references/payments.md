# Payments — full reference

Merit Aktiva splits payments into four distinct endpoints depending on direction and partial-payment semantics. Pick the right one.

## The four endpoints

| Endpoint | Direction | Use case |
|---|---|---|
| `sendpayment` | Incoming (customer pays your sales invoice) | Full payment of a known sales invoice |
| `sendpurchasepayment` | Outgoing (you pay a vendor's invoice) | Full payment of a known purchase invoice |
| `sendprepayment` | Either direction | Customer/vendor prepays; not yet linked to an invoice |
| `sendsettlement` | Either direction | Allocate an existing prepayment or credit to specific invoices |

## `POST /api/v2/sendpayment` — customer pays your sales invoice

```json
{
  "Customer": { "Id": "<customer-guid>" },
  "InvoiceNo": "2026-0042",
  "PaymentDate": "20260512",
  "Amount": 992.00,
  "IBAN": "EE382200221020145685",
  "CurrencyCode": "EUR",
  "CurrencyRate": 1.0,
  "RefNo": "20260042"
}
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `Customer.Id` (or full Customer) | GUID | yes | |
| `InvoiceNo` | string | yes | Existing sales-invoice number |
| `PaymentDate` | YYYYMMDD | yes | |
| `Amount` | decimal(18,2) | yes | **Must equal invoice total exactly.** |
| `IBAN` | string | yes | Must match a payment method configured in Merit (Settings → Banks) |
| `BankId` | GUID | optional | Alternative to `IBAN` |
| `CustomerName` | string | optional | Must match the customer's `Name` if supplied |
| `RefNo` | string | optional | Bank-side reference number |
| `CurrencyCode` | string | optional | Required if non-EUR |
| `CurrencyRate` | decimal | optional | ECB if omitted (v2) |

## `POST /api/v2/sendpurchasepayment` — you pay a vendor

Same shape, swap `Customer` for `Vendor` and `InvoiceNo` is the vendor's invoice number you recorded via `sendpurchinvoice`.

```json
{
  "Vendor": { "Id": "<vendor-guid>" },
  "InvoiceNo": "DE-2026-INV-7788",
  "PaymentDate": "20260512",
  "Amount": 100.00,
  "IBAN": "EE382200221020145685",
  "CurrencyCode": "EUR"
}
```

## The "no partials" rule

`sendpayment` and `sendpurchasepayment` require `Amount == invoice_total`. If you try to pay €500 of a €1,000 invoice via `sendpayment`, the API rejects.

For partials:

1. **Use `sendprepayment` to record the cash receipt** (no invoice link yet):

   ```json
   POST /api/v2/sendprepayment
   {
     "Customer": { "Id": "<customer-guid>" },
     "PaymentDate": "20260512",
     "Amount": 500.00,
     "IBAN": "EE..."
   }
   ```

   Returns a `PrepaymentId`.

2. **Use `sendsettlement` to allocate the prepayment to one or more invoices**:

   ```json
   POST /api/v2/sendsettlement
   {
     "Customer": { "Id": "<customer-guid>" },
     "Allocations": [
       { "PrepaymentId": "<prepayment-guid>", "InvoiceId": "<invoice-guid>", "Amount": 500.00 }
     ]
   }
   ```

3. When the second €500 arrives, repeat — either a second `sendprepayment` + `sendsettlement`, or now that the invoice's outstanding balance equals the incoming payment, use a normal `sendpayment` for the full remaining €500.

The prepayment-then-settle pattern is also the right way to handle:

- Customer overpayments (credit appears as a prepayment with no invoice link).
- Down payments on a contract.
- Foreign-currency settlements where the exchange-rate difference creates a small balance.

## Listing payments

```json
POST /api/v2/getpayments
{ "PeriodStart": "20260501", "PeriodEnd": "20260512" }
```

Returns array with `PaymentId`, `PaymentDate`, `Amount`, `InvoiceNo`, `CustomerId` / `VendorId`, `RefNo`, `IBAN`, status.

## `getpaymenttypes`

```json
POST /api/v2/getpaymenttypes
{}
```

Returns configured payment methods (bank accounts, cash, card). Cache like taxes and accounts.

## Deletion

`deletepayment` is constrained — the FAQ states: "it has complex relations with invoices throughout general ledger records." Permitted only for payments without GL postings or downstream relations. In practice:

- If you posted the payment seconds ago and nothing else happened, `deletepayment` works.
- After the bank statement reconciliation or any subsequent posting, you must post a reversing payment.

## Reversing a wrongly-recorded payment

Post a payment of opposite direction:

- Wrongly recorded incoming payment → record an outgoing payment of the same `Amount` against the same invoice, then re-record the correct incoming payment. Net effect: clean books, two extra journal lines.

## Currency-conversion gotcha

If you pay a USD invoice from a EUR bank account, supply `CurrencyCode: "USD"` and an explicit `CurrencyRate` if you want to override the ECB default. Tiny rate differences (ECB vs actual bank rate) create exchange-rate gains/losses that land on the default FX account (typically 6500-series in Estonian RTJ).

## Source

- https://api.merit.ee/connecting-robots/reference-manual/payments/create-payment/
- https://api.merit.ee/faq/ (no-partials confirmation, deletion constraints)
