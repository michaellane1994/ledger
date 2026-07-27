# Ledger — Agent Instructions

## Project Overview

Ledger is a household expense tracker built as single self-contained HTML files. Full context is in [LEDGER_BRIEF.md](LEDGER_BRIEF.md) — read it before making any changes.

## Two Versions

- `index.html` (this repo, single source of truth) — Michael & Lili's personal version (`l3_` localStorage prefix). Edit this file directly; commit and push to update GitHub Pages.
- `General Template/index-selfhost.html` (iCloud symlink) — Generic shareable template (`lt_` localStorage prefix); pushes to the `expensetracker` repo.

Unless explicitly told otherwise, apply changes to **both files**.

> **Note:** The personal site's iCloud master at `~/Library/Mobile Documents/com~apple~CloudDocs/Personal Site/` was retired on 2026-05-05 and archived to `~/ledger-archive/personal-site-2026-05-05/`. The repo's `index.html` is now the only working copy.

## Code Style

- No em-dashes (`--`) anywhere
- All CSS in `<style>` block at top
- All JS in single `<script>` block at bottom
- JS sections separated by comments like `// ── SECTION NAME ──`
- Always run `node --check` on extracted JS before shipping

## Testing

Open the HTML file directly in a browser — no build step needed. Google Sheets sync requires deployment to GitHub Pages (CORS blocks it locally).

Deploy: `git add . && git commit -m "description" && git push`

## Key Rules (from LEDGER_BRIEF.md)

- Never increment savings balances directly — always derive from contributions via `recalcAccountBalance()`
- Pull always fully replaces local data — never merge
- Always include `fromTxnId` and `category` in Income sheet writes
- Always call `rebuildCAT()` after modifying `CATS`
- **If you modify the Apps Script code block** (inside the `<pre id="apps-script-code">` element), always tell the user explicitly: "The Apps Script has changed — you need to copy the updated code from the Setup Guide into your Google Sheets Apps Script editor and redeploy."
- **Keep brand assets in sync with material branding changes** (wordmark, product name, palette). After any branding change, refresh all three and verify visually before committing:
  - **Social/link-preview thumbnail** — `ledger-saas/public/og-image.png` (1200×630, referenced by `og:image`/`twitter:image`). Source: `ledger-saas/public/og-image-source.html`; re-render with headless Chrome (`--headless=new --window-size=1200,630 --screenshot=og-image.png`, let webfonts load first).
  - **Email emblem logo** — `ledger-saas/public/email-logo.png` (used in transactional email headers). Source: `ledger-saas/public/email-logo-source.html`; render with transparent bg (`--default-background-color=00000000 --window-size=240,240`).
  - **Transactional email templates** — the shared `emailShell()` + templates in `ledger-saas/src/worker.js` (welcome, subscription, trial-ending, payment-failed). Keep palette/fonts/logo on-brand. (Worker changes require a deploy to take effect.)

## General Template: Sample Data & Reset

The template (`General Template/index.html`) ships with onboarding affordances the personal site does not have. Keep both working when schemas change.

- **`loadSampleData()`** — seeds a demo dataset (~60 txns across 3 months, salary/freelance incomes, 2 savings accounts with contributions, 1 sample mortgage). Wipes existing local data first (with confirm). Used by the welcome-banner "Load sample data" button and Settings → "✨ Load sample data". Update this function whenever you add a new data type or field that should appear in the demo.
- **`clearLocalData()`** — full local wipe (txns, uploads, incomes, savings, contribs, learned, trash, mortData, mortBalances, mortVoluntaryPmts). Does NOT touch Google Sheets. Keep the wipe list in sync with new state arrays as they're added.
- **`_wipeAllLocalState()`** — shared helper used by both of the above; always reset new state variables here when introducing them.
- Any new data type added to the template must be reflected in `loadSampleData` (seed some examples), `_wipeAllLocalState` (reset), and the Sheets push/pull payloads.

These affordances are template-only by design (the personal site has real data and should never ship a "Load sample data" button). Do not port them to `Personal Site/expense-tracker-clean.html`.

## Claude Code Skills

Skills extend Claude's capabilities via `SKILL.md` files. Invoke with `/skill-name` or let Claude auto-load when relevant.

**Docs:** https://code.claude.com/docs/en/skills

### Creating a skill

```
.claude/skills/<skill-name>/SKILL.md   # project-level (this repo)
~/.claude/skills/<skill-name>/SKILL.md # personal (all projects)
```

### SKILL.md format

```yaml
---
name: skill-name
description: What it does and when to use it (Claude uses this to auto-load)
disable-model-invocation: true  # only you can invoke (e.g. deploy, commit)
user-invocable: false           # only Claude can invoke (background knowledge)
context: fork                   # run in isolated subagent
agent: Explore                  # subagent type (Explore, Plan, general-purpose)
allowed-tools: Read, Grep       # tools allowed without per-use approval
argument-hint: "[filename]"     # shown in autocomplete
---

Skill instructions here. Use $ARGUMENTS for user-passed args, $0/$1 for positional.
Use !`shell command` to inject dynamic context (runs before Claude sees it).
```

### Key frontmatter fields

| Field | Effect |
|-------|--------|
| `disable-model-invocation: true` | Only you can invoke (use for side-effect workflows like deploy/commit) |
| `user-invocable: false` | Only Claude auto-loads (use for background reference knowledge) |
| `context: fork` | Runs in isolated subagent, no conversation history |
| `allowed-tools` | Tools pre-approved when skill is active |

### Project skills in this repo

- `.claude/skills/deploy/SKILL.md` — deploy to GitHub Pages
- `.claude/skills/budget-targets/SKILL.md` — monthly budget targets per category with progress bars on tiles
- `.claude/skills/category-trends/SKILL.md` — trend arrows on tiles showing % change vs prior period
- `.claude/skills/fixed-discretionary/SKILL.md` — tag categories as fixed/variable; hero split display
- `.claude/skills/savings-rate/SKILL.md` — savings rate % in dashboard hero stats
- `.claude/skills/on-track/SKILL.md` — spending pace indicator vs 3-month historical average
- `.claude/skills/recurring-detection/SKILL.md` — Monthly Commitments card: auto-detected recurring merchants
- `.claude/skills/google-sheets-backend/SKILL.md` — How to set up a static site with Google Sheets as a backend via Apps Script
