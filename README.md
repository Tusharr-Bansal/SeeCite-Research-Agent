# SeeCite

**Ask questions about your research papers and get answers that cite the exact passage.**

You create a project, upload the PDFs from your literature review (any field), and ask questions. SeeCite searches the papers using keyword search (BM25) and meaning-based search (vectors in Postgres/pgvector) together, then writes an answer with numbered citations. The code checks that every citation points to a real passage.

By default it runs one search per question, which is fast and uses one LLM call. You can switch on **Deep search**, where an agent plans several searches, checks whether it has enough evidence, and searches again if it doesn't (even inside one specific paper).

There's a built-in demo project with the 29 open-access papers from my MySteth literature review on pediatric heart-sound screening.

**Live demo:** https://seecite.onrender.com

## Results

I tested it on the demo papers with questions I checked by hand: 45 questions answered by a single passage, and 13 that need several papers. Each question is tied to an evidence *quote* rather than a chunk ID, so the test set still works if I change how papers are split.

- **No made-up facts.** I reviewed 60 answers blind. None contained a claim its sources didn't support. **83% were fully faithful**; the rest had small overstatements (e.g. the paper said "may", the answer said "primarily").
- **0 invalid citations** across all 174 answers I evaluated.
- **It says "not found"** instead of guessing when the papers don't have the answer.
- **91% of answers correct** with the default single search (graded by an AI judge against verified evidence).
- **Hybrid search beats vector search by 14 points:**

| Search method (45 questions, 29 papers) | Recall@10 | MRR |
|---|---|---|
| **Hybrid: BM25 + vector, merged by rank (default)** | **0.78** | **0.53** |
| BM25 only | 0.73 | |
| Vector only (bge-small, with query prefix) | 0.64 | |
| Postgres `ts_rank_cd` | 0.49 | |

### What testing taught me

- **It caught a regression.** When I made the prompts general (not heart-sound specific), the agent started rewriting questions into worse search queries. The fix was to always also search the question exactly as typed, which recovered **17 points** of evidence recall ([0016](docs/decisions/0016-search-the-original-question.md)).
- **One search was as good as the agent,** so it became the default. Both got 0.87 correct, but the agent took 3 LLM calls and ~8 s versus 1 call and ~4 s. Now the free tier handles about three times as many questions ([0017](docs/decisions/0017-agent-evaluation-results.md), [0019](docs/decisions/0019-single-search-default.md)).
- **The AI judge was too generous.** It rated 98% faithful; my review found 83%. I report my number ([0017](docs/decisions/0017-agent-evaluation-results.md), [0018](docs/decisions/0018-claim-level-judge.md)).
- **LLM-written test questions make search look better than it is.** Using the LLM's wording, recall@10 was 0.96. When I reworded the same questions naturally, it dropped to 0.57. That's why I checked the test set by hand.
- I tried a reranker and dropped it: no real gain, and 3–7 s slower per question.

<sub>How the review worked: faithfulness labels were drafted with model help, then I checked each one against the cited passages. The judge never saw my labels and I never saw the judge's verdicts. The reviewed answers were generated before the "keep the hedges" prompt change ([0019](docs/decisions/0019-single-search-default.md)). Scripts and raw results are in `eval/`.</sub>

## How it works

```
upload PDF ─► pdftotext ─► clean (headers, reference lists, number grids) ─► sentence chunks ≤350 tokens
          ─► bge-small-en-v1.5 embeddings (local ONNX) ─► Postgres: vector(384) + tsvector, per project

default:  question ─► hybrid search (BM25 + vector, RRF) ─► ANSWER with [S#] citations ─► validated

deep:     question ─► PLAN search queries ─► hybrid search ─► CHECK: enough evidence?
                                                ▲                      │ no: new queries,
                                                └──────────────────────┘ or search one paper
                                                                       │ yes
                                                          ANSWER with [S#] citations ─► validated
```

Every design choice, what I rejected, and the numbers behind it are in **[`docs/decisions/`](docs/decisions/README.md)**.

## Repository layout

```
src/seecite/       the application
  pdf_extract.py     PDF → clean page text
  chunking.py        sentence-based chunks, sized in model tokens
  embeddings.py      local bge-small embeddings, cached
  retrieval.py       vector, BM25 and hybrid search, per project
  agent.py           single search (default) and the plan → search → check → answer agent (deep)
  llm.py             Gemini client: retries, daily-quota detection, response cache
  ingest.py          upload pipeline and metadata extraction; builds the demo project
  app.py             FastAPI: projects, uploads, questions, rate limits
  static/index.html  the web page
eval/              test set, retrieval and agent evaluations, results
tests/             unit tests (pytest)
deploy/            demo project data (loaded on start if missing or outdated)
docs/              decision records
Dockerfile, render.yaml   deployment (Render + Neon)
```

## Run it locally

You need Python 3.12+, Postgres 16 with pgvector, and `pdftotext` (poppler-utils).

```bash
pip install -r requirements.txt
cp .env.example .env            # add your GEMINI_API_KEY (never commit .env)
# set DATABASE_URL in .env if not postgresql://pcg:pcg@localhost:5432/seecite
psql "$DATABASE_URL" -c "CREATE EXTENSION IF NOT EXISTS vector;"
python -c "from huggingface_hub import snapshot_download; snapshot_download('Qdrant/bge-small-en-v1.5-onnx-Q', local_dir='models/bge-small-en-v1.5-onnx-Q', allow_patterns=['*.json','*.onnx','vocab.txt'])"
PYTHONPATH=src uvicorn seecite.app:app --port 7860     # open http://localhost:7860
PYTHONPATH=src python -m pytest -q tests
```

On start, the app creates its tables and loads the demo project if it's missing or older than `deploy/demo_data.sql.gz`. User projects are never touched.

## Deploy

Render (free web service, Docker) plus Neon (free Postgres with pgvector). An external cron job calls `/api/ping` every 10 minutes to keep Render awake. That endpoint doesn't touch the database, so Neon can still go to sleep. See [0015](docs/decisions/0015-hosting-render-neon.md).

## Limits

- No accounts: a project's link is its key (random and unguessable). Don't upload confidential papers.
- Scanned PDFs without a text layer are rejected. Tables and figures are only partly captured.
- The free LLM tier allows a limited number of questions per day. The app tells you when it runs out.
