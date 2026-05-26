# CLAUDE.md

Guidance for agents working on this repo.

## Repo shape

This is a skills repo: each `skills/<name>/SKILL.md` is a self-contained Agent
Skill. `.claude-plugin/marketplace.json` lists every skill — ours and the
vendored marimo ones — as a single plugin.

## Vendored marimo skills (`vendor/marimo-team-skills/`)

`vendor/marimo-team-skills/` is a **plain vendored snapshot** of
[`marimo-team/skills`](https://github.com/marimo-team/skills): the files are
committed straight into this repo, **not** a git submodule. We dropped the
submodule on purpose — the Claude Code marketplace installer and `npx skills`
don't initialize submodules
([claude-code#17293](https://github.com/anthropics/claude-code/issues/17293)),
so a submodule would install as empty `vendor/` directories.

**Do not hand-edit anything under `vendor/marimo-team-skills/`** — it is upstream
code. Fix issues upstream and re-sync instead.

### Updating the vendored marimo skills

To pull the latest upstream, shallow-reclone and overwrite the directory:

```bash
tmp=$(mktemp -d)
git clone --depth 1 https://github.com/marimo-team/skills "$tmp"
rm -rf vendor/marimo-team-skills
mkdir -p vendor/marimo-team-skills
cp -a "$tmp"/. vendor/marimo-team-skills/
rm -rf vendor/marimo-team-skills/.git    # drop upstream's git metadata; commit the files
rm -rf "$tmp"
git add vendor/marimo-team-skills
git commit -m "Re-sync vendored marimo-team/skills"
```

After re-syncing, reconcile `.claude-plugin/marketplace.json`'s `skills` array
if upstream added or removed any `skills/<name>/` directories, and confirm
`vendor/marimo-team-skills/LICENSE` is still Apache-2.0.

marimo's skills are Apache-2.0 (`vendor/marimo-team-skills/LICENSE`); this repo's
own skills are MIT.
