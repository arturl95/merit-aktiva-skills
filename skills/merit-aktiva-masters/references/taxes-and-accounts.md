# Taxes and accounts — the per-session caches

Every transactional Merit endpoint requires GUIDs and codes that are unique per company. Resolve them once per session via the two endpoints below, cache the results, and reuse for every subsequent call.

## `POST /api/v1/gettaxes`

Body: `{}` (or empty body — most servers accept both, but `{}` is canonical).

### Response shape

```json
[
  { "Id": "11111111-...", "Code": "KM24",  "Name": "Käibemaks 24%",        "NameEN": "VAT 24%",  "NameRU": "НДС 24%", "TaxPct": 24.00 },
  { "Id": "22222222-...", "Code": "KM13",  "Name": "Käibemaks 13%",        "NameEN": "VAT 13%",  "NameRU": "НДС 13%", "TaxPct": 13.00 },
  { "Id": "33333333-...", "Code": "KM9",   "Name": "Käibemaks 9%",         "NameEN": "VAT 9%",   "NameRU": "НДС 9%",  "TaxPct": 9.00  },
  { "Id": "44444444-...", "Code": "KM0",   "Name": "Käibemaks 0%",         "NameEN": "VAT 0%",   "NameRU": "НДС 0%",  "TaxPct": 0.00  },
  { "Id": "55555555-...", "Code": "EL24",  "Name": "Pöördkäibemaks EL 24%", "NameEN": "Reverse VAT EU 24%", "TaxPct": 24.00 },
  { "Id": "66666666-...", "Code": "EXP",   "Name": "Eksport 0%",            "NameEN": "Export 0%", "TaxPct": 0.00 },
  { "Id": "77777777-...", "Code": "KMv",   "Name": "Käibemaksuvaba",        "NameEN": "VAT-exempt", "TaxPct": 0.00 }
]
```

The `Code` strings above are conventions — actual codes vary slightly per company because tenants can edit them. **Always look up by `Code` AND `TaxPct` together** to disambiguate. Never hard-code `Id` values across companies.

### Cache structure

Build two indexes when you cache:

```
by_code = { tax.Code: tax for tax in taxes }            # exact-code lookup
by_pct  = { tax.TaxPct: [tax, ...] for tax in taxes }   # may have multiple per rate (e.g. domestic 24% vs reverse-charge 24%)
```

For lookups by rate, when more than one matches the same `TaxPct`, distinguish by `Code` substring (e.g. `EL` prefix → reverse charge; `KM` prefix → domestic).

### When to refresh

Once per session is enough. Invalidate the cache only when:

- The user changed a VAT code in the Merit UI mid-session.
- A new VAT rate took effect (e.g. annual EU rate changes).
- You receive a 400 on a `sendinvoice` whose `TaxId` resolves correctly in your cache.

## `POST /api/v1/getaccounts`

Body: `{ "UsageFilter": 0 }` for the full chart of accounts. Filter values:

| `UsageFilter` | Returns |
|---|---|
| 0 (default) | All accounts |
| 1 | Cost accounts (expenses) |
| 2 | Cost contra-accounts |
| 3 | Purchase VAT accounts |

### Response shape

```json
[
  {
    "AccountID":     "guid",
    "NoActive":      false,
    "Code":          "1010",
    "Name":          "Kassa",
    "NameEN":        "Cash",
    "NameRU":        "Касса",
    "TaxName":       null,
    "TaxNameEN":     null,
    "LinkedVendorName": null,
    "IsParent":      false,
    "ReportdescName":   "Käibevara",
    "ReportdescNameEN": "Current assets",
    "CashflowName":     "...",
    "CashflowNameEN":   "..."
  }
]
```

### Estonian RTJ defaults (verify per tenant)

| Code | Name | Use |
|---|---|---|
| 1010 | Kassa | Cash on hand |
| 1020 / 1110 | Pangakontod | Bank accounts |
| 12xx (1210, 1230) | Nõuded ostjate vastu | Accounts receivable |
| 1230 | Sisendkäibemaks | Input VAT |
| 1370 | Material in stock | Stock (varied codes) |
| 2110 | Võlad tarnijatele | Accounts payable |
| 2120 | Käibemaksu kohustus | Output VAT |
| 2380 | Töötasude võlg | Wages payable |
| 2390 | Maksuvõlad | Payroll/other taxes payable |
| 2410 | Käibemaks | (Older charts) VAT control account |
| 3000 / 3010 / 3050 | Müügitulu / Eksport / EL müük | Sales / Export / EU sales |
| 5210 | Ostetud kaubad | Cost of goods purchased |
| 5510 | Sõidukite kulud | Vehicle costs |
| 6080 | Esinduskulud | Representation costs |
| 6090 | Erisoodustused | Fringe benefits |

These are the codes that **Merit Aktiva ships as a default for small Estonian companies**, but every tenant can rename or renumber. Always read from `getaccounts` rather than assume.

### Cache structure

```
accounts_by_code = { a.Code: a for a in accounts }
```

When code-mapping in skills (e.g. when `estonian-bookkeeping` says "credit account 3000"), look up `accounts_by_code["3000"]` and fail loudly if missing rather than guessing.

## Combined initialization snippet (pseudo-code)

```python
def init_session():
    taxes = sign_and_post("/api/v1/gettaxes", {})
    accounts = sign_and_post("/api/v1/getaccounts", {"UsageFilter": 0})
    return {
        "tax_by_code": {t["Code"]: t for t in taxes},
        "tax_by_pct": defaultdict(list, ...),
        "account_by_code": {a["Code"]: a for a in accounts}
    }
```

Persist the result for the duration of the session. Most production integrations also persist across sessions with a 24-hour TTL.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/tax-list/
- https://api.merit.ee/connecting-robots/reference-manual/accounts-list/
