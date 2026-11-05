# 0010 — Test set: drafted by an LLM, checked by me, tied to evidence quotes

**Date:** 2026-10-22 · **Status:** Accepted

## Context

To measure search quality I need questions where I know where the answer is. Writing ~50 by hand takes hours. Trusting an LLM to write them makes the test as unreliable as the system it's testing.

## Decision

1. **The LLM drafts, I check.** For each paper, 2 substantial passages are sampled (≥200 tokens, mostly text, not page 1) with a fixed random seed per paper. Gemini drafts a question, an answer and an exact **evidence quote** for each. 52 passages → 51 drafts.
2. **I review every draft** in a private labelling page (`eval/labeling/index.html`): **Good / Fix / Drop**, plus an **Ambiguous** flag when several papers could answer it.
3. **I write my own questions** without looking at any passage.
4. **A retrieved chunk counts as a hit if it contains the evidence quote** (see 0008).
5. **Results are reported separately** for drafted vs my own questions, and clear vs ambiguous ones.

## Biases and how I handled them

| Bias | Why it happens | What I did |
|---|---|---|
| **Copying words** | A question written while looking at a passage reuses its words, which flatters keyword search | Asked for paraphrasing, measured word overlap (median 0.33, max 0.82), rewrote copy-paste questions, and wrote my own as a separate set |
| **Assuming one answer** | Many papers describe the same dataset or method | "Ambiguous" flag; scored separately |
| **Only single-passage questions** | Drafting from one chunk can't produce cross-paper questions | My own questions cover multi-paper ones |
| **Wrong quotes** | The LLM sometimes changes text when "copying" | Every quote is checked against its passage; 3 of 51 failed |

## Review results (2026-10-22)

Of 51 drafts: **24 kept, 21 rewritten, 6 dropped**. 10 of the 45 kept are ambiguous. I also wrote **13 multi-paper questions**.

Why I rewrote or dropped drafts:
- **Vague subject** (most common): "the model" or "the framework" when asking for one paper's number. I named the method (EHST, HnLTS, FunnelNet…) without naming the paper.
- **Copying the passage,** or the opposite, **over-paraphrasing** ("a self-supervised speech representation architecture" instead of "wav2vec 2.0").
- **Reverse lookups:** describing the answer and asking for its name.
- **Summaries of other people's work** (related-work sections), where the real source isn't in the corpus. Dropped.
- **Wrong answers from broken tables** (the ChildVox table, where half the cells survived extraction). Dropped.
- **Inflated totals** ("594 features" when 594 was one group of several). Rewritten.
- **A duplicate** from two overlapping neighbouring chunks. The sampler now skips neighbours.

**Lesson:** 53% of the LLM's questions needed my changes. Unchecked, they would have overstated recall by about 40 points (see 0011).
