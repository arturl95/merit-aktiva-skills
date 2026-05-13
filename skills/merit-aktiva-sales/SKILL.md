---
name: merit-aktiva-sales
description: Create, list, credit, and email sales invoices in Merit Aktiva; trigger e-invoices via Omniva/Telema/bank. Use when issuing customer invoices, posting credit notes, or pulling sales-side data.
last-verified: 2026-05-12
---

# merit-aktiva-sales

The sales side of Merit Aktiva. Covers `sendinvoice` (v2), credit invoices, listing/incremental sync, e-invoicing, PDF retrieval, and email delivery.

## When to use

- "Post an invoice", "send a sales invoice", "bill the customer".
- "Credit note", "void invoice X" — issue a credit invoice (no direct update).
- "List this month's invoices", "find unpaid invoices".
- "Email the invoice", "get the invoice PDF".
- Triggering e-invoices via Omniva/Telema/bank for customers configured for the network.

## Pre-flight (every call)

1. Confirm `MERIT_API_ID` / `MERIT_API_KEY` are loaded — see `merit-aktiva-core`.
2. Initialize the per-session caches via `merit-aktiva-masters` (`gettaxes`, `getaccounts`).
3. Resolve the customer's `Id` via `getcustomers` (or create new via `sendcustomer`).
4. Resolve each line's item via `getitems` (or create via `senditems`).
5. Resolve the right `TaxId` from the `gettaxes` cache. If unsure which VAT code applies, consult `estonian-bookkeeping`.
6. Build the payload. Round each row to 2 decimals **before** summing `TaxAmount` and `TotalAmount`.
7. **Show the payload to the user and wait for confirmation.** Only then POST.

## `sendinvoice` happy path (minimal)

```json
POST /api/v2/sendinvoice
{
  "Customer": { "Id": "<customer-guid-from-getcustomers>" },
  "InvoiceNo": "2026-0042",
  "DocDate": "20260512",
  "DueDate": "20260526",
  "InvoiceRow": [
    {
      "Item": { "Code": "CONSULT-HR", "Description": "Consulting", "Type": 2 },
      "Quantity": 8,
      "Price": 100.00,
      "TaxId": "<24pct-tax-guid>"
    }
  ],
  "TaxAmount": [{ "TaxId": "<24pct-tax-guid>", "Amount": 192.00 }],
  "TotalAmount": 800.00
}
```

Response:

```json
{ "CustomerId": "...", "InvoiceId": "...", "InvoiceNo": "2026-0042", "RefNo": "20260042", "NewCustomer": false }
```

See [sendinvoice-schema.md](references/sendinvoice-schema.md) for every field and three full examples (minimal, mid, rich).

## The five cast-iron rules

1. **Round each row to 2 decimals before summing.** Server validates `TaxAmount` and `TotalAmount` against the sum of rounded row totals. Half-cent drift on a multi-row invoice will reject.
2. **`TaxId` is required on every row.** Even 0% lines. Pull the right GUID from the `gettaxes` cache.
3. **Different VAT rates need different item codes.** KMD INF reconciliation otherwise fails. If you have one item being sold under both domestic 24% and intra-EU 0%, register two `Code`s (e.g. `CONSULT-HR` and `EU-CONSULT-HR`).
4. **No invoice update.** Delete (`POST /api/v1/deleteinvoice` with `{ "Id": "<invoice-guid>" }`) and recreate (`sendinvoice`).
5. **`InvoiceNo` must be unique.** There is no server-side locking and no "next number" endpoint. Track your own counter. For high-concurrency, use a UUID-prefixed scheme (e.g. `INV-2026-<short-uuid>`).

## Confirmation guardrail

Before any `sendinvoice` / `sendinvoicebyemail` / `deleteinvoice` call:

1. Show the full JSON payload in a fenced block.
2. Summarise the effect in plain English:
   > Will create invoice **2026-0042** dated **2026-05-12**, due **2026-05-26**, for customer **Acme OÜ**: 8h of consulting @ €100/h. Net €800.00, VAT 24% €192.00, **Total €992.00**.
3. Wait for explicit "yes" / "go" / "post it" / equivalent before POSTing.

For batch jobs (e.g. posting 30 monthly retainers), show a summary table of all rows and require a single batch confirmation, OR let the user request per-row review.

## Credit invoices

Use `sendinvoice` with `AccountingDoc: 5` and supply `ItemCostAmount` on each row that is a stock item (`Type: 1`). See [credit-invoices.md](references/credit-invoices.md) for the full pattern.

## E-invoicing

Send a normal `sendinvoice`. The customer's `EInvOperator` field (set at customer creation/update) determines how Merit dispatches:

- 1 — no e-invoice (PDF only)
- 2 — Omniva e-arvekeskus / Telema EDI
- 3 — bank full e-invoice channel
- 4 — bank limited

For Apix/Fitek aggregators, set `ApixEInv` on the customer. B2G e-invoicing is mandatory; B2B is optional unless the buyer registered in the e-arve äriregister (since 1 Jul 2025). See [e-invoicing.md](references/e-invoicing.md).

## PDFs and email

```json
POST /api/v2/getsalesinvpdf
{ "Id": "<invoice-guid>" }
```

Optional: `"DelivNote": true` returns the invoice without prices (delivery note layout).

Response: `{ "FileName": "...", "FileContent": "<base64-pdf>" }`.

```json
POST /api/v2/sendinvoicebyemail
{ "Id": "<invoice-guid>", "Email": "billing@customer.ee" }
```

If `Email` is omitted, Merit uses the customer's stored `Email` field.

See [invoice-pdf-and-email.md](references/invoice-pdf-and-email.md) for attaching a custom PDF to a new invoice.

## Listing and incremental sync

```json
POST /api/v1/getinvoices
{ "PeriodStart": "20260101", "PeriodEnd": "20260512", "UnPaid": false, "DateType": 0 }
```

Also available as `POST /api/v2/getinvoices` (adds dimensions fields in the response) and `POST /api/v2/getinvoices2` (filter by `InvNo`, `CustName`, or `CustId` instead of period).

`DateType`: 0 = filter by document date (default), 1 = filter by changed date. Use `DateType: 1` for incremental sync — set `PeriodStart`/`PeriodEnd` to a window covering changes since the last sync. The period cannot exceed 3 months. 500-row cap per response — narrow the window if you hit it.

## Invoice detail lookup

```json
POST /api/v2/getinvoice
{ "Id": "<invoice-guid>" }
```

Optional: `"AddAttachment": true` includes any attached file in the response. Returns header, line rows, and payments arrays. See API docs for the full response shape.

## When to read each reference

- [sendinvoice-schema.md](references/sendinvoice-schema.md) — full v2 schema, every field, three examples.
- [credit-invoices.md](references/credit-invoices.md) — credit-note pattern with stock-item gotcha.
- [e-invoicing.md](references/e-invoicing.md) — operator routing, GLNCode/PartyCode/ApixEInv, B2G vs B2B.
- [invoice-pdf-and-email.md](references/invoice-pdf-and-email.md) — `getsalesinvpdf`, `sendinvoicebyemail`, attaching custom PDFs.

## Cross-references

- For auth and conventions: `merit-aktiva-core`.
- For resolving customer / item / TaxId: `merit-aktiva-masters`.
- For which VAT code applies: `estonian-bookkeeping`.
- For recording the payment that closes the invoice: `merit-aktiva-purchases-payments`.
