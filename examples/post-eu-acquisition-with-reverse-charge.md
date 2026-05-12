# Example — Post an EU acquisition with reverse charge

**Scenario**: Post a purchase invoice from a German cloud provider (Hetzner Online GmbH, DE VAT registered) for €500 of EU consulting services. Intra-EU B2B service → reverse charge applies (Art. 44 / VAT Act §10).

**Skills used**: `merit-aktiva-core`, `merit-aktiva-masters`, `merit-aktiva-purchases-payments`, `estonian-bookkeeping`.

---

## Step 1 — Pre-flight

Env vars set. Caches initialized (`gettaxes`, `getaccounts`).

From `gettaxes`, find the EU-services reverse-charge entry. Typical `Code`: `EL24` or similar, with `TaxPct: 24.00` and the name mentioning "pöördkm" (reverse VAT). Cache as `eu_services_reverse_id`.

Why this code and not regular 24%: see `estonian-bookkeeping/references/vat-rates-2026.md` row 4 — intra-EU services hit KMD line 7 (other reverse-charge acquisitions) plus the offsetting input/output VAT.

## Step 2 — Resolve vendor

```
POST /api/v1/getvendors    body: {"VatRegNo": "DE812871812"}
```

If not found, create:

```json
POST /api/v2/sendvendor
{
  "Name": "Hetzner Online GmbH",
  "RegNo": "HRB 53984",
  "VatRegNo": "DE812871812",
  "VatAccountable": true,
  "VendorType": 1,
  "CountryCode": "DE",
  "Address": "Industriestr. 25",
  "City": "Gunzenhausen",
  "PostalCode": "91710",
  "Email": "billing@hetzner.com",
  "CurrencyCode": "EUR",
  "PaymentDeadLine": 14,
  "BankAccount": "DE89 3704 0044 0532 0130 00",
  "SWIFT_BIC": "COBADEFFXXX"
}
```

Confirm with user before POST. Response gives `Id`.

## Step 3 — Build the purchase invoice payload

```json
{
  "Vendor": { "Id": "<vendor-guid>" },
  "DocNo": "DE-2026-INV-7788",
  "DocDate": "20260508",
  "TransactionDate": "20260508",
  "DueDate": "20260522",
  "CurrencyCode": "EUR",
  "InvoiceRow": [
    {
      "Description": "Cloud hosting May 2026",
      "Quantity": 1,
      "Price": 500.00,
      "GLAccountCode": "5530",
      "TaxId": "<eu_services_reverse_id>"
    }
  ],
  "TaxAmount": [
    { "TaxId": "<eu_services_reverse_id>", "Amount": 120.00 }
  ],
  "TotalAmount": 500.00
}
```

Notes:

- `DocNo` is **Hetzner's** invoice number (printed on their PDF), not your own counter.
- `Price` is the net amount (what Hetzner billed) — they did NOT add VAT because intra-EU B2B services use reverse charge (their invoice should note "Reverse charge — VAT due by recipient").
- `TotalAmount` is the **net** (cash payable to vendor) = €500.00.
- `TaxAmount` is the self-charged Estonian VAT at 24% on €500 = €120.00. This appears in KMD as output VAT, then immediately as input VAT, so the net VAT cash position is zero.

## Step 4 — Show payload, confirm

> Will post purchase invoice **DE-2026-INV-7788** from **Hetzner Online GmbH** (DE812871812) dated **2026-05-08**, due **2026-05-22**.
> Cloud hosting €500.00 net.
> EU reverse-charge: self-account €120.00 input VAT and €120.00 output VAT (net VAT cash = 0).
> **Cash payable to vendor: €500.00.**

Wait for "yes".

## Step 5 — POST

```
POST /api/v2/sendpurchinvoice    body: <payload above>
```

Response:

```json
{
  "VendorId": "<guid>",
  "PurchInvoiceId": "<guid>",
  "DocNo": "DE-2026-INV-7788",
  "NewVendor": false
}
```

## Step 6 — Auto-bookings (what Merit posts)

The reverse-charge tax code triggers Merit to post:

- Dr 5530 IT-kulud €500.00
- Dr 1230 Sisendkäibemaks €120.00 (input VAT)
- Cr 21xx Võlad tarnijatele €500.00 (AP to Hetzner)
- Cr 2120 Käibemaksu kohustus €120.00 (output VAT)

Net result: AP €500 to Hetzner, expense €500, VAT cash zero, VAT declarations populated.

## Step 7 — Pay the vendor (later)

When the bank transfer goes out:

```
POST /api/v2/sendpurchasepayment
{
  "Vendor": { "Id": "<vendor-guid>" },
  "InvoiceNo": "DE-2026-INV-7788",
  "PaymentDate": "20260520",
  "Amount": 500.00,
  "IBAN": "EE382200221020145685",
  "CurrencyCode": "EUR"
}
```

Confirm payload, then post.

## KMD impact (this month)

- **Line 7** (other reverse-charge acquisitions): +€500.00
- **Line 4** (VAT due 24%): +€120.00
- **Line 5** (deductible input VAT): +€120.00

Net effect on month-end KMD: €0 cash to EMTA from this transaction (output and input offset), but both lines are populated for declarative purposes.

## Common variations

### Intra-EU goods purchase (not services)

Use a different tax code — typically `EL kaup pöördkm 24%`. KMD hits line 6 (intra-EU acquisition of goods, base) + 6.1 (taxable) instead of line 7. Mechanics are identical.

### Non-EU services (e.g. from US supplier)

Use `3. riigi teenus pöördkm 24%`. KMD hits line 7 (other reverse-charge acquisitions). Mechanics identical.

### Vendor charges VAT incorrectly

If the EU vendor billed Estonian VAT (mistake — should be 0% reverse charge for B2B), ask them to credit and reissue with reverse-charge wording. If they refuse, post as a regular foreign-VAT purchase and chase reclaim separately — but this is unusual.

## Cross-references

- VAT mapping → `estonian-bookkeeping/references/vat-rates-2026.md` rows 3 (goods), 4 (services), 5 (non-EU services).
- Full `sendpurchinvoice` schema → `merit-aktiva-purchases-payments/references/purchase-invoices.md`.
- Vendor schema → `merit-aktiva-masters/references/vendors.md`.
