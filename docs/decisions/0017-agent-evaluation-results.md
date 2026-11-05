# 0017 — Agent evaluation results

**Date:** 2026-10-22 · **Status:** Final

## Setup

- **Compared:** the agent (general prompts + the fix from 0016) vs a single hybrid search using the same answer prompt and evidence limit.
- **Questions:** 45 single-passage questions with a verified evidence quote, plus my 13 multi-paper questions.
- **Judge:** `gemma-4-26b-a4b-it`, a different model from the one answering (`gemini-3.5-flash-lite`), so it doesn't favour its own answers.
- **Metrics:** *Evidence* (did the right passage reach the answer step?), *Correct* (judge's verdict), *Credit* (correct 1, partial 0.5), *No unsupported claims* (judge's faithfulness verdict).

Scripts: `eval/eval_agent.py`, `eval/compare_runs.py` → `eval/results/comparison.json`.

## Results

| System | Evidence (45) | Correct (45) | Credit (45) | No unsupported claims (58) |
|---|---|---|---|---|
| Single search | 0.82 | 0.87 | 0.90 | 1.00 |
| **Agent (final)** | **0.89** | 0.87 | 0.91 | 0.98 |
| Agent before the 0016 fix (partial run) | 0.71 (41) | 0.78 (41) | 0.82 (41) | 1.00 (40) |
| Agent v1, heart-sound prompts | 0.82 | 0.89 (44) | 0.92 (44) | 0.98 (56) |

**Paired differences** (95% interval; if it includes 0, it could be noise):

| Comparison | Evidence | Credit |
|---|---|---|
| Agent − single search | +0.07 [−0.04, +0.18] | +0.01 [−0.09, +0.11] |
| Agent − agent before fix | **+0.17 [+0.05, +0.32]** | +0.10 [−0.01, +0.22] |
| Agent − agent v1 | +0.07 [+0.00, +0.16] | +0.00 [−0.03, +0.03] |

**Cost per question (median):**

| | LLM calls | Input tokens | Time |
|---|---|---|---|
| Single search | 1 | 5.1k | 4.0 s |
| Agent | 3 | 8.7k | 8.4 s |

On multi-paper questions, the agent cites 2.6 papers per answer vs 2.3 for single search. That's breadth, not necessarily quality.

## What it means

1. **The 0016 fix worked:** +17 points of evidence, the one clearly significant result.
2. **The agent doesn't beat a single search on this test set.** +7 points evidence and equal correctness, both within noise, at 3× the LLM calls and 2× the time. Hybrid search is already strong (it finds the passage 82% of the time), which leaves the agent little room. Single-passage questions are also where an agent should help least.
3. **The judge says faithfulness is near perfect (98–100%).** That only means something if the judge agrees with a human, so I checked (below).

## Checking the judge against my own review

**Sample:** 60 answers, 30 per system: every answer the judge didn't rate "supported", plus a random draw. Shown **blind** (no system name, no judge verdict). Scripts: `eval/make_faith_sample.py`, `eval/judge_agreement.py`.

**How I labelled:** a model drafted a verdict and a one-line reason for each answer, then I checked each one against the cited passages and set the final label. My rules:
- a claim not in the cited passage, stated as fact → at least Partly;
- a clearly marked interpretation is fine;
- naming which paper is meant is fine;
- a cited passage that *contradicts* a grouped claim → Partly;
- "No claims" only when the answer says nothing factual.

*Limitation:* reviewing drafts can nudge the reviewer toward the draft. Fully independent labelling would avoid that.

**The judge vs my labels:**

| Metric | AI judge |
|---|---|
| Accuracy (4 classes) | 0.82 |
| Accuracy if you always say "supported" | 0.77 |
| Clean answers it passed | **1.00** (50/50) |
| Its flags that were real problems | **1.00** (1/1) |
| Real problems it caught | **0.10** (1/10) |
| Cohen's κ (less meaningful here, since most answers are clean) | 0.41 [0.12, 0.67] |

| My label ↓ / Judge → | Supported | Partly | Unsupported | No claims |
|---|---|---|---|---|
| Supported | 44 | 0 | 0 | 2 |
| Partly | **9** | 1 | 0 | 0 |
| No claims | 0 | 0 | 0 | 4 |

**What this shows:**
1. **The judge is right when it flags something, but it rarely flags.** It caught only 1 of 10 problems. Its 82% accuracy is just 5 points above always saying "supported".
2. **The real errors are overstatements, not inventions.** None of the 60 answers was Unsupported. All 10 Partly answers have the same kind of subtle problem:
   - a hedge dropped ("may stem" → "primarily due to");
   - a quantifier widened ("some" → all);
   - a detail from the question presented as the paper's ("standard", "infant");
   - a method credited to the wrong paper;
   - two sentences merged into one wrong meaning.
3. **Real faithfulness is lower than the judge said:** 83% (50/60), not 98%.
4. **By system** (revealed after labelling):

   | | Clean | Partly |
   |---|---|---|
   | Single search | 0.90 | 3/30 |
   | Agent | 0.77 | 7/30 |

   The agent writes longer answers across more passages, which gives more room to overstate. With 30 each, this is a hint, not a proven difference.

## Consequences

- I don't trust the automatic faithfulness score on its own. The README reports my reviewed figure (83% fully faithful, 0% unsupported).
- I tried a stricter judge afterwards (0018). It did worse.
- Possible next steps: make single search the default, and write harder multi-step questions where an agent should matter. (Done in 0019: single search is now the default, and the prompt now asks to keep hedges.)
