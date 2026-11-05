# 0019 — Single search is the default; the agent becomes "Deep search"

**Date:** 2026-10-22 · **Status:** Accepted. Changes 0012 (agent no longer the default) and 0015 (daily budget counted in LLM calls).

## Context

From 0017:
- Both single search and the agent got **0.87 correct**.
- The agent's +7 points of evidence could be noise.
- The agent costs 3 LLM calls and ~8 s per question, vs 1 call and ~4 s.
- In my faithfulness review, single search was at least as good (90% vs 77% fully faithful, small sample).

On the free tier, the real limit is 500 LLM requests a day, and an agent question uses three. The review also showed that **every** faithfulness problem was an overstatement (a dropped hedge, a widened quantifier, an added detail), never an invented fact. And three papers from my review were still missing from the demo.

## 1. Single search by default, agent on request

The ask endpoint takes `mode: "fast" | "deep"`, default `"fast"`:
- **fast:** one hybrid search on the question as typed, then one answer call.
- **deep:** the plan → search → check → answer agent (0012 + 0016).

The page has a **Deep search** checkbox, off by default.

**Rejected:** running the agent only when the single search looks weak. Knowing it's weak needs the check step, which costs an LLM call, so you lose most of the savings. Worth revisiting with a bigger corpus.

## 2. Daily budget counted in LLM calls

The old cap was 120 questions/day, assuming ~3 calls each. Now it's **400 calls/day** (`SEECITE_GLOBAL_CALLS_PER_DAY`): fast questions cost 1, deep ones 3. The other ~100 calls are left for reading paper titles on upload. That allows up to ~400 fast questions a day instead of 120. Per-visitor limits (5/minute, 30/day) still count questions.

The rate limiter is now **all-or-nothing** across rules: a request rejected by the per-minute limit doesn't use up the daily budget.

## 3. The answer prompt keeps the source's wording strength

Added to the answer prompt (both modes):

> Keep the strength of what the passages say. Copy their hedges and quantifiers ("may", "likely", "some", "primarily") instead of strengthening them, and do not add details (a number, scope, dataset, purpose) that the cited passage does not state. Cite each passage only for what it actually says.

## 4. The demo now has all 29 papers

| Paper | Chunks |
|---|---|
| Wu et al. 2024, murmur detection and recording quality (MDPI *Applied Sciences*) | 57 |
| Alotaibi et al. 2025, Whisper for heart sounds (*Connection Science*) | 21 |
| Rodrigues et al. 2026, detecting poor-quality recordings (SSRN preprint) | 26 |

Total: 1,089 chunks (was 985). I checked that all 985 old chunks are unchanged (same text, same embeddings). New papers can only affect *ranking*.

**Small leftover:** the Rodrigues PDF has a rotated "Preprint not peer reviewed" watermark that comes out as stray fragments ("ev ie", "we d") in 17 of its 26 chunks. Harmless for search; not worth special handling.

## 5. The demo data is versioned

Before, the demo loaded only if it was missing, so the live site would have kept the 26-paper version forever. Now:
- The database stores a hash of the demo dump file.
- On start, if the hash differs, the demo is replaced in **one transaction**. User projects are never touched.

The first test caught a bug: pg_dump output clears `search_path`, so table names must be written as `public.projects`.

| Test | Result |
|---|---|
| Empty database | loaded: 29 papers, 1,089 chunks |
| Restart with same dump | no reload |
| Old version + a user project | demo replaced, user project kept |
| Broken dump file | startup fails, old demo stays intact |

**The export script now refuses bad exports.** An earlier run without the right permissions had silently written an empty 4 KB dump. `deploy/export_demo.sh` now aborts if the dump has fewer than 100 chunks, and only replaces the file on success.

## Results with the new default

Single search with the new prompt on 29 papers (v4), vs single search from 0017 (v3: old prompt, 26 papers). Saved in `eval/results/agent_eval_v4_fast.json`.

| Single-passage questions | v3 | **v4 (default)** | Difference [95% CI] |
|---|---|---|---|
| Evidence reached the answer | 0.82 (45) | 0.80 (45) | −0.02 [−0.07, 0.00] |
| Correct | 0.87 (45) | 0.91 (44) | |
| Credit (partial = ½) | 0.90 (45) | 0.93 (44) | +0.01 [−0.06, +0.08] |
| No unsupported claim (judge) | 1.00 (58) | 0.98 (56) | −0.02 [−0.05, 0.00] |
| Invalid citations | 0 | 0 | |
| LLM calls / time | 1 / 4.0 s | 1 / 4.0 s | |

(The judge failed on 2 questions in v4. The prompt and the new papers changed together, so their effects are mixed.)

**Nothing got clearly better or worse.** Individual changes:
- **One question lost its evidence to the new papers.** The right passage had been 14th of 14; two passages from the new papers pushed it out, so the answer now honestly says it's not found. That's the expected cost of a bigger corpus with a fixed limit.
- **Two Festag questions improved.**
- **The Noponen answer got worse** (see below).

### Faithfulness on the answers that motivated the prompt

I re-checked the three single-search answers I'd marked *Partly* in 0017, plus any answer whose judge verdict changed. These are the questions that motivated the prompt change, so this is a sanity check, not a new measurement.

| Question | Before | Problem | After |
|---|---|---|---|
| Niizumi: ICBHI2017 vs SPRS | Partly | "may" → "primarily"; "some" → all | **Supported**: keeps "may" and "likely" |
| Despotovic: Open Heart collection | Partly | wrong purposes | **Supported** |
| Khan: "standard" HMM segmentation | Partly | "standard" and four states not in passage | **Supported**: says the passages don't describe a standard method |
| Noponen: duration/frequency rule | Supported | — | **Partly**: swapped 90% sensitivity and 91% specificity |

All three hedge problems were fixed, but one new error appeared: swapped numbers. The prompt doesn't target that, and citation checks can't catch it (the citation is valid). The AI judge did catch it. With only 4 answers, I can't claim a new faithfulness rate, so the 83% from 0017 stands until a fresh blind review.

## Consequences

- About 3× more questions fit in the free daily quota, and answers come in ~4 s.
- The agent's code, tests and evaluation stay. Deep search is one checkbox away if a bigger corpus needs it.
- Every new demo dump deploys itself, and a bad dump fails loudly instead of quietly emptying the demo.
- The README and the demo's results panel now show the default mode's numbers. Retrieval on 29 papers: hybrid recall@10 unchanged at **0.78**, MRR 0.54 → 0.53, vector-only 0.64.
