# Research Engineering Skills for Agents

A curated set of six [Agent Skills](https://code.claude.com/docs/en/skills) for
doing **high-quality engineering on research-grade codebases** — the kind of work
where correctness is grounded in primary sources, architecture is kept deep and
navigable, and project knowledge is written down so it survives across sessions.

Each skill is a self-contained `SKILL.md` (plus supporting files) usable by any
agent that understands the Skills convention — Claude Code, Codex, Cursor, and others.

## Skills

| Skill | What it does |
| ----- | ------------ |
| [`improve-codebase-architecture`](skills/improve-codebase-architecture/) | Surface architectural friction and propose *deepening* refactors (shallow → deep modules), presented as a visual before/after HTML report, then a grilling loop to design the chosen refactor. |
| [`consult-references`](skills/consult-references/) | Before non-trivial changes, read the actual paper or vendored repo in `references/` instead of guessing. Grounds algorithms and formulas in primary sources. |
| [`read-arxiv-paper`](skills/read-arxiv-paper/) | Ingest an arXiv paper into `references/papers/<slug>/` using the highest-fidelity format available (HTML → ar5iv → LaTeX → PDF) and write a structured `NOTES.md`. |
| [`maintain-memory-md`](skills/maintain-memory-md/) | Keep per-directory `CLAUDE.md`/`AGENTS.md` files honest and in sync with the code, plus a root Progress log. Bootstraps them in projects that have none. |
| [`pr-plan-tracking`](skills/pr-plan-tracking/) | Maintain lightweight per-PR plans, progress logs, and findings under `plans/`. Start a plan, log progress, record a finding, complete a PR. |
| [`marimo-notebook-tests`](skills/marimo-notebook-tests/) | Wire up a Python project so files are simultaneously marimo interactive notebooks and pytest test modules — executable docs that stay green in CI. |

Together they form a loop: **read the literature → ground the code in it → keep the
architecture deep → record what you learned → track it on the PR.**

## Install

### Option A — Claude Code plugin marketplace

This repo ships a `.claude-plugin/marketplace.json`, so it works as a Claude Code
marketplace out of the box. From inside Claude Code:

```
/plugin marketplace add 0x0f0f0f/research-engineering-skills
/plugin install research-engineering-skills@research-engineering-skills
```

The first command registers the marketplace; the second installs the
`research-engineering-skills` plugin (all six skills). Run `/plugin` to manage
installed plugins.

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

## Repository layout

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
├── README.md
└── LICENSE
```

The `skills/<name>/SKILL.md` layout is what both registries discover, so a single
structure serves the Claude marketplace and `npx skills` simultaneously.

## License

MIT — see [LICENSE](LICENSE).
