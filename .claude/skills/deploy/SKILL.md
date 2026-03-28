---
name: deploy
description: Deploy Ledger to GitHub Pages. Stages all changes, commits with a message, and pushes to main.
disable-model-invocation: true
argument-hint: "<commit message>"
---

Deploy Ledger to GitHub Pages.

Commit message: $ARGUMENTS

Steps:
1. Run `git fetch origin` to check for any remote changes.
2. Run `git diff origin/main -- index.html` to see if the remote index.html differs from the local one.
   - If there ARE differences, read both the remote version (`git show origin/main:index.html`) and the local `expense-tracker-clean.html`, identify what changed on the remote (e.g. custom categories, savings account types, income types, any hardcoded config), and merge those changes into `expense-tracker-clean.html` before proceeding. Tell the user what you found and merged.
   - If there are NO differences, proceed.
3. Copy `Personal Site/expense-tracker-clean.html` → `index.html` (this is what gets deployed).
4. Syntax check: run `node --check index.html`. If `node` is not in PATH (command not found), skip this step and proceed — do not block the deploy.
5. Stage `index.html` explicitly: `git add index.html`. Do NOT attempt to stage `General Template/expense-tracker-template.html` via git — it is a symlink target and git will error with "beyond a symbolic link". Template changes are edited in place and do not need to be committed to this repo.
6. Commit with the provided message (append "Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>").
7. Push to `main` with `git push`.
8. After the push completes, always tell the user explicitly: "Pushed to GitHub — https://michaellane1994.github.io/ledger/" if successful, or "Push failed — [error]" if not.

Note: Google Sheets sync only works after deployment (CORS blocks local testing).
