# Payments — full reference

Merit Aktiva splits payments into four distinct endpoints depending on direction and partial-payment semantics. Pick the right one.

## The four endpoints

| Endpoint | Direction | Use case |
|---|---|---|
| `sendpayment` | Incoming (customer pays your sales invoice) | Full payment of a known sales invoice |
| `sendPaymentV` | Outgoing (you pay a vendor's invoice) | Full payment of a known purchase invoice |
| `sendprepayment` | Either direction | Customer/vendor prepays; not yet linked to an invoice |
| `sendsettlement` | Either direction | Allocate an existing prepayment or credit to specific invoices |

> **Important:** The purchase-invoice payment endpoint is `sendPaymentV` (capital P and V), **not** `sendpurchasepayment`. The latter does not exist in the Merit API.

## `POST /api/v2/sendpayment` — customer pays your sales invoice

```json
{
  "CustomerName": "Acme OÜ",
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
| `CustomerName` | string(150) | yes | Must match an existing customer in Merit |
| `InvoiceNo` | string(35) | yes | Existing sales-invoice number |
| `PaymentDate` | YYYYMMDD | yes | |
| `Amount` | decimal(18,2) | yes | **Must equal invoice total exactly.** |
| `IBAN` | string(50) | optional | Must match a payment method configured in Merit (Settings → Banks) |
| `BankId` | GUID | optional | Alternative to `IBAN` |
| `RefNo` | string(36) | optional | Bank-side reference number |
| `CurrencyCode` | string | optional (v2) | Required if non-local currency |
| `CurrencyRate` | decimal(18,7) | optional (v2) | ECB if omitted |

## `POST /api/v2/sendPaymentV` — you pay a vendor's purchase invoice

```json
{
  "VendorName": "Hetzner Online GmbH",
  "BillNo": "DE-2026-INV-7788",
  "PaymentDate": "20260512",
  "Amount": 100.00,
  "IBAN": "EE382200221020145685",
  "BankId": "<bank-guid>",
  "CurrencyCode": "EUR"
}
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `BankId` | GUID | yes | Bank identifier |
| `IBAN` | string(50) | yes | Bank account IBAN |
| `VendorName` | string(150) | yes | Must match vendor name in Merit |
| `PaymentDate` | YYYYMMDD | yes | Format: YYYYmmddHHii |
| `BillNo` | string(35) | yes | The vendor's invoice number as recorded via `sendpurchinvoice` |
| `RefNo` | string(36) | yes | Reference number |
| `Amount` | decimal(18,2) | yes | |
| `CurrencyCode` | string | conditional | Required for v2 if non-local currency |
| `CurrencyRate` | decimal(18,7) | optional | ECB if omitted |

## The "no partials" rule

`sendpayment` and `sendPaymentV` require `Amount == invoice_total`. If you try to pay €500 of a €1,000 invoice via `sendpayment`, the API rejects.

For partials:

> **Caution:** `sendprepayment` and `sendsettlement` are **not listed** in the official Merit API reference manual. They may exist as undocumented or internal endpoints, or the official partial-payment flow may differ. Treat the below pattern as unverified until confirmed against a live Merit instance.

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
{
  "PeriodStart": "20260501",
  "PeriodEnd": "20260512",
  "PaymentType": 0,
  "BankId": "<bank-guid>",
  "DateType": 0
}
```

> Period length max 3 months. `DateType`: 0 = DocumentDate, 1 = ChangedDate.

Response fields include: `PIHId`, `BankName`, `CounterPartType`, `CounterPartName`, `CurrencyCode`, `CurrencyRate`, `DocumentDate`, `DocumentNo`, `Direction`, `Amount`, `CounterPartId`, `EInvSentDate`. v2 also includes: `DocId`, `ChangedDate`, `PaymAPIDetails` (object with `PaymId`, `DocNo`, `DocAmount`, `PaidAmount`, `CurrencyCode`, `CurrencyRate`, `DocId`).

## `getpaymenttypes`

```json
POST /api/v2/getpaymenttypes
{ "Type": 1 }
```

`Type`: 1 = purchases, 2 = expense reports, 3 = sales.

Response fields: `Id` (GUID), `Name`, `CurrencyCode`, `SourceType` (1 = cash/bank; 2 = reporting entities; 3 = GL account), `BankType`. Cache like taxes and accounts.

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
