---
name: category-trends
description: Add trend indicators to dashboard category tiles showing % change vs the previous comparable period. Arrow + % in red/green.
---

# Feature: Category Trend Indicators

Show a small trend badge on each category tile: an arrow and % change vs the previous comparable period. Gives instant context on whether a category is creeping up or stable.

## Trend badge placement

Inside each `.cat-tile`, below the amount, above the bar:

```
Groceries
$840
↑ 12%    ← trend badge
████████░░  (bar)
14.2%
```

Styling: small text (10px), red for increase (spending more), green for decrease (spending less), grey if no prior data.

## Previous period logic

| Current period | Compare against |
|---|---|
| Single month (selMonth) | Previous calendar month |
| Latest Month | Month before the latest month with data |
| Last 3 Months | The 3 months before that window |
| YTD | Same Jan-to-current-month span, prior year |

## Calculation

```js
// For a category key c.key:
const curr = byCat[c.key] || 0;
const prev = byCatPrev[c.key] || 0;
// Only show if prev > 0 (no prior data = no badge)
const pct = prev > 0 ? ((curr - prev) / prev * 100) : null;
```

Build `byCatPrev` the same way as `byCat`, but using the previous window's transactions. Reuse the same transfer/incomeCategory logic.

## Display rules

- If `pct === null`: no badge (no prior data)
- If `|pct| < 2%`: show `—` (no meaningful change)
- If pct > 0: `↑ X%` in red (spending rose)
- If pct < 0: `↓ X%` in green (spending fell)

## Both files

Apply to both `expense-tracker-clean.html` and `expense-tracker-template.html`.
