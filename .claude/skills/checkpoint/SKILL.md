---
name: checkpoint
description: Reconcile the project notes (memory files, LEDGER_SAAS_ROADMAP.md, skills) against what git actually shows, and fix whatever has drifted. Run at the end of a work session or before /clear.
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, Grep, Glob
---

Bring the written record back in line with reality, so the next session starts from
facts instead of stale claims.

Do this yourself, in this session. Do not spawn subagents.

## Ground truth (already gathered)

Personal site (`ledger`) recent commits:
!`cd "/Users/michaellane/CLAUDE CODE/ledger" && git log -15 --date=short --pretty="%ad %s"`

Personal site working tree:
!`cd "/Users/michaellane/CLAUDE CODE/ledger" && git status --short && git log origin/main..HEAD --oneline`

SaaS site (`ledger-saas`) recent commits:
!`cd "/Users/michaellane/CLAUDE CODE/ledger-saas" && git log -15 --date=short --pretty="%ad %s"`

SaaS working tree:
!`cd "/Users/michaellane/CLAUDE CODE/ledger-saas" && git status --short && git log origin/main..HEAD --oneline`

Memory files by last-modified (oldest first — oldest are likeliest to be stale):
!`ls -lt --time-style=+%Y-%m-%d "/Users/michaellane/.claude/projects/-Users-michaellane-CLAUDE-CODE-ledger/memory/" 2>/dev/null || ls -lTt "/Users/michaellane/.claude/projects/-Users-michaellane-CLAUDE-CODE-ledger/memory/"`

Launch flag (must be `true` until the Stripe live flip):
!`grep -n "TEST_MODE = " "/Users/michaellane/CLAUDE CODE/ledger-saas/public/app/index.html"`

## What to do

### 1. Read the current record

Read `MEMORY.md` (the index) and the "Current status" section at the top of
`LEDGER_SAAS_ROADMAP.md`. Read individual memory files only when a commit above
suggests one may be affected — don't read all 29 every time.

### 2. Find the drift

Compare the commits against what the notes claim. The failure modes seen in practice:

- **"Pending" that already shipped.** The commonest one. A memory said the Savings
  Goals SaaS mirror was pending for weeks after it shipped. Any memory saying
  pending / queued / not yet built / mirror needed is worth checking against git.
- **Shipped but never recorded.** Work that landed with no memory or roadmap entry,
  especially decisions ("we chose X over Y and here's why") — those aren't
  recoverable from a diff later.
- **Superseded facts.** A setting that was off and is now on, a file that moved, a
  parser that now exists.
- **Stale index lines.** `MEMORY.md` descriptions drift from their file's contents.
- **Stale skills.** Check `.claude/skills/*/SKILL.md` for references to paths, repos
  or workflows that no longer apply. The `deploy` skill was found in Aug 2026 still
  referencing the retired `Personal Site/expense-tracker-clean.html` and pushing the
  deprecated General Template.

### 3. Fix it

- **Update the existing file** rather than adding a near-duplicate memory. Delete
  memories that turned out to be wrong.
- Keep each memory one fact, with its `description` frontmatter accurate — that
  line is what future-you searches on.
- Update the matching one-line pointer in `MEMORY.md` whenever a file's substance
  changes. Put launch-relevant entries near the top.
- Refresh the roadmap's "Current status" date, critical path, and anything the
  commits contradict.
- Convert relative dates to absolute before writing.

### 4. Guardrails

- **The `ledger` repo is public.** `LEDGER_SAAS_ROADMAP.md` is gitignored and stays
  that way — never re-add it, and never write the CRA BN, BC registration number,
  legal name on record, ops gmail, or any secret into a tracked file. Those belong
  in the local memory files only. See `feedback_verify_deploy_target`.
- **Don't record what the repo already tells you.** Code structure and fix history
  live in git. Memory is for decisions, constraints, external state, and the "why".
- **Don't commit or push** as part of a checkpoint. Memory files live outside git;
  the roadmap is gitignored. If skill files changed, say so and let the user decide.

### 5. Report

Close with a short list of what changed and what you deliberately left alone. If
nothing had drifted, say that plainly — a clean checkpoint is a real result, not a
reason to invent edits.
