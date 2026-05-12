# E-invoicing

Merit Aktiva dispatches e-invoices via the configured network operator. The API itself does not require special parameters — a normal `sendinvoice` call triggers e-invoice delivery if the customer is configured for it.

## How it works

1. The customer record has an `EInvOperator` field (set via `sendcustomer` / `updatecustomer`, or in the Merit UI).
2. When you `sendinvoice` for that customer, Merit composes the e-invoice XML (Estonian e-arve standard, schema EVS 923610:2014) and pushes it through the configured operator.
3. The customer receives the e-invoice in their accounting system or banking app, typically within minutes.

## `EInvOperator` enum

| Value | Operator | Notes |
|---|---|---|
| 1 | None | PDF/email only; no e-invoice dispatch |
| 2 | Omniva e-arvekeskus / Telema EDI | Most common B2B EDI network in Estonia |
| 3 | Bank — full e-invoice channel | SEB, Swedbank, LHV — full e-arve with attachments |
| 4 | Bank — limited | Reduced metadata; used for some legacy bank connections |

For Apix / Fitek (Unifiedpost) aggregators, set `EInvOperator: 1` and supply `ApixEInv` with the buyer's Apix identifier.

## Customer-side identifiers

Some operators require additional identifiers on the customer record:

| Field | When required |
|---|---|
| `GLNCode` | EDI on Telema/Omniva; 13-digit GS1 Global Location Number |
| `PartyCode` | Network-specific party code |
| `ApixEInv` | Apix/Fitek aggregator identifier |
| `BankAccount` | Required for bank-channel e-invoices (3, 4) |

Example: configuring a customer for Omniva e-arve:

```json
POST /api/v2/sendcustomer
{
  "Name": "Acme OÜ",
  "RegNo": "12345678",
  "NotTDCustomer": false,
  "CountryCode": "EE",
  "Email": "ar@acme.ee",
  "EInvOperator": 2,
  "GLNCode": "1234567890123"
}
```

## B2G — mandatory

Estonian public-sector buyers are mandated to receive e-invoices via the **Riigi e-arvekeskus** network. Selling to a state body? Their record will typically have `EInvOperator: 2` with a `PartyCode` corresponding to their unit. The e-invoice format must be Peppol BIS / EVS 923610.

If a B2G customer's record isn't configured, `sendinvoice` will succeed (the API doesn't enforce e-invoice for B2G) but the buyer's AP system will reject the invoice. Cross-check `EInvOperator` and `PartyCode` are set before posting.

## B2B — opt-in (with a 2025 twist)

For B2B, e-invoicing is optional **unless the buyer has registered in the e-arve äriregister** (the public list of companies opted in to receive e-invoices). The 1 Jul 2025 amendment to the Accounting Act requires the seller to issue an e-invoice on request when the buyer is registered.

Practical: if the customer asks for e-invoice or you see them in the äriregister list at https://earveregister.rik.ee/, configure them with the appropriate `EInvOperator` once and forget about it.

## Triggering an e-invoice for an existing invoice

If you posted an invoice without e-invoice and want to resend it via the network:

```json
POST /api/v2/sendeinvoice
{ "Id": "<invoice-guid>" }
```

(Endpoint exists per the per-tenant API surface; documented as part of sales-invoices. Confirm availability with `gettenants` or via the Merit UI; some companies have to enable e-arve in their license.)

## Inbound e-invoices

E-invoices received from suppliers land automatically in the company's purchase-invoice inbox. Read them via:

```json
POST /api/v2/getpurchaseinvoices
{ "PeriodStart": "20260501", "PeriodEnd": "20260512" }
```

They appear as draft purchase invoices that can be approved (via `sendpurchinvoiceforapproval` flow) or directly read.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/sales-invoices/create-sales-invoice/
- https://support.merit.ee/ (operator configuration help — UI-side)
- https://earveregister.rik.ee/ (e-arve äriregister — list of B2B opt-ins)
