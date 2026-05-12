# VAT (käibemaks) 2026 — full reference

Source: [EMTA — Value added tax](https://www.emta.ee/en/business-client/taxes-and-payment/value-added-tax). Cross-verified against PwC Tax Summaries, EY Estonia, Grant Thornton EE.

## Rates

| Rate | Scope | Effective from |
|---|---|---|
| **24% standard** | Default for almost all domestic goods/services | 1 Jul 2025 (permanent) |
| **13% reduced** | Accommodation services (with/without breakfast) | 1 Jan 2025 |
| **9% reduced** | Books and workbooks (physical and electronic); press publications; listed medicines; medical devices for disabled persons | Books: 1 Aug 2022; press: 1 Jan 2025 |
| **5%** | Effectively phased out | Appears only on transitional credit invoices |
| **0%** | Exports of goods, intra-Community supply of goods (B2B), international transport, B2B services to taxable persons in another EU MS (Art. 44 / VAT Act §10), certain vessel/aircraft transactions | |
| **Exempt** | Financial services, insurance, healthcare, education, certain real-estate transactions (with option to tax) | |

## Recent timeline

- 20% → 22%: 1 Jan 2024
- 22% → 24%: 1 Jul 2025 (originally a 3-year defence-funding measure; made permanent in 2025)
- Accommodation 9% → 13%: 1 Jan 2025
- Press 5% → 9%: 1 Jan 2025

## VAT registration

- **Threshold**: €40,000 of Estonia-place-of-supply turnover in a running 12-month period.
- Application must be filed within **3 business days** of crossing the threshold.
- **Limited VAT liability** for intra-EU acquisitions: triggers at €10,000/year of acquisitions.
- Voluntary registration is allowed at any turnover level.

## Reverse charge (the recipient self-accounts for VAT)

1. **Intra-EU acquisitions of goods** (B2B).
2. **Services from non-resident taxable persons** under the general B2B rule (Art. 44 / VAT Act §10).
3. **Domestic reverse charge** for:
   - Scrap metal.
   - Gold of investment quality.
   - Immovables where the option to tax is exercised.
   - Emission allowances.

For each, the recipient books both input VAT (deductible) and output VAT (payable) on the same transaction. In Merit Aktiva, this is handled by a dedicated `pöördkm` (reverse VAT) tax code; the system auto-generates the offsetting entries.

## Deadlines

- **KMD** (käibedeklaratsioon, VAT return) — **20th of the month following the tax period** (calendar month).
- **KMD INF** (annex listing all invoices ≥ €1,000 net per partner per period) — same deadline as KMD; submitted together.
- **VD** (recapitulative statement for intra-EU supplies) — 20th.

## E-invoicing

- **Mandatory for B2G** (sales to public sector) — Peppol / EVS 923610:2014 format.
- **B2B remains optional**, except: if the buyer registers in the e-arve äriregister, the seller must issue an e-invoice on request (effective 1 Jul 2025 amendment to the Accounting Act).
- Broader B2B mandate is under consultation (aligned with EU ViDA, 2030–2035 horizon). Not in force as of 2026-05-12.

## KMD line legend (form KMD 2025)

| Line | Meaning |
|---|---|
| 1 | Taxable supply at 24% (base) |
| 2 | Taxable supply at 13% (base) |
| 2.1 | Taxable supply at 9% (base) |
| 3 | Zero-rated supply (incl. exports, intra-EU) |
| 3.1 | Intra-EU supplies of goods/services to taxable persons |
| 3.1.1 | Intra-EU supply of goods only |
| 3.2 | Export of goods |
| 4 | VAT due at 24% |
| 4.1 | VAT due at 13% |
| 4.2 | VAT due at 9% |
| 5 | Total deductible input VAT |
| 6 | Intra-EU acquisition of goods (base) |
| 6.1 | Taxable acquisition of goods (post-reverse-charge calculation) |
| 7 | Acquisition of other goods/services subject to reverse charge (services from EU/non-EU) |
| 8 | Exempt supplies |
| 9 | Import VAT shifted to KMD (postponed accounting) |
| 10 / 11 | Self-supply / fictitious supply |

## Fuel for passenger cars — the 50% rule

If a passenger car is used for both business and private purposes, only **50% of the input VAT** on related expenses (fuel, repairs, leasing) is deductible. Book the non-deductible half as expense.

Practical pattern for a fuel purchase of €120 incl. 24% VAT:

- Net fuel: €96.77; VAT: €23.23.
- Deductible VAT: €11.62 (half of 23.23).
- Non-deductible VAT becomes part of expense: €11.61.
- Dr 5510 Vehicle costs €108.38; Dr 1230 Input VAT €11.62; Cr 21xx AP €120.

## Representation costs

Input VAT on representation is fully deductible. The CIT angle: representation is **tax-free up to €50/month + 2% of payroll**. Any monthly amount above this is taxable as a distribution at 22/78 — declared on TSD annex 5.

## Cash-basis VAT (kassapõhine käibemaksuarvestus)

Available to VAT-registered persons with **turnover ≤ €200,000/year**. Output VAT due when invoice paid; input VAT recoverable when paid. Election by written notice to EMTA. See [small-business-schemes.md](small-business-schemes.md).

## Source

- https://www.emta.ee/en/business-client/taxes-and-payment/value-added-tax
- https://www.emta.ee/en/business-client/taxes-and-payment/value-added-tax/calculation-and-refund-vat/passenger-cars-and-vat-accounting
- https://taxsummaries.pwc.com/estonia/corporate/other-taxes
