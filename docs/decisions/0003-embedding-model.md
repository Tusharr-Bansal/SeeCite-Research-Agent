# 0003 — Embeddings: local `bge-small-en-v1.5`

**Date:** 2026-10-21 · **Status:** Accepted (revisit if search recall is weak)

## Context

Vector search needs a model that turns text into vectors so similar meanings end up close together. My constraints:
- Free hosts often have ~512 MB of RAM, so the model must be small.
- Re-embedding everything should be fast and free.
- The same text should always give the same vector, so evaluations can be repeated.

## Decision

`BAAI/bge-small-en-v1.5`, the quantized ONNX version (`Qdrant/bge-small-en-v1.5-onnx-Q`, 67 MB), run locally with `fastembed` (no PyTorch). It gives 384-dimension vectors.

Quick sanity check (cosine similarity):

| Pair | Similarity |
|---|---|
| "phonocardiogram murmur detection" vs "heart sound segmentation" | 0.835 |
| "phonocardiogram murmur detection" vs "stock market prices" | 0.454 |

It separates related from unrelated text. Whether it finds the *right* passages is measured later (0011).

## Alternatives

- **`gemini-embedding-001` (API):** probably better on scientific text, but every call uses free quota, and every question would need an API call just to embed it.
- **sentence-transformers + PyTorch:** same model, but PyTorch adds hundreds of MB.
- **Larger bge models:** better, but ~3× slower and bigger. Only worth it if the small one turns out to be the bottleneck.

## Consequences

- Quantization costs a little accuracy for size and speed.
- bge was trained mostly on general web text, so medical terms like "S3 gallop" may embed imperfectly. That's one reason I also use keyword search (0004).
