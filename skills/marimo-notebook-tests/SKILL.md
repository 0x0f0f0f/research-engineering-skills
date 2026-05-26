---
name: marimo-notebook-tests
description: Set up a Python project so files are simultaneously marimo interactive notebooks and pytest test modules — assertions are the source of truth, cells render the results. Use when the user wants notebooks that double as tests, asks to wire up marimo with pytest, or wants executable documentation/tutorials that stay green in CI.
---

# marimo-notebook-tests

Set up a Python project so that `notebooks/*.py` are simultaneously
marimo interactive notebooks and pytest test modules.

Assumes `marimo` is installed as a dev dependency alongside `pytest`.

## Follow marimo's notebook authoring guidelines

This skill owns the *notebook-as-test wiring* — the `test_*` / `@app.cell`
split below. It does **not** redefine how marimo cells are written. That is
owned by marimo's official `marimo-notebook` skill, which is **already vendored
into this plugin** at
[`vendor/marimo-team-skills/skills/marimo-notebook/`](../../vendor/marimo-team-skills/skills/marimo-notebook/)
and installed alongside this one. **Do not run `npx skills add marimo-team/skills`
to reinstall it** — it ships with this plugin; reinstalling would duplicate it.
Load the bundled `marimo-notebook` skill and **always follow it** when writing
the cell bodies.

Apply its rules verbatim. The load-bearing ones, repeated here so they are
never skipped:

- **PEP 723 script metadata** at the top of every notebook file
  (`# /// script` … `# dependencies = ["marimo", …]` … `# ///`).
- **Variables between cells define the reactivity** — let marimo order cells
  from their parameters; don't wrap cell bodies in needless `if`/`try`.
- **Never mutate an object across cells** — create a new object instead, so
  the dependency graph stays correct.
- **Always create UI elements; only vary the *data source* by mode** via
  `mo.app_meta().mode == "script"` (synthetic/default data in script mode,
  widget values interactively). Don't conditionally hide widgets.
- **Render equations** with `mo.md(r"""$...$""")` so the notebook mirrors the
  paper's notation.
- **Don't over-prefix variables with underscores** — reserve `_` for a
  cell-local throwaway or two, not as a habit.
- **Run `marimo check <notebook.py>`** before committing to catch the common
  mistakes automatically.

When marimo's guidance and the wiring below ever disagree, marimo wins on cell
style; this skill only dictates the `test_*` / cell split and the execution
modes.

## The pattern

Every notebook file has three layers:

```python
import marimo
app = marimo.App(width="medium")


# 1. test_* functions at module level — importable, pytest-visible.
#    Convention: return results so cells can reuse them for display
#    without running the logic twice.
def test_something():
    from mylib import thing
    result = thing()
    assert result == expected
    return result


# 2. @app.cell functions — marimo reactive UI.
#    mo is imported once in the first cell and flows as a dependency.
@app.cell
def _():
    import marimo as mo
    return (mo,)

@app.cell
def _(mo):
    result = test_something()   # assertion always runs inside the cell
    return (mo.md(f"✓ result: {result}"),)


# 3. entry point — launches the notebook UI when run directly
if __name__ == "__main__":
    app.run()
```

## Execution modes

| command                            | behaviour                                                   |
| ---------------------------------- | ----------------------------------------------------------- |
| `pytest notebooks/`                | finds `test_*` functions at module level, runs assertions   |
| `marimo run notebooks/foo.py`      | interactive notebook; cells call `test_*` then render prose |
| `python notebooks/foo.py`          | launches the notebook UI                                    |
| `from notebooks.foo import test_x` | plain import, marimo never invoked                          |

## pyproject.toml

```toml
[project.optional-dependencies]
test = ["pytest>=7", "marimo>=0.9", "discopy>=1.0"]   # marimo is a dev dep

[tool.pytest.ini_options]
testpaths = ["tests"]          # default: fast unit tests only
# to include notebook integration tests: pytest tests/ notebooks/
```

## Key invariants

- `test_*` functions **never import marimo** — pure logic, no UI coupling.
- `import marimo as mo` lives in the **first cell only**; all subsequent
  cells receive `mo` as a parameter via marimo's dependency injection.
- `@app.cell` returns a `Cell` object, not the original function, so
  pytest will **not** accidentally treat cell functions as tests.
- Multiple cells named `_` are fine: marimo registers each before the
  name is rebound; pytest ignores underscore-prefixed names.
- Cells call `test_*` first, then display — the assertion is the
  single source of truth, the cell just renders the result.

## Where to put them

- `tests/tutorials/` — preferred for a maths/PL library. Sits inside the
  existing test tree alongside unit tests, directly mirroring the
  `test/tutorials/` convention in JuliaSymbolics/Metatheory.jl. Since pytest
  recurses into `tests/`, no extra `testpaths` config is needed.
- `notebooks/` — alternative if docs-primary identity is preferred.
