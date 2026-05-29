---
name: read-arxiv-paper
description: Ingest an arXiv paper into the project reference library. Read the full text from alphaXiv markdown (the NOTES.md base), copy exact equations from ar5iv / arXiv-HTML MathML alttext, and pull the arXiv LaTeX e-print source for TikZ diagram code, into references/papers/<slug>/, then write a structured NOTES.md. Use whenever the user gives an arXiv/alphaXiv ID or URL, or asks to "ingest", "read", "summarize", or "add" a paper.
---

# Read an arXiv paper into the project reference library

## When to trigger

Any of:

- User gives an arXiv ID (e.g. `2507.19457`, `1905.05033v2`) or URL (`arxiv.org/abs/...`, `arxiv.org/pdf/...`, `arxiv.org/html/...`, `ar5iv.labs.arxiv.org/...`, `alphaxiv.org/...`).
- User says "read this paper", "summarize this paper", "add this paper to references", "ingest this paper".
- User asks Claude to consult literature on a topic and the source is on arXiv.

## Goal

Produce, inside the project, a self-contained directory `references/papers/<slug>/` containing:

- `NOTES.md` — the _actual_ artifact you and future sessions will read, written from the readable full text.
- The fetched **readable** content (`alphaxiv.md`) and the **exact-math** source (`src/` from the LaTeX tarball, or `paper.html`).
- An `arxiv-id.txt` so the paper can be re-fetched.
- A `SOURCE.md` recording where each layer came from, so the next session knows whether equations can be trusted verbatim.

## Three jobs, three sources

A paper does three different jobs in your library, and each wants a different
format. Don't treat these as a first-wins cascade — collect the layers the paper
actually needs:

1. **Readable full text → NOTES.md prose.** alphaXiv serves clean, LLM-friendly
   markdown of the whole paper in one GET. Default first fetch; the base you read
   and summarize from.
2. **Exact mathematics → verbatim equations.** ar5iv / arXiv-HTML render the
   LaTeX via LaTeXML to **MathML carrying the original LaTeX in each node's
   `alttext`** — copyable, exact, and far easier than wrangling raw `.tex`
   macros. This is the preferred equation source; reach for it whenever any
   equation matters.
3. **Diagram code → TikZ source.** The one thing the HTML layers do *not* give
   you: LaTeXML renders TikZ to SVG (or drops it) and `tikz-cd`/string-diagram
   coverage is partial — exactly the diagrams category-theory papers live on
   (Bonchi, Sobociński, Zanasi). For the actual diagram source you need the
   **arXiv LaTeX e-print tarball**, which also backstops equations when HTML
   rendering chokes on exotic macros.

alphaXiv full text degrades complex displayed math and cannot represent TikZ at
all, so it is the readable base, never the math or diagram authority.

## Layer 1 — alphaXiv markdown (readable, default)

alphaXiv exposes two plain-markdown endpoints keyed by the bare paper ID. No
auth, no JSON.

```bash
# (optional) AI-generated structured overview — quick orientation
curl -sS -o overview.md "https://www.alphaxiv.org/overview/<id>.md"

# full extracted paper text as markdown — the base for NOTES.md
curl -sS -o alphaxiv.md "https://www.alphaxiv.org/abs/<id>.md"
```

- `overview/<id>.md` returns alphaXiv's AI analysis of the paper. Good for a
  fast map; **not** a substitute for the text. Skip if you want only ground truth.
- `abs/<id>.md` returns the full extracted paper text, page by page. This is the
  readable base — read it end-to-end.

Sanity-check the result: file is more than a few KB and contains the paper's
section headings, not an error blob. **404 on overview** → report not generated
yet; use the full text. **404 on full text** → not processed yet; fall back to
ar5iv/arXiv-HTML (below) for the readable layer.

## Layer 2 — ar5iv / arXiv HTML (exact math, promoted)

The preferred source for equations. arXiv's own HTML (2023+) and ar5iv (older
corpus) both run LaTeXML, which emits **MathML carrying the original LaTeX in
each node's `alttext`** — so you copy exact equations without ever touching raw
`.tex`. It doubles as a clean readable layer when alphaXiv 404s.

```bash
curl -L -sS -o paper.html "https://arxiv.org/html/<id>"             # arXiv's own HTML (2023+)
curl -L -sS -o paper.html "https://ar5iv.labs.arxiv.org/html/<id>"  # older corpus
```

Check it actually rendered — arXiv can return HTTP 200 with an abstract/landing
or error page instead of the rendered paper. Treat it as real only if: file >10KB,
contains `<math` (or `MathJax`), shows the **paper body** (real sections, not just
the abstract/metadata stub), and does **not** contain "No HTML for this paper". If
what you got is an abstract/fallback page, do **not** record it as the exact-math
source in `SOURCE.md` — drop to Layer 3 instead. For each equation you cite, copy
the LaTeX from its `alttext` attribute (or the MathML block) — this is exact, not
extracted.

What this layer does **not** reliably give you: **TikZ diagram source.** LaTeXML
renders TikZ to SVG when it can, and `tikz-cd`/string-diagram coverage is partial,
so diagrams arrive as images or not at all. For diagram code, drop to Layer 3.

## Layer 3 — arXiv LaTeX e-print source (diagram code + math backstop)

The ground truth for **diagrams** — the `.tex` holds the exact TikZ/`tikzcd`
source the HTML layers can't surface — and the backstop for equations when HTML
rendering chokes on exotic macros. Pull this for any diagram-heavy paper
(category theory, signal-flow, anything Bonchi-flavoured).

```bash
curl -L -sS -o source.tar.gz "https://arxiv.org/e-print/<id>"
file source.tar.gz
```

- `gzip compressed` / `POSIX tar archive`: `mkdir -p src && tar -xzf source.tar.gz -C src` (try `tar -xf` if gzip fails — some are bare tar). For big tarballs (>50MB), extract selectively:
  ```bash
  tar -xzf source.tar.gz -C src --wildcards '*.tex' '*.bib' '*.bbl' '*.sty' 2>/dev/null
  ```
- `PDF document`: the author opted out of source; there is no LaTeX. Lean on Layer 2 for math and flag lossiness.
- HTML: rate-limited or wrong ID. Note it and rely on Layers 1–2.

Remove `source.tar.gz` after extraction. Find the entry point:
`grep -l '\\documentclass' src/*.tex 2>/dev/null`; the one with `\begin{document}`
is root — follow `\input`/`\include`. For diagrams, copy the relevant
TikZ/`tikzcd` block **verbatim** — never invent one. Use the `.tex` to confirm any
equation the HTML layer rendered oddly.

## Layer 4 — PDF (last resort)

```bash
curl -L -sS -o paper.pdf "https://arxiv.org/pdf/<id>"
pdftotext -layout paper.pdf paper.txt
```

Equations will be lossy. Flag this loudly in NOTES.md and SOURCE.md.

## Procedure

### 1. Extract / normalize the paper ID

Parse the bare ID from whatever the user provides:

| Input | Paper ID |
|-------|----------|
| `https://arxiv.org/abs/2401.12345` | `2401.12345` |
| `https://arxiv.org/pdf/2401.12345` | `2401.12345` |
| `https://arxiv.org/html/2401.12345v2` | `2401.12345v2` |
| `https://www.alphaxiv.org/abs/2401.12345` | `2401.12345` |
| `https://alphaxiv.org/overview/2401.12345` | `2401.12345` |
| `https://ar5iv.labs.arxiv.org/html/2401.12345` | `2401.12345` |
| `2401.12345v2` | `2401.12345v2` |
| `2401.12345` | `2401.12345` |
| old-style `cs/0701001` | `cs/0701001` |

Default to the latest version unless the user pinned one (the endpoints accept versioned IDs).

### 2. Pick a slug and a citation key

**Slug** — the directory name. Short, kebab-case, descriptive. Examples:
`interacting-hopf`, `gepa`, `deflated-sharpe`.

- If the user suggested one, use it.
- Otherwise propose one from the title or known shorthand; confirm if ambiguous.
- If `references/papers/<slug>/` exists, suffix `-v2`, `-2024`, etc., or ask.

**Citation key** — the stable handle written into prose, code comments, and the
YAML front-matter `key:` field. Format **`<first-author-surname>-<year>-<title-slug>`**,
kebab-case, ASCII-folded:

- First author surname only, lowercased, accents stripped (Sobociński → `sobocinski`,
  Van der Cruysse → `vandercruysse`).
- `<year>` is the 4-digit publication year.
- `<title-slug>` is a short descriptive slug — reuse the directory slug unless it
  collides with another paper by the same author/year (then disambiguate with a
  suffix `-i`, `-ii`, or a distinguishing word).
- Examples: `zhang-2022-relational-e-matching`, `tiurin-2024-e-hypergraphs`,
  `bonchi-2022-string-diagrams-i`.

The point of the key is that a language model recognizes `Zhang et al. 2022,
"Relational E-Matching"` and triggers its trained knowledge **without following
the link**. Carry the author + year + title in every citation for exactly that
reason — never a bare path or bare arXiv id.

### 3. Fetch metadata

```bash
curl -sS "http://export.arxiv.org/api/query?id_list=<id>" -o metadata.atom
```

Parse the Atom XML for `<title>`, `<author><name>`, `<published>`, `<summary>`.
If the `arxiv` Python package is available, use it instead:

```python
import arxiv
paper = next(arxiv.Client().results(arxiv.Search(id_list=["<id>"])))
# paper.title, paper.authors, paper.published, paper.summary, paper.categories
```

### 4. Fetch the content

1. **Layer 1 (always):** alphaXiv `alphaxiv.md` (+ optional `overview.md`) — readable base.
2. **Layer 2 (whenever any equation matters — default yes):** ar5iv/arXiv-HTML `paper.html` — exact math via MathML `alttext`. Also the readable fallback if Layer 1 404s.
3. **Layer 3 (whenever diagrams matter, or HTML math fails):** LaTeX e-print `src/` — TikZ source + equation backstop. Mandatory for diagram-heavy papers.
4. **Layer 4:** PDF only if everything above fails.

Record what you obtained in each layer — this drives the fidelity claim in SOURCE.md.

### 5. Create the directory

```bash
mkdir -p references/papers/<slug>
echo "<arxiv-id>" > references/papers/<slug>/arxiv-id.txt
```

Move the fetched files in (`alphaxiv.md`, `overview.md`, `src/`, `paper.html`, `metadata.atom`).

### 6. Read the paper

- Read `alphaxiv.md` end-to-end for the narrative — this is your NOTES.md base.
- For every equation you cite, pull the **exact** form from `src/*.tex` (or the
  HTML `alttext`); do not trust the extracted markdown for displayed math.
- For figures/string diagrams, record the caption and what the text says, and
  copy the TikZ source from `src/` if the diagram is load-bearing.

### 7. Write SOURCE.md

```markdown
# Source provenance

- **arXiv ID:** <id> (version <v>)
- **Fetched:** YYYY-MM-DD
- **Readable layer:** alphaXiv full text (`alphaxiv.md`) | ar5iv/arXiv HTML | PDF-extracted text
- **Exact-math layer:** ar5iv/arXiv-HTML MathML (`paper.html`) | LaTeX e-print (`src/`) | none (PDF-only — lossy)
- **Diagram layer:** TikZ source in `src/` | rendered SVG/image only | none
- **Equation fidelity:** exact (MathML `alttext` / `.tex`) | lossy (PDF extraction — verify against paper.pdf)
- **Files:**
  - `alphaxiv.md` — readable full text (NOTES.md base)
  - `paper.html` — ar5iv/arXiv HTML (exact equations via MathML alttext)
  - `src/` — LaTeX source (TikZ diagram code + math backstop), entry point `<main>.tex`
  - `metadata.atom` — arXiv API metadata
  - `arxiv-id.txt` — for re-fetching
```

### 8. Write NOTES.md

Target 100–250 lines. The primary artifact future sessions read.

**NOTES.md opens with a well-formed YAML front-matter block** — the formal
reference metadata schema (see "## Reference metadata schema" below). This block
is the single source of truth: `references/INDEX.md` is rendered from it, and the
`summary`/link fields let a session decide whether to open the paper at all.

````markdown
---
type: paper
key: <surname-year-slug>
title: "<Full paper title>"
authors: [<Surname>, <Surname>, ...]   # surnames in author order
year: <YYYY>
venue: "<e.g. POPL 2022>"              # omit if unknown
arxiv: "<id>"                           # bare id, no URL
doi: "<10.xxxx/...>"                    # omit if none
url: "https://arxiv.org/abs/<id>"
slug: <dir-slug>
summary: >
  1–3 sentence abstract. What problem, what method, what result.
  This is the super-short summary the index shows; keep it tight.
consult_for: >
  One sentence: when to re-read this in THIS project.
---

# <Paper title>

**Cite as:** <Surname> et al. (<year>), *<short title>* · **arXiv:** [<id>](https://arxiv.org/abs/<id>) · **Source:** see `SOURCE.md`

## TL;DR

2–4 sentences. What problem, what method, what result.

## When to consult this paper

Concrete situations in _this project_ where you'd re-read this. Be specific to the codebase if you have that context.

## Key contributions

- ...

## Method

The substantive part. Explain the algorithm/approach clearly enough to re-implement
from this alone. Include key equations _verbatim_, copied from the LaTeX source
(do not paraphrase math):

\```
$$ \hat{SR} = \frac{(SR - SR_0)\sqrt{N-1}}{\sqrt{1 - \gamma_3 SR + \frac{\gamma_4 - 1}{4} SR^2}} $$
\```

For diagrammatic papers, copy the load-bearing TikZ/`tikzcd` block too.

## Results

What they measured, on what, with what numbers. Skip if purely theoretical.

## Assumptions & limitations

The fine print that will bite you if you forget it.

## Open questions / things to verify

Anything unclear, or that the project should empirically check before relying on.

## References to follow up

Other cited papers worth reading. Titles + arXiv IDs.

## File map

- `alphaxiv.md` — readable full text
- `paper.html` — exact equations (MathML alttext)
- `src/<main>.tex` — TikZ diagram code + math backstop
- `SOURCE.md` — provenance
- `arxiv-id.txt` — for re-fetching
````

### 9. Update references/INDEX.md

INDEX.md is a catalog **rendered from the front-matter** of each reference. Append
or update one entry under `## Papers`, in this fixed shape so every entry is
parallel and cites author + year + title:

```
- **[<key>]** <Surname> et al. (<year>), *<title>* — `papers/<slug>/`.
  [arXiv:<id>](https://arxiv.org/abs/<id>) · doi:<doi>.
  <summary, one line>. **Consult for:** <when>.
```

Use the same `<key>` and link that you wrote into the front-matter. Otherwise
create a minimal INDEX with `## Repos`, `## Papers`, and `## Web` sections.

### 10. Report back

- Slug used, path written.
- **Which layers you obtained** (alphaXiv full text? LaTeX source? PDF-only?) and what that means for equation fidelity.
- One-sentence TL;DR.
- Suggest the user review `NOTES.md` and correct any mischaracterization.

## Reference metadata schema

Every reference in `references/` — paper, repo, or web page — carries a YAML
front-matter block at the top of its primary markdown file (`NOTES.md` for
papers and web pages, `AGENTS.md` for vendored repos). This is the formal,
machine-readable schema; `references/INDEX.md` is a view rendered from it.

**Shared fields (all types):**

| field | required | notes |
|-------|----------|-------|
| `type` | yes | `paper` \| `repo` \| `web` |
| `key` | yes | `<surname>-<year>-<slug>` citation handle (see step 2) |
| `title` | yes | full title, quoted |
| `authors` | yes | list of surnames in author order (`author` string ok for one) |
| `year` | yes | publication year (`YYYY`); omit only for undated tools |
| `summary` | yes | 1–3 sentence abstract (block scalar `>`) |
| `consult_for` | recommended | one line: when to re-read in this project |
| link | yes | **at least one** of `arxiv`, `doi`, `url` |

**Paper** (`papers/<slug>/NOTES.md`): adds `arxiv` (bare id), `doi`, `venue`,
`slug`, `url`.

**Repo** (`repos/<name>/AGENTS.md`): `type: repo`, plus `name`, `repo_url`
(upstream), `entry_points` (list of paths to start from), `vendored: true`, and
`paper:` (the citation key of the originating paper, if one exists). Key a repo
to its paper's author+year where there is one (`willsey-2021-egg`); for pure
tools with no canonical paper, key by name (`polars`, `discopy`).

```yaml
---
type: repo
key: willsey-2021-egg
name: egg
title: "egg: Fast and Extensible Equality Saturation"
authors: [Willsey, Nandi, Wang, Flatt, Tatlock, Panchekha]
year: 2021
repo_url: "https://github.com/egraphs-good/egg"
arxiv: "2004.03082"
paper: willsey-2021-egg
entry_points: ["src/egraph.rs", "src/run.rs", "src/explain.rs"]
vendored: true
summary: >
  Rust e-graph + equality-saturation library; the classic match→merge→rebuild loop.
consult_for: >
  Classic EqSat loop, rebuild, explain (spanning-tree LCA), test ports.
---
```

**Web** (`web/<slug>/NOTES.md`): `type: web`, plus `url`, `fetched` (YYYY-MM-DD),
`slug`.

```yaml
---
type: web
key: flatt-2022-egraph-intersection
title: "Limitations of E-Graph Intersection"
authors: [Flatt]
year: 2022
url: "https://www.oflatt.com/egraph-union-intersect.html"
fetched: "2026-05-25"
slug: oflatt-egraph-union-intersect
summary: >
  E-graph union is O(n log n); intersection of the full congruence closure is not
  finitely representable in general, only approximable over a chosen term set.
consult_for: >
  Composing/merging e-graphs, e-hypergraph intersection, proof-certificate meets.
---
```

**Citing a reference elsewhere** (prose, plans, code comments): always carry
author + year + title so a reader (human or model) recognizes the source without
following the link. Canonical forms:

- prose: `Zhang et al. (2022), "Relational E-Matching"` — append `[key]` and/or
  the path when it aids navigation.
- code comment:
  ```python
  # Relational E-Matching — Zhang et al. 2022 [zhang-2022-relational-e-matching]
  # See references/papers/relational-e-matching/NOTES.md
  ```

Never cite with a bare path (`references/papers/foo/`) or bare id (`arXiv 2108.02290`)
alone — those don't trigger a model's latent knowledge of the paper.

## Constraints

- Never overwrite an existing `references/papers/<slug>/` without confirmation.
- Never commit.
- If `references/` doesn't exist and you're not at an obvious project root, ask.
- **Don't paraphrase equations — copy them from the LaTeX source.**
- If the paper isn't on arXiv (DOI-only, journal-only), tell the user and offer to read a PDF they provide.

## Edge cases

- **Restricted networks:** some sandboxes allowlist outbound hosts and will 403
  `arxiv.org`/`alphaxiv.org`. If a fetch returns `Host not in allowlist` (or a
  bare 403 with a tiny body), stop and tell the user the environment blocks the
  host rather than silently writing an empty paper.
- **alphaXiv full text 404:** not processed yet — use ar5iv/arXiv-HTML for the readable layer, and still pull the LaTeX source for math.
- **Author opted out of LaTeX source** (e-print returns a PDF): no exact-math layer; lean on arXiv/ar5iv MathML, else flag PDF lossiness.
- **Withdrawn papers:** all endpoints error. Detect and report.
- **Multi-file LaTeX with broken `\input` paths:** if extraction missed a referenced file, note it; don't summarize a partial paper as if complete.
- **Non-English papers:** rare; read what you can, flag in NOTES.md.

## Why this layering

alphaXiv's hosted markdown is a single GET that returns clean, whole-paper text
tuned for LLM reading — better than throwing a 40-page PDF at the model — so it's
the default readable layer. But extracted markdown isn't authoritative for *math*:
displayed equations get mangled and TikZ vanishes. ar5iv/arXiv-HTML fix the math —
LaTeXML emits MathML with the original LaTeX in `alttext`, so equations copy out
exactly without parsing raw `.tex`; that's why the HTML layer sits *above* the
tarball for equations. What the HTML layers *can't* do is diagrams: TikZ renders to
SVG (or drops), and `tikz-cd`/string-diagram coverage is partial — precisely the
diagrams category-theory papers depend on. So the LaTeX e-print stays the authority
for diagram code and the backstop when HTML math fails. Three jobs, three sources:
alphaXiv reads the paper, the HTML proves the math, the source proves the diagrams.
