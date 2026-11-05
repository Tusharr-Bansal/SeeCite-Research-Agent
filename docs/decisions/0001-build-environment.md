# 0001 — Where the code is built and run

**Date:** 2026-10-21 · **Status:** Accepted

## Decision

I built this project with Claude Code working in a cloud Linux workspace, with my Windows folder linked for delivery. The cloud workspace had Postgres 16 + pgvector and could reach the Gemini API; my linked folder couldn't, so:

- All code is built and tested in the cloud workspace.
- The repo is copied to my folder at each working milestone.
- Once the repo was on GitHub, GitHub became the source of truth.

Docker Hub was blocked in the workspace, so I used a native Postgres 16 with the `postgresql-16-pgvector` package for development. Production uses hosted Postgres, so this doesn't matter there.

## Consequences

- The workspace is temporary, so I had to sync often.
- Paper PDFs couldn't be downloaded there, so I supplied them myself.

## Problems I hit

- **SQLite on a network drive:** the LLM response cache (SQLite) failed on the synced output folder (`attempt to write a readonly database`), because file locking doesn't work reliably there. I moved the working copy to local disk.
- **Stale file copies (2026-10-22):** copying a file to my computer a second time sometimes delivered the *old* version while reporting success. Three files (`index.html`, `app.py`, `eval_agent.py`) were committed and deployed stale, so a header bug I'd fixed still showed on the site. I found it by comparing MD5 checksums of all 57 files. Now each sync uses a fresh folder, and every file is checksum-compared before I commit.

**Lesson:** a tool saying "written" doesn't mean the content is right. Check the result.
