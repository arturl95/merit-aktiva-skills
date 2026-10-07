# Bank statement import (camt.053)

Merit Aktiva imports bank statements in **ISO 20022 camt.053** XML — the format every Estonian bank has used since 2018. The endpoint is `sendcamt53`; Merit then auto-matches lines to open invoices.

## Where to get the file

| Bank | Path |
|---|---|
| Swedbank | Internetbank → Statements → Format: ISO XML (camt.053) |
| SEB | Online banking → Account statements → ISO XML (camt.053) |
| LHV | Online banking → Statements → ISO XML (camt.053) |
| Luminor | Online banking → Statements → ISO 20022 format |

For high-volume needs, all four banks offer **automated daily push** via FIDAVISTA or the EU PSD2 AISP API to a configured endpoint — outside the scope of this skill but worth wiring up.

## The endpoint

```
POST /api/v2/sendcamt53
Content-Type: application/xml (or text/xml)

<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:iso:std:iso:20022:tech:xsd:camt.053.001.02">
  ...camt.053 XML content...
</Document>
```

The request body is the **raw XML file** — not a JSON wrapper. Supported schema versions: `camt.053.001.02` and `camt.053.001.10`.

> **Important:** The endpoint name is `sendcamt53`, not `sendbankstatement`. The body is raw XML, not a JSON object with a `FileContent` base64 field. The Merit auth signature is still appended to the URL as query parameters (same as all other endpoints).

Response: a plain-text Estonian message such as `"Imporditi 22 makserida (ridu kokku 27)"` (imported 22 payment rows, 27 total). There is no structured JSON response with `ImportId`, `LinesTotal`, etc.

## Auto-matching logic

Merit attempts to match each statement line to an open invoice in this order:

1. **`RefNo` (viitenumber)** — the Estonian reference-number standard. If the bank line carries a viitenumber matching an outstanding invoice's `RefNo`, it auto-matches and posts a `sendpayment` / `sendPaymentV` automatically. This is the highest-confidence path.
2. **Counterparty IBAN + amount** — if the line has the customer/vendor's known IBAN and the amount matches an outstanding invoice within ±€0.05, it matches.
3. **Amount + date proximity** — last resort; the line is suggested but flagged as needing manual confirmation.

Lines with no match end up as **"to reconcile"** entries in the Merit UI (Bank → Reconcile). Use `getpayments` with `DateType: 1` (ChangedDate) to poll for newly-created payment records after an import.

## Reading imported lines — `PaymentImports`

```
GET /api/v2/PaymentImports?bankId=<bank-guid>&bookingDateFrom=YYYY-MM-DD&bookingDateTo=YYYY-MM-DD&withLines=true
(signature query params appended as usual; signed body is the empty string)
```

- Dates here are ISO `YYYY-MM-DD`, unlike most Merit endpoints.
- Max window is 3 months. Longer returns 400 `Valideerimine ebaõnnestus (PeriodTooLong)`.
- Get `bankId` from `getbanks`.

Verified response row (live test, 2026-10):

```json
{ "BookingDate": "2026-10-06T00:00:00", "AccServiceRRef": "<bank ref>", "PartName": "<counterparty name>",
  "Amount": 1.00, "CurrencyCode": "EUR", "Document": "", "Description": "<remittance text>" }
```

- `PartName` is returned although the docs don't list it. `RefNo` is added when the bank line carries a structured reference.
- **`PaymentImports` returns only lines that are not yet confirmed.** Once a line is linked and confirmed ("Kinnita") in the UI, it disappears from the response. Linked-but-unconfirmed lines stay and still show `Document: ""`. So "unmatched/unconfirmed bank lines" = the `PaymentImports` result, with `BookingDate` older than your SLA.

### Auto-matching in practice (verified)

- An incoming payment carrying the sales invoice's viitenumber (`<CdtrRefInf><Ref>`) was auto-linked (green).
- An outgoing payment with the exact amount, vendor name and "Arve <BillNo>" in the text was **not** auto-linked, not even after "Seo uuesti", when the vendor had several open items. Plan your own matcher for non-reference lines.
- A line marked as prepayment in the Võlgnevused window ("Ettemaksu summa") books D1350 / C bank on confirm.
