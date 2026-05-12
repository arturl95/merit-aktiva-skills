---
name: estonian-bookkeeping
description: Apply current (2026) Estonian tax rules — VAT (käibemaks) 24/13/9, distributed-profit CIT 22/78, payroll taxes, KMD codes, fringe benefits, security tax status. Use for any Estonian bookkeeping decision (with or without Merit Aktiva).
last-verified: 2026-05-12
---

# estonian-bookkeeping

The domain-knowledge layer for any decision that involves Estonian tax. Independent of Merit Aktiva — usable whenever you need the right rate, the right account, or the right VAT code.

## When to use

- Choosing the VAT code for an invoice or purchase.
- Computing CIT on a dividend distribution or fringe benefit.
- Mapping a transaction to KMD lines.
- Answering "what's the rate of X in Estonia in 2026".
- Reviewing a month-end payroll calculation.

## 2026 rate cheatsheet

All rates verified 2026-05-12 against EMTA. See [vat-rates-2026.md](references/vat-rates-2026.md), [corporate-tax.md](references/corporate-tax.md), and [payroll-taxes.md](references/payroll-taxes.md) for full context and citations.

| Tax | Rate | Notes |
|---|---|---|
| **VAT standard** | **24%** | Permanent since 1 Jul 2025 |
| VAT reduced (accommodation) | 13% | Since 1 Jan 2025 |
| VAT reduced (books, press, certain medicines) | 9% | Books since 2022, press since 1 Jan 2025 |
| VAT zero | 0% | Exports, intra-EU supply (B2B), international transport |
| VAT exempt | — | Financial, insurance, healthcare, education, some real estate |
| VAT registration threshold | €40,000/year | Running 12-month rolling |
| **Corporate income tax (on distributions)** | **22% = 22/78 of net** | 24% plan cancelled Dec 2025; reduced 14/86 abolished 1 Jan 2025 |
| **Personal income tax** | **22%** | 24% plan cancelled Dec 2025 |
| Basic exemption (general) | **€700/mo** flat | Tax-hump abolished 1 Jan 2026 |
| Basic exemption (pensioner) | €776/mo | Automatic |
| **Social tax** | **33%** | On gross remuneration, employer pays |
| Social tax minimum monthly base | €886 → €292.38 min | Triggers on any payment to a person |
| Unemployment insurance — employee | 1.6% | Withheld |
| Unemployment insurance — employer | 0.8% | On top of gross |
| II pillar — default | 2% | Employee can elect 4% or 6% (Jan–Nov applications effective next year) |
| Minimum wage (1 Apr 2026) | €946/mo, €5.67/hr | |
| Fringe benefit — car ≤5 years | €1.96 × kW / month | |
| Fringe benefit — car >5 years | €1.47 × kW / month | |
| Representation tax-free cap | €50/mo + 2% of payroll | Since 1 Jan 2025 |
| Excise duties | +10% across the board | 1 Jan 2026 |

## Security tax (julgeolekumaks)

**Not in force.** Abolished by the Riigikogu on 19 Jun 2025 before it took effect. There is **no** 2% PIT add-on, **no** 2% CIT on accounting profit, **no** 1.6% wage levy. Defence is funded from the permanent VAT/excise hikes instead.

## Transaction → account → KMD code cheatsheet

Default RTJ chart-of-accounts codes shown below — verify per tenant via `merit-aktiva-masters` (`getaccounts`).

| # | Bookkeeping case | Debit / Credit (default) | Merit VAT code (typical) | KMD lines |
|---|---|---|---|---|
| 1 | Domestic sale 24% | Dr 12xx Receivables / Cr 3000, Cr 2120 | Käive 24% | 1 + 4 |
| 2 | Domestic purchase 24% (fully deductible) | Dr 5xxx, Dr 1230 / Cr 21xx | Sisendkm 24% | 5 |
| 3 | Intra-EU acquisition of goods (reverse charge) | Dr 5210, Dr 1230 / Cr 21xx; self-VAT: Dr 1230 / Cr 2120 | EL kaup pöördkm 24% | 1+4 / 5 / 6+6.1 |
| 4 | Intra-EU services (reverse charge, Art. 44) | same as #3 | EL teenus pöördkm 24% | 1+4 / 5 / 7 |
| 5 | Services from outside EU (reverse charge) | same as #4 | 3. riigi teenus pöördkm | 1+4 / 5 / 7 |
| 6 | Export of goods (non-EU, 0%) | Dr 12xx / Cr 3010 | Eksport 0% | 3 + 3.2 |
| 7 | Intra-EU supply of goods (B2B, 0%) | Dr 12xx / Cr 3050 | EL kaup 0% | 3 + 3.1 + 3.1.1; + VD |
| 8 | Export of services (B2B EU, 0%) | Dr 12xx / Cr 3050 | EL teenus 0% | 3 + 3.1; + VD |
| 9 | Fuel for passenger car (mixed use, 50% deductible) | Dr 5510 (gross 50% + net 50%), Dr 1230 (50% of VAT) / Cr 21xx | Sisendkm 24% sõiduauto 50% | 5 reduced |
| 10 | Representation cost | Dr 6080, Dr 1230 / Cr 21xx; CIT on overage via TSD annex 5 | Sisendkm 24% | 5; TSD annex 5 |
| 11 | Fringe benefit — company car | Dr 6090 (kW × €1.96 or €1.47) / Cr 2390 | n/a (taxes only) | TSD annex 4 |
| 12 | VAT-exempt sale | Dr 12xx / Cr 3xxx | Maksuvaba käive | 8 |

KMD line legend:
- 1 — taxable supply 24% (base); 4 — VAT due at 24%.
- 2 — taxable supply 13%; 4.1 — VAT due at 13%.
- 2.1 — taxable supply 9%; 4.2 — VAT due at 9%.
- 3 — zero-rated supply (3.1 intra-EU, 3.1.1 intra-EU goods, 3.2 export of goods).
- 5 — deductible input VAT.
- 6 — intra-EU acquisition of goods (base); 6.1 taxable.
- 7 — other reverse-charge acquisitions.
- 8 — exempt supplies.
- 9 — import VAT (postponed accounting).

## Verify-before-filing rule

This skill's `last-verified` date is the last manual cross-check against emta.ee. Before submitting any KMD, TSD, INF, or annual report, **re-verify the rate values and forms against the live emta.ee page** linked in each reference file. Estonia legislates frequently; rates can change with as little as one-quarter notice.

## When to read each reference

- [vat-rates-2026.md](references/vat-rates-2026.md) — VAT scope by category, reverse-charge cases, registration thresholds, e-invoicing rules, KMD line legend in detail.
- [corporate-tax.md](references/corporate-tax.md) — 22/78 math with worked examples, fringe-benefit gross-up, representation cost rules, gifts/donations.
- [payroll-taxes.md](references/payroll-taxes.md) — PIT, social tax, UI, II pillar, basic exemption mechanics, employer-cost example, board-member-fee specifics.
- [deadlines.md](references/deadlines.md) — TSD (10th), KMD/INF (20th), annual report, VD, Intrastat, OSS/IOSS.
- [small-business-schemes.md](references/small-business-schemes.md) — cash-basis VAT, OÜ vs FIE, board member fees, entrepreneur account (ettevõtluskonto).

## Cross-references

- For the Merit Aktiva VAT-code GUIDs: `merit-aktiva-masters`.
- For posting transactions that consume these mappings: `merit-aktiva-sales`, `merit-aktiva-purchases-payments`, `merit-aktiva-ledger-reports`.
