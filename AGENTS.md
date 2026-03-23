# AGENTS.md — AI Agent Onboarding

**StudyBuddy AI OnDemand Edition** — Backend-powered STEM tutoring platform (Grades 5–12)

> Read this before touching any code. It covers the system design, conventions,
> layer boundaries, and the most common mistakes to avoid.

> **Requirements tracking:** feature requirements, non-functional requirements, and architectural decisions are tracked in [REQUIREMENTS.md](REQUIREMENTS.md). Check the status column before implementing — a requirement may be `deferred` or already `in-progress`.
> **Backend performance architecture:** [BACKEND_ARCHITECTURE.md](BACKEND_ARCHITECTURE.md) — caching strategy, connection pools, hot path rules, SLOs, deployment topology.

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

### Security

- **Rate limiting:** apply at the middleware layer (not in individual route handlers). Use `slowapi` or a gateway (nginx/Caddy). Auth endpoints: 10 req/min per IP. Content endpoints: 100 req/min per student JWT.
- **Secrets:** all secrets via environment variables. Never commit `.env` files. In production, source from AWS Secrets Manager, Vault, or equivalent. The `config.py` in each layer uses `pydantic-settings` with `env` fields — no defaults for secrets (fail fast if missing).
- **CORS:** configure `CORSMiddleware` with an explicit `allow_origins` list from env var `ALLOWED_ORIGINS`. Default deny.
- **Input validation:** define all request bodies as Pydantic models. Never access `request.body()` directly in a route handler for business logic.
- **Password hashing:** bcrypt with cost factor ≥ 12. Run in thread pool executor to avoid blocking the event loop.

### Compliance

- **Age gate:** the registration handler checks `dob` (if provided) or infers age from `grade`. If under 13, set `account_status = "pending_parental_consent"` and send a consent email to the provided guardian address. Block content access until `account_status = "active"`.
- **Account deletion:** `DELETE /auth/account` must remove or anonymise all records referencing the `student_id` in a single transaction: `students`, `sessions`, `progress_answers`, `subscriptions`. Cancel the Stripe subscription first (call Stripe API), then delete local records. Return 200 only after all data is gone.
- **Forgot-password emails:** always respond HTTP 200 regardless of whether the email exists. Log internally but never surface to the caller.

### Observability

- **Structured logging:** use `get_logger("component")` from `utils/logger.py`. Every log entry must include `component`, `action`, and any relevant IDs (`student_id`, `unit_id`, `session_id`). Never log passwords, tokens, or Stripe keys.
- **Sentry:** initialise `sentry_sdk` in `backend/main.py` and `mobile/main.py` using `SENTRY_DSN` from env. Use `sentry_sdk.set_user({"id": student_id})` at the start of authenticated requests. Strip PII from breadcrumbs with a `before_send` hook.
- **Health check depth:** `GET /health` must verify: (1) a simple DB query (`SELECT 1`), (2) a Redis ping, (3) that the Content Store path exists and is readable. Return 503 if any check fails.

### Pipeline

- **Idempotency:** at the start of each unit build, read the existing `meta.json`. If `content_version` matches the target version and all expected files are present, skip unless `--force` is passed.
- **Schema validation:** after each Claude call, run `jsonschema.validate(response, SCHEMA[content_type])`. On `ValidationError`, log the raw response and retry (up to 3 times). After 3 failures, append to `pipeline_failures.json` and continue to the next unit.
- **Cost tracking:** maintain a running `tokens_used` counter across the pipeline run. If `tokens_used * TOKEN_COST_USD > MAX_PIPELINE_COST_USD`, abort the run and log a warning. Default `MAX_PIPELINE_COST_USD = 50.0`.

### Performance

- **Connection pools initialised once at worker startup** — create `asyncpg` and `aioredis` pools in the FastAPI `lifespan` context manager, not per-request. Store them on `app.state`. Inject via `Depends(get_db)` and `Depends(get_redis)` dependencies.
- **Cache read order: L1 → L2 → DB** — always check the in-process `TTLCache` first (zero network cost), then Redis (< 1 ms), then PostgreSQL (5–20 ms). Never skip a cache level on the hot read path.
- **Explicit cache invalidation on state changes** — entitlement cache must be invalidated when: (a) a lesson is accessed (increment `lessons_accessed`), (b) subscription status changes, (c) student transfers school. Curriculum resolver cache must be invalidated when: (a) student joins or leaves a school, (b) a new curriculum is activated. Use a helper `invalidate_student_cache(student_id)` that deletes both keys atomically.
- **Write endpoints return before the DB write** — `POST /progress/answer` and `POST /analytics/lesson/end` dispatch a Celery task and return `200 OK` immediately. Never await the DB write on the request path for these endpoints.
- **Celery task routing** — pipeline tasks go to the `pipeline` queue (separate high-CPU workers). Light IO tasks (email, webhook) go to the `io` queue. Never put pipeline tasks on the `default` queue — a large pipeline run would starve email delivery.

### School & Curriculum

- **Curriculum resolver:** implement as FastAPI middleware or a dependency (`get_resolved_curriculum_id(student=Depends(get_current_student))`). Every content endpoint calls this dependency — never inline the resolution logic per-endpoint.
- **XLSX parsing:** use `openpyxl` (not `xlrd`; xlrd dropped xlsx support). Iterate rows explicitly; do not assume the first row is always the header. Strip whitespace from all cell values. Return a structured `ParseResult` with `units: list[UnitRow]` and `errors: list[RowError]`.
- **Pipeline async dispatch:** use Celery with Redis as broker, or Python's `asyncio.create_task` for Phase 8. Store job state (`pending → running → done/failed`) in Redis keyed by `job_id` with a 24-hour TTL. The status polling endpoint reads from Redis.
- **School content isolation:** the Content Store path prefix for school curricula is `{curriculum_id}/` (a UUID). This provides natural isolation — no school can ever guess another school's curriculum_id to retrieve content.

### Analytics

- **Lesson view events use the offline queue:** `POST /analytics/lesson/start` and `POST /analytics/lesson/end` are queued in the same local SQLite `event_queue` as progress events. They share the same SyncManager flush path. Include `event_type: "lesson_start" | "lesson_end"` to distinguish them server-side.
- **Attempt number is server-authoritative:** never trust the client's `attempt_number`. The backend computes it from the database before inserting a new session. If a client submits an `attempt_number`, discard it.
- **Passing threshold config:** the pass/fail threshold (default 70%) is a server-side config value (`QUIZ_PASS_THRESHOLD = 0.70`), not hardcoded per endpoint. Teachers cannot currently change it; but it must not be a magic number scattered through the codebase.

### Reporting

- **All report endpoints use `get_read_db`** — never `get_db`. Add a linter check or assertion to enforce this at startup if possible.
- **Report response shape includes `data_as_of`** — every report endpoint response must include a top-level `"data_as_of": "ISO8601"` field containing the refresh timestamp of the underlying materialized view. Read this from a `mv_refresh_log` table or `pg_stat_user_tables.last_autoanalyze`.
- **Materialized views are the only source for report data** — do not write ad-hoc `SELECT … GROUP BY` queries in report endpoints. All aggregation lives in the materialized view definitions. This keeps query latency predictable and testable.
- **CSV export is always async** — `POST /reports/.../export` returns `{job_id}` immediately. The Celery task generates the CSV, writes it to a temp S3 path, and returns a pre-signed download URL valid for 1 hour. Never stream CSV bytes synchronously from an API response.
- **Alert thresholds default to sensible values** — ship `report_alert_settings` with defaults (`pass_rate_threshold=50`, `feedback_volume_threshold=3`, `inactive_days=14`) so the alert system works out-of-the-box without teacher configuration.

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
14. **Accepting Stripe webhooks without signature verification** — always call `stripe.Webhook.construct_event(payload, sig_header, STRIPE_WEBHOOK_SECRET)` first. A `SignatureVerificationError` must return HTTP 400 immediately. Failing to verify allows anyone to fake subscription events.
15. **No idempotency on the Stripe webhook handler** — log the `stripe_event_id` to the `stripe_events` table before processing. If the ID already exists, return HTTP 200 immediately without re-processing. Stripe retries on failure; double-processing activates subscriptions twice or credits accounts twice.
16. **Inline SQL strings with f-string interpolation** — always use SQLAlchemy ORM or parameterized queries. Never build a SQL string with user-controlled data inside it. FastAPI + Pydantic validate shape but not content; SQL injection is still possible if you bypass the ORM.
17. **Emitting a password reset response that leaks whether an email exists** — `POST /auth/forgot-password` must always return HTTP 200 regardless of whether the email is registered. A different response for registered vs. unregistered emails enables email enumeration.
18. **Blocking on bcrypt in an async endpoint** — bcrypt is CPU-bound and will block the FastAPI event loop. Run it in a thread pool: `await asyncio.get_event_loop().run_in_executor(None, bcrypt.checkpw, ...)`.
19. **Pipeline writing content while students are mid-session** — use `content_version` in `meta.json`. Bump the version atomically (write to a temp path, rename). Mobile caches by version; students mid-session complete against the version they started. Never overwrite in place.
20. **Calling the pipeline without pinning the Claude model ID** — if `CLAUDE_MODEL` is not set to a specific model string, a future SDK update could silently switch models and produce inconsistently styled content mid-grade. Pin explicitly in `pipeline/config.py`.
21. **No cache size limit on the mobile device** — without a `MAX_CACHE_MB` limit, downloading all grades × 3 languages × audio can exhaust storage on low-end devices. The cache manager must evict least-recently-used entries when the limit is approached.
22. **Launching app version checks after user-facing content loads** — check `GET /app/version` early in the app startup flow, before rendering content screens. An outdated client hitting a changed API will produce cryptic errors, not a clear upgrade prompt.
23. **Resolving curriculum_id from grade number at request time without caching** — curriculum resolution (student.school_id → active curriculum for grade + year → curriculum_id) involves a DB lookup. Cache the resolved `curriculum_id` in Redis per student (same TTL as entitlement cache). Invalidate on school transfer, curriculum activation, or subscription change.
24. **Generating content for a custom curriculum without validating unit codes first** — unit codes must be unique within a curriculum. A duplicate unit code in an XLSX upload causes two units to write to the same Content Store path, silently overwriting each other. Validate uniqueness before writing any units.
25. **Treating XLSX parsing errors as exceptions** — the upload handler must return HTTP 400 with a structured list of row-level errors (row number, column, error description) rather than a 500. Teachers cannot fix what they cannot see. Never let a pandas/openpyxl exception bubble up as a 500 to the client.
26. **Blocking the pipeline job on the HTTP request** — `POST /curriculum/pipeline/trigger` must return a `job_id` immediately (202 Accepted) and run the pipeline asynchronously (Celery task or background thread). Pipeline runs for large curricula take minutes; do not hold the HTTP connection open.
27. **Allowing unrestricted access to school curricula** — when `restrict_access = true`, the content resolver must verify `school_enrolments` membership before serving any unit. Do not rely on the client knowing which units to request; enforce on every content endpoint.
28. **No attempt_number on quiz sessions** — without `attempt_number`, analytics cannot distinguish a student's first attempt from their fifth. Compute `attempt_number` server-side as `COUNT(*) + 1` from prior sessions for (student_id, unit_id, curriculum_id) before inserting the new session row. Never compute it client-side.
29. **Storing teacher credentials in the student JWT** — teachers and students have separate JWT secrets and separate auth endpoints. A student JWT must never grant access to teacher endpoints, and vice versa. Middleware must check both `role` and `resource ownership` (e.g., a teacher can only access analytics for their own school).
30. **Sending pipeline failure emails with no actionable detail** — when a unit fails pipeline generation, the email to the teacher must include: unit name, unit code, failure reason (schema validation vs. API error), and a link to the admin panel. "Your pipeline failed" with no detail forces a support request.
31. **Calling a synchronous library inside an async FastAPI route** — any blocking call (synchronous DB driver, `time.sleep`, `requests.get`, synchronous file I/O) blocks the entire uvicorn event loop and stalls all concurrent requests on that worker. Use `asyncpg` for DB, `aioredis` for Redis, `httpx.AsyncClient` for HTTP. Wrap unavoidable blocking calls (bcrypt, CPU-bound processing) with `await asyncio.get_event_loop().run_in_executor(None, fn, *args)`.
32. **Proxying audio through the API server** — never stream MP3 bytes from S3 through a FastAPI response. Generate a pre-signed S3/CloudFront URL and return it to the client. Streaming audio ties up a worker for the entire download duration (potentially minutes on a slow mobile connection).
33. **Not sizing the connection pool relative to worker count** — total PostgreSQL connections = `num_workers × pool_max_size`. With 4 workers and `pool_max=20` that is 80 connections. Add Celery workers and you can hit PostgreSQL `max_connections` (default 100) instantly. Always deploy PgBouncer in transaction-pooling mode; set `pool_max` per worker to 10–20; set PostgreSQL `max_connections` to ≥ 200.
34. **Invalidating Redis cache but forgetting the CDN** — when a content version bump occurs, clearing Redis L2 is not enough. CloudFront may still serve the old `lesson_en.json` from edge caches for up to 1 hour (or the configured TTL). On pipeline completion, call `cloudfront.create_invalidation(paths=[f"/{curriculum_id}/{unit_id}/*"])` for every rebuilt unit.
35. **Deploying Redis without AOF persistence** — if Redis restarts with `appendonly no`, all cached entitlements, JWT refresh tokens, rate limit counters, and pipeline job states are lost. Every student is immediately logged out. Every rate limit counter resets. Set `appendonly yes` and `appendfsync everysec` before going to production.
36. **Running report queries against the PostgreSQL primary** — report aggregations (`GROUP BY`, `AVG`, `COUNT` across sessions and lesson_views) are expensive and long-running. They must route to the read replica. Always use the `get_read_db` dependency in report endpoints, never `get_db`. Running these on the primary competes with entitlement checks and progress writes.
37. **Refreshing all materialized views on every report request** — `REFRESH MATERIALIZED VIEW` takes a write lock and can take seconds on large tables. Only Celery Beat should trigger routine refreshes (nightly). The on-demand refresh endpoint (`POST /reports/.../refresh`) must be rate-limited (once per 30 minutes per school) and dispatched as a Celery task, never awaited synchronously.
38. **Not showing the "last updated" timestamp on reports** — materialized views are stale by design (up to 24 hours). Without a visible timestamp, teachers may act on yesterday's data thinking it is current. Always include `data_as_of: ISO8601` in every report response body.
39. **Sending weekly digest emails in a single synchronous loop** — the digest task may need to send to hundreds of teachers. Dispatch one Celery sub-task per recipient (`send_digest_email.apply_async(args=[teacher_id])`). Never block the Beat task waiting for email delivery.
45. **Calling `log.info()` directly for business events instead of `emit_event()`** — scattered log calls produce inconsistent field names across components, breaking Grafana queries and alert rules that depend on specific label values. All business events must go through `core/events.py emit_event()`; this is the single call point that writes the structured log entry AND increments the Prometheus counter.
46. **Exposing `GET /metrics` publicly** — the Prometheus metrics endpoint reveals internal state (queue depths, auth failure rates, DB pool usage) that aids an attacker. Block it at nginx with an `allow <prometheus_ip>; deny all;` directive, and require `METRICS_TOKEN` bearer auth as a second layer. Never forward it through the public load balancer.
47. **Writing audit_log entries synchronously in the request path** — `write_audit_log()` performs a DB insert. Run it as a fire-and-forget Celery task to avoid adding DB write latency to the response. The audit event does not need to be visible before the API returns 200.
48. **Not attaching `correlation_id` to Sentry events** — Sentry issues are hard to correlate with log entries without a shared ID. Set `sentry_sdk.set_tag("correlation_id", current_correlation_id.get())` in the request middleware so every captured exception carries the same ID as the log stream.
42. **Fetching Auth0 JWKS on every token exchange request** — the JWKS endpoint is a remote HTTP call. Cache the key set in L1 TTLCache with a 24-hour TTL; refresh only on `kid` mismatch (unknown key ID). Never hit the JWKS endpoint inline on the student login hot path.
43. **Using the external Auth0 `id_token` directly on content or progress endpoints** — the `id_token` does not contain `grade`, `locale`, `school_id`, or `account_status`. Always complete the token exchange first and use the internal JWT for all subsequent requests. Middleware that accepts raw Auth0 tokens on non-exchange endpoints is a security gap.
44. **Checking `account_status` in PostgreSQL inside the auth middleware** — this adds a DB query to every authenticated request. Store suspension state in a Redis set (`suspended:{id}`, TTL = JWT lifetime). The middleware reads only from Redis after signature verification; PostgreSQL is never queried on the hot path for status checks.
40. **Computing learning streaks from raw `progress_sessions` on every dashboard request** — a streak query scans every session row for the student, which grows unboundedly. Store `{current, longest, last_active_date}` in a Redis hash (`streak:{student_id}`) and update it via a Celery task on the first progress event of each calendar day. The dashboard reads only the Redis hash — zero DB queries for streak data.
41. **Returning another student's progress from `/student/dashboard` or `/student/progress`** — these endpoints must compare the `student_id` extracted from the JWT against the owner of the requested data. Never rely on a `student_id` query parameter; derive it exclusively from the verified JWT payload.

---

## Phase Checklist

### Phase 1 — Backend Foundation
- [ ] FastAPI skeleton with `/health` (deep check: DB + Redis + Content Store)
- [ ] PostgreSQL schema: `students`, `teachers`, `admin_users`, `sessions`
- [ ] Alembic migration scaffold; all schema changes via migrations
- [ ] Auth0 tenant configured; JWKS URL in `config.py`; JWKS cached in L1 TTLCache (24-hr TTL)
- [ ] `POST /auth/exchange` — verify Auth0 id_token → upsert student → issue internal JWT + refresh token
- [ ] `POST /auth/teacher/exchange` — same pattern for teachers
- [ ] `POST /auth/refresh` — Redis refresh token → new internal JWT; no Auth0 call
- [ ] `POST /auth/forgot-password` — calls Auth0 Management API; always returns 200
- [ ] `POST /admin/auth/login` — bcrypt verify in executor → admin JWT (ADMIN_JWT_SECRET)
- [ ] `POST /admin/auth/forgot-password` + `POST /admin/auth/reset-password` — Redis one-time token (TTL 1 hr)
- [ ] Admin account lockout: 5 failed logins → 15-minute Redis cooldown
- [ ] `account_status` field on `students`, `teachers`, `schools`; auth middleware checks Redis `suspended:{id}`
- [ ] PATCH status endpoints for students/teachers/schools (product_admin/super_admin only)
- [ ] Cascade suspension: suspending a school suspends all its teachers/students via Celery task
- [ ] Auth0 block sync on suspension via Celery best-effort task
- [ ] Parental consent flow: Auth0 age-gate for under-13; block content until `account_status = active`
- [ ] Curriculum endpoints: list grades, get grade detail
- [ ] `PATCH /student/profile` endpoint
- [ ] `DELETE /auth/account` endpoint (queues GDPR erasure + Auth0 user deletion)
- [ ] Mobile: Auth0 SDK integration; exchange → store internal JWT securely; locale from JWT
- [ ] `core/observability.py`: Prometheus metric objects, HTTP middleware, `GET /metrics` endpoint (token-protected), `GET /health` deep check
- [ ] `core/events.py`: `emit_event(category, event_type, **ctx)` — structured log + metric counter; `write_audit_log()` — async Celery dispatch
- [ ] Correlation ID middleware: UUID injected per request into `contextvars.ContextVar`; returned as `X-Correlation-Id` header
- [ ] `audit_log` PostgreSQL table created via Alembic migration
- [ ] Sentry SDK initialised in backend and mobile; `before_send` strips PII; `correlation_id` tag set in middleware
- [ ] CORS policy configured via `ALLOWED_ORIGINS` env var
- [ ] HTTPS enforced; HTTP → HTTPS redirect configured
- [ ] asyncpg connection pool initialised in FastAPI lifespan context
- [ ] aioredis connection pool initialised in FastAPI lifespan context
- [ ] PgBouncer configured in Docker Compose (transaction-pooling mode)
- [ ] bcrypt run in thread pool executor (not blocking event loop)

### Phase 2 — Content Pipeline + Delivery (English)
- [ ] `pipeline/build_grade.py` CLI working end-to-end (`--grade N --lang en`)
- [ ] Pipeline is idempotent: skip units already at target `content_version` unless `--force`
- [ ] Pipeline pins Claude model ID in `pipeline/config.py`; never uses implicit latest
- [ ] Pipeline validates each generated JSON against schema before writing; retries up to 3×
- [ ] Pipeline emits per-unit structured logs (tokens, cost estimate, duration)
- [ ] Pipeline emits a JSON run summary on completion
- [ ] Pipeline spend cap: abort if estimated cost exceeds `MAX_PIPELINE_COST_USD`
- [ ] Content Store populated for at least one grade in English
- [ ] Backend Content Service endpoints live
- [ ] Rate limiting on content endpoints (100 req/min per student JWT)
- [ ] Entitlement middleware: track `lessons_accessed`, return HTTP 402 after 2 for free tier
- [ ] `POST /content/{unit_id}/report` endpoint
- [ ] `GET /app/version` endpoint
- [ ] Mobile: lesson + quiz loaded from backend, not Claude
- [ ] Mobile: local SQLite cache with `content_version + lang` check; enforce `MAX_CACHE_MB` eviction
- [ ] Mobile: app version check on launch; prompt upgrade if below minimum version
- [ ] Mobile: `SubscriptionScreen` stub (no real payment)
- [ ] Structured log aggregation platform configured (CloudWatch / ELK / Loki)
- [ ] Uptime monitoring on `/health` (external ping, 60-second interval)
- [ ] Prometheus + Alertmanager deployed; `GET /metrics` scraped by Prometheus (token-protected)
- [ ] Alertmanager rules file (`docs/prometheus/alerts.yml`) deployed with critical/high/medium groups
- [ ] Notification channels configured: PagerDuty (critical), Slack `#studybuddy-alerts` (high/medium), SES email (pipeline/payments)
- [ ] Grafana provisioned with six dashboards (Platform Overview, Content & Cache, Student Activity, Backend Health, Security, SLO Burn Rate)
- [ ] Pipeline emits metrics to Prometheus Pushgateway on completion
- [ ] DB pool state + Celery queue depth metrics polled by Celery Beat every 30 s
- [ ] nginx `allow <prometheus_ip>; deny all;` for `/metrics` location block
- [ ] Redis L2 cache: entitlement and curriculum resolver with 300s TTL
- [ ] In-process L1 cache (cachetools TTLCache) for curriculum tree and JWT keys
- [ ] nginx configured: TLS, rate limiting, gzip, keep-alive upstream
- [ ] Celery workers configured: pipeline queue + default queue
- [ ] Progress write endpoints use fire-and-forget Celery task pattern

### Phase 3 — Progress Tracking
- [ ] Progress endpoints: session, answer, session/end
- [ ] Mobile: events posted after each answer
- [ ] Mobile: result screen shows backend-confirmed score
- [ ] `GET /progress/student` returns full raw history
- [ ] `GET /student/dashboard` — aggregated summary card (streak, completion %, next unit, recent activity)
- [ ] `GET /student/progress` — curriculum map with per-unit status badges (`not_started`, `in_progress`, `needs_retry`, `completed`)
- [ ] `GET /student/stats?period=7d|30d|all` — usage statistics including daily_activity[]
- [ ] Streak counter initialised in Redis on first progress event; Celery task increments on first event per calendar day
- [ ] Materialized view `mv_student_curriculum_progress` created and refreshed async on session end
- [ ] Dashboard response cached at L1 + L2; invalidated on `POST /progress/session/end`
- [ ] JWT ownership check: `/student/*` endpoints reject requests where JWT `student_id` ≠ data owner
- [ ] Mobile: `ProgressDashboardScreen`, `CurriculumMapScreen`, `StatsScreen`

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
- [ ] CloudFront CDN configured in front of S3 bucket
- [ ] Audio endpoint returns pre-signed CloudFront URL; does not proxy bytes
- [ ] Cache-Control headers set on S3 objects (lesson JSON: 1hr, MP3: 24hr)
- [ ] CDN invalidation triggered on content version bump

### Phase 5 — Subscription + Payments
- [ ] PostgreSQL `subscriptions` table
- [ ] PostgreSQL `stripe_events` table (dedup + audit log)
- [ ] `POST /subscription/checkout` → Stripe Checkout Session URL
- [ ] `POST /subscription/webhook` → validate signature, dedup by `stripe_event_id`, handle lifecycle events
- [ ] Handle `invoice.payment_failed` with 3-day grace period before locking content
- [ ] Force-expire entitlement cache in Redis on cancellation / downgrade
- [ ] `DELETE /auth/account` cancels Stripe subscription before deleting local records
- [ ] Entitlement cache in Redis (5-minute TTL)
- [ ] Alert configured for Stripe webhook delivery failures
- [ ] Mobile: `SubscriptionScreen` with real plan cards and Stripe checkout

### Phase 6 — Experiment Visualization
- [ ] Pipeline: detect `assessments.labs`; generate `experiment_{lang}.json`
- [ ] `GET /content/{unit_id}/experiment` endpoint (404 if no lab)
- [ ] Mobile: `ExperimentScreen` — step-by-step card layout
- [ ] Mobile: "🔬 Experiment" button on SubjectScreen (visible only when 200)
- [ ] Experiment content cached alongside lesson JSON

### Phase 7 — Admin Dashboard + Analytics
- [ ] Admin endpoints protected by separate admin JWT role (not student JWT)
- [ ] Admin API: content build status, per-unit/language regeneration
- [ ] `GET /admin/pipeline/status` — last run time, units built/missing/failed
- [ ] Audit log for all admin actions (who triggered what, when)
- [ ] Subscription analytics: MRR, churn, conversion
- [ ] Aggregate analytics: struggle rate per question
- [ ] Content report review queue: surface units exceeding report threshold
- [ ] Simple dashboard UI

### Phase 8 — School & Teacher Registration + Curriculum Upload
- [ ] PostgreSQL schema: `schools`, `teachers`, `curricula`, `curriculum_units` (Alembic migration)
- [ ] `POST /schools/register` — auto-approve; create school_admin teacher account
- [ ] Teacher JWT with `{teacher_id, school_id, role}` payload
- [ ] Teacher auth middleware: separate from student JWT; role-checked on all teacher endpoints
- [ ] `GET /curriculum/template` — serve pre-formatted XLSX template
- [ ] `POST /curriculum/upload` — parse XLSX (openpyxl), validate, return row-level errors, store as `curriculum_units`
- [ ] `POST /curriculum/pipeline/trigger` — dispatch async pipeline job; return `job_id`
- [ ] Pipeline reads units from DB when `curriculum_id` is provided (not from local JSON file)
- [ ] Content Store path: `{curriculum_id}/{unit_id}/…`
- [ ] `GET /curriculum/pipeline/{job_id}/status` — read job state from Redis
- [ ] Email teacher on pipeline completion or failure (per-unit failure detail)
- [ ] `python pipeline/seed_default.py --year 2026` seeds default curricula from `data/grade*_stem.json`

### Phase 9 — Student–School Association + Curriculum Routing
- [ ] PostgreSQL schema: `school_enrolments` (Alembic migration)
- [ ] `POST /schools/{school_id}/enrolment` — teacher uploads student email roster
- [ ] Student registration: match email against pending enrolments; set `student.school_id` on match
- [ ] Student registration: optional `enrolment_code` field
- [ ] Curriculum resolver dependency: resolves `curriculum_id` from student + grade + year; cached in Redis
- [ ] Curriculum resolver fallback: unaffiliated students → `default-{year}-g{grade}`
- [ ] `restrict_access` enforcement: HTTP 403 for non-enrolled students on restricted curricula
- [ ] `PUT /curriculum/{curriculum_id}/activate` — activates, archives previous
- [ ] Mobile: Settings shows enrolled school name and "Leave school" option
- [ ] Cache invalidation: clear curriculum resolver cache on school transfer or curriculum activation

### Phase 10 — Extended Analytics + Student Feedback
- [ ] PostgreSQL schema: `lesson_views`, `feedback` (Alembic migration)
- [ ] Alembic migration: add `attempt_number`, `curriculum_id`, `passed` to `sessions`
- [ ] `POST /analytics/lesson/start` and `POST /analytics/lesson/end` endpoints
- [ ] Mobile: lesson start/end events queued in `event_queue` alongside progress events
- [ ] Attempt number: computed server-side before session insert
- [ ] `passed` flag: set on `POST /progress/session/end` based on `QUIZ_PASS_THRESHOLD`
- [ ] `GET /analytics/student/me` — student self-service metrics
- [ ] `GET /analytics/school/{school_id}/class` — teacher class analytics
- [ ] Struggle flag: query surfacing units where mean_attempts > 2 or pass_rate < 50%
- [ ] `POST /feedback` — rate-limited to 5/student/hour; store in `feedback` table
- [ ] `GET /admin/feedback` — paginated, filterable
- [ ] Mobile: "Give Feedback" button on lesson, result, and Settings screens
- [ ] Mobile: Progress screen shows student analytics dashboard
