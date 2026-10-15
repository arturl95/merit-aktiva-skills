---
name: merit-aktiva-offers
description: Create, list, read, update, approve/reject and convert sales offers (pakkumised), sales orders (tellimused) and prepayment invoices (ettemaksuarved) in Merit Aktiva via sendoffer / getoffers / getoffer / setofferstatus / updateoffer / offer2inv. Use whenever the user wants a quote, offer, proposal, hinnapakkumine, pakkumine, sales order or prepayment invoice for a customer, wants to turn a spreadsheet or price list into a quote, or wants to invoice an accepted quote — even if they say "invoice" but the customer has not yet agreed to the price.
last-verified: 2026-10-15
---

# merit-aktiva-offers

The pre-invoice side of Merit Aktiva. One document type (`Müük → Pakkumised`) covers three things, chosen by `DocType`: a **quote** (pakkumine, 1), a **sales order** (müügitellimus, 2) and a **prepayment invoice** (ettemaksuarve, 3). None of them touches the general ledger or VAT returns until it is converted into a sales invoice with `offer2inv`, so a quote is the safe place to put a price a customer has not yet accepted.

## When to use

- "Make a quote / offer / pakkumine / hinnapakkumine for X", "send them a proposal for these items".
- "Turn this Excel / price list into an offer."
- "Create a sales order" or "prepayment invoice" (ettemaksuarve) before delivery.
- "They accepted the quote, invoice it" → `offer2inv`, not a fresh `sendinvoice`, so the invoice keeps the quote's lines and the quote shows status 6.
- "Mark the quote as sent / rejected / cancelled", "what quotes are open".

Do **not** use this for a normal post-delivery invoice; that is `merit-aktiva-sales`.

## Pre-flight (every call)

1. Credentials and signing: `merit-aktiva-core`.
2. Caches: `gettaxes`, `getitems`, `getcustomers` from `merit-aktiva-masters`. An offer accepts the same `Customer` object as `sendinvoice`: pass `{ "Id": <guid> }` for an existing customer, or a full object (`Name`, `RegNo`, `VatRegNo`, `NotTDCustomer`, `CountryCode`, address, `Email`) and Merit creates the customer with the offer.
3. **Units of measure must already exist, matched by name.** `UOMName` that is not in the company's list fails the whole call with `Puudub mõõtühik nimega X`. Read the list first with `GET /api/v2/units` (it is a GET; POST returns 405) and map to what exists. Typical Estonian defaults are `tk`, `km`, `tund`, `kuu`, `kg`, `m²`, `m³`, `ltr`; `h`, `m` and `kompl` usually do **not** exist. Add a missing unit with `POST /api/v2/senduom` only if the user wants it; otherwise fall back to `tk`. See [offer-lifecycle.md](references/offer-lifecycle.md#units-of-measure).
4. **Pick the next `OfferNo` yourself.** It is required and must be unique, and there is no "next number" endpoint. List recent offers with `getoffers` and continue the series the company already uses (Merit's UI default is `P-1000NN`).
5. Round each row to 2 decimals before summing `TaxAmount` and `TotalAmount` (half-up, same rule as invoices).
6. Show the payload and a plain-language summary, then wait for an explicit yes before POSTing. An offer is harmless for the books, but it still creates a customer and a numbered document the user will send out.

## `sendoffer` happy path

```json
POST /api/v2/sendoffer
{
  "Customer": { "Id": "<customer-guid>" },
  "OfferNo": "P-100011",
  "DocDate": "20261015",
  "ExpireDate": "20261029",
  "DeliveryDate": "20261029",
  "DocType": 1,
  "DocStatus": 1,
  "CurrencyCode": "EUR",
  "Hcomment": "Porsche Ring läbipääsusüsteem, 7 ust. Objekt: Audru ringrada, Pärnu.",
  "OfferRow": [
    { "Item": { "Code": "24% kaup", "Description": "Uste kontroller", "Type": 3, "UOMName": "tk" },
      "Quantity": 1, "Price": 1120.90, "TaxId": "<24pct-tax-guid>" },
    { "Item": { "Code": "24% teen", "Description": "Paigaldus ja seadistus", "Type": 2, "UOMName": "tund" },
      "Quantity": 8, "Price": 100.00, "TaxId": "<24pct-tax-guid>" }
  ],
  "TaxAmount": [{ "TaxId": "<24pct-tax-guid>", "Amount": 461.02 }],
  "TotalAmount": 1920.90
}
```

Response (the field names say "Invoice" but it is the offer):

```json
{ "CustomerId": "...", "InvoiceId": "<offer-guid>", "InvoiceNo": "P-100011", "RefNo": null, "NewCustomer": null }
```

Keep `InvoiceId`; it is the `Id` for `getoffer`, `setofferstatus`, `updateoffer` and `offer2inv`.

Full field list and enums: [sendoffer-schema.md](references/sendoffer-schema.md).

## Rules that bite

1. **`DocType` 1/2/3 and `DocStatus` 1–7 are integers**, and both are effectively required: omit them and Merit may store the wrong kind of document. A new quote is `DocType 1`, `DocStatus 1` (created). For a prepayment invoice (`DocType 3`) `ExpireDate` is the due date.
2. **`ExpireDate` and `DeliveryDate` are required dates** (`YYYYMMDD`). The UI calls `ExpireDate` "Kehtiv kuni"; two weeks is the common default.
3. **Items are created on the fly** if `Code` is unknown and `Type` is given, exactly as on invoices. Reuse the company's generic sales items (for example `24% kaup` for goods and `24% teen` for services) unless the user keeps a real item catalogue; the `Description` on the row is what prints.
4. **Rows need `TaxId` every time**, even for 0 % lines.
5. **`getoffers` needs all four fields** `DateType`, `PeriodStart`, `PeriodEnd`, `UnPaid`; leaving one out returns a bare HTTP 500 `FormatException` ("Midagi läks valesti") that looks like an outage. Period max 3 months. `v1/getoffers` does not exist (404).
6. **There is no delete for offers via the API.** `setofferstatus` with `NewStatus 7` cancels one; `updateoffer` changes only `DeliveryDate`, `HComment`, `FComment`. Anything else means cancel and recreate with a new number.
7. **No API to email or PDF an offer.** `getsalesinvpdf` / `sendinvoicebyemail` take invoice Ids only. Tell the user to send the quote from Merit (Müük → Pakkumised → open → Saada e-postiga) and offer to set the customer's email on the card so it is pre-filled.

## Confirmation guardrail

Before `sendoffer`, `setofferstatus`, `updateoffer`, `offer2inv` or `sendPaymentO`:

1. Show the JSON payload in a fenced block.
2. Summarise:
   > Will create **quote P-100011** for **Rada OÜ** (new customer, reg 12112827), dated 15.10.2026, valid to 29.10.2026, 11 lines. Net €4 280.70, VAT 24 % €1 027.37, **total €5 308.07**. Nothing is booked until it is converted to an invoice.
3. Wait for an explicit yes.

For a quote built from a spreadsheet, show the user a line table (description, qty, unit, price, net) plus the net/VAT/gross totals and state which price column you used (cost vs. margin price) before asking.

## Lifecycle after creation

| Want | Call |
|---|---|
| Read it back, with rows | `POST /api/v2/getoffer { "Id" }` |
| List open quotes | `POST /api/v2/getoffers { "DateType": 0, "PeriodStart", "PeriodEnd", "UnPaid": false }` and filter `DocType`/`DocStatus` |
| Mark sent / accepted / rejected / cancelled | `POST /api/v2/setofferstatus { "Id", "NewStatus", "Comment" }` (Comment required for 5) |
| Fix delivery date or comments | `POST /api/v2/updateoffer { "OfferNo", "DeriveryDate", "HComment", "FComment" }` (sic, the field is misspelt in the API) |
| Customer paid a prepayment against it | `POST /api/v2/sendPaymentO` |
| Customer accepted → invoice | `POST /api/v2/offer2inv { "Id" }` → returns the new `InvoiceId`/`InvoiceNo`; then send the invoice with `merit-aktiva-sales` |

Details and status semantics: [offer-lifecycle.md](references/offer-lifecycle.md).

## Cross-references

- Auth, signing, rate limits: `merit-aktiva-core`.
- Customers, items, tax codes, units: `merit-aktiva-masters`.
- The invoice that `offer2inv` produces, e-invoicing and email: `merit-aktiva-sales`.
- Which VAT code applies: `estonian-bookkeeping`.
