---
name: budget-targets
description: Add monthly budget targets per expense category. Each category tile shows a progress bar vs budget. Tiles turn amber at 75%, red at 100%.
argument-hint: "[optional: personal-only | template-only]"
---

# Feature: Budget Targets per Category

Add a monthly budget target to each expense category. The budget is optional — tiles with no budget set behave exactly as today.

## Data

Add a `budget` field (number, monthly dollar target) to each entry in `CATS`. Persisted via `saveCats()` alongside the existing `key`, `label`, `color`, `kw` fields.

```js
// Example CATS entry with budget
{ key:'groceries', label:'Groceries', color:'#3d8b5e', kw:[...], budget: 600 }
```

`normCat()` should default `budget` to `0` if absent (keeps backward compatibility).

## Settings UI

In Settings → Categories (where users already edit category name/colour/keywords), add a budget input field per row:

```
[colour swatch] [name input] [$budget input] [keywords input] [delete]
```

- Number input, placeholder `No budget`, min 0
- On change: update `CATS[i].budget`, call `saveCats()`

## Dashboard Tile

When `budget > 0`, replace the existing `ct-bar` (% of total) with a budget progress bar:

- Bar fills from 0 to `budget`, colour = category colour
- At 75%: bar turns amber (`#f59e0b`)
- At 100%+: bar turns red (`#dc2626`), amount label also turns red
- Below the bar, show: `$amt / $budget` in muted small text
- If `budget === 0`, tile renders exactly as today (% of total spend bar)

## Period handling

Budget is always a monthly figure. When the dashboard is in a multi-month period (3m, YTD), scale the budget accordingly:

- Latest Month: budget × 1
- Last 3 Months: budget × 3
- YTD: budget × months elapsed so far this year (min 1)

## Both files

Apply to both `expense-tracker-clean.html` and `expense-tracker-template.html`.
