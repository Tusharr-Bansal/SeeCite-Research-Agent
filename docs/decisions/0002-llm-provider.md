# 0002 — LLM: Gemini free tier, pinned to `gemini-3.5-flash-lite`

**Date:** 2026-10-21 · **Status:** Accepted

## Context

I need an LLM to draft test questions, write search queries, write cited answers and judge faithfulness. Budget: free, or at most ~$15.

- **Claude API:** my Claude credit doesn't cover API calls, so it would cost extra.
- **Grok:** not reachable from my workspace.
- **Gemini:** has a free tier and was reachable.

## Decision

Gemini free tier, pinned to an exact model: `gemini-3.5-flash-lite`.

When I tested (2026-10-21), the 2.5 models returned 404 (not available to new users), and the larger Flash models returned 503 (overloaded). `gemini-flash-lite-latest` worked, but it's an alias.

**Why pin instead of using `-latest`:** Google can point the alias at a new model at any time. If that happens between two evaluation runs, the numbers change for reasons unrelated to my code. Pinning means any change in results is caused by something I did. I also log the model version returned with every response.

## Alternatives

- **Bigger Flash models:** overloaded during testing. The model name is a setting, so I can swap and compare later.
- **Claude API:** costs money here. The LLM code sits behind one small interface, so switching providers wouldn't touch the agent.

## Consequences

- **Rate limits:** the client needs throttling, retries with backoff for 429/503, and a response cache so re-running an evaluation doesn't use up quota.
- **Data use:** on the free tier Google may use prompts to improve its products. Fine here, because the papers are open access. Nothing private goes through this key.
- **Judge bias:** a judge from the same model family tends to favour its own answers. My hand labels measure how much the judge agrees with a human.
