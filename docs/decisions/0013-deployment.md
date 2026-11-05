# 0013 — Deployment: one container on Hugging Face, plus safeguards

**Date:** 2026-10-22 · **Status:** Hosting replaced by [0015](0015-hosting-render-neon.md) (free Hugging Face Docker Spaces now need PRO). The safeguards still apply.

## Context

I wanted a free public link that works when a recruiter clicks it. The app needs Postgres with pgvector, the 67 MB embedding model, ~1 GB RAM, and access to the Gemini API.

## Plan

One Docker container on a free Hugging Face Space (2 vCPU, 16 GB RAM):
- Base image `pgvector/pgvector:0.8.0-pg16` plus Python.
- **Postgres inside the container,** listening only on localhost. On start, `deploy/start.sh` loads the papers from a 2.3 MB database dump, then starts the API.
- **FastAPI** serving one page and a few API endpoints.
- **The API key** as a Space secret, never in the image or repo.

## Safeguards for a public demo on a free key

| Risk | Safeguard |
|---|---|
| Strangers using up the free quota | Per-visitor limits (5/min, 30/day) and a global cap of 300 questions/day, all configurable |
| Bursts causing provider 429s | One question at a time |
| Huge or abusive inputs | 3–500 characters, checked before any work |
| Prompt injection | The agent has **no write tools**; it can only search read-only papers. The worst case is a bad answer, not a bad action |
| Database exposure | Postgres only listens on localhost |
| Provider overload | Retries with backoff, then a friendly "model is busy" message instead of an error trace |

## Testing (2026-10-22)

I ran the startup script as a non-root user on an empty data folder, the way the container would:
- **Found a real bug:** Postgres tried to put its socket in `/var/run/postgresql`, which a non-root user can't write to. Hugging Face runs containers as a non-root user, so this would have failed in production. Fixed by using `/tmp`.
- After the fix: 985 chunks loaded, health check OK, a bad API key gave a clean error, and the 6th request in a minute got HTTP 429.
- The page worked at desktop (light) and phone (dark) widths, with citations linking to their passages.

## Alternatives

- **Render/Railway + hosted Postgres:** more accounts, and free tiers had only 512 MB RAM. (I ended up here anyway, see 0015.)
- **Gradio or Streamlit:** faster to write, but a plain HTML page gives full control over how citations are shown.
- **Pre-computed answers:** free, but then it's not a live system.

## Consequences

- Free Spaces sleep after ~48 h, so the first visit takes 1–2 minutes.
- Rate limits are in memory and reset on restart. Fine for a demo; production would use Redis or the database.
