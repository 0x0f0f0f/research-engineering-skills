---
name: pr-plan-tracking
description: Manage per-PR planning notes, progress logs, and findings in the project's `plans/` directory. Use this skill whenever the user mentions a PR's plan, progress, status, or findings — including phrases like "log progress", "what did we do today", "start a plan for X", "record this finding", "wrap up the PR", "mark the PR done", "ready to merge", or any time work on a PR is starting, advancing, or completing. Also use proactively at the end of a meaningful work session on a PR, even if the user hasn't explicitly asked to log anything, and whenever a non-obvious design decision is made that future contributors would want to know about.
---

# PR Plan Tracking

This skill maintains a lightweight, file-based record of work happening on PRs in a project. Everything lives under `plans/` at the project root:

```
plans/
├── PROGRESS.md              # Append-only log of progress entries, newest at top
├── FINDINGS.md              # Append-only log of important discoveries/decisions
├── <plan-name>.md           # One file per active PR
└── completed/
    └── <plan-name>.md       # Plans moved here when PR is merged
```

The skill has four core operations: **start a plan**, **log progress**, **record a finding**, and **complete a PR**. Pick the one that matches what the user is doing and follow the workflow below. If multiple apply in one turn (e.g. the user shares progress and a finding in the same message), do both.

## Identifying the current PR

Before any operation, figure out which plan we're working on. The plan name is usually derived from the git branch. Resolve it in this order:

1. **Check git**: run `git branch --show-current` to see the current branch. Normalize it to a plan name by stripping common prefixes (`feature/`, `feat/`, `fix/`, `bugfix/`, `chore/`) and using the rest as the slug (e.g. `feature/add-cpcv-eval` → `add-cpcv-eval`).
2. **Check `plans/` for an existing plan file** matching that slug. If `plans/<slug>.md` exists, use it.
3. **Check `plans/PROGRESS.md`** for the most recent entry to see which plan was being worked on — useful when the branch name is ambiguous (`main`, `dev`) or when the user is mid-session.
4. **Ask the user** if there's still ambiguity, or if the resolved plan doesn't seem to match what they're describing. Keep the question short: "Is this for the `<slug>` plan, or something else?"

Never invent a plan name silently. If no plan exists yet for the current branch and the user is logging progress or a finding, ask whether to start a new plan first.

## Timestamps

Always use ISO datetime format: `YYYY-MM-DD HH:MM` (24-hour, local time). Get the current time with `date +"%Y-%m-%d %H:%M"`. Use this exact format consistently across PROGRESS.md and FINDINGS.md — it's how entries line up with each other when cross-referencing.

---

## Operation 1: Start a new plan

Triggered by phrases like "start a plan for X", "let's plan out Y", "new PR for Z", or when the user describes work that has no existing plan file.

1. Resolve the plan slug from the current branch (or ask if on a generic branch).
2. Check that `plans/<slug>.md` doesn't already exist. If it does, ask whether to overwrite or append.
3. Create `plans/<slug>.md` with this structure:

```markdown
# <Plan title — human-readable, not the slug>

## Goal

<One or two sentences on what this PR accomplishes and why.>

## Approach

<Bullet points or short paragraphs on how. Include alternatives considered if relevant.>

## Tasks

- [ ] <Concrete task>
- [ ] <Concrete task>

## Open questions

- <Anything unresolved>
```

4. Fill in what the user has told you. Leave sections empty (with a placeholder comment) rather than fabricating content. If the user gave a rough verbal plan, structure it — don't just paste it.
5. Add a first PROGRESS.md entry noting that the plan was started (see Operation 2).

## Operation 2: Log progress

Triggered by "log progress", "log this", "wrap up for today", "what did we do", or proactively at the end of a meaningful session.

1. Resolve the current plan.
2. Get the current timestamp.
3. Prepend a new entry to `plans/PROGRESS.md` (newest first). If the file doesn't exist, create it with a heading first:

```markdown
# Progress Log

## 2026-05-24 14:30 — <plan-slug>

- <What got done, in past tense, concrete>
- <Next bullet>

### Next up

- <What's queued for the next session, if known>
```

4. Keep entries scannable. Bullets, not paragraphs. Focus on what changed or got decided, not on narration ("I tried X, then Y, then Z" → just say what worked). If a session produced no real progress (just exploration), say so briefly rather than padding.
5. If the plan file has a `## Tasks` checklist, also tick off any completed items in `plans/<slug>.md` itself — keep the plan and progress log in sync.

## Operation 3: Record a finding

Triggered by "record this", "this is important", "let's note that", or whenever a non-obvious design decision or discovery comes up — especially "we chose X because Y was happening otherwise" style insights. Be willing to trigger this proactively when something interesting surfaces; future contributors (and future Claude sessions) benefit a lot from these.

1. Get the current timestamp.
2. Prepend a new entry to `plans/FINDINGS.md` (newest first). Create the file with a heading if it doesn't exist:

```markdown
# Findings

## 2026-05-24 14:30 — <short title>

**Context:** <what we were trying to do>

**Finding:** <what we discovered or decided>

**Why it matters:** <implication for future work, or what would break if we forgot this>

_Related plan: <plan-slug>_
```

3. The title should be specific enough to skim — "ASOF JOIN ordering matters for ALFRED vintages" not "DB issue". Findings outlive any single PR, so write them so they make sense without the surrounding session context.

## Operation 4: Complete a PR

Triggered by "PR is done", "ready to merge", "mark the PR complete", "wrap this up". **Only do this on explicit user request** — don't infer completion from a long progress log or from the user saying "tests pass".

1. Resolve the current plan.
2. Confirm with the user before moving files: "Mark `<slug>` as complete and move it to `plans/completed/`?" This is destructive-ish (changes file location) so always confirm.
3. On confirmation:
   - Add a final progress entry to `plans/PROGRESS.md` noting completion (e.g. "PR complete — merged / ready to merge").
   - Create `plans/completed/` if it doesn't exist: `mkdir -p plans/completed`.
   - Move the plan file: `git mv plans/<slug>.md plans/completed/<slug>.md` (use `git mv` if the repo tracks plans/, otherwise plain `mv`).
4. Don't move or alter PROGRESS.md or FINDINGS.md — they remain at the top level as the project's full history.

---

## Style and tone for entries

- **Concrete over abstract.** "Switched to DuckDB ASOF JOIN for PIT joins" beats "improved data infrastructure".
- **Past tense for progress, present/imperative for plans, declarative for findings.**
- **No fluff.** Skip "Successfully completed", "Great progress on", etc. Just the facts.
- **Link to code when useful** — file paths, function names, commit hashes if available. Don't fabricate hashes; only include ones from `git log` output.
- **Don't duplicate.** If a finding is recorded in FINDINGS.md, the progress entry can reference it ("see FINDINGS: <title>") instead of restating.

## When the `plans/` directory doesn't exist yet

If the user invokes the skill on a project with no `plans/` directory, create it (`mkdir -p plans`) before the first operation. Don't ask permission for this — it's the natural setup step. Mention briefly that you created it.

## A note on proactive use

The user opted into this skill triggering on creating plans, logging progress, recording findings, and completing PRs — all four. That means at natural breakpoints in a session (end of a working block, a meaningful design decision, finishing a chunk of tasks), it's appropriate to suggest logging without waiting to be asked. Keep suggestions light: "Want me to log this to PROGRESS.md?" rather than just doing it unprompted, since the user is the one who knows whether the session is actually done.
