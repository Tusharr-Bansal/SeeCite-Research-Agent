# 0008 — Chunking: whole sentences, ≤350 tokens, ~60-token overlap

**Date:** 2026-10-21 · **Status:** Accepted

## Context

Search returns chunks, not whole papers.
- **Chunks too big:** they mix topics, the embedding gets blurry, and the LLM reads lots of irrelevant text.
- **Chunks too small:** facts get separated from their context, and citations show fragments.

There's also a hard limit: `bge-small-en-v1.5` reads at most **512 tokens** and **silently cuts off** the rest.

## Decision

- **Count size in the model's own tokens,** not words. Scientific words like "phonocardiogram" are several tokens each.
- **At most 350 tokens per chunk,** safely under 512.
- **Split on sentence boundaries** so citations read as complete statements. The splitter knows abbreviations like "et al.", "Fig.", "e.g." and "vs." Over-long "sentences" (usually flattened tables) are split by words.
- **~60 tokens of overlap** between neighbouring chunks, so a fact on a boundary appears whole in at least one of them.
- **Track page ranges** for page-level citations.
- **Stable IDs** like `guo2017_calibration:012`.

The 5-paper pilot gave 226 chunks, median 306 tokens, max 334, **none over 512**. Unit tests in `tests/test_chunking.py` cover abbreviations, the size limit, overlap, no lost sentences, and splitting over-long text.

## Alternatives

- **Fixed-size windows:** simplest, but cut sentences in half.
- **Split by section headings:** detecting headings in PDF text is unreliable across 29 layouts.
- **Semantic chunking:** costs an embedding per sentence, with mixed results in published comparisons.
- **Search small chunks, return the bigger section:** a known upgrade if answers lack context. Not needed yet.

## Why test labels aren't tied to chunks

Each test question is tied to an **evidence quote**. A retrieved chunk counts as correct if it contains that quote. If labels were chunk IDs, changing the chunk size would break the whole test set, and I could never tune chunk size fairly.
