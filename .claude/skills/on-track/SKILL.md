---
name: on-track
description: Show a spending pace indicator on the dashboard for single-month views. Compares current month's daily spend rate to the 3-month historical average.
---

# Feature: On-Track Spending Pace Indicator

When viewing a single month (selected month or Latest Month), show a spending pace badge near the hero total. It tells the user whether they're on track relative to their typical spending pace — the earlier in the month they see it, the more useful it is.

## When to show

Only show for single-month views: `dashPeriod === ''` (selMonth active) or `dashPeriod === 'latest'`.
Hide for 3m, YTD, and all-time views.

## Calculation

```js
// Current month pace
const today = new Date();
const daysElapsed = (viewMonth === currentCalendarMonth)
  ? today.getDate()
  : daysInMonth(viewMonth); // full month if it's already past
const dailyRate = total / daysElapsed;
const projectedMonthly = dailyRate * daysInMonth(viewMonth);

// Historical baseline: average monthly spend over last 3 full months
// (exclude the current/viewed month)
const last3Months = getMonths().filter(m => m < viewMonth).slice(0, 3);
const baseline = last3Months.length
  ? last3Months.reduce((s, m) => s + spendForMonth(m), 0) / last3Months.length
  : null;

const pct = baseline ? ((projectedMonthly - baseline) / baseline * 100) : null;
```

`spendForMonth(m)` = sum of `effectiveAmt` for all expense transactions in month `m`.

## Display

Show inline in the hero section, below the period label:

```
On track  (within ±5%)
↑ 18% above pace   (red)
↓ 12% below pace   (green)
```

- Within ±5%: "On track" in muted
- Above: red with arrow and %
- Below: green with arrow and %
- No baseline data: hide the indicator entirely

If the month is not yet over, prefix with: "Projected: $X,XXX / $baseline avg"

## Both files

Apply to both `expense-tracker-clean.html` and `expense-tracker-template.html`.
