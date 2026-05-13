# Customers — full reference

## `POST /api/v2/sendcustomer`

Create or upsert a customer. The `Id` field, if present, switches to update mode (other fields ignored except the ones you supply for the update).

### Fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `Id` | GUID | no | If present, others ignored (update) |
| `Name` | string(150) | yes (create) | Unique |
| `RegNo` | string(30) | recommended | 8-digit Estonian äriregister code |
| `VatRegNo` | string(30) | optional | `EE` + 9 digits for Estonia; country prefix + national number for EU |
| `NotTDCustomer` | bool | yes (create) | `true` for physical persons / foreign companies |
| `CountryCode` | string(2) | yes (create) | ISO 3166-1 alpha-2, e.g. `EE` |
| `Address` | string | no | |
| `City` | string | no | |
| `County` | string | no | |
| `PostalCode` | string | no | |
| `PhoneNo`, `PhoneNo2` | string | no | |
| `HomePage` | string | no | |
| `Email` | string | no | Used by `sendinvoicebyemail` if set |
| `CurrencyCode` | string(4) | no | Default `EUR` |
| `PaymentDeadLine` | int | no | Days (e.g. 14) |
| `OverDueCharge` | decimal(5,2) | no | % per year |
| `RefNoBase` | string(36) | no | Base for auto-generated payment reference numbers |
| `SalesInvLang` | enum | no | `ET`, `EN`, `RU`, `FI` (EE); `PL`, `EN`, `RU` (PL) |
| `Contact` | string(35) | no | Primary contact name |
| `EInvPaymId` | string(20) | no | E-invoice payment identifier |
| `EInvOperator` | enum | no | See below |
| `BankAccount` | string(50) | no | IBAN |
| `ApixEInv` | string(20) | no | Apix/Fitek aggregator identifier |
| `GroupInv` | bool | no | Aggregate invoices in group billing |

### v2-only additions

| Field | Type | Notes |
|---|---|---|
| `GLNCode` | string(10) | Global Location Number for e-invoicing |
| `PartyCode` | string(20) | Network party code |
| `PayerReceiverName` | string(150) | When payer differs from receiver |
| `Dimensions` | array of `{ DimId, DimCode, DimValueId }` | Custom dimensions; `DimId` is Int |
| `CustGrCode` / `CustGrId` | string(20) / GUID | Customer group |
| `ShowBalance` | bool | Show running balance on invoices |
| `BankOnSalesInvoice` | GUID | Specific bank account to show on invoices |
| `Comments` | array of `{ Comment, CommDate }` | |

### `EInvOperator` enum

| Value | Meaning |
|---|---|
| 1 | Not in service registry (no e-invoice) |
| 2 | Omniva e-arvekeskus / Telema EDI |
| 3 | Bank full e-invoice channel |
| 4 | Bank limited |

For Apix/Fitek aggregators, fill `ApixEInv` instead and leave `EInvOperator` at 1.

### Response

```json
{ "Id": "8a3b...-...", "Name": "Acme OÜ" }
```

## `POST /api/v1/getcustomers`

Filter the customer list. **Never call with an empty filter** — large unfiltered queries can return ASP.NET stack traces inside a 200.

### Filter fields

| Field | Type | Notes |
|---|---|---|
| `Id` | GUID | If present, others ignored |
| `RegNo` | string | Exact match |
| `VatRegNo` | string | Exact match |
| `Name` | string | Broad case-insensitive substring |
| `WithComments` | bool | Include `Comments[]` in each row |
| `CommentsFrom` | date | Comments from this date onward |
| `ChangedDate` | YYYYMMDD | Incremental cursor — returns customers modified on or after this date |

### Response object fields

`CustomerId`, `Name`, `RegNo`, `VatRegNo`, `Contact`, `PhoneNo`, `PhoneNo2`, `FaxNo`, `Address`, `City`, `County`, `PostalCode`, `CountryName`, `CountryCode`, `Email`, `HomePage`, `CurrencyCode`, `CustomerGroupId`, `CustomerGroupName`, `PaymentDeadLine`, `OverdueCharge`, `NotTDCustomer`, `BankName`, `BankAccount`, `SalesInvLang`, `RefNoBase`, `GLNCode`, `PartyCode`, `TelemaEdi`, `InvSendPref`, `ChangedDate`, `Comments[]`, `Dimensions[]`.

`Dimensions` array objects: `{ Id, DimId, DimValueId, DimCode }`.

## `POST /api/v1/updatecustomer`

Update an existing customer. `Id` is required; all other fields are optional and only the supplied fields are changed.

### Fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `Id` | GUID | yes | Identifies the customer to update |
| `Name` | string(150) | no | |
| `CountryCode` | string(2) | no | |
| `Address` | string(100) | no | |
| `City` | string(30) | no | |
| `County` | string(100) | no | |
| `PostalCode` | string(15) | no | |
| `PhoneNo` | string(50) | no | |
| `PhoneNo2` | string(50) | no | |
| `Email` | string(80) | no | |
| `RegNo` | string(30) | no | |
| `VatRegNo` | string(30) | no | |
| `SalesInvLang` | string(2) | no | |
| `RefNoBase` | string(36) | no | |
| `EInvPaymId` | string(20) | no | |
| `EInvOperator` | int | no | |
| `BankAccount` | string(50) | no | |
| `CustGrCode` | string(20) | no | |
| `CustGrId` | GUID | no | |
| `Contact` | string(35) | no | |
| `ApixEInv` | string(20) | no | |
| `GroupInv` | bool | no | |
| `PaymentDeadLine` | int | no | |
| `OverDueCharge` | decimal(5,2) | no | |
| `NotTDCustomer` | bool | no | |
| `PayerReceiverName` | string(150) | no | |
| `CurrencyCode` | string(4) | no | |
| `HomePage` | string(80) | no | |
| `GLNCode` | string(10) | no | |
| `PartyCode` | string(20) | no | |
| `ShowBalance` | bool | no | |
| `BankOnSalesInvoice` | GUID | no | |
| `Dimensions` | array | no | `{ DimId, DimValueId, DimCode }` |
| `Comments` | array | no | `{ Comment, CommDate }` |

Response: not documented; treat as success/error HTTP status.

## `POST /api/v2/sendcustomergroup`

Create or update a customer group.

| Field | Type | Req | Notes |
|---|---|---|---|
| `Id` | GUID | no | If present, updates existing |
| `Name` | string(64) | yes | |
| `Code` | string(20) | yes | |

Response: `{ Id, Name, Code }`.

## `POST /api/v2/getcustomergroups`

Returns all customer groups. Body: `{}`. Response: array of `{ Id, Name, Code }` (all string/GUID fields).

## Incremental sync pattern

```
high_water = read_last_sync_date()                # YYYYMMDD or null
filter     = { "ChangedDate": high_water }
results    = POST /api/v1/getcustomers (filter)
upsert_local(results)
write_last_sync_date(max(r.ChangedDate for r in results) - 1 day)
```

Subtracting 1 day on write protects against clock skew between Merit and your system.

## Rich example

```json
POST /api/v2/sendcustomer
{
  "Name": "Acme Logistics OÜ",
  "RegNo": "11223344",
  "VatRegNo": "EE100200300",
  "NotTDCustomer": false,
  "CountryCode": "EE",
  "Address": "Pärnu mnt 100",
  "City": "Tallinn",
  "County": "Harju maakond",
  "PostalCode": "10115",
  "PhoneNo": "+372 5555 1234",
  "Email": "billing@acme-logistics.ee",
  "HomePage": "https://acme-logistics.ee",
  "CurrencyCode": "EUR",
  "PaymentDeadLine": 14,
  "OverDueCharge": 12.0,
  "SalesInvLang": "ET",
  "Contact": "Mari Maasikas",
  "EInvOperator": 2,
  "GLNCode": "1234567890123",
  "Dimensions": [
    { "DimCode": "REGION", "DimValueId": "a1b2c3d4-..." }
  ],
  "Comments": [
    { "Comment": "Annual rebate 5% on >€50k turnover", "CommDate": "20260101" }
  ]
}
```

## Source

- https://api.merit.ee/connecting-robots/reference-manual/customers/create-customer/
- https://api.merit.ee/connecting-robots/reference-manual/customers/get-customer-list/
