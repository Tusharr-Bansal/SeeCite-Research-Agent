# 0020 — The product is renamed from Citewise to SeeCite

**Date:** 2026-10-27 · **Status:** Accepted

## Context

Tushar wanted a new name and proposed two: **KeyCite** and **SeeCite**. Before choosing, a web search checked whether either is already in use (a quick check, not a formal trademark search):

- **KeyCite** is Thomson Reuters' citation-checking service in Westlaw: a trademarked name, in the same field (citations and research), and taught by name in law schools. Using it would be a trademark risk, and search results for the name would be all Westlaw. **Rejected.**
- **SeeCite:** no product found under this exact name. Two near-matches:
  - it sounds like **SciCite**, AllenAI's citation-intent dataset (Cohan et al., NAACL 2019);
  - it's close to **scite.ai**, an AI tool for exploring citations.

  Acceptable for a portfolio project, and the name says what the product does: you *see* the passage behind every *citation*. It's written with a capital C (**SeeCite**) so it doesn't read as "SciCite".

## Decision

Everything that is part of the product now says **SeeCite**:

- the Python package `src/citewise/` → `src/seecite/` (imports, `uvicorn seecite.app:app`, `python -m seecite.ingest`);
- every setting `CITEWISE_*` → `SEECITE_*` (for example `SEECITE_GLOBAL_CALLS_PER_DAY`, `SEECITE_EMBED_THREADS`). The Render service sets only `DATABASE_URL` and `GEMINI_API_KEY`, so nothing on Render needs changing;
- the page title, logo and text; the README and study guide; the evaluation and labelling scripts; the Dockerfile; `pyproject.toml`;
- the default local database name in `config.py`, `.env.example` and `deploy/export_demo.sh` (`.../citewise` → `.../seecite`).

Kept on purpose:

| Kept | Why |
|---|---|
| The browser's saved project list | Read from the old key `citewise.projects` if the new `seecite.projects` is empty, so returning visitors keep their projects. |
| The Render service name `citewise` in `render.yaml` | If the service is managed by a Blueprint, a new name there could create a *second* service rather than rename this one. |
| The live URL `citewise-3i23.onrender.com` | Render doesn't change a service's onrender.com address when the service is renamed. A new address needs a new service or a custom domain. |
| The GitHub repository `Citewise-Research-Agent` and links to it | Renaming the repository is Tushar's call; GitHub redirects the old links after a rename. |
| Decision records 0014–0019 | Written when the product was Citewise; left as written, like 0001–0013 (see the index). |
| Evaluation results (`eval/results/`) | Measurements; nothing in them depends on the name. |

## Be ready to explain

- Why the name was checked before it was adopted: a name collision with a product in the same field is a trademark risk, and it buries you in search results.
- What a rename touches beyond the visible name (package, settings, saved browser state, deployment identity, links), and which parts were kept on purpose.
