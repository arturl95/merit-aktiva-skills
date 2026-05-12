# Corporate income tax 2026

Source: [EMTA — Taxation of dividends](https://www.emta.ee/en/business-client/taxes-and-payment/income-and-social-taxes/taxation-dividends), [PwC — Estonia CIT](https://taxsummaries.pwc.com/estonia/corporate/taxes-on-corporate-income).

## The distribution-based system

Estonia taxes corporate income only when **profit is distributed** (or deemed distributed). Retained earnings remain at 0% indefinitely. Triggers for CIT:

1. Dividend distribution (cash or in kind).
2. Fringe benefits to employees / board members.
3. Gifts and donations (except to listed non-profits within limits).
4. Non-business expenses.
5. Excessive interest, royalties, or related-party adjustments.
6. Hidden profit distributions.

## Rate on distributions

**22% on the gross distribution = 22/78 of the net distributed amount.**

Math: net dividend €78 → CIT €22 → gross profit consumed €100.

```
gross_profit  = net_distribution / 0.78
cit_amount    = gross_profit - net_distribution  = net_distribution * 22 / 78
```

Worked examples:

| Net distribution | Gross profit consumed | CIT owed |
|---|---|---|
| €78 | €100 | €22 |
| €1,000 | €1,282.05 | €282.05 |
| €10,000 | €12,820.51 | €2,820.51 |

The 24% rate planned for 2026 was **cancelled in December 2025**. The rate stays at 22% for 2026.

## Reduced 14/86 regime — abolished

The reduced rate for regular dividends (14/86 + 7% PIT withholding to natural persons) was abolished from 1 Jan 2025. All dividends now taxed uniformly at 22/78.

Transitional rule: pre-2025 retained earnings that were eligible for 14/86 still carry their original treatment when distributed, but new earnings do not qualify.

## Fringe benefits (erisoodustus)

Paid by the employer on a **grossed-up** basis. For a benefit of value `B`:

```
cit_on_benefit          = B * 22/78
gross_benefit           = B + cit_on_benefit
social_tax_on_benefit   = gross_benefit * 0.33
total_employer_cost     = gross_benefit + social_tax_on_benefit
                        = B + B*22/78 + (B + B*22/78) * 0.33
```

Worked example for a €100 fringe benefit:

- CIT: 100 × 22/78 = €28.21
- Gross: 100 + 28.21 = €128.21
- Social tax: 128.21 × 0.33 = €42.31
- Total employer cost: €170.52

Declared on **TSD annex 4**, due by the 10th of the following month.

### Company car benefit

| Vehicle age | Benefit per kW per month |
|---|---|
| ≤5 years | €1.96 |
| >5 years | €1.47 |

The benefit base is `kW × rate`. Apply the gross-up above on top.

A 150 kW, 3-year-old car: monthly benefit = 150 × 1.96 = €294. Total employer cost ≈ €501/month.

### Sports / health benefit

€400/year per employee is tax-free (since several years; unchanged in 2026). Above this, taxable as fringe benefit.

## Representation costs

Tax-free cap (since 1 Jan 2025): **€50/month + 2% of payroll**.

Practical:

- Calculate monthly payroll for the period (gross wages × employees).
- Tax-free representation ceiling = €50 + 2% × payroll.
- Any representation spending above the ceiling is a distribution; pay 22/78 CIT on the excess via **TSD annex 5**.

## Gifts and donations

- Gifts to **listed non-profits** (the TMIN list maintained by EMTA): exempt up to **3% of payroll** OR **10% of prior-year profit** — taxpayer chooses the more favourable ceiling.
- Gifts to other recipients: taxable as 22/78 distribution.

## Advance payments

None for the standard CIT regime. CIT on distributions is paid by the 10th of the month following distribution on **TSD**.

Credit-institution surtax: 18% on quarterly profit — separate regime, not applicable to normal OÜs.

## Source

- https://www.emta.ee/en/business-client/taxes-and-payment/income-and-social-taxes/taxation-dividends
- https://www.emta.ee/en/business-client/taxes-and-payment/income-and-social-taxes/fringe-benefits
- https://taxsummaries.pwc.com/estonia/corporate/taxes-on-corporate-income
- https://www.ey.com/en_ee/insights/tax/significant-tax-changes-in-estonia-in-2025-2026
