---
name: deploy
description: Deploy the Ledger sites. Personal → michaellane1994/ledger (GitHub Pages). Trove SaaS → michaellane1994/ledger-saas (Cloudflare, householdtrove.com).
disable-model-invocation: true
argument-hint: "<commit message>"
---

Deploy a Ledger site. Commit message: $ARGUMENTS

**Name which site you are deploying before you touch git**, and deploy one site per run
unless the user explicitly asks for both. Both repos are **public**.

## Before anything

1. **Both sites are repo-as-master.** Edit the file in the repo, commit, push. There is no
   `cp` from iCloud — the iCloud "Saas Model" copy is stale since 2026-05-04 and copying
   from it would destroy months of work. See `SITE_LAYOUT.md`.
2. **Never deploy the General Template** (`expensetracker`). Deprecated 2026-07-08.
3. **Syntax check first** — required by CLAUDE.md, and `node` is not on PATH:
   - Extract the largest inline `<script>` block from the HTML to a temp `.js`
   - `ELECTRON_RUN_AS_NODE=1 "/Applications/Antigravity IDE.app/Contents/MacOS/Electron" --check <file>`
   - Do not push if it fails.
4. **Scan the diff before committing** for anything that must not be public: CRA BN,
   BC registration number, legal-name-on-record, the ops gmail, API keys, real transaction
   data. If any appear, stop and tell the user.
5. **Stage explicit paths.** Never `git add .` — it can pick up untracked scratch files.

## Personal site → michaellane1994/ledger

1. `git fetch origin` and check `git diff origin/main -- index.html` for remote-only changes
   (a remote agent or another machine may have pushed). If they differ, reconcile before
   committing rather than blowing them away.
2. Syntax check `index.html` as above.
3. `git add index.html` (plus any other explicitly changed files).
4. Commit with the provided message, appending:
   `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`
5. `git push origin main`.
6. Report: "Personal site pushed — https://michaellane1994.github.io/ledger/"

Note: Google Sheets sync only works once deployed (CORS blocks local testing).

## Trove SaaS → michaellane1994/ledger-saas

Working dir: `/Users/michaellane/CLAUDE CODE/ledger-saas/`

1. `git fetch origin`, check for remote-only changes as above.
2. Identify what changed:
   - `public/app/index.html` — the app
   - `public/index.html` — the marketing landing page
   - `src/worker.js` — the backend Worker
   - `public/{privacy,terms,accessibility}.html`, `sitemap.xml`, `robots.txt` — legal/SEO
3. Syntax check any changed HTML's inline JS.
4. **If you touched `sitemap.xml`, `robots.txt`, or a canonical tag:** use the
   **extensionless** URLs (`/privacy`, not `/privacy.html`). Cloudflare Pages 307-redirects
   the `.html` form, so pointing SEO signals at it creates redirect hops.
5. `git add <explicit paths>`, commit with attribution, `git push origin main`.
6. Cloudflare auto-deploys from `main`. **Verify it actually went live** before reporting —
   poll the real URL rather than assuming:
   ```bash
   until curl -s https://householdtrove.com/<changed-path> | grep -q "<something new>"; do sleep 5; done
   ```
7. Report: "Trove SaaS pushed and verified live — https://householdtrove.com"

## Reminders that bite

- **Apps Script**: if the change touched the `<pre id="apps-script-code">` block on the
  personal site, tell the user explicitly: *"The Apps Script has changed — copy the updated
  code from the Setup Guide into your Google Sheets Apps Script editor and redeploy."*
- **Branding**: if the change touched the wordmark, product name, or palette, refresh
  `og-image.png`, `email-logo.png`, and the `emailShell()` templates in `src/worker.js`
  (see CLAUDE.md).
- **`TEST_MODE`**: still `true` in `public/app/index.html` and must stay that way until the
  Stripe live-mode flip. Never flip it as a side effect of another change.
