# Decision log

I wrote down each design decision when I made it: the situation, what I chose, what I rejected and why, and any numbers I measured. If a decision changes later, I add a new record instead of rewriting the old one, so you can see how my thinking changed.

| # | Decision | Status |
|---|----------|--------|
| [0001](0001-build-environment.md) | Where the code is built and run | Accepted |
| [0002](0002-llm-provider.md) | LLM: Gemini free tier, pinned to `gemini-3.5-flash-lite` | Accepted |
| [0003](0003-embedding-model.md) | Embeddings: local `bge-small-en-v1.5` (quantized ONNX) | Accepted |
| [0004](0004-database.md) | One Postgres database for vector and keyword search | Accepted (keyword ranking settled in 0011) |
| [0005](0005-secrets.md) | Handling the API key | Accepted |
| [0006](0006-documentation-format.md) | Decisions are documented in the repo | Accepted |
| [0007](0007-pdf-extraction.md) | PDF text with pdftotext; remove headers, references and number tables | Accepted |
| [0008](0008-chunking.md) | Chunks of whole sentences, ≤350 tokens, ~60-token overlap | Accepted |
| [0009](0009-corpus.md) | Demo corpus: my 29-paper MySteth literature review | Accepted |
| [0010](0010-gold-set-design.md) | Test set: LLM-drafted, checked by me, tied to evidence quotes | Accepted |
| [0011](0011-retrieval-method.md) | Search: hybrid vector + BM25; reranker tried and dropped | Accepted |
| [0012](0012-agent-loop.md) | Agent: a simple plan → search → check → answer loop | Accepted |
| [0013](0013-deployment.md) | Deployment safeguards; one container on Hugging Face | Hosting replaced by 0015 |
| [0014](0014-generalize-to-projects.md) | From a heart-sound tool to any field: projects and uploads | Accepted |
| [0015](0015-hosting-render-neon.md) | Hosting on Render + Neon, keep-alive ping, daily quota | Accepted |
| [0016](0016-search-the-original-question.md) | The agent always searches the question as typed | Accepted; changes 0012 |
| [0017](0017-agent-evaluation-results.md) | Agent vs single search: same accuracy at 1/3 the cost; 83% fully faithful; AI judge too generous | Final |
| [0018](0018-claim-level-judge.md) | A stricter AI judge, tested once: it did worse | Negative result |
| [0019](0019-single-search-default.md) | Single search is the default, agent becomes "Deep search"; all 29 papers in the demo | Accepted; changes 0012, 0015 |
| [0020](0020-rename-to-seecite.md) | Renamed from Citewise to SeeCite | Accepted |

**About old names:** records 0001–0013 were written when the project was the "PCG Research Agent" (package `pcg_agent`), and 0014–0019 when it was called **Citewise** (package `citewise`, settings `CITEWISE_*`). The code now lives in `src/seecite/` and settings are `SEECITE_*`.
