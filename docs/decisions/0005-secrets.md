# 0005 — Handling the API key

**Date:** 2026-10-21 · **Status:** Accepted

- The Gemini key lives only in `.env` (`GEMINI_API_KEY=...`), which is in `.gitignore`. The repo has `.env.example` with a placeholder.
- The key never goes into code, logs or chat. When checking it, only the first and last 4 characters are shown.
- In deployment, it's a secret environment variable on the host, not baked into the Docker image.

A better option, where a proxy attaches the key so it never reaches the build machine, wasn't available in my setup.
