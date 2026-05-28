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

## What `plans/` is for — and what it is NOT

`plans/` and `FINDINGS.md` are **agent memory**: TODO lists, design plans, important
decisions, theory, and actual discoveries that stay true. They are **NOT a changelog
of every code change** — that is what `git log` is for. Before writing an entry, ask
"will this still be useful, and still true, in a month?" If it's just "changed X to
Y", it belongs in the commit message, not here. FINDINGS.md is for durable
conclusions, not a running tally.

**PROGRESS.md is COMPACT current-status-by-plan, NOT a session changelog.** It holds,
per active plan, what's *done* and what's *next* — plus a short "recent milestones"
list of ONE-LINERS (newest first). It must stay short (aim < ~80 lines): a long
PROGRESS duplicates `git log`, drifts, and confuses agents. So when logging progress:
**update the relevant plan's status block IN PLACE** and **prepend a single one-line
milestone** — do NOT append a multi-paragraph session entry (the blow-by-blow is in
`git log` + the commit message). If PROGRESS has grown into verbose per-session
entries, compact it (status-by-plan + one-line milestones; old detail → git) with a
dated `> Compacted` note.

**Never log benchmark numbers in `plans/`/`FINDINGS.md` during iterative perf work.**
Concrete figures (wall times, "1.7× faster", MB) go stale the instant the next run
lands and actively confuse future agents — the values in two entries disagree and
nobody knows which is current. Instead: (1) commit the actual benchmark table to a
**single results file** (e.g. `bench/results/latest.md`), overwritten each full
run, so `git log -p` is the trustworthy history; and (2) display the fresh table to
the user. FINDINGS may record the *durable conclusion* ("the vectorized path wins at
scale"; "the warmup cost amortizes, it's not constant-factor") but **not the digits** —
point to the results file for those. This applies to repo work generally, not just
this skill.

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
2. **Update that plan's status block in `plans/PROGRESS.md` IN PLACE** — move finished
   work from "next" to "done", revise "next". (Create PROGRESS with a `## Status by
   plan` section + a `## Recent milestones` list if it doesn't exist.)
3. **Prepend ONE line** to the `## Recent milestones` list (newest first):
   `- <date>  <plan>: <what shipped, one line>`. The blow-by-blow goes in the commit
   message + `git log`, NOT here.

```markdown
# Progress

## Status by plan
### <plan-slug>
- **Done:** <terse>
- **Next:** <terse>

## Recent milestones (one line each, newest first; detail in git log)
- 2026-05-24  <plan>: <one-line what shipped>
```

4. Do NOT write multi-paragraph per-session entries — that's the changelog PROGRESS is
   explicitly not. Keep the whole file short (aim < ~80 lines); if it has bloated into
   session logs, compact it (see "What `plans/` is for").
5. If the plan file has a `## Tasks` checklist, also tick off completed items in
   `plans/<slug>.md` — keep plan and status in sync.

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
2. **Verify completion against the code, item by item.** A plan is "complete" only when EVERY task is actually done in the codebase — not just claimed in the plan text. Walk each `- [ ]`/`- [x]` and confirm against the current source (grep the symbol, read the function, run the test). Plans drift: a `[x]` can describe a feature later removed (e.g. a `backend=` arg, a JAX path), and a `[ ]` can already be done. If any item is genuinely unfinished — and is NOT explicitly blocked-upstream or deferred-to-another-PR — the plan is NOT complete: leave it active and report what remains. Don't move a plan with real open work.
3. Confirm with the user before moving files: "Mark `<slug>` as complete and move it to `plans/completed/`?" This is destructive-ish (changes file location) so always confirm.
4. On confirmation:
   - Add a final progress entry to `plans/PROGRESS.md` noting completion (e.g. "PR complete — merged / ready to merge").
   - Create `plans/completed/` if it doesn't exist: `mkdir -p plans/completed`.
   - **Compact the plan as you move it.** The completed copy should be *maximally informative but tight*: keep the goal, what actually shipped (with the load-bearing design decisions, invariants, and commit hashes), and the non-goals/follow-ups. Cut the per-session blow-by-blow, intermediate measurements, abandoned approaches, and superseded numbers — that detail lives in git history (`git log -- plans/<slug>.md`). Write the new compacted file at `plans/completed/<slug>.md` and `git rm` the original (a plain `git mv` keeps the bloat; we want the trimmed version). Add a one-line `> Compacted on completion; full history in git.` note at the top.
5. Don't move or alter PROGRESS.md or FINDINGS.md — they remain at the top level as the project's full history.

**Pruning FINDINGS.md (do this when it sprawls, or when asked).** FINDINGS.md is append-only *by default*, but an iterative optimization loop produces entries whose numbers get superseded round-over-round and entries about features that were later removed. When it becomes confusing (conflicting numbers, dead-feature references), compact it: KEEP the durable, still-true conclusions (general design facts, publishable results, invariants); REMOVE process-noise (intermediate wall times, abandoned micro-experiments) and findings about deleted code. Consolidate a multi-entry saga into one current-state finding. Nothing is lost — it's in `git log -p plans/FINDINGS.md`. Add a dated `> Compacted` note at the top explaining what was cut and where to find it.

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
