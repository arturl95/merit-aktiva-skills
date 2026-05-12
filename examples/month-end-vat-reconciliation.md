# Example — Month-end VAT reconciliation

**Scenario**: It's 18 May 2026. The KMD for April is due by 20 May. Before filing on emta.ee, verify that the GL output-VAT and input-VAT balances reconcile to what the KMD declares.

**Skills used**: `merit-aktiva-core`, `merit-aktiva-masters`, `merit-aktiva-sales`, `merit-aktiva-purchases-payments`, `merit-aktiva-ledger-reports`, `estonian-bookkeeping`.

---

## Goal

Verify three numbers reconcile:

1. **Output VAT from sales** — sum of all 24%/13%/9% VAT collected on sales invoices in April.
2. **Input VAT from purchases** — sum of deductible input VAT from purchase invoices and reverse-charge bookings.
3. **GL account movements** — the period movement on accounts 2120 (output VAT) and 1230 (input VAT).

If they don't agree, the KMD will be wrong. Find the discrepancy before filing.

## Step 1 — Pull period sales

```
POST /api/v2/getinvoices
{ "PeriodStart": "20260401", "PeriodEnd": "20260430" }
```

Returns headers only; for full row-level VAT data, call `getinvoice` per record OR rely on the headers (each invoice header reports `VatTotal`, `TotalAmount`, etc.).

For a thorough reconciliation, pull full details per invoice and group by `TaxId`:

```python
totals_by_tax = defaultdict(lambda: {"base": 0, "vat": 0})
for inv in invoices:
    full = post("/api/v2/getinvoice", {"Id": inv["InvoiceId"]})
    for row in full["Lines"]:
        net = round(row["Quantity"] * row["Price"] * (1 - row.get("DiscountPct", 0)/100), 2)
        tax_id = row["TaxId"]
        tax_pct = tax_cache_by_id[tax_id]["TaxPct"]
        vat = round(net * tax_pct / 100, 2)
        totals_by_tax[tax_id]["base"] += net
        totals_by_tax[tax_id]["vat"] += vat
```

## Step 2 — Pull period purchases

```
POST /api/v2/getpurchaseinvoices
{ "PeriodStart": "20260401", "PeriodEnd": "20260430" }
```

Same shape; group by `TaxId`. Distinguish:

- **Regular input VAT** (domestic purchases at 24%/13%/9%) — appears on KMD line 5.
- **Reverse-charge** (intra-EU goods, intra-EU services, non-EU services) — appears as both output VAT (line 4) and input VAT (line 5), plus the acquisition base on lines 6/6.1/7.

## Step 3 — Map to KMD lines

Build the KMD line totals. Cross-reference `estonian-bookkeeping/references/vat-rates-2026.md` for the line legend.

```
KMD line 1  (taxable supply 24% base)         = sum(sales bases @ 24%)
KMD line 2  (taxable supply 13% base)         = sum(sales bases @ 13%)
KMD line 2.1 (taxable supply 9% base)         = sum(sales bases @ 9%)
KMD line 3  (zero-rated supply)                = sum(sales bases @ 0% incl. export, intra-EU)
KMD line 4  (VAT due 24%)                     = sum(sales VAT @ 24%) + sum(reverse-charge VAT @ 24%)
KMD line 4.1 (VAT due 13%)                    = sum(sales VAT @ 13%)
KMD line 4.2 (VAT due 9%)                     = sum(sales VAT @ 9%)
KMD line 5  (total deductible input VAT)      = sum(purchase input VAT, fully deductible) + sum(reverse-charge input VAT)
KMD line 6, 6.1 (intra-EU acq of goods)       = sum(intra-EU goods acquisitions)
KMD line 7  (other reverse-charge acq)        = sum(intra-EU services + non-EU services acquisitions)
KMD line 8  (exempt supplies)                 = sum(exempt sales)
```

Net VAT payable = (line 4 + 4.1 + 4.2) − line 5.

## Step 4 — Pull GL movements

```
POST /api/v2/getglbatchesfull
{ "PeriodStart": "20260401", "PeriodEnd": "20260430" }
```

Or, simpler: pull the financial position as of period start and period end, take the delta:

```
POST /api/v2/getfinpos { "AsOfDate": "20260331" }
POST /api/v2/getfinpos { "AsOfDate": "20260430" }
```

Find accounts 1230 (Sisendkäibemaks) and 2120 (Käibemaksu kohustus). The period movement is the difference between the two as-of values.

## Step 5 — Reconcile

| Number | Source | April figure |
|---|---|---|
| Output VAT (KMD lines 4 + 4.1 + 4.2) | Step 3 | €X |
| GL output VAT movement (account 2120) | Step 4 | €X |
| Input VAT (KMD line 5) | Step 3 | €Y |
| GL input VAT movement (account 1230) | Step 4 | €Y |
| Net VAT payable | KMD math | €(X − Y) |

The KMD-derived and GL-derived numbers should agree to ±€0.01 (cumulative rounding).

## Step 6 — When they don't agree

Common discrepancy causes:

1. **Reverse-charge entries that didn't net**. Symptom: GL output and input VAT both move, but by different amounts on a specific transaction. Find the transaction by listing GL entries for accounts 1230 and 2120, looking for unmatched debit/credit pairs.
2. **VAT on a credit invoice that hit the wrong period**. Symptom: the credit's `TransactionDate` is in a different VAT period than the original sale. Decide whether to file the correction in April or restate March.
3. **A `sendglbatch` that booked to 2120/1230 directly without a matching invoice**. Symptom: GL movement larger than KMD. Find via `getglbatchesfull` filtered to those accounts; investigate whether the entry was legitimate (rare) or a coding error.
4. **A sale or purchase invoice with the wrong `TaxId`**. Symptom: VAT total on KMD doesn't match what was charged on the invoice. Delete + recreate the invoice with the right `TaxId` (sales) or post a correcting purchase invoice.
5. **An invoice still in approval queue** (not yet posted). Symptom: KMD numbers exclude an invoice you remember posting. Approve it in the Merit UI, then re-run reconciliation.

## Step 7 — Show the reconciliation to the user

Format as a table. Highlight any discrepancy ≥ €0.02. If clean, recommend filing the KMD.

If discrepancies exist, propose fixes one at a time, each with a confirmation step (no auto-corrections to a tax period that has already been declared internally).

## Step 8 — File the KMD

KMD filing happens on emta.ee, not via Merit's API (EMTA provides their own e-MTA portal and X-Road integration; Merit does not currently expose a `submit_kmd` endpoint). Export the KMD figures from Merit (UI → Reports → VAT report) and enter into e-MTA.

(If/when EMTA opens a public submission API and Merit exposes it, this section will get an update. Current state: manual filing via the e-MTA web UI.)

## Step 9 — Save a copy

Once filed, save a copy of the final KMD figures with the reconciliation worksheet to the company's archive (out of scope for this example, but a useful practice).

## Common reconciliation patterns to encode

- "Show me April VAT" → run steps 1–5, present table.
- "Why is output VAT off by €5?" → fall into step 6 investigation.
- "What if I post this adjustment?" → simulate the GL change against the KMD totals before posting.

## Cross-references

- KMD line legend → `estonian-bookkeeping/references/vat-rates-2026.md` §KMD line legend.
- Reporting endpoints → `merit-aktiva-ledger-reports/references/reports.md`.
- Reverse-charge mechanics → `examples/post-eu-acquisition-with-reverse-charge.md`.
- GL entry/reversal patterns → `merit-aktiva-ledger-reports/references/gl-transactions.md`.
