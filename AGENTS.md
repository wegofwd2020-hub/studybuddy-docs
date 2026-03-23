# AGENTS.md — AI Agent Onboarding

**StudyBuddy AI OnDemand Edition** — Backend-powered STEM tutoring platform (Grades 5–12)

> Read this before touching any code. It covers the system design, conventions,
> layer boundaries, and the most common mistakes to avoid.

---

## Project Status

**Phase: Architecture & Documentation**
Implementation has not begun. The canonical design is in [ARCHITECTURE.md](ARCHITECTURE.md).
The [phased plan](ARCHITECTURE.md#phased-implementation-plan) defines the build order.

Predecessor project (for UI/prompt reference): [studybuddy_free](https://github.com/wegofwd2020-hub/studybuddy_free)

---

## System Architecture Summary

Three distinct runtime contexts:

```
1. Content Pipeline (offline, run by operator)
   CLI scripts → Anthropic API → Content Store (filesystem/S3)

2. Backend (always-on server)
   FastAPI → PostgreSQL + Redis + Content Store
   Serves curriculum, pre-built content, and records progress

3. Mobile App (student device)
   Kivy → local SQLite cache + backend REST API
   Never calls Anthropic directly
```

**The Anthropic API key never leaves the backend environment.** Students authenticate with email + password and receive a JWT.

---

## Repository Layout (target state)

```
StudyBuddy_OnDemand/
  backend/
    main.py              ← FastAPI app entry point
    config.py            ← All env-based config (DB URL, JWT secret, API key)
    src/
      auth/              ← register, login, JWT issue/refresh
      curriculum/        ← serve grade JSON files
      content/           ← serve pre-built lesson/quiz/tutorial JSON
      progress/          ← record session, answers, session end
      admin/             ← build status, regenerate, analytics
    tests/               ← pytest; no live API calls (mock everything)
    requirements.txt

  mobile/
    main.py              ← Kivy app entry (thin client)
    config.py            ← Backend URL, JWT path, SQLite path
    src/
      api/               ← HTTP client (replaces claude_client.py)
      ui/                ← Kivy screens (reused from Free edition, adapted)
      logic/             ← sync manager, local cache, progress queue
      utils/             ← logger (same as Free edition)
    data/                ← grade JSON files (curriculum only, no AI content)

  pipeline/
    build_grade.py       ← CLI: generate all content for a grade
    build_unit.py        ← CLI: regenerate a single unit
    prompts.py           ← Prompt builders (shared with Free edition)
    config.py            ← Anthropic key (env), output path, model

  data/                  ← grade5_stem.json … grade12_stem.json (source of truth)
  docs/                  ← additional design documents
```

---

## Layer Rules

Dependencies flow **downward only**. Never import from a higher layer.

```
mobile/src/ui/          → mobile/src/logic/, mobile/src/api/
mobile/src/logic/       → mobile/src/api/
mobile/src/api/         → (external: backend REST)

backend/src/content/    → backend/src/auth/ (for auth checks only)
backend/src/progress/   → backend/src/auth/
backend/src/admin/      → backend/src/content/, backend/src/curriculum/

pipeline/               → prompts.py, Anthropic API, Content Store
                          (completely independent of backend + mobile at runtime)
```

---

## Key Conventions

### Configuration
- **Backend:** all config via environment variables. Never hardcode database URLs, JWT secrets, or the Anthropic API key. Use `python-decouple` or `pydantic-settings`.
- **Mobile:** `config.py` holds `BACKEND_URL`, local SQLite path, JWT file path. No Anthropic key.
- **Pipeline:** Anthropic API key from `ANTHROPIC_API_KEY` env var. Output path from `CONTENT_STORE_PATH`.

### Authentication (Backend)
- Registration: `POST /auth/register` → bcrypt-hashed password → JWT
- All other endpoints require `Authorization: Bearer <token>` header
- JWT payload: `{student_id, grade, exp}`
- Refresh tokens stored in Redis with TTL

### Content Store Layout
```
{CONTENT_STORE_PATH}/
  G8-MATH-001/
    lesson.json
    quiz_set_1.json
    quiz_set_2.json
    quiz_set_3.json
    tutorial.json
    meta.json          ← {generated_at, model, content_version, unit_id}
```
Always key by `unit_id` (e.g. `G8-MATH-001`). Never by grade or subject name — those change; unit IDs don't.

### Quiz Set Rotation (Mobile)
The app tracks `last_quiz_set` per unit in local SQLite and picks the next set (1→2→3→1) on each attempt. Never show the same set twice in a row.

### Progress Event Queue (Mobile)
Progress events are written to a local SQLite `event_queue` table immediately. A `SyncManager` background thread flushes the queue to the backend whenever the network is available. Events must be idempotent — include a client-generated `event_id` (UUID) so the backend can de-duplicate on retry.

### Logging
Same structured JSON approach as the Free edition:
```python
from src.utils.logger import get_logger, LogContext
log = get_logger("component")   # "api", "sync", "cache", "pipeline", "auth"
```
Never use `print()`. Backend logs go to stdout (captured by the container runtime). Mobile logs go to the same rolling file structure as the Free edition.

### Testing
- **Backend:** pytest + `httpx.AsyncClient` for endpoint tests. Mock PostgreSQL with `pytest-asyncio` + an in-memory SQLite or `testing.postgresql`. Never hit a real database in CI.
- **Mobile:** pytest for logic (sync manager, cache, queue). No Kivy widget tests in CI.
- **Pipeline:** pytest with a mock `urllib.request.urlopen` (same pattern as Free edition).

---

## Content Generation Conventions

Prompt builders live in `pipeline/prompts.py` and are shared with / adapted from the Free edition. Each builder returns `(user_prompt, system_prompt)`.

**Grade-language calibration** — always call `_lang(grade)` in every prompt. The Free edition's `_LANG_LEVEL` dict is the reference.

**Token budgets for pipeline calls** (not constrained by mobile timeouts):

| Content type | Recommended max_tokens |
|---|---|
| Lesson synopsis | 600 |
| Quiz set (8 questions) | 2000 |
| Practice set (4 questions) | 800 |
| Tutorial | 1200 |

These can be higher than in the Free edition because the pipeline runs on a server with no timeout pressure.

---

## Common Pitfalls

1. **Calling Anthropic from the mobile app** — the mobile app has no API key and must never call `api.anthropic.com` directly. All AI content comes from the backend Content Service.
2. **Storing the Anthropic key in the mobile app** — even as a build-time constant. It must only exist in backend environment variables.
3. **Forgetting idempotency on progress events** — the sync manager will retry failed posts. The backend must deduplicate by `event_id`, not insert duplicates.
4. **Serving content directly from the pipeline output path** — in production, put a CDN or at minimum a Content Service in front. Mobile clients must never have direct filesystem access to the Content Store.
5. **Regenerating all content in place while students are active** — use `content_version` in `meta.json`. The mobile app caches by `unit_id + content_version`; a version bump triggers a re-download, but only after the current session ends.
6. **Hardcoding grade numbers** — use `curriculum_loader` to discover grades from `data/grade*_stem.json` filenames. The backend Curriculum Service does the same.
7. **Blocking the mobile UI thread with sync** — all network calls (backend API, sync flush) must run in daemon threads with `@mainthread` callbacks for UI updates (same pattern as Free edition).
8. **Label `text_size` in Kivy** — always bind `text_size` to the label's actual width. Never use `text_size=(9999, None)` with `halign`. Use `wrap_label()` from `widgets.py`. (Lesson learned in the Free edition — documented here to avoid repeating.)

---

## Phase Checklist

### Phase 1 — Backend Foundation
- [ ] FastAPI skeleton with `/health`
- [ ] PostgreSQL schema: `students`, `sessions`
- [ ] Auth endpoints: register, login, refresh
- [ ] Curriculum endpoints: list grades, get grade detail
- [ ] Mobile: replace local registration with backend auth
- [ ] Mobile: JWT stored securely; sent on every request

### Phase 2 — Content Pipeline + Delivery
- [ ] `pipeline/build_grade.py` CLI working end-to-end
- [ ] Content Store populated for at least one grade
- [ ] Backend Content Service endpoints live
- [ ] Mobile: lesson + quiz loaded from backend, not Claude
- [ ] Mobile: local SQLite cache with `content_version` check

### Phase 3 — Progress Tracking
- [ ] Progress endpoints: session, answer, session/end
- [ ] Mobile: events posted after each answer
- [ ] Mobile: result screen shows backend-confirmed score
- [ ] `GET /progress/student` returns full history

### Phase 4 — Offline Sync
- [ ] SQLite event queue on device
- [ ] SyncManager flushes queue on foreground/network
- [ ] Backend deduplicates by `event_id`
- [ ] Content freshness check on app start

### Phase 5 — Admin Dashboard
- [ ] Admin API: content build status, per-unit regeneration
- [ ] Aggregate analytics: struggle rate per question
- [ ] Simple dashboard UI
