# Ledger — Site Layout & Sync Workflow

**Last updated:** 2026-08-13 (repo-as-master for both active sites; stale iCloud copies flagged)

Where each site lives, how edits reach GitHub, and what the local automation does.
Read this before touching any site or running any `cp`.

## Active sites

Two sites are actively maintained. A third exists but is deprecated.

| # | Site | Master (edit this) | Git working dir | Remote | Live URL | localStorage |
|---|---|---|---|---|---|---|
| 1 | **Personal** (Michael & Lili's real data) | `ledger/index.html` — **repo is master** | `/Users/michaellane/CLAUDE CODE/ledger/` | `michaellane1994/ledger` | `michaellane1994.github.io/ledger/` | `l3_` |
| 2 | **Trove SaaS** (paid, Drive sync, Stripe) | `ledger-saas/public/app/index.html` (app)<br>`ledger-saas/public/index.html` (landing) — **repo is master** | `/Users/michaellane/CLAUDE CODE/ledger-saas/` | `michaellane1994/ledger-saas` | `householdtrove.com` (Cloudflare) | `lt_` |
| — | ~~Self-host / family template~~ | *(deprecated 2026-07-08 — do not edit)* | `/Users/michaellane/CLAUDE CODE/expensetracker/` | `michaellane1994/expensetracker` | GitHub Pages | `lt_` |

### ⚠️ The iCloud copies are stale — do not `cp` from them

Older versions of this doc described sites 2 and 3 as having **iCloud masters** that you
copy into the repo. That is no longer true, and following it would destroy work:

| iCloud folder | Last modified | Repo equivalent | Verdict |
|---|---|---|---|
| `CloudDocs/Saas Model/index.html` | **2026-05-04** | `ledger-saas/public/app/index.html` (2026-08-12) | **3+ months stale. Ignore it.** |
| `CloudDocs/General Template/index-selfhost.html` | — | `expensetracker/index.html` | Deprecated site; not maintained |
| `CloudDocs/Personal Site/` | retired 2026-05-05 | — | Archived to `~/ledger-archive/personal-site-2026-05-05/` |

**Both active sites are now repo-as-master.** Edit the file in the repo, commit, push. No
`cp` step, no iCloud round-trip, for either one.

## Edit workflow

### Personal site (#1)

1. Edit `ledger/index.html` directly.
2. Extract the inline JS and syntax-check it (see below) — required by CLAUDE.md.
3. `git add index.html && git commit && git push`.
4. GitHub Pages serves it at `michaellane1994.github.io/ledger/`.

### Trove SaaS (#2)

1. Edit `ledger-saas/public/app/index.html` (the app) or `public/index.html` (the landing).
2. Syntax-check as above.
3. `git add <file> && git commit && git push`.
4. Cloudflare auto-deploys from `main`; live within ~a minute at `householdtrove.com`.

Worker changes (`src/worker.js`) need a deploy to take effect.

### Syntax check (no `node` on PATH)

`node` is not installed on this machine. Use the Node runtime bundled with the IDE:

```bash
ELECTRON_RUN_AS_NODE=1 "/Applications/Antigravity IDE.app/Contents/MacOS/Electron" --check extracted.js
```

Extract the JS first — it's the single largest inline `<script>` block at the bottom of
the HTML. `jq` is also unavailable; use `python3` for JSON work.

## Repo hygiene

**Both repos are public.** Never commit business identifiers (CRA BN, BC registration
number, legal name on record), secrets, or the ops gmail. Deliberately kept out of git:

| File | Where it lives | Why |
|---|---|---|
| `LEDGER_SAAS_ROADMAP.md` | `ledger/`, gitignored | Internal strategy + business state |
| `SECURITY.md` | `ledger-saas/`, gitignored | Security posture detail |
| Memory files | `~/.claude/projects/-Users-michaellane-CLAUDE-CODE-ledger/memory/` | Outside any repo |

Superseded pre-rebrand backups were archived out of both repos on 2026-08-13 to
`~/ledger-archive/workspace-cleanup-2026-08-13/`. Git history still holds every version,
so don't re-add backup copies — use `git show <ref>:<path>` instead.

## Auto-pull (all three repos)

A LaunchAgent per repo pulls every 5 minutes so remote agents and other-machine pushes
flow back automatically. Verified loaded 2026-08-13.

| Repo | Script | LaunchAgent label |
|---|---|---|
| `ledger` | `~/ledger-archive/auto-pull.sh` | `com.michaellane.ledger-pull` |
| `expensetracker` | `~/ledger-archive/auto-pull-expensetracker.sh` | `com.michaellane.expensetracker-pull` |
| `ledger-saas` | `~/ledger-archive/auto-pull-ledger-saas.sh` | `com.michaellane.ledger-saas-pull` |

Plists live at `~/Library/LaunchAgents/<label>.plist`. Logs at `~/ledger-archive/auto-pull*.log`.

**Behavior (identical for all three):** runs every 300s; skips if the working tree is dirty
(in-progress edits are never clobbered); uses `git pull --ff-only` so it never auto-merges;
silent unless it skips, pulls, or errors.

```bash
launchctl list | grep -E "ledger-pull|expensetracker-pull|ledger-saas-pull"
tail -f ~/ledger-archive/auto-pull*.log
launchctl unload ~/Library/LaunchAgents/com.michaellane.ledger-pull.plist   # disable one
bash ~/ledger-archive/auto-pull.sh                                          # force-run now
```

## What's not automated

- Auto-pull never pushes. If a remote agent commits while you have local commits, the pull
  skips (non-fast-forward). Resolve manually.
- Session context. A new chat gets `CLAUDE.md`, `MEMORY.md`, and skill names automatically,
  plus recent git state via the `SessionStart` hook (`~/.claude/hooks/ledger-git-context.py`).
  Run `/checkpoint` to reconcile the notes against git when they drift.

## Remote agents

Agents scheduled via `/schedule` run in Anthropic's cloud and only see the GitHub repo they
clone — no iCloud, no local filesystem, and **no memory files**. Both active sites are
repo-as-master, so remote agents work cleanly for either; auto-pull catches your local copy
up within 5 minutes.

## Quick path reference

| Purpose | Path |
|---|---|
| Personal site source | `/Users/michaellane/CLAUDE CODE/ledger/index.html` |
| SaaS app source | `/Users/michaellane/CLAUDE CODE/ledger-saas/public/app/index.html` |
| SaaS landing source | `/Users/michaellane/CLAUDE CODE/ledger-saas/public/index.html` |
| SaaS Worker (backend) | `/Users/michaellane/CLAUDE CODE/ledger-saas/src/worker.js` |
| Roadmap (gitignored) | `/Users/michaellane/CLAUDE CODE/ledger/LEDGER_SAAS_ROADMAP.md` |
| Memory files | `~/.claude/projects/-Users-michaellane-CLAUDE-CODE-ledger/memory/` |
| Archives | `~/ledger-archive/` |
