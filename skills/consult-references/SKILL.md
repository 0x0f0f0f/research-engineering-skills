---
name: consult-references
description: Before non-trivial changes, consult the project's references/ directory — vendored repos and paper notes — to ground the work in primary sources instead of guessing. Trigger when about to design, implement, or modify code that involves an algorithm, formula, or API that's covered by a paper or reference repo in references/; when the user mentions a concept that matches a reference (e.g. "DSR", "CPCV", "GEPA", "Pareto selection"); or when the user explicitly says "check references", "consult the literature", "look up X in the repos". Also use to bootstrap references/ structure in a new project.
---

# Consult the references library

## Why this exists

Most LLM mistakes on technical code come from confidently misremembering an algorithm or API. The fix is dirt cheap: read the actual paper or repo before writing the code. This skill makes that reflex automatic.

The convention assumes a project layout like:

```
references/
  INDEX.md           # one-line "what's here and when to consult it"
  repos/
    <name>/          # git submodule or shallow clone, read-only
  papers/
    <slug>/
      NOTES.md       # human-readable summary (the primary thing you read)
      main.tex       # or paper.pdf — the source of truth for equations
      arxiv-id.txt
```

If the project doesn't have this layout yet, use Mode B to set it up.

## When to trigger

**Consult mode (most common):**
- About to implement or change code that touches an algorithm covered by a reference (e.g. modifying walk-forward scoring → check `papers/deflated-sharpe/`).
- About to call into an API of a vendored repo (e.g. building a mutation operator → check `repos/gepa/`).
- User uses a term that matches a paper slug or repo name.
- User says "what does the GEPA paper say about X", "check CPCV before changing this", "consult references".
- You're about to write code from memory for something with mathematical or algorithmic precision (Sharpe-family metrics, cross-validation schemes, optimization algorithms).

**Bootstrap mode:**
- User says "set up references", "initialize references", "create the references directory".
- Project has no `references/` directory and the work clearly needs one.

## Mode A: Consult

### 1. Read the index first

```
references/INDEX.md
```

This is hand-curated and the source of truth for what's available and when to use each. If it doesn't exist, fall back to listing `references/repos/` and `references/papers/` and inferring from directory names.

### 2. Identify what's relevant

Match the task to entries in INDEX.md. Be generous — it's cheaper to check one extra reference than to misimplement something.

### 3. Read in the right order

**Papers:** `papers/<slug>/NOTES.md` first. Only open the `.tex` source when you need to verify an equation, algorithm, or assumption exactly. The LaTeX is there precisely so you can copy formulas verbatim instead of paraphrasing them.

**Repos:** start from the entry point listed in INDEX.md. If none listed, scan the top-level `README.md` and then the module that names suggest is relevant. Use grep/glob to find the specific functions or types you'll be calling or imitating — don't read the whole repo.

### 4. Ground the work

When you write the code or design:
- For formulas: quote the LaTeX block in a comment above the implementation, with the paper slug and section reference. Example:
  ```python
  # Deflated Sharpe Ratio — Bailey & López de Prado 2014, eq. 9
  # See references/papers/deflated-sharpe/NOTES.md
  # $$ \hat{SR} = \frac{(SR - SR_0)\sqrt{N-1}}{\sqrt{1 - \gamma_3 SR + \frac{\gamma_4 - 1}{4} SR^2}} $$
  ```
- For algorithms: cite the paper slug + section/figure in a comment.
- For APIs imitated from a vendored repo: cite the source file in a comment.

### 5. If something contradicts your prior

The reference wins. Update the code, and if you find that the paper's NOTES.md was misleading, note the discrepancy and offer to update NOTES.md.

### 6. If nothing relevant is in references/

Tell the user. Suggest fetching the missing paper (the `read-arxiv-paper` skill handles this) or vendoring the missing repo before proceeding. Don't fall back to "winging it from memory" silently.

## Mode B: Bootstrap references/

When setting up a fresh references directory:

1. Create the structure:
   ```bash
   mkdir -p references/repos references/papers
   ```

2. Create `references/INDEX.md`:
   ```markdown
   # References

   Hand-curated index of repos and papers this project depends on.
   Update whenever you add something.

   ## Repos

   - `repos/<name>/` — <one-line description>. Consult for: <when>.
     Entry point: `<path within the repo>`.

   ## Papers

   - `papers/<slug>/` — <one-line description>. Consult for: <when>.
   ```

3. Create `references/README.md` with a short explanation of the convention (so anyone — human or agent — opening the project understands it).

4. If the user named specific repos to vendor, suggest:
   - For external repos they'll occasionally update: `git submodule add <url> references/repos/<name>`
   - For frozen reference points: `git clone --depth 1 <url> references/repos/<name>` and add `references/repos/*/.git` to `.gitignore` (or just commit the snapshot).

5. If the user named papers, hand off to the `read-arxiv-paper` skill for each.

6. Add a pointer in the root `CLAUDE.md`:
   ```
   ## References

   Before non-trivial changes, consult `references/INDEX.md` for relevant repos and papers.
   Use the `consult-references` skill.
   ```

## What this skill is NOT

- Not a search engine over arbitrary docs. The whole point is that `references/` is small, curated, and project-specific.
- Not for general programming knowledge. Don't open a paper to remember Python syntax.
- Not a substitute for tests. Grounding in the paper doesn't prove the implementation is correct.

## Rules of thumb

- The cost of consulting is one read of a ~200-line NOTES.md. The cost of not consulting is rediscovering why your Sharpe ratio is wrong three weeks later. Consult.
- If you find yourself wanting to "explain the algorithm" and there's a relevant paper in `references/`, open it first. Your explanation will be better and you'll catch the assumptions you'd have missed.
- A reference that's never consulted is dead weight. If you notice `references/INDEX.md` lists something nobody uses, suggest removing it.
