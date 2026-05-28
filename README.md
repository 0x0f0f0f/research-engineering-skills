# 🔬 Research Engineering Skills for Agents

A curated set of six [Agent Skills](https://code.claude.com/docs/en/skills) for
doing **high-quality engineering on research-grade codebases** — the kind of work
where correctness is grounded in primary sources, architecture is kept deep and
navigable, and project knowledge is written down so it survives across sessions.

Each skill is a self-contained `SKILL.md` (plus supporting files) usable by any
agent that understands the Skills convention — Claude Code, Codex, Cursor, and others.

## ✨ The skills

| Skill | What it does |
| ----- | ------------ |
| 🏛️ [`improve-codebase-architecture`](skills/improve-codebase-architecture/) | Surface architectural friction and propose *deepening* refactors (shallow → deep modules), presented as a visual before/after HTML report, then a grilling loop to design the chosen refactor. |
| 📚 [`consult-references`](skills/consult-references/) | Before non-trivial changes, read the actual paper or vendored repo in `references/` instead of guessing. Grounds algorithms and formulas in primary sources. |
| 📥 [`read-arxiv-paper`](skills/read-arxiv-paper/) | Ingest an arXiv paper into `references/papers/<slug>/`: read the full text from alphaXiv markdown, copy exact equations from ar5iv/arXiv-HTML MathML, pull TikZ diagram code from the LaTeX source, and write a structured `NOTES.md`. |
| 🧠 [`maintain-memory-md`](skills/maintain-memory-md/) | Keep per-directory `CLAUDE.md`/`AGENTS.md` files honest and in sync with the code as a current-state description (session history lives in `plans/` via `pr-plan-tracking`, not in `CLAUDE.md`). Bootstraps them in projects that have none. |
| 📋 [`pr-plan-tracking`](skills/pr-plan-tracking/) | Maintain lightweight per-PR plans, progress logs, and findings under `plans/`. Start a plan, log progress, record a finding, complete a PR. |
| 🧪 [`marimo-notebook-tests`](skills/marimo-notebook-tests/) | Wire up a Python project so files are simultaneously marimo interactive notebooks and pytest test modules — executable docs that stay green in CI. |

## 🟢 Bundled marimo skills

This plugin **vendors [`marimo-team/skills`](https://github.com/marimo-team/skills)**
directly under [`vendor/marimo-team-skills/`](vendor/marimo-team-skills/) — the
files are committed into this repo (**not** a git submodule), so they always come
along on a plain `git clone`, the Claude Code marketplace install, and
`npx skills`. We deliberately avoided a submodule: the marketplace installer does
not initialize submodules
([claude-code#17293](https://github.com/anthropics/claude-code/issues/17293)),
which would leave the `vendor/` paths empty. Each skill is listed in
`marketplace.json`, so installing this plugin also gives you marimo's own
skills — `marimo-notebook` (the canonical notebook-authoring rules that
`marimo-notebook-tests` defers to), `implement-paper`, `implement-paper-auto`,
`auto-paper-demo`, `jupyter-to-marimo`, `streamlit-to-marimo`, `anywidget`,
`wasm-compatibility`, `add-molab-badge`, and `marimo-batch`.

The vendored copy is a **snapshot of upstream**; to re-sync, shallow-reclone and
overwrite the directory — see [`CLAUDE.md`](CLAUDE.md) for the exact steps. These
skills are © the marimo team under **Apache-2.0** — see
[`vendor/marimo-team-skills/LICENSE`](vendor/marimo-team-skills/LICENSE); this
repo's own skills remain MIT.

## 🔁 How to use them — the loop

The six skills aren't a grab-bag; they chain into one workflow. A typical run from
"here's a paper" to "the refactor is merged and remembered" looks like this:

1. **📥 Ingest the papers.** Point [`read-arxiv-paper`](skills/read-arxiv-paper/)
   at an arXiv ID or URL. It reads the full text from alphaXiv markdown, copies
   exact equations from ar5iv/arXiv-HTML MathML, and pulls TikZ diagram code from
   the arXiv LaTeX source, unpacks them into `references/papers/<slug>/`, and writes
   a structured `NOTES.md` you'll actually re-read — equations copied *verbatim*,
   not paraphrased.

2. **📚 Read the papers — and the implementation.** Before any non-trivial change,
   [`consult-references`](skills/consult-references/) opens the relevant `NOTES.md`
   (and the `.tex` when an equation has to be exact) plus the vendored repos in
   `references/repos/`. Algorithms get grounded in the source, not in memory.

3. **💡 Get the insights — and formalize the problem.** With the literature loaded,
   [`improve-codebase-architecture`](skills/improve-codebase-architecture/) walks the
   code for *shallow* modules and friction, naming things in the project's own domain
   vocabulary (`CONTEXT.md`) and a precise architecture glossary
   (*deep / shallow / seam / leverage*). The **deletion test** separates pass-throughs
   from modules that earn their keep.

4. **📋 Propose the improvements as a plan.** The architecture skill presents candidate
   *deepening* refactors as a visual before/after HTML report; pick one, and
   [`pr-plan-tracking`](skills/pr-plan-tracking/) turns it into a per-PR plan under
   `plans/` — tasks, open questions, and a progress log.

5. **🚀 Execute the plan.** Implement the refactor, citing the paper slug and the
   equation right above the code. [`marimo-notebook-tests`](skills/marimo-notebook-tests/)
   makes the change verifiable: files that are *simultaneously* marimo notebooks and
   pytest modules, so the docs stay green in CI.

6. **🧠 Record what you learned.** [`maintain-memory-md`](skills/maintain-memory-md/)
   keeps per-directory `CLAUDE.md`/`AGENTS.md` honest as a current-state description;
   `pr-plan-tracking` logs the session progress and load-bearing findings under `plans/`.
   The next session — human or agent — gets back up to speed in 20 seconds.

> **A concrete run:** ingest the *Deflated Sharpe Ratio* paper → consult it before
> touching walk-forward scoring → notice the scoring module is *shallow* (a thin
> wrapper hiding the real bug in how it's called) → plan a deepening refactor →
> implement it with the DSR equation quoted verbatim above the function → log the
> finding so nobody re-derives it three weeks later.

Then the loop closes and starts again: **read the literature → ground the code in it →
keep the architecture deep → record what you learned → track it on the PR.**

## 🧩 How this compares to general-purpose agent rules

If you've seen Karpathy's [`andrej-karpathy-skills`](https://github.com/multica-ai/andrej-karpathy-skills)
(the viral CLAUDE.md / skill, 220k+ combined stars), it packages four behavioural
rules — *think before coding*, *keep it simple*, *make surgical changes*, *drive toward
a verifiable goal*. It's a short, agent-agnostic **discipline layer** that sharpens how
an agent behaves on *any* codebase. Great baseline.

These skills work a layer deeper and more specific: they're **workflow skills for
research-grade codebases**, and they leave behind durable artifacts — paper notes,
grounded citations, architecture reports, PR plans, memory files. The difference is
concreteness:

- Where the Karpathy rules say *think before coding*, `consult-references` says **what to
  read first and where the equation lives**.
- Where they say *keep it simple*, `improve-codebase-architecture` gives a **vocabulary**
  (deep / shallow / seam) and a **test** (the deletion test) for what "simple" means
  *structurally* — and proposes the refactor, not just the principle.
- Where they say *drive toward a verifiable goal*, `marimo-notebook-tests` and
  `pr-plan-tracking` make the goal **executable and tracked**.

They compose well: keep the Karpathy rules as your behavioural baseline and add these for
the *read-the-paper-then-deepen-the-module* loop. If your work is grounded in literature
and you want the agent to leave behind something the next session can build on, this set
goes further.

## 📦 Install

### Option A — Claude Code plugin marketplace

This repo ships a `.claude-plugin/marketplace.json`, so it works as a Claude Code
marketplace out of the box. From inside Claude Code:

```
/plugin marketplace add 0x0f0f0f/research-engineering-skills
/plugin install research-engineering-skills@research-engineering-skills
```

The first command registers the marketplace; the second installs the
`research-engineering-skills` plugin (the six authored skills plus the bundled
marimo skills, which are vendored directly so they install without any extra
steps). Run `/plugin` to manage installed plugins.

### Option B — `npx skills` (Vercel Labs)

GitHub *is* the registry for [`npx skills`](https://github.com/vercel-labs/skills);
no submission step is required. To install every skill in this repo into the
current project:

```bash
npx skills add 0x0f0f0f/research-engineering-skills
```

Useful variants:

```bash
# install globally (available to all your projects)
npx skills add 0x0f0f0f/research-engineering-skills -g

# target specific agents
npx skills add 0x0f0f0f/research-engineering-skills --agent claude-code cursor

# list / search
npx skills list
npx skills find architecture
```

Skills land in the agent's config directory (`.claude/skills/` or `.agents/skills/`).

### Option C — manual / git submodule

Copy a `skills/<name>/` directory into your project's `.claude/skills/` (or
`.agents/skills/`), or vendor the whole repo:

```bash
git submodule add https://github.com/0x0f0f0f/research-engineering-skills \
  .agents/research-engineering-skills
```

## 🗂️ Repository layout

```
research-engineering-skills/
├── .claude-plugin/
│   └── marketplace.json        # Claude Code marketplace manifest (one plugin, six skills)
├── skills/
│   ├── improve-codebase-architecture/
│   │   ├── SKILL.md
│   │   ├── LANGUAGE.md          # architecture glossary (deep/shallow/seam/leverage)
│   │   ├── HTML-REPORT.md       # report scaffold + diagram patterns
│   │   ├── INTERFACE-DESIGN.md
│   │   ├── DEEPENING.md
│   │   ├── CONTEXT-FORMAT.md    # bundled: domain-context doc format
│   │   └── ADR-FORMAT.md        # bundled: ADR format
│   ├── consult-references/SKILL.md
│   ├── read-arxiv-paper/SKILL.md
│   ├── maintain-memory-md/SKILL.md
│   ├── pr-plan-tracking/SKILL.md
│   └── marimo-notebook-tests/SKILL.md
├── vendor/
│   └── marimo-team-skills/         # vendored copy of marimo-team/skills (Apache-2.0, not a submodule)
│       └── skills/                 # marimo-notebook, implement-paper, jupyter-to-marimo, …
├── README.md
└── LICENSE
```

The `skills/<name>/SKILL.md` layout is what both registries discover, so a single
structure serves the Claude marketplace and `npx skills` simultaneously.

## 📄 License

MIT — see [LICENSE](LICENSE).
