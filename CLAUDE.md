# Ledger — Agent Instructions

## Project Overview

Ledger is a household expense tracker built as single self-contained HTML files. Full context is in [LEDGER_BRIEF.md](LEDGER_BRIEF.md) — read it before making any changes.

## Two Versions

- `Personal Site/expense-tracker-clean.html` — Michael & Lili's personal version (`l3_` localStorage prefix)
- `General Template/expense-tracker-template.html` — Generic shareable template (`lt_` localStorage prefix)

Unless explicitly told otherwise, apply changes to **both files**.

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
