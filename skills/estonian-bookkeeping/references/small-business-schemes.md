# Small-business schemes

## Cash-basis VAT (kassapõhine käibemaksuarvestus)

Available to VAT-registered persons with turnover **≤ €200,000/year**.

Mechanics:
- **Output VAT** due when the customer pays the invoice (not when issued).
- **Input VAT** recoverable when you pay the supplier (not when received).
- Election: file a written notice with EMTA. The election applies to all transactions; you cannot mix accrual and cash for VAT.
- Useful for cash-flow-tight micro-businesses that issue net-30 invoices to slow payers.

Caveat: the bookkeeping is more involved because each invoice's VAT line floats until payment lands. Merit Aktiva supports this regime; ensure the company's `Settings → VAT` is configured for cash basis before posting.

Source: [EMTA — VAT](https://www.emta.ee/en/business-client/taxes-and-payment/value-added-tax).

## OÜ vs FIE

### OÜ (osaühing — private limited company)

- **CIT**: distribution-based (22/78 on dividends). Retained earnings 0%.
- **Share capital**: minimum €0.01 since February 2023 (formerly €2,500).
- **Liability**: shielded — shareholders not personally liable beyond capital.
- **Reporting**: mandatory annual report (majandusaasta aruanne) within 6 months of year-end.
- **Use case**: default vehicle for businesses with >€1,000–€2,000 monthly turnover.

### FIE (füüsilisest isikust ettevõtja — sole proprietor)

- **Income tax**: 22% PIT on **net business income** (revenue minus deductible expenses).
- **Social tax**: 33% on net business income, with the €886 monthly minimum base when registered/active.
- **Reporting**: Form E annually by 30 April.
- **Liability**: unlimited personal liability for business debts.
- **Use case**: very low-volume, intermittent business activity. Most active small businesses incorporate as OÜ.

Tax math example for a FIE earning €1,000/mo net business income:

- PIT: 22% × €1,000 = €220
- Social tax: 33% × €1,000 = €330
- After-tax: €450 (45% effective)

Compare to an OÜ that retains the same €1,000 and never distributes: €0 tax.

## Board member fees

See [payroll-taxes.md](payroll-taxes.md) for the full mechanics. Headline:

- 22% PIT + 33% social tax.
- **No** unemployment insurance (either side).
- **No** II pillar.
- Triggers the €886 minimum social-tax base regardless of fee amount.

Practical: paying yourself a €100 board fee costs €292.38 in social tax. A €1,000 fee costs €330 in social tax (above the minimum threshold). Plan around the minimum carefully.

## Entrepreneur account (ettevõtluskonto)

Flat-tax scheme for micro-entrepreneurs via LHV Pank. Designed for B2C personal services and platform work.

| II pillar status | Tax rate on gross receipts |
|---|---|
| Not in II pillar | **20%** |
| 2% II pillar | **22%** |
| 4% II pillar | **24%** |
| 6% II pillar | **26%** |

- Tax is **auto-debited** from each incoming transfer to the LHV entrepreneur account.
- **No expense deduction** — the rate applies to gross receipts.
- Threshold: €40,000/year. Above this, you must register as VAT payer / FIE / OÜ.
- **B2B blocked**: you cannot sell to legal persons in the course of their business. Intended for B2C: tutoring, cleaning, photography, platform delivery, etc.
- The minimum €292.38/month social-tax-equivalent contribution is still needed to maintain health insurance (Estonian state healthcare is contribution-based).

Source: [EMTA — Entrepreneur account](https://www.emta.ee/en/private-client/taxes-and-payment/taxable-income/entrepreneur-account).

## Choosing between schemes

A rough decision tree for a small Estonian operator:

1. **Receipts < €10k/year, B2C only** → entrepreneur account (simplest).
2. **Receipts €10k–€40k/year, mixed B2B/B2C, low expenses** → FIE.
3. **Receipts > €40k/year, OR B2B-heavy, OR significant expenses, OR you want to retain earnings** → OÜ.

VAT registration becomes mandatory at €40,000 turnover regardless of legal form. Cash-basis VAT is available up to €200,000 turnover.

## Source

- https://www.emta.ee/en/private-client/taxes-and-payment/taxable-income/entrepreneur-account
- https://www.emta.ee/en/business-client/taxes-and-payment/value-added-tax
- https://www.emta.ee/en/business-client/taxes-and-payment/income-and-social-taxes/social-tax
