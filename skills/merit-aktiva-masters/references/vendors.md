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
| `PhoneNo` | string(50) | no | |
| `PhoneNo2` | string(50) | no | |
| `HomePage` | string(80) | no | |
| `Email` | string(80) | no | |
| `CurrencyCode` | string(4) | no | Default `EUR` |
| `PaymentDeadLine` | int | no | Days |
| `OverDueCharge` | decimal(5,2) | no | |
| `RefNoBase` | string(36) | no | |
| `BankAccount` | string(50) | no | IBAN (used by `sendpayment`) |
| `SWIFT_BIC` | string(30) | no | BIC code |
| `ReceiverName` | string(150) | no | When payee differs from vendor name |

### v2-only additions

| Field | Type | Notes |
|---|---|---|
| `Dimensions` | array of `{ DimId, DimCode, DimValueId }` | `DimId` is Int |
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
| `WithComments` | bool | Include `Comments[]` in each row |
| `CommentsFrom` | date | Comments from this date onward |
| `ChangedDate` | YYYYMMDD | Incremental cursor |

### Response

`VendorId`, `VendorType`, `Name`, `RegNo`, `VatRegNo`, `Contact`, `PhoneNo`, `PhoneNo2`, `FaxNo`, `Address`, `City`, `County`, `PostalCode`, `CountryName`, `CountryCode`, `Email`, `HomePage`, `CurrencyCode`, `VendorGroupId`, `VendorGroupName`, `PaymentDeadLine`, `OverdueCharge`, `VatAccountable`, `BankAccount`, `ReferenceNo`, `ChangedDate`, `Dimensions[]`.

Note: the API response field for bank reference is `ReferenceNo` (not `RefNoBase`). `SWIFT_BIC`, `BankName`, and `ReceiverName` are send-only fields not returned in getvendors. `Comments[]` is returned when `WithComments: true`.

## `POST /api/v1/updatevendor` (also available as v2)

Update an existing vendor. `Id` is required; all other fields are optional.

| Field | Type | Notes |
|---|---|---|
| `Id` | GUID | Required — identifies the vendor |
| `Name` | string(150) | |
| `CountryCode` | string(2) | |
| `Address` | string(100) | |
| `City` | string(30) | |
| `County` | string(100) | |
| `PostalCode` | string(15) | |
| `PhoneNo` | string(50) | |
| `PhoneNo2` | string(50) | |
| `Email` | string | |
| `HomePage` | string(80) | |
| `RegNo` | string(30) | |
| `VatRegNo` | string(30) | |
| `VatAccountable` | bool | |
| `SalesInvLang` | string(2) | EE, FI, PL |
| `BankAccount` | string(50) | |
| `SWIFT_BIC` | string(30) | v1 only |
| `ReferenceNo` | string(36) | |
| `VendGrCode` | string(20) | |
| `VendGrId` | GUID | |
| `PayerReceiverName` | string(150) | v2 only |
| `Dimensions` | array | v2 only; `{ DimId, DimValueId, DimCode }` |

Response: not documented; treat as success/error HTTP status.

## `POST /api/v2/sendvendorgroup` / `POST /api/v2/getvendorgroups`

Vendor groups mirror customer groups. `sendvendorgroup` fields: `Id` (GUID, optional), `Name` (string(64), required), `Code` (string(20), required). Response: `{ Id, Name, Code }`.
`getvendorgroups` body: `{}`. Response: array of `{ Id, Name, Code }`.

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
  "CurrencyCode": "EUR",
  "PaymentDeadLine": 14,
  "BankAccount": "DE89 3704 0044 0532 0130 00",
  "SWIFT_BIC": "COBADEFFXXX",
  "Dimensions": [
    { "DimCode": "COST_CENTER", "DimValueId": "11111111-..." }
  ]
}
```

## Source

- https://api.merit.ee/connecting-robots/reference-manual/vendors/create-vendor/
