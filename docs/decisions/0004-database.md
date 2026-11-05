# 0004 — One Postgres database for vector and keyword search

**Date:** 2026-10-21 · **Status:** Accepted (keyword ranking settled in 0011)

## Decision

Hybrid search needs a vector index (meaning) and a keyword index (exact words) over the same chunks. I put both in one **Postgres 16** database:
- **pgvector** for the embeddings.
- Postgres **full-text search** (`tsvector` + GIN index) for keywords.

Both run on the same `chunks` table, so their results use the same chunk IDs and are easy to merge.

## Alternatives

- **A vector database (Qdrant, Chroma, Weaviate) plus a separate keyword engine:** two systems to deploy and keep in sync. Overkill for ~100 papers.
- **FAISS in memory:** fast, but no keyword search and no persistence.
- Postgres is already on my resume, and free hosted Postgres with pgvector is easy to find.

## Open question at the time

Postgres's built-in `ts_rank` isn't BM25: it doesn't boost rare words like "S3 gallop" the way BM25 does. I planned to compare `ts_rank_cd`, BM25 in Python, and a Postgres BM25 extension once the test set existed, and pick by recall@10. Result: BM25 in Python (0011).

At this size (a few thousand chunks), the choice of vector index type barely matters.
