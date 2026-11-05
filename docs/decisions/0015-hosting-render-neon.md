# 0015 — Hosting on Render (app) + Neon (database)

**Date:** 2026-10-22 · **Status:** Accepted (replaces the Hugging Face plan in 0013)

## Context

When I created the Hugging Face Space, it returned **402 Payment Required**: free Docker Spaces now need a PRO subscription. The app needs ~280 MB RAM plus Postgres with pgvector.

| Option | Cost | Notes |
|---|---|---|
| Hugging Face PRO | $9/month | Already packaged |
| Render free + Neon free | $0, no card | 512 MB, 0.1 CPU, sleeps after 15 min; Neon 1 GB, 100 compute-hours/month |
| Azure for Students | $100 credit | More setup, credit runs out |
| Heroku (Student Pack) | $13/month credit | Needs verification |
| Koyeb / Fly.io / Cloud Run / Oracle | | Card required |

I chose **Render + Neon**, with a timed ping to keep Render awake.

## Decision

- **App on Render** (free, Docker, Singapore), set up from `Dockerfile` + `render.yaml`. Secrets are entered in Render's dashboard, never in the repo.
- **Database on Neon** (free, Singapore, pgvector). Because the database is outside the container, uploaded projects survive restarts.
- **First start on an empty database:** the app creates the tables, then loads the demo from `deploy/demo_data.sql.gz` (data only, so it needs no special permissions). Tested: loads in 0.5 s, identical data.
- **Keep-alive:** cron-job.org calls `/api/ping` every 10 minutes.
  - `/api/ping` **never touches the database.** Neon sleeps after 5 idle minutes and counts compute hours while awake. Pinging the database every 10 minutes would keep it awake all month and burn the free hours for nothing.
  - Every 10 minutes, not 14.5, because cron timing drifts and could miss Render's 15-minute cutoff.
  - Running 24/7 uses ~744 of Render's 750 free hours a month: enough for exactly one always-on service.
- **Tuned for a tiny host:** 1 ONNX thread, 1 Uvicorn worker, and Neon's **direct** (not pooled) connection string, because psycopg's prepared statements don't work through PgBouncer's pooling.

## Consequences

- **Uploads are slow** on 0.1 CPU: minutes per paper. The page says so. Questions are fine.
- Render can restart free services anytime, which resets in-memory rate limits and caches.
- Neon adds ~1 s to the first request after it has slept.

## Problem: the free daily quota

The evaluation stopped at 83 of 116 answers with `429 RESOURCE_EXHAUSTED`: the free tier allows **500 requests/day** for `gemini-3.5-flash-lite`, reset at midnight Pacific.

1. **Retrying a daily quota is pointless.** Visitors were waiting ~30 s through retries just to be told "try again in a minute", which wasn't true. Now `llm.py` recognises the daily quota error and the app replies in under a second: "Today's free AI quota is used up. It resets at midnight Pacific time." Per-minute limits still retry.
2. **My cap was too high.** 300 questions × ~3 LLM calls ≈ 900 calls, more than 500. I lowered it to 120 questions/day. *(Later replaced in [0019](0019-single-search-default.md) by a cap of 400 LLM calls/day.)*

The live app and my evaluations share the same key and the same daily quota, so I run evaluations on quiet days.

## Problem: `pdftotext` missing from the image

While writing the README setup steps, I noticed the Dockerfile didn't install `pdftotext`. Every upload would have failed in production while the pre-loaded demo kept working, which makes it easy to miss. Fixed by installing `poppler-utils`.

**Lesson:** test every external tool the app uses after deploying, not just the path that pre-loaded data allows.

## Problem: the first upload on Render ran out of memory

The first live upload crashed the instance (over 512 MB). The restart recovery worked: the paper was marked "please upload again". I measured memory while embedding 64 chunks:

| Batch size | Peak memory | Speed |
|---|---|---|
| 32 (old default) | **634 MB** | 0.64 s/chunk |
| 8 | 387 MB | 0.60 s/chunk |
| **4 (chosen)** | **309 MB** | 0.56 s/chunk |
| 2 | 278 MB | 0.56 s/chunk |

The idle app uses ~240 MB, and big batches added ~400 MB that ONNX Runtime keeps holding. Batch size 4 leaves ~200 MB of headroom, and it's actually faster on one thread.

**Lesson:** my earlier 280 MB measurement was the idle app. Measure memory on the heaviest path (uploading), not the common one.

## Hiding health checks from the logs

Render calls `/api/ping` every few seconds from its internal network, which buried real requests in the logs. I wanted those hidden but the cron-job.org pings kept, to show the keep-alive works. Render's checks come from a **private** IP (`10.x.x.x`); cron-job.org comes through the proxy with a **public** IP. A log filter (`HideInternalHealthChecks` in `app.py`) drops `/api/ping` lines from private IPs only. Tested in `tests/test_logging.py`. If Render changes how this works, the worst case is the lines come back.

## Problem: questions failed after the database slept

**Symptom:** every question failed with "Something went wrong" until I restarted Render.

**Cause:** the search code opened one database connection at startup and kept it forever. Neon closes all connections when it sleeps after ~5 idle minutes. Because the ping deliberately doesn't touch the database, the app stayed up while its connection died. The upload worker had the same problem.

**Fix:** before each question, the app runs `SELECT 1` and reconnects if needed (which also wakes Neon). The upload worker opens a fresh connection per paper. Tested by force-closing the connection; tests in `tests/test_reconnect.py`.

**Lesson:** if the database is allowed to sleep, any connection can disappear between requests.
