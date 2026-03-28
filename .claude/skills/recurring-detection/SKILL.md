---
name: recurring-detection
description: Detect recurring merchants (appearing in 3+ of the last 6 months) and surface them as a Monthly Commitments card on the dashboard. Shows the spending floor.
---

# Feature: Recurring / Monthly Commitments Detection

Auto-detect merchants that recur every month and surface them as a dedicated "Monthly Commitments" section on the dashboard. This makes the spending floor visible — the amount that goes out regardless of lifestyle choices.

## Detection logic

```js
function detectRecurring() {
  const last6 = getMonths().slice(0, 6); // up to 6 most recent months
  if (last6.length < 3) return []; // not enough data

  // Group non-income, non-savings transactions by normalised merchant
  const byMerchant = {};
  txns.forEach(t => {
    if (t.incomeType || t.savingsAccountId || t.amount <= 0) return;
    const key = t.merchant.toLowerCase().trim().substring(0, 25);
    if (!byMerchant[key]) byMerchant[key] = { merchant: t.merchant, category: t.category, months: new Set(), amounts: [] };
    const m = t.date.substring(0, 7);
    if (last6.includes(m)) {
      byMerchant[key].months.add(m);
      byMerchant[key].amounts.push(t.amount);
    }
  });

  // Keep merchants appearing in 3+ of the last 6 months
  return Object.values(byMerchant)
    .filter(r => r.months.size >= 3)
    .map(r => ({
      merchant: r.merchant,
      category: r.category,
      frequency: r.months.size, // out of 6
      avgAmount: r.amounts.reduce((s,a)=>s+a,0) / r.amounts.length,
    }))
    .sort((a, b) => b.avgAmount - a.avgAmount);
}
```

## Dashboard card

Add a "Monthly Commitments" card below the category grid and above the charts:

```
Monthly Commitments                              Est. $2,340/mo
─────────────────────────────────────────────────────────────
● Netflix         Bills        $18       6/6 months
● Goodlife        Gym          $55       5/6 months
● Telus           Bills        $89       6/6 months
● Shaw            Bills        $120      6/6 months
● Rent            Home         $2,100    6/6 months
...
```

- Show category colour dot
- Show merchant name, category, avg amount, and frequency badge (e.g. "6/6")
- Footer: total estimated monthly commitment
- Collapsed by default (show first 5, expand on click)
- Card only renders if at least 3 recurring merchants are found

## Period awareness

Recurring detection always uses all-time data (last 6 months globally), not the current dashboard filter. The card is informational context, not a filtered view.

## Both files

Apply to both `expense-tracker-clean.html` and `expense-tracker-template.html`.
