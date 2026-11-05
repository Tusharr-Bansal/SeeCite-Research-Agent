# 0018 — A stricter AI judge, tested once: it did worse

**Date:** 2026-10-22 · **Status:** Done. **Negative result:** κ dropped from 0.41 to 0.20.

## Context

In 0017 the AI judge caught only **1 of 10** answers with a faithfulness problem, and every miss was an overstatement. The old judge read each answer as a whole and gave one verdict, which is exactly where small overstatements slip through. I wanted one attempt at a stricter judge, tested on the **same 60 labels** (no new labelling needed).

## Rules I set before running it

The risk is tweaking the judge until it agrees with these 60 labels, which would measure overfitting, not quality. So:
- **One method, decided in advance, run once.** No prompt changes after seeing results.
- **Same judge model** (`gemma-4-26b-a4b-it`, temperature 0) and same inputs. Only the method changes.

## The method (`eval/judge_v2.py`)

1. **Split the answer into individual claims.** "Information not found" statements and naming which paper is meant don't count as claims.
2. **For each claim:** the judge gives an exact quote from the cited passages that supports it, and a label:
   - `supported`: the quote says it, at the same strength (keeping hedges like *may* and quantifiers like *some*) and with every detail;
   - `overstated`: the passage only supports a weaker or narrower version, or the claim adds a detail;
   - `wrong_source`: the cited passage doesn't support it, or contradicts it;
   - `unsupported`: nothing supports it.
3. **Quote check in code:** a `supported` claim's quote must actually be in the passages (fuzzy match, ≥85%). Otherwise it counts as only partly supported.
4. **The final verdict is computed in code:** no claims → `no_claims`; all supported → `supported`; more than half unsupported → `unsupported`; anything else → `partly`.

**One rule fixed before the first run:** my first draft would have labelled a one-claim answer with a small overstatement `unsupported`, which contradicts my labelling rubric ("unsupported" means the main claims aren't backed). I corrected it before any judge output existed.

**Fixes made while running** (to how the judge runs, not how it judges):
- The judge sometimes added stray text after its JSON. The parser now reads the first JSON object only.
- On one answer the judge kept generating until the server timed out. I capped output at 2048 tokens and re-ran all 60 under that cap. Dropped connections now retry.
- The run hit the per-minute token limit, and retries ignored the server's "retry in 25s". `llm.py` now waits as long as the server asks (tested in `tests/test_llm.py`).

Answers the judge still can't handle are recorded as `judge_failed`, never given a guessed label.

**Known limitation:** my rubric categories came from labelling these same 60 answers, so results on them will look better than reality. A proper check needs a held-out set.

## Results

The judge failed on 1 of 60 answers, so the comparison uses the other **59** (the old judge is re-scored on the same 59).

| Judge (same model, same 59) | Cohen's κ [95% CI] | Accuracy | Binary κ | Problems caught | False alarms |
|---|---|---|---|---|---|
| Old: one verdict per answer | **0.41** [0.12, 0.67] | 0.81 | 0.16 | 1 / 10 | 0 / 49 |
| **New: per claim + quote check** | **0.20** [−0.03, 0.42] | 0.73 | −0.05 | 1 / 10 | 7 / 49 |
| New, without the quote check | 0.28 [0.06, 0.51] | 0.78 | 0.02 | 1 / 10 | 4 / 49 |

**The new method made the judge worse.** It still caught only 1 of 10 problems and added false alarms.

## Why it failed (from the judge's own output; nothing re-run)

1. **It finds the right evidence but doesn't notice the claim is stronger.** In all 9 misses, the judge quoted the exact sentence containing the hedge, then still said `supported`. For example:
   - claim "differs **primarily due to** noise", quote "the difference… **may stem** from data";
   - claim "**specifically trained on infant**…", quote "trained **primarily** on child-centered";
   - a claim citing S1 and S2, where it only checked S1 and missed that S2 contradicts it.

   My instructions explicitly named hedges and quantifiers, so this is a limit of the judge model (a 26B mixture-of-experts with ~4B active parameters), not of the prompt.
2. **False alarms came from two mechanical issues:**
   - The judge turned "the passages don't mention X" into a claim (against instructions), so 3 of 4 "no claims" answers became "unsupported".
   - The quote check failed on 3 claims from garbled PDF tables, where the quote was reordered.

## Decision

- **Keep the old judge and its κ = 0.41.** The README and resume numbers don't change (83% faithful by my review).
- **No more changes based on these 60 labels.** The two mechanical fixes are obvious, but making them now and re-scoring would be tuning to the test set. They need a held-out set.
- **What would probably work:** a stronger judge model, since every miss was a reasoning failure on evidence the judge had already found. On the free tier, the stronger Gemini models allow ~20 requests a day, so this means a paid key (a few cents) or three days of quota.

## Takeaways

- **Test the evaluator, not just the system.** The judge looked fine at 98% until I checked it against my labels.
- **Deciding the method in advance and running it once** is what makes a negative result trustworthy.
- **Small models find the right evidence but compare meaning poorly:** they spot the sentence but miss that "may" became "primarily".
