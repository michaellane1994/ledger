# Ledger — Site Layout & Sync Workflow

**Last updated:** 2026-05-05 (personal site migrated out of iCloud)

This doc captures where each of the three Ledger sites lives, how edits flow to GitHub, and how the local automation works. Refer to this any time you (or a future agent) needs to touch one of the sites.

## Three Sites at a Glance

| # | Site | Source file (master) | Git working dir | Remote | Live URL | localStorage prefix |
|---|---|---|---|---|---|---|
| 1 | **Personal** (Michael & Lili's real data) | `/Users/michaellane/CLAUDE CODE/ledger/index.html` _(repo IS master)_ | `/Users/michaellane/CLAUDE CODE/ledger/` | `github.com:michaellane1994/ledger.git` | `michaellane1994.github.io/ledger/` | `l3_` |
| 2 | **Self-host / family template** (free, Drive sync) | `~/Library/Mobile Documents/com~apple~CloudDocs/General Template/index-selfhost.html` _(iCloud master)_ | `/Users/michaellane/CLAUDE CODE/expensetracker/` | `github.com:michaellane1994/expensetracker.git` | Check Pages config | `lt_` |
| 3 | **SaaS** (paid, Drive sync, Stripe gates) | `~/Library/Mobile Documents/com~apple~CloudDocs/Saas Model/index.html` _(iCloud master)_ | `/Users/michaellane/CLAUDE CODE/ledger-saas/public/index.html` | `github.com:michaellane1994/ledger-saas.git` | `michaellane1994.github.io/ledger-saas/` | `lt_` |

> **Personal site migration note (2026-05-05):** the iCloud folder at `~/Library/Mobile Documents/com~apple~CloudDocs/Personal Site/` was archived to `~/ledger-archive/personal-site-2026-05-05/` and the repo became the single source of truth. The `Personal Site` symlink in this repo was removed at the same time. Sites 2 and 3 still use iCloud as the editing source.

## Edit Workflow

### Personal site (#1)

1. Edit `/Users/michaellane/CLAUDE CODE/ledger/index.html` directly.
2. `git add index.html && git commit -m "..." && git push`.
3. GitHub Pages picks it up at `michaellane1994.github.io/ledger/`.

No `cp` step. No iCloud sync. Repo is the only place the file lives.

### Self-host (#2)

1. Edit `~/Library/Mobile Documents/com~apple~CloudDocs/General Template/index-selfhost.html` (iCloud master).
2. `cp` to `/Users/michaellane/CLAUDE CODE/expensetracker/index.html`.
3. `cd` into the repo, commit, push.

### SaaS (#3)

1. Edit `~/Library/Mobile Documents/com~apple~CloudDocs/Saas Model/index.html` (iCloud master).
2. `cp` to `/Users/michaellane/CLAUDE CODE/ledger-saas/public/index.html`.
3. `cd` into the repo, commit, push.

## Auto-Pull (Personal Site Only)

A LaunchAgent on this Mac pulls the personal repo every 5 minutes so remote agents and other-machine pushes flow back automatically.

**Files:**

- Script: `~/ledger-archive/auto-pull.sh`
- LaunchAgent plist: `~/Library/LaunchAgents/com.michaellane.ledger-pull.plist`
- Log: `~/ledger-archive/auto-pull.log`

**Behavior:**

- Runs every 300s (5 min).
- Checks `git status --porcelain` first; if you have uncommitted changes, it skips with a log entry. Your in-progress edits are never clobbered.
- Uses `git pull --ff-only` so it never auto-merges. If history diverged, it skips and logs.
- Silent on "Already up to date"; only logs skips / actual pulls / errors.

**Inspect:**

```bash
launchctl list | grep ledger-pull        # confirm loaded; PID 0 = idle, exit code 0 = healthy
tail -f ~/ledger-archive/auto-pull.log    # watch activity
```

**Disable:**

```bash
launchctl unload ~/Library/LaunchAgents/com.michaellane.ledger-pull.plist
```

**Re-enable:**

```bash
launchctl load ~/Library/LaunchAgents/com.michaellane.ledger-pull.plist
```

**Force-run now:**

```bash
bash ~/ledger-archive/auto-pull.sh
```

## What's NOT Automated

- Sites 2 (self-host) and 3 (SaaS) have no auto-pull. If a remote agent ever pushes to those repos, you'll need to `git pull` them manually.
- The auto-pull does not push. If a remote agent commits and you also have local commits, the pull will be skipped (non-fast-forward). Resolve manually.
- iCloud sync between devices is irrelevant for site #1 now. Sites 2 and 3 still rely on iCloud for that.

## Remote Agents

Agents scheduled via `/schedule` run in Anthropic's cloud and only have access to the GitHub repo they clone. They cannot reach iCloud or your local filesystem.

For site #1 this is seamless: agent clones, edits, pushes; auto-pull catches up your local copy within 5 minutes.

For sites #2 and #3, a remote agent could in theory be queued, but its push would only update the GitHub repo — your local iCloud master would still be stale. You'd need to manually pull and reverse-sync to iCloud, which defeats the iCloud-master pattern. So in practice, only schedule remote agents for the personal repo.

## Migrating Sites 2 / 3 Out of iCloud Later (Optional)

If you ever want sites 2 and 3 to behave like site 1 (repo-as-master, no iCloud, optional auto-pull), the steps for each are:

1. `mv ~/Library/Mobile\ Documents/com~apple~CloudDocs/<Site>/ ~/ledger-archive/<site>-YYYY-MM-DD/`
2. Remove any symlinks in dependent repos that point at the archived path.
3. Update CLAUDE.md and the memory routing table to reflect the new master.
4. Optional: copy the `auto-pull.sh` and plist patterns and create `<site>-pull.plist` for that repo.

About 10 minutes per site.

## Quick Path Reference

| Purpose | Path |
|---|---|
| Personal site source (master) | `/Users/michaellane/CLAUDE CODE/ledger/index.html` |
| Self-host source (iCloud master) | `~/Library/Mobile Documents/com~apple~CloudDocs/General Template/index-selfhost.html` |
| SaaS source (iCloud master) | `~/Library/Mobile Documents/com~apple~CloudDocs/Saas Model/index.html` |
| Personal site iCloud archive | `~/ledger-archive/personal-site-2026-05-05/` |
| Auto-pull script | `~/ledger-archive/auto-pull.sh` |
| Auto-pull plist | `~/Library/LaunchAgents/com.michaellane.ledger-pull.plist` |
| Auto-pull log | `~/ledger-archive/auto-pull.log` |
