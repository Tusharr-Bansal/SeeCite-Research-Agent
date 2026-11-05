# 0016 — The agent always searches the question as typed

**Date:** 2026-10-22 · **Status:** Accepted · Changes 0012

## Context

After I made the prompts general (0014), I re-ran the evaluation and the agent got worse. The run was cut short by the daily quota, so these are partial numbers (~40 questions each):

| | Correct | Evidence reached the answer |
|---|---|---|
| Single search (question as typed) | 0.88 | 0.81 |
| Agent, new general prompts | **0.78** | **0.72** |
| Agent, old heart-sound prompts | 0.89 | 0.82 |

5 answers went from correct to incorrect, and 1 the other way. In 4 of those 5, the right passage never reached the answer step.

**Cause, from the search logs:** the agent only searched the queries its planner wrote. For "How can S1 and S2 be told apart using timing alone?":
- the old planner wrote *"differentiating S1 and S2 using timing alone"*, close to the question, and found the answer;
- the new planner wrote *"systolic time interval S1 and S2 differentiation timing heart sound phonocardiogram"*, stuffed with keywords, and missed it.

The plain question found the passage. The agent was throwing away its best query. This is a design flaw, not a prompt bug: any planner can rewrite a question worse. Tweaking the prompt until the numbers recover would just hide it.

## Decision

In the first round, the agent always searches the question **exactly as typed**, alongside its planned queries. Those results are **guaranteed half the evidence slots** (7 of 14, placed first). The planned queries fill the rest.

- No extra LLM calls (searching is local).
- The agent's own queries can add evidence but can't push out the plain search's top 7.
- The check step knows the question was already searched, so it won't suggest it again.

**Why half (7):** it's a principle. Half for the user's wording, half for the agent's exploring. To be transparent: I did look at where the right passage ranks in the plain search (rank 1–7 in 31 of 34 cases, 9–14 in 3). Picking a cutoff from test-set ranks would be tuning on the test set, so I kept 7 instead of raising it.

## Alternatives

- **Go back to the heart-sound prompt:** not possible now that the product is general.
- **Tell the planner to stay close to the user's wording:** asks the model to do something code can guarantee.
- **Give the plain search all the slots:** then the agent can't add anything.
- **Skip planning for simple questions:** needs a way to detect "simple", which is a separate decision.

## Evidence

- Unit test: `test_anchored_keeps_the_original_query_results_first_and_whole`.
- Forcing the planner to return the bad query, the S1/S2 question now gets the right passage, with the same 3 LLM calls.
- The full re-run is in 0017. The pre-fix results are saved in `eval/results/agent_eval_v2a_before_anchor.json` for a fair before/after comparison.
