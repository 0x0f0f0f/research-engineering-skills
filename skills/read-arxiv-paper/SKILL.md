---
name: read-arxiv-paper
description: Ingest an arXiv paper into the project reference library. Fetch the readable full text from alphaXiv markdown FIRST (the base for NOTES.md), then pull the arXiv LaTeX e-print source for the COMPLETE, exact mathematics and TikZ diagrams, unpack into references/papers/<slug>/, read it end-to-end, and write a structured NOTES.md. Use whenever the user gives an arXiv/alphaXiv ID or URL, or asks to "ingest", "read", "summarize", or "add" a paper to their reference library.
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

## The two-layer strategy

A paper has two jobs in your library, and they want different formats:

1. **Readable full text → for NOTES.md prose.** alphaXiv serves clean,
   LLM-friendly markdown of the whole paper in a single GET. This is the
   **default first fetch** and the base you read and summarize from.
2. **Exact mathematics and diagrams → for verbatim equations.** alphaXiv's
   markdown is *extracted* text — great for prose and most inline math, but it
   degrades complex displayed equations and **cannot represent TikZ string
   diagrams** (the bread and butter of category-theory papers — Bonchi,
   Sobocinski, Zanasi et al.). For anything math- or diagram-heavy, also pull
   the **arXiv LaTeX e-print source**, which contains the `.tex` with exact
   equations and the TikZ source for every diagram.

Do **not** treat these as a first-wins cascade. Get layer 1 always; get layer 2
whenever equations or diagrams matter (which, for this library, is almost
always). ar5iv/arXiv-HTML and PDF are fallbacks when a layer is unavailable.

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

## Layer 2 — arXiv LaTeX e-print source (exact math + TikZ diagrams)

```bash
curl -L -sS -o source.tar.gz "https://arxiv.org/e-print/<id>"
file source.tar.gz
```

- `gzip compressed` / `POSIX tar archive`: `mkdir -p src && tar -xzf source.tar.gz -C src` (try `tar -xf` if gzip fails — some are bare tar). For big tarballs (>50MB), extract selectively:
  ```bash
  tar -xzf source.tar.gz -C src --wildcards '*.tex' '*.bib' '*.bbl' '*.sty' 2>/dev/null
  ```
- `PDF document`: the author opted out of source; there is no LaTeX. Use Layer 3/4 for math and flag lossiness.
- HTML: rate-limited or wrong ID. Note it and rely on Layer 1 + Layer 3.

Remove `source.tar.gz` after extraction. Find the entry point:
`grep -l '\\documentclass' src/*.tex 2>/dev/null`; the one with `\begin{document}`
is root — follow `\input`/`\include`. Copy equations **verbatim** from the `.tex`
into NOTES.md (preferring `\begin{equation}`/`$$` blocks). For diagrams, copy the
relevant TikZ/`tikzcd` block or describe it from the source — never invent one.

## Layer 3 — ar5iv / arXiv HTML (rendered alternative)

When you want rendered MathML instead of wrangling LaTeX macros, or when Layer 1
is unavailable for the readable text:

```bash
curl -L -sS -o paper.html "https://arxiv.org/html/<id>"          # arXiv's own HTML (2023+)
curl -L -sS -o paper.html "https://ar5iv.labs.arxiv.org/html/<id>"  # older corpus
```

Check it actually rendered: file >10KB, contains `<math` / `MathJax`, and does
**not** contain "No HTML for this paper". Equations are MathML with LaTeX in
`alttext` — copy the `alttext`.

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

### 2. Pick a slug

Short, kebab-case, descriptive. Examples: `interacting-hopf`, `gepa`, `deflated-sharpe`.

- If the user suggested one, use it.
- Otherwise propose one from the title or known shorthand; confirm if ambiguous.
- If `references/papers/<slug>/` exists, suffix `-v2`, `-2024`, etc., or ask.

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

1. **Layer 1 (always):** `alphaxiv.md` (+ optional `overview.md`).
2. **Layer 2 (whenever math/diagrams matter — default yes):** LaTeX e-print `src/`.
3. Fall back to **Layer 3** (ar5iv/HTML) for the readable layer if Layer 1 404s, or for rendered math if you don't want raw LaTeX.
4. **Layer 4** PDF only if everything above fails.

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
- **Readable layer:** alphaXiv full text (`alphaxiv.md`) | ar5iv HTML | PDF-extracted text
- **Exact-math layer:** LaTeX e-print source (`src/`) | arXiv/ar5iv MathML | none (PDF-only — lossy)
- **Equation fidelity:** exact (copied from .tex / MathML alttext) | lossy (PDF extraction — verify against paper.pdf)
- **Diagrams:** TikZ source in `src/` | image-only | none
- **Files:**
  - `alphaxiv.md` — readable full text (NOTES.md base)
  - `src/` — LaTeX source (exact math + TikZ), entry point `<main>.tex`
  - `metadata.atom` — arXiv API metadata
  - `arxiv-id.txt` — for re-fetching
```

### 8. Write NOTES.md

Target 100–250 lines. The primary artifact future sessions read.

````markdown
# <Paper title>

**arXiv:** <id> · **Authors:** <first author et al., year> · **Source:** see SOURCE.md

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
- `src/<main>.tex` — exact math + TikZ
- `SOURCE.md` — provenance
- `arxiv-id.txt` — for re-fetching
````

### 9. Update references/INDEX.md

If it exists, append/update under `## Papers`:

```
- `papers/<slug>/` — <one-line description>. Consult for: <when>.
```

Otherwise create a minimal one with `## Repos` and `## Papers` sections.

### 10. Report back

- Slug used, path written.
- **Which layers you obtained** (alphaXiv full text? LaTeX source? PDF-only?) and what that means for equation fidelity.
- One-sentence TL;DR.
- Suggest the user review `NOTES.md` and correct any mischaracterization.

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

## Why alphaXiv first, then LaTeX source

alphaXiv's hosted markdown is a single GET that returns clean, whole-paper text
already tuned for LLM reading — strictly better than throwing a 40-page PDF at the
model, and faster than walking arXiv's HTML. So it's the default readable layer.
But extracted markdown is not authoritative for *math*: displayed equations get
mangled and TikZ string diagrams vanish entirely. The arXiv LaTeX e-print is the
ground truth for both, so for any paper where the equations or diagrams matter —
category theory, signal-flow, anything Bonchi-flavoured — we pull the source and
copy the math verbatim. Two layers, two jobs: alphaXiv reads the paper, the source
proves it.
