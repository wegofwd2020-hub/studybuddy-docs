# StudyBuddy OnDemand — Change Log

Tracks all decisions, design changes, and implementation milestones.

---

## Design Decisions

| # | Decision | Rationale |
|---|---|---|
| D-01 | Backend-only Anthropic API calls | Removes student API key requirement; eliminates on-device latency |
| D-02 | Pre-generated content pipeline | Lessons and quizzes available instantly; no timeout risk on mobile |
| D-03 | 3 quiz sets per unit | Student can retake a topic without seeing identical questions |
| D-04 | Content versioned by `content_version` in `meta.json` | Allows selective regeneration without disrupting in-progress students |
| D-05 | Offline-first with SQLite event queue | Progress never lost on spotty connections; synced when online |
| D-06 | Per-answer progress logging | Enables adaptive difficulty and struggle analytics in Phase 7 |
| D-07 | JWT authentication with `locale` in payload | Stateless; works offline between refreshes; locale travels with token |
| D-08 | Grade gating on backend | Controls which grades are visible per student; natural upgrade path |
| D-09 | Shared `prompts.py` between pipeline and Free edition | Consistent grade-language calibration; one place to improve prompts |
| D-10 | Content Store keyed by `unit_id` | Unit IDs are stable; grade/subject names can change without breaking storage |
| D-11 | Freemium: 2 free lessons, then subscription | Lowers registration barrier; shows value before asking for payment; no time limit on trial |
| D-12 | Subscription via Stripe (hosted checkout) | Removes PCI scope from backend; Stripe handles card storage and compliance |
| D-13 | Content pre-generated per language (en, fr, es) | No runtime translation; content quality consistent across languages; new languages added in pipeline only |
| D-14 | Locale authoritative from JWT only | Prevents content language mismatch; students cannot request a language not on their account |
| D-15 | TTS audio pre-generated in pipeline (Amazon Polly / Google TTS) | No per-request latency; audio cached offline with lesson JSON; provider swappable |
| D-16 | Experiment visualization JSON generated only for lab units | Avoids generating empty content; mobile detects availability via HTTP 200/404 on content endpoint |
| D-17 | Student streak stored in Redis, not computed from DB | Streak query over unbounded `progress_sessions` is expensive; Redis hash updated once per calendar day by Celery; dashboard reads only the hash |
| D-20 | External auth (Auth0) for students and teachers; internal JWT issued after token exchange | Delegates password storage, brute-force defence, email verification, and password reset to Auth0; our backend never stores a student or teacher password hash |
| D-21 | Local bcrypt auth for internal product team (developer/tester/product_admin/super_admin) | Internal tooling must not depend on external provider uptime; `ADMIN_JWT_SECRET` is separate to prevent credential cross-use |
| D-22 | Suspension enforced via Redis set in auth middleware; Auth0 block is best-effort async | Avoids DB query on every authenticated request; 15-min TTL matches JWT lifetime for instant effective revocation |
| D-18 | Student dashboard cached at L1+L2 (60 s TTL), invalidated on session end | Dashboard is read on every app open; caching eliminates DB fan-out on hot path; short TTL ensures freshness |
| D-19 | `/student/*` report endpoints derive student_id exclusively from JWT | Prevents one student accessing another's progress; no client-supplied student_id accepted |
| D-22 | Suspension enforced via Redis set **with no TTL** in auth middleware; Auth0 block is best-effort async | Avoids DB query on every authenticated request; no TTL ensures suspension persists until explicitly lifted — a TTL would silently restore access |
| D-23 | Static in-code RBAC (`ROLE_PERMISSIONS` map); no DB-backed permission table | Seven fixed roles with deterministic permissions; zero hot-path DB queries for permission checks; map can be replaced with DB-backed system later without changing call sites |
| D-24 | AlexJS automated language analysis in the pipeline before human review | Catches insensitive language (profanity, gendered phrasing, ableist terms) at generation time; warnings feed review queue without blocking pipeline throughput |
| D-25 | Content versioning at subject level, not unit level | Subject is the atomic publication unit; avoids partially reviewed content reaching students; rollback always restores a coherent subject snapshot |
| D-26 | Block and publish require separate permission levels (`review:approve` vs `content:publish`) | Two-party check: a school_admin can approve for their school but only a product_admin can make it live on the platform |

---

## Implementation Log

*No implementation has begun. Log entries will be added here as phases are completed.*

---

## Pending

| # | Item | Phase |
|---|---|---|
| P-01 | FastAPI project skeleton | 1 |
| P-02 | PostgreSQL schema design (`students`, `sessions`, `subscriptions`) | 1 |
| P-03 | Auth service (register / login / refresh) — with `locale` field | 1 |
| P-04 | Curriculum service | 1 |
| P-05 | Mobile thin-client auth flow (with language picker at registration) | 1 |
| P-06 | Content generation pipeline CLI (`build-grade --grade N --lang en`) | 2 |
| P-07 | Content service endpoints (lesson, quiz, practice, tutorial) | 2 |
| P-08 | Entitlement middleware (free-tier 2-lesson limit, HTTP 402) | 2 |
| P-09 | Mobile SQLite content cache (`unit_id + content_version + lang`) | 2 |
| P-10 | `SubscriptionScreen` stub | 2 |
| P-11 | Progress service endpoints | 3 |
| P-12 | Mobile progress event posting | 3 |
| P-28 | Student progress reports: `GET /student/dashboard`, `GET /student/progress`, `GET /student/stats` | 3 |
| P-29 | Streak counter in Redis + Celery daily updater | 3 |
| P-30 | Materialized view `mv_student_curriculum_progress` + async refresh on session end | 3 |
| P-31 | Mobile: `ProgressDashboardScreen`, `CurriculumMapScreen`, `StatsScreen` | 3 |
| P-13 | SQLite offline event queue + SyncManager | 4 |
| P-14 | Backend event deduplication | 4 |
| P-15 | Multi-language pipeline (`--lang fr,es`) | 4 |
| P-16 | TTS worker (`tts_worker.py`): lesson text → MP3 | 4 |
| P-17 | Audio delivery endpoint + mobile "🔊 Listen" button | 4 |
| P-18 | Language picker in mobile Settings; locale switch + cache clear | 4 |
| P-19 | Mobile i18n: static UI string dictionaries (en/fr/es) | 4 |
| P-20 | Subscription service: Stripe checkout + webhook handler | 5 |
| P-21 | `SubscriptionScreen` with real plan cards and checkout flow | 5 |
| P-22 | Entitlement cache in Redis | 5 |
| P-23 | Experiment pipeline: detect `assessments.labs`, generate `experiment_{lang}.json` | 6 |
| P-24 | Experiment content endpoint (`GET /content/{unit_id}/experiment`) | 6 |
| P-25 | `ExperimentScreen` mobile UI (step-by-step lab guide) | 6 |
| P-26 | Admin API + dashboard | 7 |
| P-27 | Subscription analytics (MRR, churn, conversion) | 7 |
| P-32 | RBAC: `core/permissions.py` `ROLE_PERMISSIONS` map + `require_permission()` dependency | 7 |
| P-33 | AlexJS pipeline integration: `pipeline/alex_checker.py` + `alex_report.json` per unit | 2 |
| P-34 | `content_subject_versions` PostgreSQL schema + Alembic migration | 2 |
| P-35 | Content endpoint guard: `status = published` AND no active block | 2 |
| P-36 | Content review queue + annotation + rating + approve/reject endpoints (Phase 7) | 7 |
| P-37 | Dictionary/thesaurus integration: Datamuse + Merriam-Webster API | 7 |
| P-38 | Publish + rollback + CDN invalidation for content versioning | 7 |
| P-39 | Content block + lift endpoints; school-scoped and platform-wide | 7 |
| P-40 | Student marked-text feedback: mobile UI + `POST /content/{unit_id}/feedback/marked` | 7 |

---

---

## 2026-03-23 (Phase 1 Gap Closure)

### Phase 1 readiness — all gaps closed (this session)
- Gap analysis identified 10 critical Phase 1 blockers; all resolved
- Created PHASE1_SETUP.md: Auth0 tenant setup (7 steps), complete .env.example with all
  previously missing vars, PostgreSQL DDL for all Phase 1 tables, PgBouncer config,
  nginx config, Kivy/Python PKCE flow, conftest.py testing setup, standard API contracts,
  design clarifications
- ARCHITECTURE.md patches:
  - Fixed suspension TTL bug: `suspended:{id}` Redis key has NO TTL; explicitly deleted on
    reactivation only (prior text said TTL = 15 min, which would silently restore access)
  - Added `POST /auth/logout` to Auth endpoints table (deletes refresh token from Redis)
  - Expanded Secrets Management table from 6 → 16 rows; added all Auth0 vars, ADMIN_JWT_SECRET,
    SENTRY_DSN, METRICS_TOKEN; added JWT algorithm note (HS256 Phase 1, RS256 deferred)
  - Added PARENTAL_CONSENT entity to ERD with `consent_token`, `token_expires_at`, `ip_address`
  - Added Standard Response Envelopes section (error shape + pagination shape) to API Design

## 2026-03-23

### Documentation
- Added REQUIREMENTS.md: full requirements tracking with ~150 FR/NFR entries, ADR log
- Expanded ARCHITECTURE.md: security controls, compliance (COPPA/GDPR), observability,
  content quality, multi-tenancy/curriculum model, school & teacher management,
  curriculum upload, student-school association, extended analytics, student feedback,
  Phases 8–10
- Added BACKEND_ARCHITECTURE.md: performant backend design — three-level caching,
  async-throughout application model, PgBouncer, Celery worker topology, nginx gateway,
  circuit breakers, critical DB indexes, materialized views, SLOs, deployment topologies
- Expanded AGENTS.md: 22 additional pitfalls (9–30), 6 new convention sections,
  updated phase checklists for Phases 1–7, new checklists for Phases 8–10

### Cross-file consistency updates (this session)
- ARCHITECTURE.md: added BACKEND_ARCHITECTURE.md companion doc link; expanded Technology
  Stack table with gunicorn, asyncpg, PgBouncer, CDN, cachetools, Celery, circuitbreaker,
  and Docker/ECS deployment rows; added Performance Targets subsection in Observability;
  updated System Overview diagram with PgBouncer and CDN nodes; added backend architecture
  notes to Phase 1, 2, and 4 in the Phased Implementation Plan
- REQUIREMENTS.md: updated NFR-PERF-001/002/003 to align with SLOs from
  BACKEND_ARCHITECTURE.md; added NFR-PERF-006/007, NFR-REL-010/011/012,
  NFR-OBS-012/013; added ADR-015 through ADR-018; bumped version to 0.3.0
- AGENTS.md: added BACKEND_ARCHITECTURE.md reference line; added pitfalls 31–35
  (event loop blocking, audio proxying, connection pool sizing, CDN invalidation, Redis
  AOF persistence); added Performance conventions section; updated Phase 1/2/4 checklists

### Observability Stack — Grafana Hooks, Notifications, Event Logging (this session)
- ARCHITECTURE.md: replaced thin Observability section with full stack design — 8 subsections:
  observability stack diagram, Prometheus metrics (HTTP auto + 15 custom business/infra metrics),
  Grafana dashboard reference (6 dashboards), Alertmanager rules YAML (critical/high/medium groups),
  notification routing (PagerDuty/Slack/SES), structured event logging system (8 categories,
  emit_event() API, correlation ID pattern), audit_log table schema with 9 event types,
  Sentry integration with Slack integration
- BACKEND_ARCHITECTURE.md: expanded Technology Stack table with prometheus_client,
  prometheus-fastapi-instrumentator, Alertmanager, Grafana, OpenTelemetry, events.py, audit_log
- REQUIREMENTS.md: replaced NFR-OBS (13 items) with 22 detailed requirements (NFR-OBS-001 to 022)
- AGENTS.md: added pitfalls 45–48 (emit_event() discipline, /metrics exposure, audit_log async,
  Sentry correlation_id); expanded Phase 1 checklist (observability.py, events.py, correlation ID,
  audit_log table, Sentry); expanded Phase 2 checklist (Prometheus, Alertmanager, Grafana, nginx)

### Authentication Architecture + Account Management (this session)
- ARCHITECTURE.md: added "Authentication Architecture" section — two-track auth diagram,
  token exchange sequence diagram, account status lifecycle state diagram; updated Auth
  API endpoints table (exchange, teacher/exchange, admin/auth/*); added Account Management
  API endpoints table; updated STUDENT/TEACHER data models (external_auth_id, auth_provider,
  account_status replacing password_hash); added ADMIN_USER to ERD; updated Phase 1
- REQUIREMENTS.md: replaced FR-AUTH with 14 updated requirements reflecting external auth
  pattern; added FR-ACCTMGMT (10 requirements); added ADR-022/023/024
- AGENTS.md: added pitfalls 42–44 (JWKS caching, raw id_token on non-exchange endpoints,
  account_status DB query in middleware); expanded Phase 1 checklist with Auth0 integration,
  token exchange, admin auth, suspension enforcement
- CHANGES.md: added D-20/D-21/D-22

### Student Progress Reports (this session)
- ARCHITECTURE.md: added "Student Progress Reports" section — 3 new endpoints
  (`GET /student/dashboard`, `GET /student/progress`, `GET /student/stats`), response
  payloads, status logic table, mobile screens, architecture flow diagram; updated
  Phase 3 milestone to include all three report endpoints and new mobile screens
- REQUIREMENTS.md: added FR-SPRD section with 11 requirements (FR-SPRD-001 to FR-SPRD-011)
  covering dashboard, curriculum map, usage stats, streak logic, caching, ownership checks
- AGENTS.md: added pitfalls 40–41 (streak from raw DB, student_id from query param);
  expanded Phase 3 checklist with all report endpoints, streak Redis setup,
  materialized view, cache invalidation, and mobile screens
- CHANGES.md: added design decisions D-17/D-18/D-19; added pending items P-28 to P-31

---

## Implementation Log

### Phase 1 — Backend Foundation (2026-03-24)

**Branch:** `20260324_1310` → merged to `main`
**Tests:** 38/38 passing

**Files created:**
- `backend/main.py` — FastAPI app, lifespan (asyncpg + aioredis pools), CORS, correlation ID middleware, custom HTTPException handler
- `backend/config.py` — pydantic-settings; fail-fast on missing secrets; `JWT_SECRET` ≠ `ADMIN_JWT_SECRET` enforced
- `backend/src/auth/router.py` — `POST /auth/exchange`, `/teacher/exchange`, `/refresh`, `/logout`, `/forgot-password`, `PATCH /student/profile`, `DELETE /auth/account`
- `backend/src/auth/admin_router.py` — `POST /admin/auth/login`, `/forgot-password`, `/reset-password`; 5-failure lockout → 15-min cooldown
- `backend/src/auth/service.py` — Auth0 JWKS L1 cache, internal JWT issue/verify, bcrypt in executor, upsert_student/teacher, Auth0 Management API calls
- `backend/src/auth/schemas.py` — request/response models; `ForgotPasswordRequest.email` uses minimal `@` validator (not `EmailStr`) to prevent email enumeration via 422
- `backend/src/auth/tasks.py` — Celery tasks: `sync_auth0_suspension`, `cascade_school_suspension`, `gdpr_delete_account`, `write_audit_log_task`
- `backend/src/auth/dependencies.py` — `get_current_student`, `get_current_teacher`, `require_admin` FastAPI dependencies
- `backend/src/account/router.py` — PATCH status endpoints for students/teachers/schools (RBAC-gated)
- `backend/src/core/permissions.py` — `ROLE_PERMISSIONS` dict (7 roles × 15 permissions); `require_permission()` dependency
- `backend/src/core/observability.py` — Prometheus counters/histograms, `CorrelationIdMiddleware`, `GET /health` (deep check), `GET /metrics` (token-protected)
- `backend/src/core/events.py` — `emit_event()`, `write_audit_log()` (Celery dispatch)
- `backend/src/core/cache.py`, `db.py`, `redis_client.py` — shared infrastructure helpers
- `backend/src/curriculum/router.py` — `GET /curriculum/grades`, `GET /curriculum/grade/{grade}`
- `backend/src/utils/logger.py` — structured JSON logger (structlog)
- `backend/alembic/versions/0001_phase1_initial_schema.py` — tables: `schools`, `students`, `teachers`, `admin_users`, `parental_consents`, `audit_log`; ENUM types; indexes
- `backend/tests/` — `conftest.py`, `test_auth.py`, `test_account.py`, `test_curriculum.py`, `test_health.py`, `helpers/token_factory.py`
- `mobile/src/auth/auth0_client.py` — PKCE flow, code exchange
- `mobile/src/auth/token_store.py` — secure JWT storage (0600 permissions)
- `data/grade5_stem.json` … `data/grade12_stem.json` — default STEM curriculum metadata
- `docker-compose.yml`, `infra/pgbouncer/pgbouncer.ini`, `infra/nginx/nginx.conf`, `infra/nginx/nginx.dev.conf`
- `.github/workflows/test.yml` — CI pipeline
- `dev_start.sh` — one-command dev environment launcher (start / test / stop / reset)

**Bug fixes applied during implementation:**
- Custom `HTTPException` handler: unwraps `detail` dict to root level so error envelope is always `{error, detail, correlation_id}`
- `ForgotPasswordRequest`: changed `EmailStr` → `str` + `@` check to prevent 422 leaking email format acceptance
- Admin lockout: added `headers=exc.headers or {}` to custom handler so `Retry-After` header is preserved
- Cascade suspension: moved `import cascade_school_suspension` to module level so test patching works
- Parental consent age-gate: `upsert_student` now sets `account_status='active'` for non-under-13 students (DB default was `pending`); exchange endpoint blocks `pending` with 403 `account_pending`
- `dev_start.sh`: port conflict detection, dedicated test DB on port 5433, env var isolation before pytest

**How to run:**
```bash
./dev_start.sh          # start server → http://localhost:8000/api/docs
./dev_start.sh stop     # stop containers
./dev_start.sh reset    # wipe DB and start fresh
```

**How to test:**
```bash
./dev_start.sh test     # runs 38/38 automated tests (no API key or Auth0 needed)
```

**Endpoints available after Phase 1:**

| Endpoint | Auth required | Notes |
|---|---|---|
| `GET /health` | No | DB + Redis live check |
| `GET /metrics` | Bearer token | `METRICS_TOKEN` from `.env` |
| `GET /api/v1/curriculum/grades` | Student JWT | Lists grades 5–12 |
| `GET /api/v1/curriculum/grade/{grade}` | Student JWT | Subjects + units for a grade |
| `POST /api/v1/auth/exchange` | Auth0 id_token | Issues internal JWT (student) |
| `POST /api/v1/auth/teacher/exchange` | Auth0 id_token | Issues internal JWT (teacher) |
| `POST /api/v1/auth/refresh` | Refresh token | New access JWT |
| `POST /api/v1/auth/logout` | Refresh token | Invalidates session |
| `POST /api/v1/auth/forgot-password` | None | Always returns 200 |
| `PATCH /api/v1/student/profile` | Student JWT | Update name/grade/locale |
| `DELETE /api/v1/auth/account` | Student JWT | GDPR deletion (async) |
| `POST /api/v1/admin/auth/login` | Email + password | Admin JWT (bcrypt) |
| `PATCH /api/v1/admin/accounts/students/{id}/status` | Admin JWT | Suspend/reactivate |
| `PATCH /api/v1/admin/accounts/teachers/{id}/status` | Admin JWT | Suspend/reactivate |
| `PATCH /api/v1/admin/accounts/schools/{id}/status` | Admin JWT | Suspend/cascade |

**Generate a test student JWT (no Auth0 needed):**
```bash
source venv/bin/activate && cd backend
python -c "
from src.auth.service import create_internal_jwt
from config import settings
token = create_internal_jwt(
    {'student_id': '00000000-0000-0000-0000-000000000001',
     'grade': 8, 'locale': 'en', 'role': 'student', 'account_status': 'active'},
    settings.JWT_SECRET, settings.JWT_ACCESS_TOKEN_EXPIRE_MINUTES)
print(token)
"
```

---

### Phase 2 — Content Pipeline + Delivery (2026-03-25)

**Branch:** `main` (direct)
**Tests:** 52/52 passing (+14 new)

**Files created:**

*Pipeline (`pipeline/`):*
- `config.py` — `ANTHROPIC_API_KEY`, `CONTENT_STORE_PATH`, `CLAUDE_MODEL="claude-sonnet-4-6"` (pinned); `MAX_PIPELINE_COST_USD=50`; `TTS_PROVIDER`; `REVIEW_AUTO_APPROVE`
- `prompts.py` — `build_lesson_prompt`, `build_quiz_prompt`, `build_tutorial_prompt`, `build_experiment_prompt`; grade-appropriate language complexity; locale-aware
- `schemas.py` — JSON schemas + validators for lesson, quiz (enforces exactly 8 questions), tutorial, experiment, meta; `jsonschema` library
- `alex_runner.py` — `npx alex --stdin`; 30s timeout; graceful no-op if Node/npx unavailable (`skipped=True`)
- `tts_worker.py` — AWS Polly / Google TTS / no-op; never crashes pipeline on missing deps
- `build_unit.py` — core generation: idempotency via `meta.json`, Claude API calls, schema validation (retry 3×), AlexJS, TTS synthesis, `meta.json` update, spend cap; CLI entry point
- `build_grade.py` — loads `data/gradeN_stem.json`, upserts `curricula`/`curriculum_units` in DB, calls `build_unit` per unit/lang, creates `content_subject_versions` rows, structured JSON logs, run summary; CLI entry point
- `seed_default.py` — seeds grades 5–12 idempotently into DB; CLI entry point

*Alembic migration `0002_phase2_content_schema.py`:*
- Tables: `curricula`, `curriculum_units`, `content_subject_versions`, `content_reviews`, `content_annotations`, `content_blocks`, `student_content_feedback`, `app_versions` (seeded `0.1.0`), `student_entitlements`

*Backend content service (`backend/src/content/`):*
- `schemas.py` — `LessonResponse`, `QuizResponse`, `TutorialResponse`, `ExperimentResponse`, `AudioUrlResponse`, `ReportRequest`, `FeedbackRequest`, `AppVersionResponse`
- `service.py` — `get_entitlement`, `check_content_published`, `get_content_file`, `increment_lessons_accessed`, `resolve_curriculum_id`, `check_content_block`, `get_next_quiz_set`, `get_unit_subject`; L2 Redis cache (entitlement 300s TTL, content 3600s TTL)
- `router.py` — 8 endpoints; published/block guards; free-tier 402 after 2 lessons; quiz rotation 1→2→3→1; audio returns URL not bytes; locale from JWT only

*Mobile (`mobile/src/`):*
- `logic/LocalCache.py` — thread-safe SQLite cache; LRU eviction to `MAX_CACHE_MB`
- `api/content_client.py` — async httpx: `get_lesson`, `get_quiz`, `get_tutorial`, `get_audio_url`, `get_app_version`, `report_content`
- `ui/SubscriptionScreen.py` — Kivy stub on HTTP 402; Subscribe button placeholder for Phase 5

*Observability:*
- `docs/prometheus/alerts.yml` — 3 alert groups (critical / high / medium); 8 rules

*Tests:*
- `backend/tests/test_content.py` — 9 tests: published guard, 402 entitlement gate, quiz rotation, experiment 404, report, app version, audio URL
- `backend/tests/test_pipeline.py` — 5 tests: prompt builders, schema validation, 8-question enforcement, idempotency skip, spend cap

**Key decisions made during Phase 2:**
- AlexJS is optional: pipeline proceeds without Node.js installed; warnings count defaults to 0 with `skipped=True` flag in `meta.json`
- TTS is optional: if no TTS provider configured, MP3 files are simply not generated; content endpoints still serve lesson JSON
- `resolve_curriculum_id` defaults to `default-2026-g{grade}` based on JWT grade (school enrolment resolver added in Phase 9)
- Audio endpoint returns local path URL in dev (`/static/content/...`), S3 pre-signed URL in production — same interface to mobile client

**How to run:**
```bash
./dev_start.sh          # start server → http://localhost:8000/api/docs
```

**How to test:**
```bash
./dev_start.sh test     # runs 52/52 automated tests (no API key needed)
```

**Run the content pipeline (requires Anthropic API key):**
```bash
source venv/bin/activate
export ANTHROPIC_API_KEY=sk-ant-...
export CONTENT_STORE_PATH=./data/content_store
export REVIEW_AUTO_APPROVE=true        # auto-publish in dev (skips review queue)

python pipeline/seed_default.py --year 2026          # seed curricula into DB
python pipeline/build_grade.py --grade 8 --lang en   # generate Grade 8 English content
python pipeline/build_unit.py \                      # regenerate one unit
  --curriculum-id default-2026-g8 \
  --unit G8-MATH-001 --lang en --force
```

**New endpoints available after Phase 2:**

| Endpoint | Auth | Notes |
|---|---|---|
| `GET /api/v1/content/{unit_id}/lesson` | Student JWT | 404 if unpublished; 402 after 2 lessons (free tier) |
| `GET /api/v1/content/{unit_id}/quiz` | Student JWT | 8 questions; rotates sets 1→2→3→1 per student |
| `GET /api/v1/content/{unit_id}/tutorial` | Student JWT | Remediation content |
| `GET /api/v1/content/{unit_id}/experiment` | Student JWT | 404 if unit has no lab |
| `GET /api/v1/content/{unit_id}/lesson/audio` | Student JWT | Returns URL (never raw bytes) |
| `POST /api/v1/content/{unit_id}/report` | Student JWT | Flag problematic content |
| `POST /api/v1/content/{unit_id}/feedback/marked` | Student JWT | Annotate content |
| `GET /api/v1/app/version` | Student JWT | `{min_version, latest_version}` |

**Verify content is being served (after pipeline run):**
```bash
# Generate a student JWT then call the lesson endpoint
curl -H "Authorization: Bearer <token>" \
  http://localhost:8000/api/v1/content/G8-MATH-001/lesson
```

*Last updated: 2026-03-25 — documentation audit complete after Phase 2*

---

### Phase 3 — Progress Tracking (2026-03-25)

**Branch:** `main` (branched snapshot `20260325_1608`)
**Tests:** 73/73 passing

**Files created:**
- `backend/alembic/versions/0003_phase3_progress_schema.py` — migration: `progress_sessions`, `progress_answers`, `lesson_views` tables + `mv_student_curriculum_progress` materialized view
- `backend/src/progress/__init__.py` — module init
- `backend/src/progress/schemas.py` — Pydantic request/response schemas for all progress endpoints
- `backend/src/progress/service.py` — DB logic: `create_session`, `record_answer_sync`, `end_session`, `get_raw_history`; `attempt_number` computed server-side
- `backend/src/progress/router.py` — `POST /progress/session`, `POST /progress/session/{id}/answer`, `POST /progress/session/{id}/end`, `GET /progress/student`
- `backend/src/student/__init__.py` — module init
- `backend/src/student/schemas.py` — Pydantic schemas for dashboard, progress map, stats
- `backend/src/student/service.py` — aggregation logic: dashboard (L2 Redis cached 60s), progress map (mat. view), stats (with daily_activity); streak read/write helpers
- `backend/src/student/router.py` — `GET /student/dashboard`, `GET /student/progress`, `GET /student/stats?period=7d|30d|all`
- `backend/tests/test_progress.py` — 9 progress endpoint tests
- `backend/tests/test_student.py` — 12 student aggregate endpoint tests
- `mobile/src/api/progress_client.py` — async HTTP client for all progress + student endpoints
- `mobile/src/ui/ResultScreen.py` — result screen showing backend-confirmed score; never computes score client-side
- `mobile/src/ui/ProgressDashboardScreen.py` — streak badge, subject rings, next unit card, recent activity
- `mobile/src/ui/CurriculumMapScreen.py` — full curriculum map with colour-coded status badges; tapping navigates to lesson
- `mobile/src/ui/StatsScreen.py` — period picker (7d/30d/all), stat cards, daily activity breakdown

**Files modified:**
- `backend/src/auth/tasks.py` — added `write_progress_answer_task` (fire-and-forget answer write), `update_streak_task` (Redis streak update), `refresh_progress_view_task` (REFRESH MATERIALIZED VIEW CONCURRENTLY + dashboard cache invalidation)
- `backend/main.py` — registered `progress_router` and `student_router`

**Bug fixes applied during implementation:**
- `curriculum_units` has no `grade` column — fixed service to JOIN `curricula` table to obtain grade
- `DATE(started_at)` in index expression rejected by PostgreSQL (requires IMMUTABLE) — changed to index on `started_at` directly
- `db_conn` fixture uses uncommitted transaction not visible to the client pool — fixed test helpers to use `client._transport.app.state.pool` for student inserts

**Key decisions made during Phase 3:**
- Answer writes are fire-and-forget Celery tasks; `POST /progress/session/{id}/answer` returns `200 OK` before DB write (CLAUDE.md rule: progress writes never delay client)
- `attempt_number` is always `COUNT(completed sessions) + 1` server-side — client-supplied values are ignored
- Streak stored as JSON in Redis (`streak:{student_id}`); never recomputed from raw sessions on request (D-17)
- `QUIZ_PASS_THRESHOLD = 0.60` (≥60% score required to mark unit `completed`)
- `REFRESH MATERIALIZED VIEW CONCURRENTLY` used so reads aren't blocked during refresh
- Dashboard cached 60s at L2 Redis; invalidated on every `POST /progress/session/{id}/end`

**How to run:**
```bash
./dev_start.sh
```

**How to test:**
```bash
./dev_start.sh test    # 73/73 passing
```

**Manual testing via Swagger:**
```bash
# 1. Get a student JWT (POST /api/v1/auth/exchange)
# 2. Start a session
curl -X POST http://localhost:8000/api/v1/progress/session \
  -H "Authorization: Bearer <token>" \
  -d '{"unit_id":"G8-MATH-001","curriculum_id":"default-2026-g8"}'
# 3. Record answers, end session, view dashboard
curl http://localhost:8000/api/v1/student/dashboard -H "Authorization: Bearer <token>"
```

**New endpoints added in Phase 3:**

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/v1/progress/session` | Student JWT | Open a quiz session; returns `attempt_number` (server-computed) |
| `POST` | `/api/v1/progress/session/{id}/answer` | Student JWT | Record answer (fire-and-forget Celery write) |
| `POST` | `/api/v1/progress/session/{id}/end` | Student JWT | Close session; compute `score` + `passed`; triggers streak + view refresh |
| `GET` | `/api/v1/progress/student` | Student JWT | Full raw history (sessions + answers), newest first |
| `GET` | `/api/v1/student/dashboard` | Student JWT | Aggregated dashboard card (streak, subject %, next unit, recent activity); Redis-cached 60s |
| `GET` | `/api/v1/student/progress` | Student JWT | Curriculum map with per-unit status badges (`not_started` / `in_progress` / `needs_retry` / `completed`) |
| `GET` | `/api/v1/student/stats` | Student JWT | Usage stats for `?period=7d\|30d\|all`; includes `daily_activity[]` and streak counters |

*Last updated: 2026-03-25*
