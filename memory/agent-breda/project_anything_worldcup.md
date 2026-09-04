---
name: Anything Worldcup project
description: Full-stack web app built as first real project for the ai-teams team
type: project
---

Full-stack "Anything Worldcup" app at `/Users/kirinchoi/Projects/anything-worldcup/`.
SPEC.md is now at `/Users/kirinchoi/Vaults/team-mustang/projects/anything-worldcup/SPEC.md`.

**Why:** First real project for the Roy+Breda+Hawkeye team. Purpose is proving the SDD workflow end-to-end.

**Stack:** FastAPI + SQLAlchemy (SQLite) backend (port 8801), React + Vite frontend (port 8800).

**Status:** v0.2 complete. SC-1~SC-12 all verified by Hawkeye. pytest 15/15, Playwright 22/22.

**How to apply:** When asked to continue work on this project, check `/Users/kirinchoi/Vaults/team-mustang/projects/anything-worldcup/SPEC.md` for current SC list and the git log for recent changes.

**Key implementation notes:**
- `SECRET_KEY` loaded from `backend/.env` via python-dotenv (gitignored)
- `GET /api/topics` filters out topics with < 2 items (SC-12)
- CSS reset includes `button { color: inherit; }` to fix dark-on-dark text (SC-11)
- E2E tests use unique `Date.now()` names to avoid DB state accumulation across runs
