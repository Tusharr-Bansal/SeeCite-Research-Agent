# 0009 — Demo corpus: my 29-paper MySteth literature review

**Date:** 2026-10-21 · **Status:** Accepted (replaces an earlier 5-paper plan)

## Context

The first plan was 5 hand-picked papers. Two problems:
- "RAG over 5 papers" looks like a tutorial.
- With only ~226 passages, almost anything relevant lands in the top 10, so recall@10 tells you little.

## Decision

Use all 29 papers from my literature review for MySteth 2.0 (pediatric heart-murmur screening). All are open access. Each paper has a key like `guo2017_calibration`, and its title, authors, year and URL are in `papers/manifest.json`. The PDFs aren't committed to git.

## Why it's a good test

- **Real questions from a real user.** I wrote this review, so the test questions come from actual domain knowledge.
- **Lots of overlap.** Many papers use the same dataset (CirCor), the same methods (MC Dropout, temperature scaling, self-supervised pretraining) and the same words. Search has to tell similar papers apart.
- **Mixed types:** benchmark papers, method papers, a 27-page survey, 4-page conference papers, a 2007 clinical paper, and calibration papers from other fields. This tests whether long papers crowd out short ones.

## Getting the papers

23 PDFs were ingested on 2026-10-21. Feng, Niizumi and Sondermann were added before the test set was built (26 papers). Wu, Alotaibi and Rodrigues were added on 2026-10-22, completing all **29** (see [0019](0019-single-search-default.md)).

## Consequences

- Long papers get more chunks. Balanya et al. (99 chunks) and Ismail et al. (83) together are ~20% of the corpus. If they crowd out short papers, per-paper recall will show it.
- The base MySteth paper (Jain et al. 2025) is included, so questions about it are fair.
