---
name: read-arxiv-paper
description: Fetch an arXiv paper using the best-available format (arXiv's official HTML rendering first, then ar5iv, then LaTeX source, then PDF), unpack it into the project's references/papers/slug/ directory, read it end-to-end, and write a structured NOTES.md summary. Use whenever the user gives an arXiv ID or URL, asks to "ingest", "read", "summarize", or "add" a paper, or wants to add a paper to their reference library.
---

# Read an arXiv paper into the project reference library

## When to trigger

Any of:

- User gives an arXiv ID (e.g. `2507.19457`, `1905.05033v2`) or URL (`arxiv.org/abs/...`, `arxiv.org/pdf/...`, `arxiv.org/html/...`, `ar5iv.labs.arxiv.org/...`, `alphaxiv.org/...`).
- User says "read this paper", "summarize this paper", "add this paper to references", "ingest this paper".
- User asks Claude to consult literature on a topic and the source is on arXiv.

## Goal

Produce, inside the project, a self-contained directory `references/papers/<slug>/` containing:

- The fetched paper content, in the best format obtainable
- A `NOTES.md` that is the _actual_ artifact you and future sessions will read
- An `arxiv-id.txt` so the paper can be re-fetched later
- A `SOURCE.md` recording which format you got and from where, so the next session knows whether equations can be trusted verbatim

## Format selection: HTML first

arXiv provides multiple ways to access the same paper. They differ in fidelity for LLM consumption. **Try them in this order:**

### 1. `https://arxiv.org/html/<id>` — preferred

arXiv's official HTML rendering (since 2023; formerly the ar5iv project, brought in-house). Single HTTP GET. Equations are MathML, structure is semantic HTML, no multi-file traversal. Covers most papers from late 2023 onward and an expanding backfill of the older corpus.

```bash
curl -L -sS -o paper.html "https://arxiv.org/html/<id>"
```

Then check whether the rendering actually succeeded. arXiv returns 200 with an error page for unsupported papers. Verify:

- File is >10KB
- Contains `<math` tags or `MathJax` (means equations rendered)
- Doesn't contain the string "No HTML for this paper" or similar

### 2. `https://ar5iv.labs.arxiv.org/html/<id>` — fallback for older papers

The original ar5iv project, still hosted, covers a different (older) slice of the corpus.

```bash
curl -L -sS -o paper.html "https://ar5iv.labs.arxiv.org/html/<id>"
```

Same success checks as above.

### 3. `https://arxiv.org/e-print/<id>` — LaTeX source fallback

When HTML rendering failed (rare — exotic macros, withdrawn papers, very old uploads):

```bash
curl -L -sS -o source.tar.gz "https://arxiv.org/e-print/<id>"
file source.tar.gz
```

- `gzip compressed` or `POSIX tar archive`: `tar -xzf source.tar.gz` (try `tar -xf` if gzip fails — some are bare tar). For big tarballs (>50MB), extract selectively:
  ```bash
  tar -xzf source.tar.gz --wildcards '*.tex' '*.bib' '*.bbl' 2>/dev/null
  ```
- `PDF document`: the author opted out of source. Skip to step 4.
- HTML: rate-limited or wrong ID. Stop and report.

Remove `source.tar.gz` after extraction.

Find the entry point: `grep -l '\\documentclass' *.tex 2>/dev/null`. The one with `\begin{document}` is root. Follow `\input` / `\include`.

### 4. PDF — last resort

```bash
curl -L -sS -o paper.pdf "https://arxiv.org/pdf/<id>"
pdftotext -layout paper.pdf paper.txt
```

Equations will be lossy. Flag this loudly in NOTES.md and SOURCE.md.

## Procedure

### 1. Normalize the arXiv ID

Accept any of:

- `2507.19457`
- `2507.19457v3`
- `https://arxiv.org/abs/2507.19457`
- `https://arxiv.org/pdf/2507.19457.pdf`
- `https://arxiv.org/html/2507.19457v2`
- `https://ar5iv.labs.arxiv.org/html/2507.19457`
- `https://www.alphaxiv.org/abs/2507.19457`
- old-style: `cs/0701001`

Strip to bare ID; default to latest version unless user specified one.

### 2. Pick a slug

Short, kebab-case, descriptive. Examples: `gepa`, `deflated-sharpe`, `cpcv`, `alphaevolve`.

- If the user suggested one, use it.
- Otherwise propose one based on title or known shorthand. Confirm if ambiguous.
- If `references/papers/<slug>/` exists, suffix `-v2`, `-2024`, etc., or ask.

### 3. Fetch metadata first

Use the arXiv API for clean title/authors/abstract — much easier than parsing them out of the document.

```bash
curl -sS "http://export.arxiv.org/api/query?id_list=<id>" -o metadata.atom
```

Parse the Atom XML for `<title>`, `<author><name>`, `<published>`, `<summary>`. If the `arxiv` Python package is available (`pip show arxiv`), use it:

```python
import arxiv
paper = next(arxiv.Client().results(arxiv.Search(id_list=["<id>"])))
# paper.title, paper.authors, paper.published, paper.summary, paper.categories
```

### 4. Fetch the paper (HTML cascade)

Walk the format cascade above. As soon as one succeeds, stop.

Record which format you ended up with — this matters for NOTES.md fidelity claims.

### 5. Create the directory

```bash
mkdir -p references/papers/<slug>
cd references/papers/<slug>
echo "<arxiv-id>" > arxiv-id.txt
```

Move the fetched file(s) in.

### 6. Read the paper

- **HTML:** read `paper.html` directly. The semantic tags (`<section>`, `<math>`, etc.) make navigation easy. For equations, copy the MathML blocks or the LaTeX in `alttext` attributes — both are present.
- **LaTeX:** read entry point, follow `\input`/`\include`.
- **PDF→text:** read `paper.txt`, but cross-check equations against `paper.pdf` visually if any look garbled.

For figures (`<img>` in HTML, `\includegraphics` in LaTeX), record caption and what the text says about each. Don't try to render images.

### 7. Write SOURCE.md

A one-screen file recording provenance:

```markdown
# Source provenance

- **arXiv ID:** <id> (version <v>)
- **Fetched:** YYYY-MM-DD
- **Format obtained:** HTML (arxiv.org/html) | HTML (ar5iv) | LaTeX source | PDF-extracted text
- **Equation fidelity:** exact (MathML/LaTeX) | exact (LaTeX source) | lossy (PDF extraction — verify against paper.pdf)
- **Files:**
  - `paper.html` — primary content
  - `metadata.atom` — arXiv API metadata
  - `arxiv-id.txt` — for re-fetching
```

### 8. Write NOTES.md

Target 100–250 lines. This is the primary artifact future sessions will read.

````markdown
# <Paper title>

**arXiv:** <id> · **Authors:** <first author et al., year> · **Source:** see SOURCE.md

## TL;DR

2–4 sentences. What problem, what method, what result.

## When to consult this paper

Bullet list of concrete situations in _this project_ where you'd want to re-read this.
Be specific to the user's codebase if you have that context.

## Key contributions

- ...

## Method

The substantive part. Explain the algorithm/approach clearly enough that you could
re-implement from this alone. Include the key equations _verbatim_ (LaTeX preferred,
copied from MathML alttext or .tex source — do not paraphrase math):

\```
$$ \hat{SR} = \frac{(SR - SR_0)\sqrt{N-1}}{\sqrt{1 - \gamma_3 SR + \frac{\gamma_4 - 1}{4} SR^2}} $$
\```

Include pseudocode if the paper has it.

## Results

What they measured, on what, with what numbers. Skip if purely theoretical.

## Assumptions & limitations

The fine print that will bite you if you forget it. Crucial for any paper used in
production code.

## Open questions / things to verify

Anything unclear, or that the project should empirically check before relying on.

## References to follow up

Other papers cited here that look relevant to the project. Just titles + arXiv IDs.

## File map

- `paper.html` (or `main.tex` etc.) — primary content
- `SOURCE.md` — provenance
- `arxiv-id.txt` — for re-fetching
````

### 9. Update references/INDEX.md

If `references/INDEX.md` exists, append/update an entry under `## Papers`:

```
- `papers/<slug>/` — <one-line description>. Consult for: <when>.
```

If it doesn't exist, create a minimal one with `## Repos` and `## Papers` sections.

### 10. Report back

Tell the user:

- Slug used, path written
- **Which format was obtained** (HTML / LaTeX / PDF) and why it matters for fidelity
- One-sentence TL;DR
- Suggest they review `NOTES.md` and edit if you mischaracterized anything

## Constraints

- Never overwrite an existing `references/papers/<slug>/` without confirmation.
- Never commit.
- If `references/` doesn't exist and you're not at an obvious project root, ask.
- Don't paraphrase equations — copy them.
- If the paper isn't on arXiv (DOI-only, journal-only), tell the user and offer to read a PDF they provide.

## Edge cases

- **Withdrawn papers:** all endpoints return an error page. Detect and report.
- **alphaXiv URLs:** strip to bare arXiv ID and use the cascade — alphaXiv is a viewer over arXiv content; the underlying source is the same.
- **Specific version requested:** the HTML endpoint accepts versioned IDs (`/html/2507.19457v2`).
- **Multi-file LaTeX with broken `\input` paths:** if extraction missed a referenced file, fall back one level in the cascade (LaTeX → PDF) rather than reading a partial paper.
- **Non-English papers:** rare; read what you can, flag in NOTES.md.

## Why this cascade rather than alphaxiv-open or similar

The `AsyncFuncAI/alphaxiv-open` project and similar tools are chat-over-paper services
(FastAPI + RAG + LLM). They add infrastructure without improving extraction fidelity —
most convert PDF to markdown, which loses equations. arXiv's own HTML endpoint already
does the high-fidelity conversion (it's the ar5iv pipeline, now first-party) and is a
single GET. No need for a separate service.
