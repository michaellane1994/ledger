---
name: deploy
description: Deploy both Ledger versions to GitHub Pages. Personal site → michaellane1994/ledger. General Template → michaellane1994/expensetracker.
disable-model-invocation: true
argument-hint: "<commit message>"
---

Deploy both Ledger versions to GitHub Pages.

Commit message: $ARGUMENTS

## Personal Site → michaellane1994/ledger

1. Run `git fetch origin` to check for remote changes.
2. Run `git diff origin/main -- index.html` to see if the remote index.html differs from the local one.
   - If there ARE differences, read both the remote version (`git show origin/main:index.html`) and the local `expense-tracker-clean.html`, identify what changed on the remote (e.g. custom categories, savings account types, income types, any hardcoded config), and merge those changes into `expense-tracker-clean.html` before proceeding. Tell the user what you found and merged.
   - If there are NO differences, proceed.
3. Copy `Personal Site/expense-tracker-clean.html` → `index.html`.
4. Syntax check: run `node --check index.html`. If `node` is not in PATH, skip and proceed.
5. Stage `index.html` explicitly: `git add index.html`. Do NOT stage `General Template/index.html` via git — it is a symlink target and git will error.
6. Commit with the provided message (append "Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>").
7. Push to `main` with `git push`.
8. Report: "Personal site pushed — https://michaellane1994.github.io/ledger/" or "Personal site push failed — [error]".

## General Template → michaellane1994/expensetracker

This is a separate standalone repo at `/Users/michaellane/CLAUDE CODE/expensetracker/` (if it exists locally) — but since there is no local clone, push the template file directly using the GitHub API or by cloning the repo temporarily.

**Practical steps:**
1. Check if a local clone exists: `ls "/Users/michaellane/CLAUDE CODE/expensetracker" 2>/dev/null && echo exists || echo missing`
2. If missing, clone it: `git clone git@github.com:michaellane1994/expensetracker.git "/Users/michaellane/CLAUDE CODE/expensetracker"`
3. Copy `General Template/index.html` → `/Users/michaellane/CLAUDE CODE/expensetracker/index.html`
4. Stage, commit, and push:
   ```
   git -C "/Users/michaellane/CLAUDE CODE/expensetracker" add index.html
   git -C "/Users/michaellane/CLAUDE CODE/expensetracker" commit -m "<same commit message> Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
   git -C "/Users/michaellane/CLAUDE CODE/expensetracker" push
   ```
5. Report: "Template pushed — https://michaellane1994.github.io/expensetracker/" or "Template push failed — [error]".

## Final summary to user

Always end with both results explicitly:
- "Personal site pushed — https://michaellane1994.github.io/ledger/"
- "Template pushed — https://michaellane1994.github.io/expensetracker/"

Or the relevant failure message for either.

Note: Google Sheets sync only works after deployment (CORS blocks local testing).
