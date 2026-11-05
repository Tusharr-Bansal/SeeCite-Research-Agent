# 0020 — Renamed from Citewise to SeeCite

**Date:** 2026-10-27 · **Status:** Accepted

## Context

I wanted a new name and had two ideas: **KeyCite** and **SeeCite**. I did a quick web search on both (not a formal trademark search):

- **KeyCite** is Thomson Reuters' citation checker in Westlaw: a trademark in the same field. Rejected.
- **SeeCite:** no product with this exact name. It sounds like **SciCite** (an AllenAI dataset) and is close to **scite.ai**, but that's fine for a portfolio project. The name says what it does: you *see* the passage behind every *citation*. The capital C (**SeeCite**) keeps it from reading as "SciCite".

## What changed

- Python package `src/citewise/` → `src/seecite/`.
- Settings `CITEWISE_*` → `SEECITE_*` (e.g. `SEECITE_GLOBAL_CALLS_PER_DAY`). Render only sets `DATABASE_URL` and `GEMINI_API_KEY`, so nothing there needed changing.
- Page title, logo and text; README; evaluation scripts; Dockerfile; `pyproject.toml`.
- Default local database name `citewise` → `seecite`.

## Kept on purpose

- **The browser's saved project list:** if the new `seecite.projects` key is empty, it reads the old `citewise.projects`, so returning visitors keep their projects.
- **Decision records 0014–0019** still say Citewise, because that was the name when I wrote them.
- **Evaluation results** don't depend on the name.

**Update:** the Render service, the live URL (now https://seecite.onrender.com) and the GitHub repo (`SeeCite-Research-Agent`) were renamed too.
