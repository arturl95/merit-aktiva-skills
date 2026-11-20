# Offer lifecycle — list, read, status, update, convert, prepay, units

All endpoints are `https://aktiva.merit.ee/api/v2/...`, signed like every other call (`merit-aktiva-core`). Verified 2026-10-15.

## Statuses

| `DocStatus` | Meaning | Set via |
|---|---|---|
| 1 | Created / unsent (Koostatud) | `sendoffer` default |
| 2 | Sent (Saadetud) | `setofferstatus` or sending from the UI |
| 3 | Approved by customer (Kinnitatud) | `setofferstatus` |
| 4 | Rejected (Tagasi lükatud) | `setofferstatus` |
| 5 | Comment received (Kommenteeritud) | `setofferstatus` with `Comment` |
| 6 | Invoice created (Arve koostatud) | set automatically by `offer2inv` or the UI |
| 7 | Cancelled (Tühistatud) | `setofferstatus`; the API's only "delete" |

## List — `POST /api/v2/getoffers`

```json
{ "DateType": 0, "PeriodStart": "20260715", "PeriodEnd": "20261015", "UnPaid": false }
```

- All four fields are required in practice; a missing one gives HTTP 500 `urn:FormatException` with no useful message.
- `DateType` 0 = document date, 1 = changed date (use 1 for incremental sync). Period max 3 months.
- Response rows mirror `getinvoices` (`SIHId` = offer Id, `OfferNo`, `DocumentDate`, `DueDate` = expiry, `CustomerName`, `TotalAmount`, `TaxAmount`, `TotalSum`, `DocType`, `DocStatus`, `DeliveryDate`, `HComment`, `FComment`, `ReferenceNo`, `EmailSent`, `ChangedDate`).
- To find the next free `OfferNo`, sort by `DocumentDate` and take the highest number in the series. Quotes, orders and prepayment invoices share one series.

## Read one — `POST /api/v2/getoffer`

```json
{ "Id": "<offer-guid>" }
```

Returns header fields (`OfferId`, `CustomerId`, `CustomerName`, `PayerId`, `DocDate`, `ExpireDate`, `DeliveryDate`, `OfferNo`, `DocType`, `DocStatus`, `ContactInfo`, `RefNo`, `CurrencyCode`, `HComment`, `FComment`, `TotalAmount`, `TaxAmount`, `RoundingAmount`, `ReserveItems`, `PrepaymPct`) and `OfferRow[]` with `SILId`, `ArticleCode`, `Description`, `Quantity`, `Price`, `AmountExclVat`, `VatAmount`, `AmountInclVat`, `TaxId`, `GLAccountCode`, `Dimensions`. Use it right after `sendoffer` to verify totals before telling the user the quote exists.

## Change status — `POST /api/v2/setofferstatus`

```json
{ "Id": "<offer-guid>", "NewStatus": 3, "Comment": "Accepted by phone 16.10" }
```

`NewStatus`: 1 unsent, 2 sent, 3 confirmed, 4 rejected, 5 commented (`Comment` required), 7 cancelled. Status 6 cannot be set by hand; it comes from `offer2inv`. Cancelling (7) is the only way to retire a wrong offer through the API, and the number stays used.

## Update — `POST /api/v2/updateoffer`

```json
{ "OfferNo": "P-100011", "DeriveryDate": "20261105", "HComment": "...", "FComment": "..." }
```

Only these fields can change, and the document is addressed by `OfferNo`, not Id. The misspelling `DeriveryDate` is the API's. For any other change (customer, rows, prices, dates) cancel the offer with `setofferstatus 7` and create a new one with the next number, or edit it in the UI (Müük → Pakkumised → open → pencil).

## Convert to invoice — `POST /api/v2/offer2inv`

```json
{ "Id": "<offer-guid>" }
```

Response: `{ "CustomerId", "InvoiceId", "InvoiceNo", "RefNo", "NewCustomer" }` for the **new sales invoice**. Merit copies the rows, dates the invoice today, numbers it from the invoice series and sets the offer to status 6. After this, the document is a real invoice: send it as e-invoice or email with `merit-aktiva-sales` (`sendinvoiceaseinv`, `sendinvoicebyemail`), and record its payment with `merit-aktiva-purchases-payments`. Check the customer's delivery channel first (e-arve vs email) if the company keeps such rules.

Do this only when the customer has actually accepted; converting posts revenue and VAT.

## Prepayment against an offer — `POST /api/v2/sendPaymentO`

```json
{ "BankId": "<bank-guid>", "IBAN": "EE…", "CustomerName": "Rada OÜ",
  "PaymentDate": "202610201200", "OfferNo": "P-100011", "RefNo": "…", "Amount": 2654.04,
  "CurrencyCode": "EUR" }
```

Records a customer prepayment (ettemaks) linked to the offer, typically for `DocType` 3. `PaymentDate` is `YYYYmmddHHii`. `CurrencyCode`/`CurrencyRate` only when not in the local currency. Bank Ids come from `getbanks`. Most companies book prepayments through the bank import instead; use this only when the user asks for it explicitly.

## Sending the document

There is no API to email a quote or fetch its PDF (`getsalesinvpdf` and `sendinvoicebyemail` accept invoice Ids only). The user sends it from Merit: Müük → Pakkumised → open the offer → **Saada e-postiga** (or print to PDF). If the customer card has an `Email`, it is pre-filled, so set it with `updatecustomer` when you know it. Sending from the UI moves the status to 2.

## Units of measure

```
GET  /api/v2/units          → [ { "Code": "tk", "Name": "tk" }, { "Code": "tundi", "Name": "tund" }, … ]
POST /api/v2/senduom        { "Name": "h", "NamePlural": "h", "NameEN": "h", "NamePluralEN": "h" }
```

- `units` is a **GET** (POST returns 405). Sign it like a POST with an empty body: HMAC over `apiId + timestamp` with no payload, then `?apiId=…&timestamp=…&signature=…`.
- Rows match units by **Name**, not `Code` (`tundi` has name `tund`). A fresh Estonian company has `tk`, `km`, `tund`, `kuu`, `kg`, `m²`, `m³`, `ltr`. `h`, `m`, `kompl`, `pcs` do not exist until added.
- `senduom` returns the new unit's Id, or `api-uomexsist` if the name is already there. Adding a unit is a small write; ask the user before doing it, and prefer an existing unit (`tund` for hours, `tk` for a kit or a cable length when the quantity is a count) when the user has not asked for a new one.
