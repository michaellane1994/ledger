---
name: savings-rate
description: Add a savings rate % to the dashboard hero stats. (Income - Spend) / Income shown as a % with green/red colour coding.
---

# Feature: Savings Rate

Show a savings rate percentage prominently on the dashboard. This is the single clearest signal of financial health — positive means you're building wealth, negative means you're drawing it down.

## Formula

```js
const savingsRate = totalIncome > 0
  ? ((totalIncome - total) / totalIncome * 100)
  : null;
```

Where `total` = total spend (period-filtered, category-unfiltered, same as hero total), `totalIncome` = sum of non-transfer income entries for the period.

## Display

Add to the existing hero stats row (alongside txn count, daily avg, largest purchase):

```
Savings Rate: 24%
```

- Green if ≥ 10%
- Amber if 0–9%
- Red if negative
- Show `—` if no income data for the period (avoid divide-by-zero confusion)

Use the same `.hero-stat` element pattern already in the markup.

## Period awareness

Uses the same `monthIncomes` / `totalIncome` already calculated in `renderDashboard()` — no new data pass needed. Just derive the % from existing values.

## Both files

Apply to both `expense-tracker-clean.html` and `expense-tracker-template.html`.
