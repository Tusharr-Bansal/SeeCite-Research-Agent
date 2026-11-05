# 0012 — Agent: a simple plan → search → check → answer loop

**Date:** 2026-10-22 · **Status:** Accepted. Changed by [0016](0016-search-the-original-question.md) (always search the question as typed). Since [0019](0019-single-search-default.md), this is the optional "Deep search" mode.

## Context

A single search misses evidence in two ways (0011): right paper but wrong passage, and questions worded differently from the paper. My 13 multi-paper questions also need evidence from several papers. The agent searches repeatedly and answers only from what it found.

## Decision

`agent.py`, about 200 lines, no framework:

1. **Plan:** the LLM writes 1–3 search queries, one per part of the question, with abbreviations spelled out ("STFT short-time Fourier transform").
2. **Search:** hybrid search, 6 passages per query. Results from different queries are **interleaved by rank**, so every part of the question gets evidence within the 14-passage limit.
3. **Check:** the LLM decides whether the evidence is enough. If not, it writes new queries, and can **search inside one specific paper** when it can tell which paper should have the missing detail.
4. **Answer:** the LLM answers only from the passages, cites `[S#]` after each factual sentence, points out disagreements, and says whether the question is answerable (yes / partly / no). **Code** then checks that every citation refers to a passage it was actually shown.

**Limits:** at most 3 search rounds and 6 LLM calls per question. Every step is logged (queries, hits, tokens, time, cache).

## What the first live runs showed

| Problem | Fix | Result |
|---|---|---|
| Gemini wrote `[S3, S5, S7]`, but the parser only knew `[S3]`, so no citations were recorded | Parse grouped citations (tested) | Citations recorded and checked |
| A question about "STFT settings" failed because the paper only says "short-time Fourier transform". The agent honestly said "not found", and its check step even named the right paper, but it couldn't search inside it | Spell out abbreviations + paper-scoped search | Found and cited `festag2022:005` in round 3, settings correct |
| "Which papers use MC Dropout?" missed Bondareva, who writes "deviation of the predictions" | Not fixed. "Find every paper that…" is a known weak spot, and tuning the prompt to one question would overfit | Answer correctly says it's partial |

## Alternatives

- **LangGraph:** its strengths (checkpointing, human approval steps, resumable graphs) aren't needed for a 4-step loop that finishes in under 20 s. A plain loop is easier to read and test. It would map directly onto a LangGraph graph if I ever need those features.
- **Free-form tool calling:** more flexible, but harder to limit and evaluate. The fixed loop keeps the cost predictable (2–6 LLM calls).
- **One search, then answer:** the cheapest option, and the baseline the agent has to beat.

## Consequences

- 9–16 s and 8–15k input tokens per question. The free tier allows about one question every 15–20 s.
- The agent is only as good as what search finds. "Find all papers that…" is still hard.
