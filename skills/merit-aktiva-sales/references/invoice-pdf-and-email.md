# Invoice PDFs and email delivery

## `POST /api/v2/getsalesinvpdf` — fetch an invoice PDF

Request:

```json
{ "Id": "<invoice-guid>" }
```

Optional field: `DelivNote` (bool, lowercase `"true"` or `"false"`) — if `true`, returns the invoice without prices (delivery note layout).

**Note:** The official API documentation only documents `Id` as a request field for this endpoint. The `InvoiceNo` alternative is not confirmed in the live docs.

Response:

```json
{ "FileName": "Müügiarve 2026-0042.pdf", "FileContent": "<base64-PDF>" }
```

Decode `FileContent` to get the PDF bytes. Typical use cases:

- Forward to the customer outside the e-invoice channel.
- Archive in a document management system.
- Attach to an audit trail.

## `POST /api/v2/sendinvoicebyemail` — email an invoice

```json
{
  "Id": "<invoice-guid>",
  "Email": "billing@customer.ee",
  "CC": "ar-archive@us.example.com",
  "Subject": "Invoice 2026-0042 — Consulting May",
  "Body": "Please find attached our invoice for May consulting services."
}
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `Id` | GUID | yes (or `InvoiceNo`) | |
| `Email` | string | no | Falls back to customer's stored `Email` if omitted |
| `CC` | string | no | Single CC address |
| `Subject` | string | no | Default uses Merit's template |
| `Body` | string | no | Default uses Merit's template |

The email goes out via Merit's mail relay, so the from-address is `noreply@merit.ee` (or the company's configured custom from-domain if set in Merit UI).

## Attaching a custom PDF to a new invoice

When the customer expects a PDF you generated yourself (e.g. branded layout from your CRM), attach it via the `PDF` field on `sendinvoice`:

```json
POST /api/v2/sendinvoice
{
  "Customer": { "Id": "..." },
  "InvoiceNo": "2026-0099",
  "DocDate": "20260512",
  "DueDate": "20260526",
  "InvoiceRow": [ ... ],
  "TaxAmount": [ ... ],
  "TotalAmount": ...,
  "PDF": "<base64-encoded-pdf-bytes>",
  "FileName": "Custom-INV-2026-0099.pdf"
}
```

Notes:

- The PDF must be valid; Merit doesn't render it but stores it.
- Size: keep under ~5 MB. No hard limit documented but very large payloads can exceed Merit's JSON compilation budget.
- This attached PDF is what `sendinvoicebyemail` will dispatch and what `getsalesinvpdf` will return. It replaces the Merit-rendered PDF.

## Pulling all invoices' PDFs in bulk

Common for monthly archival:

```
invoices = POST /api/v2/getinvoices { PeriodStart, PeriodEnd, UnPaid: false }
for inv in invoices:
    pdf = POST /api/v2/getsalesinvpdf { "Id": inv.InvoiceId }
    save_to_disk(pdf.FileName, base64.decode(pdf.FileContent))
```

Honor the 100/min rate limit — for 500+ invoices, throttle to ~1.5/sec or batch with sleep between requests.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/sales-invoices/create-sales-invoice/get-sales-invoice-pdf/
- https://api.merit.ee/connecting-robots/reference-manual/sales-invoices/send-invoice-by-email/
