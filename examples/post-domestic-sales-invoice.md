# Example — Post a domestic sales invoice

**Scenario**: Issue a €1,000 + 24% VAT consulting invoice to an existing Estonian customer (Acme OÜ) for 10 hours of work at €100/h.

**Skills used**: `merit-aktiva-core`, `merit-aktiva-masters`, `merit-aktiva-sales`, `estonian-bookkeeping`.

---

## Step 1 — Pre-flight

Confirm env vars are set:

```bash
echo "API_ID: ${MERIT_API_ID:0:8}..."
echo "API_KEY: ${MERIT_API_KEY:+set}"
```

Confirm the connected company is a trial company before any first write in a session (unless the user has already said "production").

## Step 2 — Initialize caches

```
POST /api/v1/gettaxes        body: {}
POST /api/v1/getaccounts     body: {"UsageFilter": 0}
```

From the `gettaxes` response, find the 24% domestic VAT entry by `Code` (typical: `KM24`) and `TaxPct: 24.00`. Cache its `Id` as `tax24_id`.

## Step 3 — Resolve customer

```
POST /api/v1/getcustomers    body: {"RegNo": "12345678"}
```

Two outcomes:

- **Found** → grab `CustomerId`. Proceed.
- **Not found** → create:
  ```json
  POST /api/v2/sendcustomer
  {
    "Name": "Acme OÜ",
    "RegNo": "12345678",
    "NotTDCustomer": false,
    "CountryCode": "EE",
    "VatRegNo": "EE123456789",
    "Email": "billing@acme.ee",
    "CurrencyCode": "EUR",
    "PaymentDeadLine": 14,
    "SalesInvLang": "ET",
    "EInvOperator": 1
  }
  ```
  Confirm with user before posting. Response gives `Id`.

## Step 4 — Resolve item

```
POST /api/v1/getitems        body: {"Code": "CONSULT-HR"}
```

If not found, create:

```json
POST /api/v2/senditems
{
  "Items": [{
    "Code": "CONSULT-HR",
    "Description": "Consulting (hourly)",
    "Type": 2,
    "Usage": 1,
    "UOMName": "h",
    "TaxId": "<tax24_id>",
    "SalesAccCode": "3000"
  }]
}
```

## Step 5 — Build the invoice payload

```json
{
  "Customer": { "Id": "<customer-guid>" },
  "InvoiceNo": "2026-0042",
  "DocDate": "20260512",
  "DueDate": "20260526",
  "InvoiceRow": [
    {
      "Item": { "Code": "CONSULT-HR", "Description": "Consulting (May, 10h)", "Type": 2 },
      "Quantity": 10,
      "Price": 100.00,
      "TaxId": "<tax24_id>"
    }
  ],
  "TaxAmount": [{ "TaxId": "<tax24_id>", "Amount": 240.00 }],
  "TotalAmount": 1000.00,
  "Hcomment": "Thank you for your business."
}
```

VAT math:
- Net: 10 × 100 = €1,000.00
- VAT 24%: €240.00
- Grand total: €1,240.00

## Step 6 — Show payload, confirm

Tell the user:

> Will create invoice **2026-0042** dated **2026-05-12**, due **2026-05-26**, for customer **Acme OÜ** (RegNo 12345678).
> 10h consulting @ €100/h = €1,000.00 net.
> VAT 24%: €240.00.
> **Grand total: €1,240.00**.

Then ask: "Confirm post?"

## Step 7 — POST

Wait for "yes". Then:

```
POST /api/v2/sendinvoice    body: <the payload above>
```

Response:

```json
{
  "CustomerId": "<guid>",
  "InvoiceId":  "<guid>",
  "InvoiceNo":  "2026-0042",
  "RefNo":      "20260042",
  "NewCustomer": false
}
```

## Step 8 — (Optional) Retrieve PDF

```
POST /api/v2/getsalesinvpdf   body: {"Id": "<InvoiceId>"}
```

Decode `FileContent` (base64) and save locally if the user wants the PDF.

## Step 9 — (Optional) Email

```
POST /api/v2/sendinvoicebyemail
{ "Id": "<InvoiceId>", "Email": "billing@acme.ee" }
```

## Outcome

- Sales invoice 2026-0042 posted, total €1,240.00 incl. 24% VAT.
- KMD line 1 (taxable supply 24%) +€1,000.00; line 4 (VAT due 24%) +€240.00.
- AR balance for Acme OÜ +€1,240.00 (account 1210).
- Revenue (account 3000) +€1,000.00.
- Output VAT (account 2120) +€240.00.

## Cross-references

- VAT code mapping → `estonian-bookkeeping/references/vat-rates-2026.md` row 1.
- Full `sendinvoice` schema → `merit-aktiva-sales/references/sendinvoice-schema.md`.
- Customer schema → `merit-aktiva-masters/references/customers.md`.
