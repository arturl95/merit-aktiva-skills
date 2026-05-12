# Chart of accounts

## `POST /api/v1/getaccounts`

Returns the company's chart of accounts. The endpoint exists only at v1.

Body:

```json
{ "UsageFilter": 0 }
```

| `UsageFilter` | Meaning |
|---|---|
| 0 (default) | All accounts |
| 1 | Cost accounts (expense side) |
| 2 | Cost contra-accounts |
| 3 | Purchase VAT accounts |

## Response object fields

| Field | Notes |
|---|---|
| `AccountID` | GUID |
| `NoActive` | bool — inactive accounts excluded from new postings |
| `Code` | Account code (e.g. `1010`, `3000`) |
| `Name` | Local-language name |
| `NameEN` / `NameRU` | Translations |
| `TaxName` / `TaxNameEN` / `TaxNameRU` | Default VAT code linked to this account (if any) |
| `LinkedVendorName` | If the account is a vendor-specific sub-ledger |
| `IsParent` | If true, this is a header line in the chart; not used directly for postings |
| `ReportdescName` / `...EN` | Financial statement category (e.g. "Current assets") |
| `CashflowName` / `...EN` | Cash-flow statement category |

## Caching

Cache `getaccounts` per session. Build an `accounts_by_code` index:

```python
accounts = sign_and_post("/api/v1/getaccounts", {"UsageFilter": 0})
by_code = {a["Code"]: a for a in accounts if not a["NoActive"]}
```

Invalidate the cache when:
- A new account is added via the Merit UI.
- An account is renamed or deactivated.

## Estonian RTJ default chart

Merit ships a default Estonian chart aligned with RTJ (small-company financial reporting standard). Common codes — **verify these per tenant** because each company can rename or renumber:

### Assets (1xxx)

| Code | Name | Use |
|---|---|---|
| 1010 | Kassa | Cash on hand |
| 1020 / 1110 | Pangakontod | Bank accounts |
| 1210 | Nõuded ostjate vastu | Accounts receivable — domestic |
| 1230 | Sisendkäibemaks | Input VAT (recoverable) |
| 1370 | Material in stock / Tooraine | Stock asset (varies by company) |
| 18xx | Põhivara | Fixed assets (1810 land, 1820 buildings, 1825 accum. depreciation, etc.) |

### Liabilities & Equity (2xxx)

| Code | Name | Use |
|---|---|---|
| 2110 | Võlad tarnijatele | Accounts payable — domestic |
| 2120 | Käibemaksu kohustus | Output VAT (payable) |
| 2150 | Viitlaekumised | Accrued liabilities |
| 2380 | Töötasude võlg | Wages payable |
| 2390 | Maksuvõlad | Payroll/other taxes payable |
| 2410 | Käibemaks (older charts) | VAT control account |
| 2510 | Osakapital | Share capital |
| 2530 | Jaotamata kasum | Retained earnings |

### Revenue (3xxx)

| Code | Name | Use |
|---|---|---|
| 3000 | Müügitulu | Sales — domestic, standard VAT |
| 3010 | Eksport | Export sales (non-EU, 0% VAT) |
| 3050 | EL müük | Intra-EU supplies (B2B, 0% VAT) |
| 3100 | Muu tulu | Other operating income |

### Cost of sales (5xxx)

| Code | Name | Use |
|---|---|---|
| 5210 | Ostetud kaubad | Cost of goods purchased |
| 5300 | Ostetud teenused | Cost of services purchased |
| 5510 | Sõidukite kulud | Vehicle expenses |
| 5520 | Bürookulud | Office expenses |
| 5530 | IT-kulud | IT/software expenses |

### Other expenses (6xxx)

| Code | Name | Use |
|---|---|---|
| 6080 | Esinduskulud | Representation costs |
| 6090 | Erisoodustused | Fringe benefits (in-kind) |
| 6210 | Amortisatsioon | Depreciation expense |
| 6500 | Finantskulud / FX | Finance costs incl. FX |

### Tax (7xxx — varies)

| Code | Name | Use |
|---|---|---|
| 7100 | Tulumaks | Corporate income tax (on distribution) |

## Looking up an account by purpose

When a skill says "post to account 3000", do not assume — look up:

```python
acc = by_code.get("3000")
if acc is None:
    raise RuntimeError("Account 3000 not in chart for this company")
```

If the company has renamed `3000` to e.g. `30100`, prompt the user to map it, or use the account whose `ReportdescName` is "Müügitulu" / "Sales revenue" as a fallback.

## Adding a new account

There is no documented `sendaccount` endpoint at v1; at v2 it exists but is rarely used because chart changes typically happen in the UI for auditability. If you need to add an account programmatically, contact Merit support to confirm the current API surface.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/accounts-list/
