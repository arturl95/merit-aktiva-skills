# Vendors — full reference

Vendors mirror customers, with three key differences: the VAT flag is `VatAccountable` instead of `NotTDCustomer`, there is no `EInvOperator` or `SalesInvLang` (vendors receive invoices, they don't send them), and bank details (`BankAccount`, `SWIFT_BIC`) are first-class fields.

## `POST /api/v2/sendvendor`

### Fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `Id` | GUID | no | If present, others ignored (update) |
| `Name` | string(150) | yes (create) | Unique |
| `RegNo` | string(30) | recommended | Estonian äriregister code for EE; national registry number elsewhere |
| `VatRegNo` | string(30) | optional | `EE` + 9 digits or country prefix + national number |
| `VatAccountable` | bool | yes (create) | `true` for VAT-registered suppliers whose input VAT can be claimed |
| `VendorType` | enum | recommended | 1 = vendor, 3 = reporting entity (used for statutory bodies) |
| `CountryCode` | string(2) | yes (create) | ISO 3166-1 alpha-2 |
| `Address`, `City`, `County`, `PostalCode` | strings | no | |
| `PhoneNo`, `PhoneNo2`, `FaxNo` | strings | no | |
| `HomePage`, `Email` | strings | no | |
| `Contact` | string(35) | no | |
| `CurrencyCode` | string(4) | no | Default `EUR` |
| `PaymentDeadLine` | int | no | Days |
| `BankAccount` | string(50) | no | IBAN (used by `sendpayment`) |
| `SWIFT_BIC` | string | no | BIC code |
| `BankName` | string | no | |
| `ReceiverName` | string(150) | no | When payee differs from vendor name |
| `Notes` | string | no | |

### v2-only additions

| Field | Type | Notes |
|---|---|---|
| `Dimensions` | array of `{ DimCode, DimValueId }` | |
| `VendGrCode` / `VendGrId` | string(20) / GUID | Vendor group |
| `Comments` | array of `{ Comment, CommDate }` | |

### Response

```json
{ "Id": "guid", "Name": "Hetzner Online GmbH" }
```

## `POST /api/v1/getvendors`

### Filter fields

| Field | Type | Notes |
|---|---|---|
| `Id` | GUID | If present, others ignored |
| `RegNo`, `VatRegNo` | string | Exact |
| `Name` | string | Broad |
| `ChangedDate` | YYYYMMDD | Incremental cursor |

### Response

`VendorId`, `Name`, `RegNo`, `VatRegNo`, `Contact`, phone/address fields, `Email`, `CurrencyCode`, `VendorGroupName`, `PaymentDeadLine`, `VatAccountable`, `BankName`, `BankAccount`, `SWIFT_BIC`, `ReceiverName`, `VendorType`, `ChangedDate`, `Comments[]`, `Dimensions[]`.

## When the vendor is an EU supplier

Set `VatAccountable: true` and supply the EU VAT number in `VatRegNo` with the country prefix (`DE`, `FR`, `FI`, etc.). The invoice you later post against this vendor will use the intra-EU reverse-charge TaxId from `gettaxes` (Code typically `EL`-prefixed). See `estonian-bookkeeping/references/vat-rates-2026.md` row 4 of the transaction cheatsheet.

## When the vendor is a non-EU supplier

Set `VatAccountable: true` if they would be VAT-registered in their country; otherwise `false`. Either way, the invoice posted against them carries the "services from outside EU" reverse-charge TaxId (Code typically `3R`-prefixed in EE templates).

## Rich example

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
  "HomePage": "https://www.hetzner.com",
  "Contact": "Accounts Receivable",
  "CurrencyCode": "EUR",
  "PaymentDeadLine": 14,
  "BankAccount": "DE89 3704 0044 0532 0130 00",
  "SWIFT_BIC": "COBADEFFXXX",
  "BankName": "Commerzbank",
  "Dimensions": [
    { "DimCode": "COST_CENTER", "DimValueId": "11111111-..." }
  ]
}
```

## Source

- https://api.merit.ee/connecting-robots/reference-manual/vendors/create-vendor/
