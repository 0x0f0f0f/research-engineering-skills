---
name: maintain-memory-md
description: Keep per-directory CLAUDE.md (or AGENTS.md) files accurate and in sync with the code as a CURRENT-STATE description (not a changelog). Trigger after finishing a task that created, deleted, moved, or non-trivially changed files; when the user says "update CLAUDE.md", "sync memory", "wrap up"; or at the end of a coding session before the user moves on. Also use to bootstrap CLAUDE.md files in a project that doesn't have them yet. Session history/progress goes in the progress log (e.g. plans/PROGRESS.md via pr-plan-tracking) and git, NOT in CLAUDE.md.
---

# Maintain CLAUDE.md files

## Why this exists

CLAUDE.md files in each directory are how project knowledge persists across sessions. They go stale fast if nobody updates them. This skill is the protocol for keeping them honest, lean, and useful — and for setting them up from scratch in a project that doesn't have them.

**CLAUDE.md describes the CURRENT STATE, not the history.** It is orientation an agent reads to know how the system works *right now* — not a changelog. Do NOT accumulate commit-level entries, LOC deltas, benchmark numbers, "shipped X then Y" narratives, or dated achievement logs in it. That archaeology bloats the system prompt every session without changing what the agent should do next. Session history goes in the **progress log** (`plans/PROGRESS.md` via the `pr-plan-tracking` skill, or the project's equivalent) and in `git log`; durable discoveries go in `plans/FINDINGS.md`. When you update CLAUDE.md, you EDIT the current-state description in place and DELETE what's no longer true — you don't append.

## When to trigger

**End-of-task triggers (most common):**

- Just finished implementing a feature, fixing a bug, or refactoring.
- Created, moved, deleted, or renamed files.
- Changed a directory's purpose, invariants, or public API.
- User says "wrap up", "update memory", "log this", "update CLAUDE.md", "sync CLAUDE.md".

**Bootstrap triggers:**

- User says "set up CLAUDE.md", "add memory files", "initialize CLAUDE.md across the project".
- Project has no root CLAUDE.md or has only a stub.

## Two modes

### Mode A: Bootstrap (project has no CLAUDE.md, or only a stub)

1. Scan the project structure (respect `.gitignore`; ignore `node_modules`, `.venv`, `dist`, `build`, `__pycache__`, etc.).
2. Identify directories that warrant their own CLAUDE.md. Rule of thumb:
   - Has its own coherent purpose (not just a folder of utilities)
   - Contains >3 source files OR is referenced from multiple places
   - Has non-obvious invariants or conventions
   - Examples: `src/strategies/`, `src/data/`, `tests/`, `references/`
   - Do NOT create CLAUDE.md for: leaf folders with 1–2 trivial files, generated output dirs, vendored deps.
3. Create the **root** `CLAUDE.md` using the template below.
4. Create per-directory `CLAUDE.md` files using the per-directory template. Keep each under 150 lines. Be honest about what you don't know — write `TODO: confirm with user` rather than guessing at invariants.
5. Report the list of files created and ask the user to skim the root one before continuing.

### Mode B: Sync (CLAUDE.md files exist; you just changed code)

Run through this checklist:

1. **List what changed this session.** What files were created, deleted, renamed, or non-trivially edited? Which directories did those changes touch?
2. **For each touched directory:** read its `CLAUDE.md` (if one exists). Check:
   - File list / key files section — is it still accurate?
   - Purpose statement — did the directory's role shift?
   - Invariants/gotchas — did you introduce a new one, or invalidate an old one?
   - Public API or entry points — changed?
     If yes to any: update that section. Keep edits minimal and surgical — do NOT rewrite the whole file.
3. **If a touched directory has no CLAUDE.md** and now meets the "warrants one" bar from Mode A, create one.
4. **Log the session to the progress log, NOT to CLAUDE.md.** Session-by-session "what changed" goes in `plans/PROGRESS.md` (use the `pr-plan-tracking` skill) or the project's progress log — never as an appended changelog inside CLAUDE.md. If CLAUDE.md has a `## Plans & Progress` section, it should be a short *pointer* to the log, not the log itself.
5. **Length / archaeology check.** CLAUDE.md is current-state orientation. If it has grown a changelog ("shipped X", "−124 LOC", dated achievement bullets, benchmark numbers), that's a smell — DELETE it (the history is in git + the progress log). Also prune to keep each file lean (aim <300–400 lines); move detailed architecture into `docs/` and reference it.
6. **Report** the diff: which files you edited, one line each. Don't dump full contents back at the user.

## Root CLAUDE.md template

```markdown
# <Project name>

<1–3 sentence description of what this project is and what its current goal is.>

## Architecture (high level)

<Bullet list of the 3–8 top-level pieces and how they relate. Link to per-directory
CLAUDE.md files. Example:>

- `src/strategies/` — trading strategy implementations (see `src/strategies/CLAUDE.md`)
- `src/data/` — point-in-time data layer (see `src/data/CLAUDE.md`)
- `src/eval/` — evaluation gates: dry run → mini backtest → Pareto → CPCV/DSR
- `references/` — external repos and papers; consult `references/INDEX.md`

## Conventions

<Project-wide rules. Keep tight. Examples:>
- Python: polars only; no pandas. River for online learning.
- No new dependencies without asking.
- Strategies must be pure functions of `(state, data) -> orders`.

## Maintenance protocol

This file is current-state orientation, not a changelog. When finishing a task:
edit the sections above so they describe how the system works NOW, and delete
what's no longer true. Do NOT append dated entries here — session history goes in
`plans/PROGRESS.md` (the `pr-plan-tracking` skill), findings in `plans/FINDINGS.md`,
commit detail in `git log`.

## Plans & Progress

History lives in `plans/` and git, not here:
- `plans/PROGRESS.md` — append-only session log.
- `plans/FINDINGS.md` — durable discoveries / decisions.
- `plans/<slug>.md` — active PR plans; `plans/completed/` — merged.

## References

Before non-trivial changes, consult `references/INDEX.md` for relevant repos and papers.
```

## Per-directory CLAUDE.md template

```markdown
# <directory name>

**Purpose:** <1–2 sentences. What problem does this directory solve in the project?>

## Key files

- `<file>` — <one-line role>
- `<file>` — <one-line role>

(Only the ones that matter for navigation. Skip trivial files.)

## Public surface

What other parts of the project import from here. What's stable vs. internal.

## Invariants & gotchas

- <Things that must hold true>
- <Non-obvious constraints>
- <Past bugs that taught us a lesson>

## Conventions specific to this directory

<Optional. Only if they differ from project-wide.>
```

(No "Recent changes" / changelog section in per-directory files either — describe
the current state and let `git log <dir>` carry the history.)

## Rules of thumb

- **Brevity beats completeness.** A CLAUDE.md nobody reads is worse than no CLAUDE.md. Cap each file at 300–400 lines.
- **Specific beats abstract.** "Strategies must return `polars.DataFrame` with columns `[ts, symbol, side, qty]`" beats "Strategies follow a clean interface."
- **Honest beats aspirational.** If something is messy, say it's messy. Don't describe how the code _should_ be organized.
- **Don't duplicate code into the file.** Reference files and functions by path; don't paste their contents.
- **AGENTS.md vs CLAUDE.md:** if the project already uses `AGENTS.md`, use that name throughout. If neither exists, use `CLAUDE.md` (Claude Code's default). Never create both — pick one and stick with it across the project.

## What NOT to put in CLAUDE.md

- **Changelogs / session history / achievement logs** — dated "shipped X" bullets, LOC deltas, commit-level granularity, test counts, benchmark numbers. This is the most common bloat. It's archaeology loaded as orientation every session and it changes nothing about what the agent does next. → `git log`, `plans/PROGRESS.md`, `plans/FINDINGS.md`, `bench/results/`.
- Long code examples (link to the file instead)
- General programming advice (Claude already knows)
- Personality instructions ("be helpful, be concise") — that's `~/.claude/CLAUDE.md` territory
- Secrets, credentials, or anything that shouldn't be in git
- Detailed tutorials — put those in `docs/`

## After running this skill

End your reply with:

- The list of CLAUDE.md files you created or edited (paths only)
- Where you logged the session (the progress log — NOT CLAUDE.md)
- Anything you flagged as `TODO: confirm with user`
