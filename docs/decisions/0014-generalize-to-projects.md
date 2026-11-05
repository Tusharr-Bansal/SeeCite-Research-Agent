# 0014 — From a heart-sound tool to any field: projects and uploads

**Date:** 2026-10-22 · **Status:** Accepted

## Context

Up to 0013, everything was built for one fixed set of papers: my MySteth review. I decided any researcher should be able to **create a project, upload their own papers, and ask questions** in any field. Search, the agent and the evaluation stay the same; everything around them changes. I renamed the product **Citewise** (later SeeCite, see 0020).

## What changed

### Data
`projects` → `papers` (status: queued / processing / ready / failed) → `chunks` (now tagged with project and paper). Deleting a project deletes its papers and chunks. The setup script **never drops data** (the old one rebuilt tables on every run, which was fine for one corpus but would wipe user uploads).

### The demo project
The MySteth papers became the built-in, read-only project `demo-mysteth`, with the **same chunk IDs**, so the test set still works. I proved nothing changed: a hash over all chunk IDs and texts matches the old database exactly, and every retrieval result reproduces.

### Search per project
- **BM25 statistics are per project.** A word that's common in one field may be rare in another.
- **Vector search is exact within a project** (no approximate index). An approximate index searched across all projects and then filtered can return too few or wrong results. At a few thousand chunks per project, exact search takes milliseconds. Switching changed only 1 of 315 rankings in my tests.

### Uploads
- One background worker processes PDFs one at a time (embedding is CPU-heavy).
- Checks: real PDF, max 25 MB, max 40 papers per project. **Scanned PDFs** (under 1,500 characters of text) are rejected with an explanation.
- **Title, first author and year** come from one LLM call on the first page, with the filename as fallback. **Links** come from an arXiv ID or DOI on the first two pages.
- The PDF is deleted after processing; only the text chunks are kept.
- After a restart, half-finished uploads are marked failed ("please upload again") instead of hanging.

**End-to-end test:** a new project with Guo 2017, Balanya 2024 and a fake PDF. The fake was rejected, metadata was correct, and a question was answered citing only that project's papers. After removing Balanya, the same question correctly got "not found".

### Prompts
The prompts no longer mention heart sounds. They include the project's name and the user's description, so field-specific vocabulary still helps.

### Access: the link is the key
No accounts. Each project ID is random (~96 bits), so it can't be guessed. Having the link gives access, like an "anyone with the link" Google Doc. The browser remembers your projects, and the page warns not to upload confidential papers.

### Limits
Per visitor per day: 10 projects, 120 uploads, 30 questions (5/minute). Overall: 300 questions/day and 300 projects.

## Consequences

- Each request opens its own database connection, so one request's queries can't end up inside another's transaction.
- The evaluation had to be re-run with the new prompts (results in 0017).

## Later: upload progress bar

On the free host a paper takes minutes, and "Processing" alone looked frozen. The bar has two stages:
1. **Uploading:** real upload progress (using `XMLHttpRequest`, since `fetch()` can't report it).
2. **Processing:** each paper saves its current stage and progress, which the page checks every 3 s: reading the PDF 2%, reading the title 5%, embedding passages 10% → 95% (most of the time), saving 97%, ready 100%. Embedding runs in batches of 8 so the bar keeps moving.

**Bug I shipped and fixed:** I first named the bar's CSS class `.bar`, but the page header already used `class="bar"`, so the header turned into a squashed pill on the live site. My browser test had only checked the papers section. I renamed it `.progress-track`, and the test now screenshots every page and checks the header's height. **Lesson:** after a CSS change, check the whole page, not just the part you changed.

New columns are added with `ALTER TABLE … ADD COLUMN IF NOT EXISTS`, because `CREATE TABLE IF NOT EXISTS` never adds columns to an existing table.

## Later: projects are deleted after 7 days

Every non-demo project is deleted **7 days after it's created** (setting: `SEECITE_PROJECT_TTL_DAYS`). This keeps the database within Neon's free 1 GB and limits how long uploads are kept.

- **Hidden immediately:** an expired project returns 404 from every endpoint, whether or not it's been deleted yet.
- **Deleted lazily:** at startup, then at most once an hour, *only during a request that already uses the database*. A separate hourly timer would wake Neon 24 times a day and waste its free compute hours (same idea as the database-free ping in 0015).
- **Users are told** on the create form, the home page and each project page, and the 404 message explains why a link stopped working.

**Tested:** a project backdated to 8 days returned 404 immediately, and after a restart it was fully deleted. A 6-day-23-hour project and the demo were untouched.

**Bug the tests caught:** the "last cleanup" time started at `0.0`, but `time.monotonic()` counts from boot, so on a recently booted host "0" looked recent and the first cleanup would be skipped. Fixed by starting at `-inf`.
