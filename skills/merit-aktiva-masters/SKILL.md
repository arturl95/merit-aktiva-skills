---
name: merit-aktiva-masters
description: Look up and create Merit Aktiva customers, vendors, items, tax codes, and chart-of-accounts entries. Use to resolve names to GUIDs before posting transactions, or to onboard new master data.
last-verified: 2026-05-12
---

# merit-aktiva-masters

Resolve human-readable names to the GUIDs every transactional Merit endpoint needs, and onboard new master data when something doesn't exist yet.

## When to use

- Before any `sendinvoice`, `sendpurchinvoice`, `sendpayment`, `sendglbatch` call — to resolve customer/vendor/item/account/tax identifiers.
- When the user asks to "add a new customer", "create a service item", "list our suppliers", etc.
- At the start of a session, to populate the per-session caches described below.

## Cache-once-per-session rule

Two endpoints return per-tenant configuration. Call each once at the start of a session and cache the result:

- **`POST /api/v1/gettaxes`** with body `{}` — returns the company's VAT codes as a list of `{ Id (GUID), Code, Name, NameEN, NameRU, TaxPct }`. **TaxId GUIDs are unique per company; never hard-code them across tenants.** Map by `Code` (`KM24`, `KM13`, `KM9`, `KM0`, …) and `TaxPct`.
- **`POST /api/v1/getaccounts`** with body `{ "UsageFilter": 0 }` — returns the chart of accounts. Filter values: 0 = all, 1 = cost accounts, 2 = cost contra-accounts, 3 = purchase VAT accounts. Most workflows use 0.

These responses are stable across a session. Invalidate the cache only when a tax rate is changed in the Merit UI or a new account is added.

## Flags you must know

### Customers — `NotTDCustomer`

Required boolean on `sendcustomer`:

- `false` — normal Estonian B2B customer with an äriregister code.
- `true` — physical person, foreign company, or anyone who shouldn't appear on "tax declaration customer" reports.

Defaults differ across Merit editions; always set it explicitly.

### Vendors — `VatAccountable`

Required boolean on `sendvendor`:

- `true` — VAT-registered supplier; their VAT can be claimed as input VAT.
- `false` — non-VAT-registered supplier (e.g. a small business, a foreign supplier without an EU VAT number).

Pair with `VatRegNo` for VAT-registered EU suppliers (format `EE123456789` for Estonia, country code prefix for others).

## Broad-search gotcha

Unfiltered `getcustomers` / `getvendors` calls can return ASP.NET stack traces inside a 200 response when the result set is too large. Always pass a filter:

- `Id` (GUID) for exact match — preferred when you have it.
- `RegNo` or `VatRegNo` for exact match by registry/VAT number.
- `Name` for broad match (case-insensitive substring).
- `ChangedDate` as a high-water-mark cursor for incremental sync.

Same applies to `getitems` — filter by `Code`, `Description`, or `LocationCode` rather than pulling everything.

## Minimum examples

### Create a customer

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

Response: `{ "Id": "<guid>", "Name": "Acme OÜ" }`. Names must be unique.

### Create a vendor

```json
POST /api/v2/sendvendor
{
  "Name": "Hetzner Online GmbH",
  "RegNo": "HRB 53984",
  "VatAccountable": true,
  "VatRegNo": "DE812871812",
  "CountryCode": "DE",
  "VendorType": 1,
  "CurrencyCode": "EUR",
  "Email": "billing@hetzner.de",
  "BankAccount": "DE89 3704 0044 0532 0130 00",
  "SWIFT_BIC": "COBADEFFXXX"
}
```

### Create a service item

```json
POST /api/v2/senditems
{
  "Items": [
    {
      "Code": "CONSULT-HR",
      "Description": "Consulting (hourly)",
      "Type": 2,
      "Usage": 1,
      "UOMName": "h",
      "TaxId": "<24%-tax-guid-from-gettaxes-cache>",
      "SalesAccCode": "30000"
    }
  ]
}
```

`Type`: 1 = stock, 2 = service, 3 = item. `Usage`: 1 = sales, 2 = purchases, 3 = both.

## Guardrails

Master-data writes (`sendcustomer`, `sendvendor`, `senditems`, `sendtax`) are state-changing. Show the payload to the user before posting and wait for confirmation. Most onboarding mistakes (wrong RegNo, wrong `NotTDCustomer`, wrong default account) propagate into every later transaction, so the confirmation step is especially valuable here.

## When to read each reference

- [customers.md](references/customers.md) — full `sendcustomer` and `getcustomers` schemas, EInvOperator enum, language codes, dimensions.
- [vendors.md](references/vendors.md) — vendor specifics: `VatAccountable`, `VendorType`, bank fields.
- [items.md](references/items.md) — `Type` and `Usage` enums, multi-location stock, per-language descriptions, GL account override fields.
- [taxes-and-accounts.md](references/taxes-and-accounts.md) — building the per-session caches; mapping to Estonian RTJ defaults.

## Cross-references

- For the auth recipe: `merit-aktiva-core`.
- For "which VAT code is right for this transaction": `estonian-bookkeeping`.
- For sales/purchase/payment writes that consume these GUIDs: the corresponding `merit-aktiva-*` skill.
