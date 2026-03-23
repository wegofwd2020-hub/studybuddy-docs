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
   CLI scripts → Anthropic API + TTS API → Content Store (filesystem/S3)
   Generates content per unit × language (en, fr, es)

2. Backend (always-on server)
   FastAPI → PostgreSQL + Redis + Content Store
   Serves curriculum, pre-built content, subscription status, and records progress

3. Mobile App (student device)
   Kivy → local SQLite cache + backend REST API
   Never calls Anthropic directly
```

**The Anthropic API key never leaves the backend environment.** Students authenticate with email + password and receive a JWT that includes their grade and locale.

---

## Repository Layout (target state)

```
StudyBuddy_OnDemand/
  backend/
    main.py              ← FastAPI app entry point
    config.py            ← All env-based config (DB URL, JWT secret, API key, Stripe keys)
    src/
      auth/              ← register, login, JWT issue/refresh
      curriculum/        ← serve grade JSON files
      content/           ← serve pre-built lesson/quiz/tutorial/experiment/audio JSON
      progress/          ← record session, answers, session end
      subscription/      ← plan management, Stripe checkout, webhook handler
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
    i18n/                ← static UI string dictionaries: en.json, fr.json, es.json

  pipeline/
    build_grade.py       ← CLI: generate all content for a grade × language(s)
    build_unit.py        ← CLI: regenerate a single unit
    tts_worker.py        ← TTS synthesis: lesson text → MP3
    prompts.py           ← Prompt builders (shared with Free edition)
    config.py            ← Anthropic key (env), TTS key (env), output path, model

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
backend/src/subscription/ → backend/src/auth/
backend/src/admin/      → backend/src/content/, backend/src/curriculum/, backend/src/subscription/

pipeline/               → prompts.py, Anthropic API, TTS API, Content Store
                          (completely independent of backend + mobile at runtime)
```

---

## Key Conventions

### Configuration
- **Backend:** all config via environment variables. Never hardcode database URLs, JWT secrets, API keys, or Stripe keys. Use `python-decouple` or `pydantic-settings`.
- **Mobile:** `config.py` holds `BACKEND_URL`, local SQLite path, JWT file path. No Anthropic key. No Stripe key.
- **Pipeline:** Anthropic API key from `ANTHROPIC_API_KEY` env var; TTS key from `TTS_API_KEY`; output path from `CONTENT_STORE_PATH`.

### Authentication (Backend)
- Registration: `POST /auth/register` → bcrypt-hashed password → JWT
- JWT payload: `{student_id, grade, locale, exp}`
- All other endpoints require `Authorization: Bearer <token>` header
- Refresh tokens stored in Redis with TTL

### Locale Handling
- Locale is part of the JWT: `"locale": "fr"`. The backend reads locale from the JWT to serve the correct language file.
- Content endpoints never accept a `?lang=` query parameter — locale is authoritative from the JWT only, preventing students from accessing languages their account isn't set for.
- The pipeline `--lang` flag controls which language columns are built. Always build `en` first.
- Fall back to `en` if a requested language file does not exist in the Content Store.

### Subscription Entitlement
- Entitlement is checked **on every content request** by middleware in `backend/src/content/`.
- Free-tier students: `lessons_accessed` is incremented on each unique lesson view (by `unit_id`). After 2 distinct units, HTTP 402 is returned.
- Subscription status is cached in Redis for 5 minutes per student to avoid DB roundtrips on every request.
- The mobile app must handle HTTP 402 on content endpoints by navigating to `SubscriptionScreen`.
- Never gate entitlement on the mobile side — the backend is the single source of truth.

### Content Store Layout
```
{CONTENT_STORE_PATH}/
  G8-MATH-001/
    lesson_en.json
    lesson_fr.json
    lesson_es.json
    lesson_en.mp3
    lesson_fr.mp3
    lesson_es.mp3
    quiz_set_1_en.json
    quiz_set_1_fr.json
    quiz_set_1_es.json
    quiz_set_2_{lang}.json
    quiz_set_3_{lang}.json
    tutorial_{lang}.json
    experiment_{lang}.json   ← only for units with assessments.labs
    meta.json                ← {generated_at, model, content_version, langs_built: ["en","fr","es"]}
```
Always key by `unit_id` (e.g. `G8-MATH-001`). Never by grade or subject name — those change; unit IDs don't.

### Text-to-Speech (Pipeline)
- TTS runs as part of the pipeline, after lesson JSON is generated.
- Input: `lesson_{lang}.json["synopsis"]` + joined `key_concepts`.
- Output: `lesson_{lang}.mp3` written to the same unit directory.
- `tts_worker.py` wraps the provider SDK; switch providers by changing `TTS_PROVIDER` in `pipeline/config.py`.
- Do not generate TTS at request time — audio must be pre-built in the pipeline.

### Experiment Visualization (Pipeline)
- Before generating experiment JSON, check `unit["assessments"]["labs"]` in the curriculum JSON.
- If `labs` is empty or absent, skip experiment generation entirely (do not write a placeholder file).
- The backend returns HTTP 404 for `GET /content/{unit_id}/experiment` when no file exists. The mobile app shows the "🔬 Experiment" button only if this endpoint returns 200.

### Quiz Set Rotation (Mobile)
The app tracks `last_quiz_set` per unit in local SQLite and picks the next set (1→2→3→1) on each attempt. Never show the same set twice in a row.

### Progress Event Queue (Mobile)
Progress events are written to a local SQLite `event_queue` table immediately. A `SyncManager` background thread flushes the queue to the backend whenever the network is available. Events must be idempotent — include a client-generated `event_id` (UUID) so the backend can de-duplicate on retry.

### i18n (Mobile UI Strings)
- UI strings (button labels, navigation, error messages) are stored in `mobile/i18n/{lang}.json`.
- Load the dictionary at app start based on the JWT locale. Fall back to `en` if a key is missing.
- Never hardcode user-facing strings in screen files — always use `i18n.t("key")`.
- AI-generated content (lesson, quiz, tutorial text) does NOT go through i18n — it is already in the correct language from the pipeline.

### Logging
Same structured JSON approach as the Free edition:
```python
from src.utils.logger import get_logger, LogContext
log = get_logger("component")   # "api", "sync", "cache", "pipeline", "auth", "subscription"
```
Never use `print()`. Backend logs go to stdout (captured by the container runtime). Mobile logs go to the same rolling file structure as the Free edition.

### Testing
- **Backend:** pytest + `httpx.AsyncClient` for endpoint tests. Mock PostgreSQL with `pytest-asyncio` + an in-memory SQLite or `testing.postgresql`. Never hit a real database in CI. Mock Stripe SDK calls.
- **Mobile:** pytest for logic (sync manager, cache, queue, i18n loader). No Kivy widget tests in CI.
- **Pipeline:** pytest with a mock `urllib.request.urlopen` (same pattern as Free edition). Mock TTS provider SDK.

---

## Content Generation Conventions

Prompt builders live in `pipeline/prompts.py` and are shared with / adapted from the Free edition. Each builder returns `(user_prompt, system_prompt)`.

**Grade-language calibration** — always call `_lang(grade)` in every prompt. The Free edition's `_LANG_LEVEL` dict is the reference.

**Language instruction** — every prompt must include an explicit language instruction:
```python
lang_names = {"en": "English", "fr": "French", "es": "Spanish"}
lang_instruction = f"Respond entirely in {lang_names[lang]}."
```

**Token budgets for pipeline calls** (not constrained by mobile timeouts):

| Content type | Recommended max_tokens |
|---|---|
| Lesson synopsis | 600 |
| Quiz set (8 questions) | 2000 |
| Practice set (4 questions) | 800 |
| Tutorial | 1200 |
| Experiment guide (5–10 steps) | 1500 |

These can be higher than in the Free edition because the pipeline runs on a server with no timeout pressure.

---

## Common Pitfalls

1. **Calling Anthropic from the mobile app** — the mobile app has no API key and must never call `api.anthropic.com` directly. All AI content comes from the backend Content Service.
2. **Storing the Anthropic key in the mobile app** — even as a build-time constant. It must only exist in backend environment variables.
3. **Forgetting idempotency on progress events** — the sync manager will retry failed posts. The backend must deduplicate by `event_id`, not insert duplicates.
4. **Serving content directly from the pipeline output path** — in production, put a CDN or at minimum a Content Service in front. Mobile clients must never have direct filesystem access to the Content Store.
5. **Regenerating all content in place while students are active** — use `content_version` in `meta.json`. The mobile app caches by `unit_id + content_version + lang`; a version bump triggers a re-download, but only after the current session ends.
6. **Hardcoding grade numbers** — use `curriculum_loader` to discover grades from `data/grade*_stem.json` filenames. The backend Curriculum Service does the same.
7. **Blocking the mobile UI thread with sync** — all network calls (backend API, sync flush) must run in daemon threads with `@mainthread` callbacks for UI updates (same pattern as Free edition).
8. **Label `text_size` in Kivy** — always bind `text_size` to the label's actual width. Never use `text_size=(9999, None)` with `halign`. Use `wrap_label()` from `widgets.py`. (Lesson learned in the Free edition — documented here to avoid repeating.)
9. **Gating entitlement in the mobile app** — the mobile app must not decide whether a student has access. It only reads the HTTP status from the backend. HTTP 200 = serve; HTTP 402 = show paywall.
10. **Hardcoding locale in prompts** — every pipeline prompt must receive a `lang` parameter and include the language instruction. A prompt missing the language instruction will produce English output regardless of the `--lang` flag.
11. **Generating TTS at request time** — audio files must be pre-built in the pipeline. The `GET /content/{unit_id}/lesson/audio` endpoint serves a file; it never calls a TTS API at request time.
12. **Showing the Experiment button for all units** — only show `ExperimentScreen` if `GET /content/{unit_id}/experiment` returns HTTP 200. Check at unit load time and cache the result.
13. **Storing Stripe keys in the mobile app** — the mobile app never touches Stripe directly. It only calls `POST /subscription/checkout` to get a checkout URL, then opens that URL in a browser.

---

## Phase Checklist

### Phase 1 — Backend Foundation
- [ ] FastAPI skeleton with `/health`
- [ ] PostgreSQL schema: `students`, `sessions`
- [ ] Auth endpoints: register (with `locale`), login, refresh
- [ ] Curriculum endpoints: list grades, get grade detail
- [ ] Mobile: replace local registration with backend auth (include language picker)
- [ ] Mobile: JWT stored securely; locale read from JWT; sent on every request

### Phase 2 — Content Pipeline + Delivery (English)
- [ ] `pipeline/build_grade.py` CLI working end-to-end (`--grade N --lang en`)
- [ ] Content Store populated for at least one grade in English
- [ ] Backend Content Service endpoints live
- [ ] Entitlement middleware: track `lessons_accessed`, return HTTP 402 after 2 for free tier
- [ ] Mobile: lesson + quiz loaded from backend, not Claude
- [ ] Mobile: local SQLite cache with `content_version + lang` check
- [ ] Mobile: `SubscriptionScreen` stub (no real payment)

### Phase 3 — Progress Tracking
- [ ] Progress endpoints: session, answer, session/end
- [ ] Mobile: events posted after each answer
- [ ] Mobile: result screen shows backend-confirmed score
- [ ] `GET /progress/student` returns full history

### Phase 4 — Offline Sync + Multi-language + TTS
- [ ] SQLite event queue on device
- [ ] SyncManager flushes queue on foreground/network
- [ ] Backend deduplicates by `event_id`
- [ ] Content freshness check on app start
- [ ] Pipeline: `--lang fr,es` generates French and Spanish content
- [ ] `tts_worker.py`: generates `lesson_{lang}.mp3` in pipeline
- [ ] `GET /content/{unit_id}/lesson/audio` endpoint
- [ ] Mobile: language picker in Settings; switches locale + clears content cache
- [ ] Mobile: "🔊 Listen" button on SubjectScreen; caches MP3 for offline

### Phase 5 — Subscription + Payments
- [ ] PostgreSQL `subscriptions` table
- [ ] `POST /subscription/checkout` → Stripe Checkout Session URL
- [ ] `POST /subscription/webhook` → handle Stripe lifecycle events
- [ ] Entitlement cache in Redis (5-minute TTL)
- [ ] Mobile: `SubscriptionScreen` with real plan cards and Stripe checkout

### Phase 6 — Experiment Visualization
- [ ] Pipeline: detect `assessments.labs`; generate `experiment_{lang}.json`
- [ ] `GET /content/{unit_id}/experiment` endpoint (404 if no lab)
- [ ] Mobile: `ExperimentScreen` — step-by-step card layout
- [ ] Mobile: "🔬 Experiment" button on SubjectScreen (visible only when 200)
- [ ] Experiment content cached alongside lesson JSON

### Phase 7 — Admin Dashboard + Analytics
- [ ] Admin API: content build status, per-unit/language regeneration
- [ ] Subscription analytics: MRR, churn, conversion
- [ ] Aggregate analytics: struggle rate per question
- [ ] Simple dashboard UI
