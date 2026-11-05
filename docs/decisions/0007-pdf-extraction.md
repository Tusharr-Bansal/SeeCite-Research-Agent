# 0007 — Getting clean text out of PDFs

**Date:** 2026-10-21 · **Status:** Accepted

## Context

Everything else (search, answers, citations) depends on the extracted text. Research PDFs are messy: two columns, ligature characters, headers on every page, reference lists, and tables that lose their structure.

## Decision

1. **Use `pdftotext` (poppler)** in default mode.
2. **Drop headers and footers:** any line that appears on at least half the pages (minimum 3). Also drop bare page numbers at the top or bottom of a page.
3. **Drop the reference list,** from "References" (including forms like `R EFERENCES`) up to a "Supplementary", "Appendix" or "S1" heading, as long as the lines after that heading don't look like citations.
4. **Drop "number grids":** paragraphs of 8+ tokens where over 60% are bare numbers. These are tables that lost their column headers.

## Why pdftotext

I compared extractors on a page of Guo et al. (a two-column paper):

| Extractor | Column order | Ligatures | Line-end hyphens |
|---|---|---|---|
| `pdftotext` | correct | fixed (`classification`) | joined |
| `pypdf` | correct | **kept as `ﬁ`** | **kept** (`proba-` / `bilities`) |
| `pymupdf`, `sort=True` | **columns mixed line by line** | fixed | kept |

Ligatures matter for keyword search: `ﬁ` is one character, not `f` + `i`, so "classification" wouldn't match `classiﬁcation`.

## Bugs found

- **References leaking:** the first version let 4 chunks of citations from Guo et al. through. The PDF prints a "Supplementary Materials" header in the middle of the reference list, which stopped the cutting early. Fix: only resume after such a heading if the next 8 lines have fewer than 3 citation signs (years, `pp.`, `doi`, venue names). Both cases are tested in `tests/test_chunking.py`.
- **Number grids:** one Guo et al. chunk was 88% numbers (`5.22% 4.29% 0.81% …`). A chunk like that can't support any answer.

## Alternatives

- **ML layout parsers (Docling, Marker, GROBID):** better at tables, but need PyTorch or Java, are slow on CPU and heavy to deploy. Not worth it for v1.
- **Keep number grids:** rejected. Important numbers usually also appear in the text, which is kept.

## Known limits

- Questions answered only by a table will fail.
- Figure axis labels sometimes survive when mixed with text.
- Equations come out as flattened symbols.
