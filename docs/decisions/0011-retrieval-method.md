# 0011 — Search: hybrid of vector search and BM25

**Date:** 2026-10-22 · **Status:** Accepted (settles the open questions in 0003 and 0004)

## What I compared

I built five search methods (`retrieval.py`) and tested them on the 45 checked single-passage questions from 0010 (21 of them reworded by me, 10 ambiguous). A question counts as found at k if a chunk containing its evidence quote is in the top k.

| Method | R@1 | R@5 | **R@10** | MRR@10 | R@10 on my rewordings (21) | Median time |
|---|---|---|---|---|---|---|
| vector (bge prefix) | 0.33 | 0.49 | 0.64 | 0.41 | 0.52 | 90 ms |
| vector (no prefix) | 0.27 | 0.44 | 0.56 | 0.35 | 0.48 | 84 ms |
| Postgres `ts_rank_cd` | 0.11 | 0.38 | 0.49 | 0.21 | 0.29 | 22 ms |
| BM25 | 0.36 | 0.64 | 0.76 | 0.48 | 0.57 | 2 ms |
| hybrid: vector + ts_rank | 0.27 | 0.56 | 0.69 | 0.40 | 0.57 | 106 ms |
| **hybrid: vector + BM25** | **0.47** | 0.62 | **0.78** | **0.54** | 0.57 | 87 ms |
| hybrid: vector + BM25, no prefix | 0.42 | 0.56 | 0.73 | 0.49 | 0.48 | 74 ms |

**Are the differences real?** Paired bootstrap on recall@10 (5,000 resamples):

| Comparison | Difference | p | Real? |
|---|---|---|---|
| hybrid vs vector | +0.13 | 0.002 | yes |
| BM25 vs ts_rank_cd | +0.27 | 0.001 | yes |
| vector with vs without prefix | +0.09 | 0.013 | yes |
| hybrid vs BM25 alone | +0.02 | 0.43 | **no** |

95% confidence interval for hybrid R@10: 0.64–0.89 overall, **0.33–0.76 on my rewordings**.

## Decision

Default search = **hybrid**: bge vector search (with its query prefix) plus BM25, merged with reciprocal rank fusion (k = 60, 50 candidates from each).

**Why hybrid, if it isn't clearly better than BM25?** It has the best score on every main metric and is never worse than either part. BM25 needs shared words; vectors handle rewording. Real users don't copy the paper's wording, so the vector half is insurance. It only adds ~85 ms, tiny next to an LLM call.

## Biggest finding: LLM-written questions make search look too good

| Questions | Word overlap with evidence | Hybrid R@10 | BM25 R@10 |
|---|---|---|---|
| LLM drafts kept as written (24) | 0.32 | **0.96** | 0.92 |
| Drafts I reworded (21) | 0.18 | **0.57** | 0.57 |

On the LLM's wording, search looks almost perfect. On the same facts asked naturally, it finds the evidence about half the time. **An unchecked synthetic test set would have overstated recall by ~40 points.** My reworded set is the honest estimate.

## Why the 10 misses happened

- **7: right paper, wrong passage.** The evidence was at rank 11–38. That's a ranking problem, the classic job for a reranker.
- **3: general questions** ("standard HMM-based segmentation") answered from other papers. These may be fine answers that the strict metric counts as misses.
- **2 not in the top 50 at all:** reworded questions that share few words with the evidence. Candidates for the agent's query rewriting.

## Consequences

- `ts_rank_cd` is dropped. BM25 runs in Python over ~1,000 chunks in ~2 ms. At 100k+ chunks it would need to move into the database or a search engine.
- With 45 questions, I can't detect differences under ~10 points. Any future "improvement" must beat that.

## Reranker experiment: rejected

Since 7 of 10 misses were ranking problems, I tried cross-encoder rerankers on the top 50 hybrid results (small models that fit a free host, CPU only):

| Reranker | R@1 | R@5 | R@10 | MRR | R@10 on rewordings | Extra time |
|---|---|---|---|---|---|---|
| none | 0.47 | 0.62 | 0.78 | 0.54 | 0.57 | 0 |
| ms-marco-MiniLM-L-6-v2 (80 MB) | 0.44 | 0.67 | 0.76 | 0.54 | 0.67 | +3.4 s |
| ms-marco-MiniLM-L-12-v2 (120 MB) | 0.47 | 0.69 | 0.78 | 0.56 | 0.62 | +6.6 s |

Larger rerankers didn't finish in time on this hardware.

**No reranker.** Overall recall didn't change, and the gain on rewordings is 1–2 questions, within noise. The cost is 3–7 s per search, and the agent can run up to 6 searches. These models were trained on web searches, not science papers. The "right paper, wrong passage" problem is now handled more cheaply by the agent's paper-scoped search (0012).
