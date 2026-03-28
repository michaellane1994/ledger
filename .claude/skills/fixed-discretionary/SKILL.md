---
name: fixed-discretionary
description: Tag expense categories as Fixed or Variable (discretionary). Dashboard hero shows the split so users can see how much of their spending is committed vs flexible.
---

# Feature: Fixed vs Discretionary Split

Tag each category as Fixed (committed spend — rent, insurance, bills, subscriptions) or Variable (discretionary — dining, shopping, activities). The dashboard then surfaces the split prominently so users immediately understand how much room they have.

## Data

Add a `fixed: boolean` field to each `CATS` entry. Default `false`. Persisted via `saveCats()`.

```js
{ key:'bills', label:'Bills', color:'#3050a8', fixed: true, kw:[...] }
{ key:'eatingout', label:'Eating Out', color:'#d4603a', fixed: false, kw:[...] }
```

`normCat()` should default `fixed` to `false` if absent.

## Default tagging (personal version)

| Category | Fixed |
|---|---|
| Bills | true |
| Home (rent/mortgage) | true |
| Gym | true |
| Puppies | false |
| Groceries | false |
| Eating Out | false |
| Liquor | false |
| Shopping | false |
| Transport | false |
| Travel | false |
| Activities | false |
| Health | false |
| Hubby / Wifey | false |

## Settings UI

In Settings → Categories, add a toggle/checkbox column "Fixed" next to each category row. On change: update `CATS[i].fixed`, call `saveCats()`.

## Dashboard display

In the hero stats row (below the big total), add two new stats:

```
Fixed: $2,400    Discretionary: $1,100
```

Calculation:
```js
const fixedTotal = expenses
  .filter(t => CAT[t.category]?.fixed)
  .reduce((s, t) => s + effectiveAmt(t), 0);
const varTotal = total - fixedTotal;
```

- `hero-fixed` and `hero-disc` stat elements
- Fixed shown in muted ink, discretionary shown in accent

## Tile badge

Each category tile gets a small "Fixed" badge (top-right corner, muted, uppercase 9px) when `fixed === true`. Variable categories show nothing.

## Both files

Apply to both `expense-tracker-clean.html` and `expense-tracker-template.html`. For template, default all categories to `fixed: false` — users set their own.
