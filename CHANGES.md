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

### Phase 1 — Backend Foundation (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0001_phase1_initial_schema.py` — Creates core identity tables: `schools` (stub FK target; school_id, name, contact_email, country, enrolment_code, status), `students` (Auth0 external users; student_id, external_auth_id, auth_provider, name, email, grade, locale, account_status, school_id, lessons_accessed; indexes on external_auth_id, school_id, account_status), `teachers` (Auth0 external users; teacher_id, school_id, external_auth_id, auth_provider, name, email, role; index on school_id), `admin_users` (internal product team bcrypt auth; admin_user_id, email, password_hash, role, totp_secret, last_login_at), `parental_consents` (COPPA compliance; consent_id, student_id FK, guardian_email, status, consent_token, token_expires_at, consented_at, ip_address; index on student_id), `audit_log` (immutable event trail; id, timestamp, event_type, actor_type/id, target_type/id, metadata JSONB, ip_address, correlation_id; indexes on timestamp, target, actor). Enums: `account_status` (pending/active/suspended/deleted), `admin_role` (developer/tester/product_admin/super_admin), `consent_status` (pending/granted/denied).
- `backend/src/auth/__init__.py` — Package init
- `backend/src/auth/schemas.py` — Auth0ExchangeRequest/Response, TokenRefreshRequest/Response, AdminLoginRequest/Response, ForgotPasswordRequest, ResetPasswordRequest, LogoutRequest
- `backend/src/auth/service.py` — `exchange_auth0_token` (JWKS verify → upsert student/teacher → issue internal JWT), `refresh_token` (Redis validate → rotate), `admin_login` (bcrypt + lockout via Redis INCR), `admin_reset_password_request` (Redis TTL token), `delete_student_gdpr`
- `backend/src/auth/router.py` — Student track: `POST /auth/exchange`, `POST /auth/refresh`, `POST /auth/logout`, `POST /auth/forgot-password`, `POST /auth/reset-password`, `DELETE /auth/me`. Teacher track: `POST /auth/teacher/exchange`
- `backend/src/auth/admin_router.py` — `POST /admin/auth/login`, `POST /admin/auth/refresh`, `POST /admin/auth/reset-password/request`
- `backend/src/auth/dependencies.py` — `get_current_student`, `get_current_teacher`, `get_current_admin` JWT verification dependencies; suspension check via Redis `suspended:{id}` key
- `backend/src/auth/tasks.py` — Celery app setup; `send_auth0_block_task`, `send_reset_email_task`; Celery Beat schedule
- `backend/src/account/router.py` — `PUT /admin/students/{student_id}/status`, `PUT /admin/teachers/{teacher_id}/status`, `PUT /admin/schools/{school_id}/status` (product_admin/super_admin only)
- `backend/src/account/schemas.py` — `AccountStatusUpdateRequest/Response`
- `backend/src/curriculum/router.py` — `GET /curriculum` (all grades + unit counts), `GET /curriculum/{grade}` (full unit tree for a grade)
- `backend/src/curriculum/schemas.py` — `CurriculumUnit`, `CurriculumGrade`, `CurriculumListResponse`
- `backend/src/core/cache.py` — L1 TTLCache wrapper (cachetools); L2 Redis helpers
- `backend/src/core/db.py` — `get_db` async context manager (asyncpg connection from pool)
- `backend/src/core/redis_client.py` — `get_redis` dependency
- `backend/src/core/events.py` — `emit_event` structured log + Prometheus counter; `write_audit_log` Celery dispatch
- `backend/src/core/observability.py` — `CorrelationIdMiddleware`, Prometheus metrics registry, `GET /health`, `GET /metrics` routes
- `backend/src/core/permissions.py` — `ROLE_PERMISSIONS` static map; `require_permission()` FastAPI dependency
- `backend/config.py` — pydantic-settings: DATABASE_URL, REDIS_URL, JWT_SECRET, TEACHER_JWT_SECRET, ADMIN_JWT_SECRET, AUTH0_DOMAIN, AUTH0_AUDIENCE, STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, S3_BUCKET_NAME, CLOUDFRONT_DISTRIBUTION_ID, FCM_SERVER_KEY, SENTRY_DSN, CONTENT_STORE_PATH, METRICS_TOKEN, and all pool/TTL/limit settings
- `backend/main.py` — FastAPI app factory; asyncpg + aioredis lifespan; CORS; CorrelationIdMiddleware; global exception handlers; routers registered under `/api/v1`
- `backend/tests/conftest.py` — `db_conn` fixture (Alembic upgrade head on test DB, yield connection, rollback per test); `app` + `client` httpx fixtures; pool fixture on `app.state`
- `backend/tests/test_auth.py` — 14 tests: exchange creates student, upsert on second login, under-13 blocked pending, suspended account, refresh valid, refresh invalid, logout deletes token, forgot-password (existing + nonexistent email), admin login valid/wrong-password, lockout after 5 failures, lockout reset on success, student token rejected on admin endpoint
- `backend/tests/test_health.py` — 5 tests: health ok, correlation ID header present, provided correlation ID echoed, metrics requires token, metrics with valid token
- `backend/tests/test_curriculum.py` — 9 tests: list returns all grades, list has unit counts, get grade 8/5/12, invalid grade too low/too high, lab units present, L1 cache hit
- `backend/tests/test_account.py` — 10 tests: suspend/reactivate student, non-admin rejected, developer role rejected, suspend nonexistent 404, suspend already suspended 409, cannot change deleted account, suspend teacher, suspend school, school suspension requires admin

**Key decisions:**

- `POST /auth/forgot-password` always returns 200 regardless of whether the email exists — returning different responses leaks registered emails.
- Suspension enforced via Redis `suspended:{id}` key with NO TTL — explicitly deleted on reactivation only; a TTL would silently restore access after expiry.
- Two-track auth: Auth0 (external) for students and teachers; local bcrypt for internal team signed with separate `ADMIN_JWT_SECRET` so a compromised student JWT can never reach admin endpoints.
- `GET /curriculum` reads from DB with L1 TTLCache; zero DB queries on cache-warm requests.

**Test count:** 38 (0 → 38)

---

### Phase 2 — Content Pipeline + English Delivery (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0002_phase2_content_schema.py` — Creates: `curricula` (curriculum_id, grade, year, name, is_default, school_id; indexes on grade and school), `curriculum_units` (composite PK unit_id+curriculum_id, subject, title, description, has_lab, sort_order; index on curriculum_id), `content_subject_versions` (version_id, curriculum_id, subject, version_number, status, alex_warnings_count, generated_at, published_at, pipeline_run_id; unique on curriculum_id+subject+version_number), `content_reviews` (review_id, version_id, reviewer_id FK, action, notes, reviewed_at), `content_annotations` (annotation_id, version_id, unit_id, content_type, start/end_offset, marked_text, annotation_text, created_by/at), `content_blocks` (block_id, curriculum_id, unit_id, content_type, reason, blocked_by, blocked_at, unblocked_at; unique on curriculum_id+unit_id+content_type), `student_content_feedback` (feedback_id, student_id, unit_id, curriculum_id, content_type, category, message; index on unit), `app_versions` (platform, min_version, latest_version; seeded with all/0.1.0/0.1.0), `student_entitlements` (PK: student_id; plan, lessons_accessed, valid_until)
- `backend/src/content/__init__.py` — Package init
- `backend/src/content/schemas.py` — `LessonResponse`, `QuizQuestion`, `QuizResponse`, `TutorialResponse`, `AudioResponse` (pre-signed URL only), `AppVersionResponse`
- `backend/src/content/service.py` — `get_lesson` (L1 → L2 → file read; checks published status + no active block; increments `lessons_accessed` + checks free-tier limit), `get_quiz` (rotates 3 sets by attempt count mod 3), `get_tutorial`, `get_audio` (returns pre-signed S3 URL; never proxies bytes), `check_entitlement` (Redis `ent:{student_id}` cached 300s), `get_app_version`
- `backend/src/content/router.py` — `GET /content/{unit_id}/lesson`, `GET /content/{unit_id}/quiz`, `GET /content/{unit_id}/tutorial`, `GET /content/{unit_id}/audio`, `GET /content/version`; entitlement gating returns 402 on free-tier breach
- `pipeline/config.py` — `ANTHROPIC_API_KEY`, `TTS_API_KEY`, `CONTENT_STORE_PATH`, `CLAUDE_MODEL` (pinned to `claude-sonnet-4-6`)
- `pipeline/prompts.py` — `build_lesson_prompt`, `build_quiz_prompt`, `build_tutorial_prompt` per grade/subject/unit/lang
- `pipeline/build_grade.py` — CLI: `--grade N --lang en,fr,es [--force] [--dry-run]`; idempotency via `meta.json` content_version check; JSON schema validation before write; spend cap abort; AlexJS integration
- `pipeline/tts_worker.py` — Lesson text → MP3 via Amazon Polly / Google TTS; writes to `{CONTENT_STORE_PATH}/curricula/{curriculum_id}/{unit_id}/lesson_{lang}.mp3`
- `backend/tests/test_content.py` — 10 tests: lesson returns content, lesson unpublished 404, free-tier limit 402, quiz rotates sets, experiment not found for non-lab, experiment 200 for lab unit, report 200, app version, content rate limit, audio returns URL

**Key decisions:**

- Audio endpoint returns a pre-signed S3/CloudFront URL — MP3 bytes are never proxied through FastAPI (performance rule #3).
- Quiz sets rotated by `attempt_count mod 3` server-side — client cannot select a set.
- `check_entitlement` cached in Redis under `ent:{student_id}` with 300s TTL; invalidated on subscription change.
- Pipeline is idempotent: checks `meta.json` at unit start; skips if `content_version` matches and all expected files exist; `--force` overrides.
- Claude model pinned in `pipeline/config.py` — never use implicit "latest".

**Test count:** 52 (38 → 52, +14 content + pipeline tests)

---

### Phase 3 — Progress Tracking (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0003_phase3_progress_schema.py` — Creates: `progress_sessions` (session_id, student_id, unit_id, curriculum_id, grade, subject, started_at, ended_at, score, total_questions, completed, attempt_number, passed; indexes on student_id, unit_id, ended_at), `progress_answers` (answer_id, session_id FK, event_id UUID unique for offline dedup, question_id, student_answer, correct_answer, correct, ms_taken, recorded_at; index on session_id), `lesson_views` (view_id, student_id, unit_id, curriculum_id, duration_s, audio_played, experiment_viewed, started_at, ended_at; indexes on student_id and student+date). Materialized view `mv_student_curriculum_progress` (per student×unit roll-up: attempts, best_score, best_pct, status badge — completed/needs_retry/in_progress/not_started, last_attempt_at; UNIQUE index for concurrent refresh).
- `backend/src/progress/__init__.py` — Package init
- `backend/src/progress/schemas.py` — `SessionStartRequest/Response`, `AnswerRequest/Response`, `SessionEndRequest/Response`, `SessionHistoryResponse`
- `backend/src/progress/service.py` — `start_session` (computes `attempt_number` server-side as COUNT+1; rejects if open session exists for same unit), `record_answer` (ownership check; fire-and-forget via Celery; deduplicates on `event_id`), `end_session` (idempotent 409 on double-end; computes passed ≥ 70%; Celery dispatches MV refresh), `get_session_history`
- `backend/src/progress/router.py` — `POST /progress/session`, `POST /progress/answer`, `POST /progress/session/{session_id}/end`, `GET /progress/history`
- `backend/src/student/__init__.py` — Package init
- `backend/src/student/schemas.py` — `DashboardResponse` (streak, units in progress, recent activity), `CurriculumMapUnit`, `CurriculumMapResponse`, `StudentStatsResponse`
- `backend/src/student/service.py` — `get_dashboard` (L1+L2 cached 60s; reads `mv_student_curriculum_progress` + Redis streak hash; invalidated on session end), `get_curriculum_map` (full unit list with status badges from MV), `get_stats` (total sessions, pass rate, audio plays, streak)
- `backend/src/student/router.py` — `GET /student/dashboard`, `GET /student/progress`, `GET /student/stats`
- `backend/src/auth/tasks.py` — Added `refresh_mv_student_progress_task` (refreshes `mv_student_curriculum_progress` concurrently); added `update_student_streak_task` (Celery Beat daily at 01:00 UTC; increments Redis streak hash or resets if >1 day gap)
- `backend/tests/test_progress.py` — 10 tests: start session 201, attempt_number increments, record answer 200, wrong session 404, other student session 403, end session computes passed, not passed below threshold, double end 409, get history, progress requires auth
- `backend/tests/test_student.py` — 11 tests: dashboard empty defaults, dashboard with data, dashboard cache, curriculum map empty, curriculum map with statuses, stats empty, stats with sessions, streak in dashboard, progress map badges, student endpoint auth required, student_id from JWT only

**Key decisions:**

- `attempt_number` is computed server-side as `COUNT(*) + 1` from prior sessions; any client-supplied value is ignored.
- Progress answers use Celery fire-and-forget — `POST /progress/answer` returns 200 immediately without awaiting a DB write.
- `mv_student_curriculum_progress` uses UNIQUE index for `REFRESH MATERIALIZED VIEW CONCURRENTLY` so reads are never blocked during refresh.
- Student dashboard cached L1 (TTLCache 60s) + L2 (Redis 60s); invalidated by `end_session` to show updated badges.

**Test count:** 73 (52 → 73, +21 progress + student tests)

---

### Phase 4 — Offline Sync + Push Notifications + Analytics (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0004_phase4_push_notifications.py` — Creates: `push_tokens` (token_id, student_id FK, device_token, platform CHECK IN android/ios/web, created_at, updated_at; UNIQUE on student_id+device_token; index on student_id), `notification_preferences` (PK: student_id; streak_reminders, weekly_summary, quiz_nudges booleans with True defaults, updated_at)
- `backend/src/notifications/__init__.py` — Package init
- `backend/src/notifications/schemas.py` — `PushTokenRegisterRequest` (device_token, platform), `PushTokenRegisterResponse`, `NotificationPreferencesRequest/Response`
- `backend/src/notifications/service.py` — `register_push_token` (UPSERT ON CONFLICT; replaces old token for same student+device pair), `deregister_push_token` (DELETE by token value), `get_preferences` (returns defaults if no row), `update_preferences`
- `backend/src/notifications/router.py` — `POST /notifications/token` (register), `DELETE /notifications/token` (deregister), `GET /notifications/preferences`, `PUT /notifications/preferences`
- `backend/src/analytics/__init__.py` — Package init
- `backend/src/analytics/schemas.py` — `LessonStartRequest/Response`, `LessonEndRequest/Response`
- `backend/src/analytics/service.py` — `record_lesson_start` (INSERT into `lesson_views`; returns `view_id`), `record_lesson_end` (validates ownership + open state; Celery fire-and-forget write of duration_s, audio_played, experiment_viewed; 409 on double-end)
- `backend/src/analytics/router.py` — `POST /analytics/lesson/start`, `POST /analytics/lesson/end`
- `backend/src/auth/tasks.py` — Added `send_streak_reminder_task`, `send_weekly_summary_task`, `send_quiz_nudge_task` (Celery Beat tasks checking notification_preferences before FCM dispatch); added `record_lesson_end_task` (fire-and-forget DB write for lesson_views)
- `pipeline/build_grade.py` — Added `--lang fr,es` support; pipeline generates `lesson_fr.json`, `lesson_es.json`, `lesson_{lang}.mp3` per unit
- `backend/tests/test_analytics.py` — 7 tests: lesson start 201, lesson end fires-and-returns-200, ownership enforced, invalid view_id 400, nonexistent view 404, double end 409, auth required
- `backend/tests/test_notifications.py` — 7 tests: register token 200, upsert idempotent, invalid platform 422, deregister, get preferences returns defaults, update and get preferences, notifications require auth

**Key decisions:**

- `POST /analytics/lesson/end` dispatches a Celery task and returns 200 immediately — never awaits a DB write on the request path.
- Push token registration uses UPSERT so re-registering the same device is always safe and idempotent.
- Multi-language pipeline generates per-language files in parallel; `meta.json` tracks `langs_built` list; `--force` re-generates all languages even if `content_version` matches.

**Test count:** 87 (73 → 87, +14 analytics + notifications tests)

---

### Phase 5 — Subscription + Payments (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0005_phase5_subscriptions.py` — Creates: `subscriptions` (subscription_id, student_id FK, plan CHECK IN monthly/annual, status CHECK IN active/cancelled/past_due, stripe_customer_id, stripe_subscription_id UNIQUE, current_period_end, grace_period_end, created_at, updated_at; indexes on student_id, student+active partial, stripe_subscription_id), `stripe_events` (PK: stripe_event_id; event_type, processed_at, outcome CHECK IN ok/error, error_detail — idempotency + audit log)
- `backend/src/subscription/__init__.py` — Package init
- `backend/src/subscription/schemas.py` — `SubscriptionStatusResponse`, `CheckoutRequest` (plan), `CheckoutResponse` (checkout_url), `CancelResponse`
- `backend/src/subscription/service.py` — `get_subscription_status` (checks `subscriptions` table; returns free tier if no row), `create_checkout_session` (Stripe hosted checkout; creates/retrieves `stripe_customer_id`; stores pending subscription), `handle_webhook` (verifies `stripe.Webhook.construct_event` first; deduplicates via `stripe_events`; handles `checkout.session.completed` → activate, `invoice.payment_failed` → grace period, `customer.subscription.deleted` → cancel; invalidates `ent:{student_id}` Redis key on each state change), `cancel_subscription`
- `backend/src/subscription/router.py` — `GET /subscription/status`, `POST /subscription/checkout`, `POST /subscription/webhook` (raw body + Stripe-Signature header), `POST /subscription/cancel`
- `backend/tests/test_subscription.py` — 12 tests: status free when no subscription, status active subscription, checkout 503 when Stripe not configured, checkout returns URL, webhook rejects invalid signature, webhook deduplicates, webhook checkout.session.completed, webhook payment_failed sets grace period, webhook subscription.deleted cancels, cancel 404 when none, cancel success, endpoints require auth

**Key decisions:**

- Stripe webhook verifies signature via `stripe.Webhook.construct_event` before any processing — invalid signatures return 400 immediately.
- Webhook handler deduplicates by `stripe_event_id` (writes to `stripe_events` table; returns 200 if already processed) — ensures idempotency across Stripe retries.
- Entitlement cache key `ent:{student_id}` is invalidated on every subscription state change so the content endpoint reflects the new plan within one Redis TTL window.

**Test count:** 99 (87 → 99, +12 subscription tests)

---

### Phase 6 — Experiment Visualization (2026-03-26)

**Files created/modified:**

- `backend/src/content/router.py` — Added `GET /content/{unit_id}/experiment` endpoint; returns 404 if `has_lab=False` for the unit or if `experiment_{lang}.json` does not exist in the Content Store
- `backend/src/content/service.py` — Added `get_experiment` (checks `has_lab` flag on `curriculum_units`; reads `experiment_{lang}.json` from Content Store; caches in Redis under `content:experiment:{curriculum_id}:{unit_id}:{lang}`)
- `backend/src/content/schemas.py` — Added `ExperimentStep`, `ExperimentResponse` (steps list, materials list, safety_notes, expected_outcome, grade_level)
- `pipeline/prompts.py` — Added `build_experiment_prompt` (lab guide prompt for has_lab units; generates structured JSON with steps, materials, safety notes)
- `pipeline/build_grade.py` — Added lab detection: only generates `experiment_{lang}.json` when unit `has_lab=True`; non-lab units produce no experiment file
- `pipeline/build_unit.py` — New CLI: `--curriculum-id UUID --unit G8-MATH-001 --lang en`; regenerates a single unit; supports `--force`
- `backend/tests/test_content.py` — Added 2 experiment tests (test_experiment_not_found_for_non_lab_unit, test_experiment_returns_200_for_lab_unit); these were counted in Phase 2 test file but activated in Phase 6

**Key decisions:**

- Experiment content is only generated (and only requested) for units where `has_lab=True`; the mobile app detects availability by HTTP status — 200 = show ExperimentScreen, 404 = hide button.
- No new DB migration required — `has_lab` column was already present on `curriculum_units` from Phase 2.

**Test count:** 100 (99 → 100, +1 net new experiment path test)

---

### Phase 7 — Admin Dashboard + Analytics + Content Review (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0006_phase7_admin_review.py` — Adds `language_rating`, `content_rating` columns to `content_reviews`; adds `archived_at` to `content_subject_versions`; indexes on status and feedback category
- `backend/src/admin/__init__.py` — Package init
- `backend/src/admin/schemas.py` — All request/response models for 16 admin endpoints; datetime fields use Python `datetime` type (Pydantic serialises to ISO 8601)
- `backend/src/admin/service.py` — Full admin business logic: review queue, open/annotate/rate/approve/reject, publish/rollback (Redis + CDN invalidation), content blocks, feedback listing, subscription analytics (MRR/churn), struggle analytics, pipeline status, Datamuse + Merriam-Webster dictionary
- `backend/src/admin/router.py` — 16 admin endpoints using chained `_require(permission)` dependency that combines `get_current_admin` + RBAC in one dependency so FastAPI's dependency graph guarantees auth runs before the permission check
- `backend/src/core/permissions.py` — Fixed `async def dependency(request: Request)` type annotation (previously untyped, causing FastAPI to treat `request` as a query parameter → 422 on all admin endpoints)
- `backend/src/auth/tasks.py` — Added `regenerate_subject_task` Celery task; added pipeline queue route
- `backend/config.py` — Added `MW_API_KEY` optional setting for Merriam-Webster
- `backend/main.py` — Registered `admin_router` under `/api/v1`
- `backend/tests/test_admin.py` — 24 tests covering all 16 endpoints plus auth rejection

**Key decisions:**

- `_require(permission)` pattern in admin router chains `get_current_admin` as a sub-dependency of the permission check. This is required because FastAPI runs independent `Depends()` branches concurrently; `require_permission` reads `request.state.jwt_payload` which `get_current_admin` sets — so they must be ordered.
- Student JWTs on admin endpoints return 401 (not 403) because ADMIN_JWT_SECRET verification fails before RBAC is checked.
- Datetime fields in schemas use `datetime` type; Pydantic v2 handles ISO 8601 serialisation automatically — no manual `.isoformat()` calls needed for dict-based responses.
- Fixed test admin UUID (`_TEST_ADMIN_ID`) ensures the JWT's `admin_id` and the DB row's `admin_user_id` match, satisfying FK constraints on `content_reviews.reviewer_id` and `content_blocks.blocked_by`.

**Test count:** 124 (100 → 124, +24 admin tests)

---

### Phase 8 — School & Teacher + Curriculum Upload + Academic Year (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0007_phase8_school_curriculum.py` — ALTER TABLE curricula/curriculum_units to add Phase 8 columns: `source_type`, `status`, `restrict_access`, `created_by`, `activated_at` (curricula); `unit_name`, `objectives`, `lab_description`, `sequence`, `content_status` (curriculum_units). Makes legacy `title`/`description` columns nullable.
- `backend/src/school/__init__.py` — Package init
- `backend/src/school/schemas.py` — Request/response models: `SchoolRegisterRequest/Response`, `SchoolProfileResponse`, `TeacherInviteRequest/Response`
- `backend/src/school/service.py` — `register_school` (school + school_admin teacher in one transaction, issues JWT immediately), `fetch_school` (school-scoped), `invite_teacher` (pending account, placeholder ext_auth_id)
- `backend/src/school/router.py` — `POST /schools/register` (public), `GET /schools/{school_id}` (teacher JWT), `POST /schools/{school_id}/teachers/invite` (school_admin only)
- `backend/src/curriculum/upload_service.py` — `parse_xlsx`, `build_xlsx_template`, `create_curriculum_from_json`, `trigger_pipeline` (Celery dispatch + Redis job state), `get_pipeline_job_status`, `seed_default_curriculum` (ON CONFLICT DO UPDATE)
- `backend/src/curriculum/schemas.py` — Added `CurriculumUnitInput`, `CurriculumUploadRequest/Response`, `UploadError`, `PipelineTriggerRequest/Response`, `PipelineJobStatusResponse`
- `backend/src/curriculum/router.py` — Added `GET /curriculum/template`, `POST /curriculum/upload`, `POST /curriculum/upload/xlsx`, `POST /curriculum/pipeline/trigger` (202), `GET /curriculum/pipeline/{job_id}/status`; route ordering critical: template + pipeline routes registered before `/{grade}`
- `backend/src/auth/tasks.py` — Added `send_pipeline_email_task`, `run_curriculum_pipeline_task` (pipeline queue), `promote_student_grades` (Celery Beat at 00:05 UTC daily; no-op unless today == `GRADE_PROMOTION_DATE`)
- `backend/config.py` — Added `GRADE_PROMOTION_DATE` (MM-DD format), `SENDGRID_API_KEY`, `EMAIL_FROM`
- `backend/main.py` — Registered `school_router` under `/api/v1`
- `backend/requirements.txt` — Added `openpyxl==3.1.5`
- `pipeline/seed_default.py` — Rewritten to use `seed_default_curriculum()` from upload_service; maps data JSON format (title → unit_name, description → single objective); supports `--grade` flag for single-grade seeding
- `backend/tests/test_school.py` — 9 tests covering registration, profile (school-scoped), teacher invite, RBAC, and auth rejection
- `backend/tests/test_curriculum_upload.py` — 21 tests covering template download, JSON upload, XLSX upload, pipeline trigger/status, parse_xlsx unit tests, and promote_student_grades no-op checks

**Key decisions:**

- Migration 0007 uses ALTER TABLE (not CREATE TABLE) because `curricula` and `curriculum_units` tables already exist from Phase 2 (0002) with a simpler schema. Legacy `title`/`description` columns made nullable to allow new inserts that use `unit_name`/`objectives` instead.
- `trigger_pipeline` catches Celery dispatch exceptions and logs a warning — pipeline dispatch failure is non-fatal; the job state is already written to Redis before the dispatch attempt.
- Teacher token from `POST /schools/register` uses `JWT_SECRET` (not `ADMIN_JWT_SECRET`) — school admins are teachers on the teacher auth track.
- `HTTPBearer(auto_error=True)` returns 403 (not 401) when no Authorization header is provided; 401 is only returned on invalid/expired tokens. Tests that check "no auth" correctly assert 403.
- Curriculum upload tests pre-register a school via `POST /schools/register` to get a teacher token with a valid school_id that satisfies the `curricula.school_id` FK constraint.

**Test count:** 159 (124 → 159, +35 Phase 8 tests)

---

### Phase 9 — Student–School Association + Curriculum Routing (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0008_phase9_enrolment.py` — Creates `school_enrolments` table (school_id FK, student_email, student_id FK nullable, status pending/active, unique on school_id+email); indexes on school_status, email, student_id
- `backend/src/school/enrolment_service.py` — `upload_roster` (upsert email list; auto-links already-registered students), `get_roster` (all rows for school), `link_student` (called on student login; matches email to pending enrolment, sets school_id on student)
- `backend/src/school/schemas.py` — Added `EnrolmentUploadRequest/Response`, `EnrolmentRosterItem`, `EnrolmentRosterResponse`
- `backend/src/school/router.py` — Added `POST /schools/{school_id}/enrolment` and `GET /schools/{school_id}/enrolment` (school_admin only for both)
- `backend/src/curriculum/resolver.py` — `get_curriculum_id` FastAPI dependency; `_resolve_from_db` resolves active school curriculum (or default fallback); enforces `restrict_access` via enrolment check (403 if absent); Redis-cached 300s under `cur:{student_id}`; `invalidate_resolver_cache_for_school` helper
- `backend/src/curriculum/schemas.py` — Added `CurriculumActivateResponse`
- `backend/src/curriculum/router.py` — Added `PUT /curriculum/{curriculum_id}/activate`; archives previous active curriculum for same (school_id, grade, year); invalidates `cur:*` cache for enrolled students
- `backend/src/auth/router.py` — After `upsert_student`, calls `link_student()` to auto-link any pending enrolment by email; re-fetches student to capture school_id set by linking
- `backend/tests/test_enrolment.py` — 17 tests covering roster upload/dedup, roster get, activation (archives previous, wrong-school 403), resolver unit tests (default fallback, school curriculum, restrict_access 403)

**Key decisions:**

- `link_student` is called on every Auth0 exchange, not just first login: idempotent (checks `status='pending'`), so safe to call on login.
- `restrict_access=False` (default): any student with the right school_id can access the school curriculum without an enrolment row.
- `restrict_access=True`: requires an active `school_enrolments` row; raises 403 "not_enrolled" otherwise.
- Curriculum resolver is cached per-student in Redis (`cur:{student_id}`); invalidated on activate (for all enrolled students) and would also be invalidated on school transfer.
- `PUT /curriculum/{curriculum_id}/activate` parses asyncpg's "UPDATE N" string to count archived curricula.

**Test count:** 176 (159 → 176, +17 Phase 9 tests)

---

### Phase 10 — Extended Analytics + Student Feedback (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0009_phase10_feedback.py` — Creates `feedback` table (student_id FK, category CHECK IN content/ux/general, unit_id, curriculum_id, message, rating 1-5, reviewed bool, reviewed_by FK admin_users); indexes on student_id and reviewed
- `backend/src/feedback/__init__.py` — Package init
- `backend/src/feedback/schemas.py` — `FeedbackSubmitRequest` (category pattern, message max 500, rating 1-5 optional), `FeedbackSubmitResponse`, `AdminFeedbackItem`, `AdminFeedbackPagination`, `AdminFeedbackListResponse`
- `backend/src/feedback/service.py` — `check_and_increment_rate_limit` (Redis INCR+EXPIRE, 5/student/hour key `feedback:rate:{student_id}:{hour}`), `submit_feedback` (INSERT returning feedback_id), `list_feedback` (paginated + filterable by category/unit/curriculum/reviewed)
- `backend/src/feedback/router.py` — `POST /feedback` (student JWT, 429 if rate-limited), `GET /admin/feedback` (admin, requires `feedback:view`)
- `backend/src/core/permissions.py` — Added `"feedback:view"` to `product_admin` permissions set
- `backend/src/analytics/schemas.py` — Added `PerUnitStudentMetric`, `ImprovementPoint`, `StudentMetricsResponse`, `PerUnitClassMetric`, `ClassMetricsResponse`
- `backend/src/analytics/service.py` — Added `get_student_metrics` (summary counters + per-unit breakdown + improvement trajectory from progress_sessions + lesson_views); added `get_class_metrics` (enrolled student aggregation with struggle_flag; fixed $1-based placeholder numbering bug); `_STRUGGLE_PASS_THRESHOLD = 50%`, `_STRUGGLE_ATTEMPTS_THRESHOLD = 2`
- `backend/src/analytics/router.py` — Added `GET /analytics/student/me` (student JWT), `GET /analytics/school/{school_id}/class` (teacher JWT, school-scoped 403 enforcement)
- `backend/main.py` — Registered `feedback_router` under `/api/v1`
- `backend/tests/test_feedback.py` — 12 tests covering submit (valid, categories, invalid category 422, message too long 422), rate limiting (429 on 6th), admin list (pagination, category filter, reviewed filter, wrong-role 403, no-auth 403)
- `backend/tests/test_analytics_extended.py` — 9 tests covering student/me (empty defaults, with data, auth required), class metrics (empty school, with data, struggle_flag low-pass-rate, wrong-school 403, auth required)

**Key decisions:**

- Rate limit key includes the hour bucket (`%Y%m%d%H`) so the 5/hour window rolls naturally without a scheduled cleanup job.
- `struggle_flag` is `True` when `first_attempt_pass_rate_pct < 50%` OR `mean_attempts_to_pass > 2` — either condition suffices.
- `get_class_metrics` fetches enrolled student IDs first, then uses `ANY(ARRAY[...]::uuid[])` to filter progress data — avoids a JOIN to `school_enrolments` inside the metrics query.
- Placeholder numbering bug fixed: original code used `$2` as first student ID placeholder (no `$1` in the query), causing `IndeterminateDatatypeError`. Fixed to start at `$1` with grade/subject appended after.

**Test count:** 197 (176 → 197, +21 Phase 10 tests)

---

### Phase 11 — Teacher Reporting Dashboard (2026-03-26)

**Files created/modified:**

- `backend/alembic/versions/0010_phase11_reports.py` — Creates `report_alert_settings` (school_id PK, threshold columns with defaults), `digest_subscriptions` (unique per school+teacher, enabled flag, timezone), `report_alerts` (triggered alert log with JSONB details, acked flag); creates 3 materialized views (`mv_class_summary`, `mv_student_progress`, `mv_feedback_summary`) with UNIQUE indexes for concurrent refresh
- `backend/src/reports/__init__.py` — Package init
- `backend/src/reports/schemas.py` — All Pydantic response models: `OverviewReport`, `UnitReport`, `StudentReport`, `CurriculumHealthReport`, `CurriculumHealthUnit`, `FeedbackReport`, `FeedbackGroup`, `TrendsReport`, `WeekSnapshot`, `ExportRequest/Response`, `AlertListResponse`, `AlertSettings/Response`, `DigestSubscribeRequest/Response`, `RefreshResponse`
- `backend/src/reports/service.py` — Full reporting logic: `get_overview` (active students, lesson views, quiz pass rates, audio play rate, struggle/no-activity unit lists, unreviewed feedback count), `get_unit_report` (per-unit pass rate, attempt distribution, score percentile, top struggling students), `get_student_report` (per-student unit history with pass/fail/in-progress status), `get_curriculum_health` (all units ranked by health tier), `get_feedback_report` (feedback grouped by unit, filterable), `get_trends` (week-over-week snapshots), `refresh_materialized_views`, `save_alert_settings`, `get_alerts`, `subscribe_digest`, `trigger_export`
- `backend/src/reports/router.py` — 12 endpoints with `_check_school()` ownership enforcement; all require teacher JWT; `POST /refresh` available to both school_admin and teacher; `GET /download/{export_id}` serves completed CSV files
- `backend/src/auth/tasks.py` — Added Celery Beat schedule entries (nightly refresh 02:00 UTC, alert eval 06:00 UTC, weekly digest Monday 08:00 UTC); added `refresh_report_views_task`, `evaluate_report_alerts_task`, `send_weekly_digest_task`, `export_report_task`
- `backend/main.py` — Registered `reports_router` under `/api/v1`
- `backend/tests/test_reports.py` — 18 tests covering all 12 endpoints

**Key decisions:**

- Reports query underlying tables directly (not materialized views); MVs exist as production performance layer only — this simplifies tests (no MV refresh step) and ensures correctness.
- Period mapping: `7d`=7 days, `30d`=30 days, `term`=September 1 of current/previous academic year. Trend periods: `4w`=4 weeks, `12w`=12 weeks, `term`=16 weeks.
- Health tier logic: Healthy ≥70% first-attempt pass AND ≤1.5 avg attempts; Watch 50–70% or 1.5–2.0 avg; Struggling <50% or >2.0 avg; No Activity = zero lesson views.
- Export flow: fire-and-forget Celery task writes CSV to `CONTENT_STORE_PATH/exports/{export_id}.csv`; separate `GET /download/{export_id}` endpoint serves the completed file.
- `AND FALSE` enrolled filter: when a school has no enrolled students, the trends query uses `AND FALSE` as the student filter so the week-by-week loop still runs and returns zero-valued entries for all requested weeks (avoids empty response).

**Bugs fixed:**

- `get_overview` feedback placeholder off-by-one: feedback query only receives `*id_uuids` (no `start` param), so placeholders must start at `$1`, not `$2`. Created separate `fb_placeholders` variable independent of the main `placeholders` string.
- `get_trends` early return on empty enrollment: removed `if not enrolled: return ...` early exit; precomputed `enrolled_filter` string with `AND FALSE` fallback ensures all `n_weeks` iterations always run.

**Test count:** 215 (197 → 215, +18 Phase 11 tests)

---

## Pending

All planned items through Phase 11 are complete. No further phases are defined.

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

*Last updated: 2026-03-25 — documentation audit complete after Phase 2 and Phase 5 complete*

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

---

## Phase 4 Implementation Log — Offline Sync + Push Notifications + Analytics

**Date:** 2026-03-25
**Branch:** main (snapshot branch `20260325_XXXX` created after commit)
**Tests after phase:** 87 passed, 0 failed

### Files Created

| File | Purpose |
|---|---|
| `backend/alembic/versions/0004_phase4_push_notifications.py` | `push_tokens` + `notification_preferences` tables |
| `backend/src/notifications/__init__.py` | Package marker |
| `backend/src/notifications/schemas.py` | Pydantic schemas for token register/delete and preferences |
| `backend/src/notifications/service.py` | DB helpers: `register_token`, `deregister_token`, `get_preferences`, `update_preferences`, `send_push_notification` (FCM HTTP v1) |
| `backend/src/notifications/router.py` | POST/DELETE `/notifications/token`, GET/PUT `/notifications/preferences` |
| `backend/src/analytics/__init__.py` | Package marker |
| `backend/src/analytics/schemas.py` | Pydantic schemas for lesson start/end events |
| `backend/src/analytics/service.py` | DB helpers: `start_lesson_view`, `verify_view_owner`, `end_lesson_view` |
| `backend/src/analytics/router.py` | POST `/analytics/lesson/start` (201), POST `/analytics/lesson/end` (fire-and-forget) |
| `backend/tests/test_notifications.py` | 7 tests: token register/upsert/delete, preferences defaults/update/auth |
| `backend/tests/test_analytics.py` | 7 tests: start/end lifecycle, ownership, 400/404/409/auth |
| `mobile/src/logic/EventQueue.py` | SQLite offline event queue: `enqueue`, `pending`, `mark_sent`, `purge_sent` |
| `mobile/src/logic/SyncManager.py` | Flush queue on foreground/network restore; dispatches `progress_answer` + `lesson_end` to API |
| `mobile/src/ui/SettingsScreen.py` | Language picker (clears content cache on change) + notification preference toggles |

### Files Modified

| File | Change |
|---|---|
| `backend/config.py` | Added `FCM_SERVER_KEY`, `S3_BUCKET_NAME`, `CLOUDFRONT_DISTRIBUTION_ID` (all Optional) |
| `backend/src/auth/tasks.py` | Added `write_lesson_end_task`, `send_push_notification_task`, `check_streak_reminders`, `check_quiz_nudges`, `send_weekly_summary`; added Beat schedule for 3 notification tasks; imported `crontab` |
| `backend/src/content/service.py` | Added `invalidate_cdn_path()` — triggers CloudFront invalidation on content version bumps (skips silently if `CLOUDFRONT_DISTRIBUTION_ID` unset) |
| `backend/main.py` | Registered `notifications_router` and `analytics_router` at `/api/v1` |
| `mobile/src/api/progress_client.py` | Added `start_lesson_view`, `end_lesson_view` (sync, for SyncManager), `register_push_token`, `get_notification_preferences`, `update_notification_preferences` |

### Key Decisions

- **`POST /analytics/lesson/end` is fire-and-forget** (same pattern as `/progress/answer`). Ownership is verified synchronously; the actual DB write is a Celery task. This avoids blocking the student's response on a DB write.
- **SyncManager debounces concurrent flushes** — a second `flush()` call while one is running is a no-op (guarded by `threading.Lock`). Events are processed oldest-first.
- **EventQueue `event_id` forwarded to backend** — the UUID is generated on enqueue and sent as the `event_id` field in the API request. Backend uses `ON CONFLICT DO NOTHING` for deduplication.
- **Language picker clears `LocalCache`** — locale change must clear cached content so the next lesson load fetches the correct language file from backend. The JWT locale is still authoritative for content requests (server-side); the stored preference is sent on next token refresh (Phase 5).
- **CDN invalidation is optional** — `invalidate_cdn_path()` silently skips when `CLOUDFRONT_DISTRIBUTION_ID` is unset (local dev / CI). In production, called from the admin pipeline trigger after content version bumps.
- **Beat tasks are best-effort** — streak reminder, quiz nudge, and weekly summary tasks catch all exceptions and log a warning rather than crashing the beat worker.

### How to Run

```bash
# Start dev environment
./dev_start.sh

# Run all tests
./dev_start.sh test

# Register push token (example)
curl -X POST http://localhost:8000/api/v1/notifications/token \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"device_token":"fcm-abc123","platform":"android"}'

# Start lesson view
curl -X POST http://localhost:8000/api/v1/analytics/lesson/start \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"unit_id":"G8-MATH-001","curriculum_id":"default-2026-g8"}'
```

### New Endpoints (Phase 4)

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/v1/notifications/token` | Student JWT | Register/refresh FCM device token |
| `DELETE` | `/api/v1/notifications/token` | Student JWT | Deregister FCM device token |
| `GET` | `/api/v1/notifications/preferences` | Student JWT | Get notification opt-in flags (defaults: all true) |
| `PUT` | `/api/v1/notifications/preferences` | Student JWT | Update notification opt-in flags |
| `POST` | `/api/v1/analytics/lesson/start` | Student JWT | Record lesson open; returns `view_id` |
| `POST` | `/api/v1/analytics/lesson/end` | Student JWT | Record lesson close with duration/flags (fire-and-forget) |

*Last updated: 2026-03-25*

---

## Phase 5 Implementation Log — Subscription + Payments

**Date:** 2026-03-25
**Branch:** main (snapshot branch created after commit)
**Tests after phase:** 99 passed, 0 failed (+12 new)

### Files Created

| File | Purpose |
|---|---|
| `backend/alembic/versions/0005_phase5_subscriptions.py` | `subscriptions` + `stripe_events` tables; partial index on active subs; unique on `stripe_subscription_id` |
| `backend/src/subscription/__init__.py` | Package marker |
| `backend/src/subscription/schemas.py` | Pydantic schemas: `CheckoutRequest`, `CheckoutResponse`, `SubscriptionStatusResponse`, `CancelResponse` |
| `backend/src/subscription/service.py` | `create_checkout_session`, `cancel_stripe_subscription`, `expire_entitlement_cache`, `already_processed`, `log_stripe_event`, `get_subscription_status`, `activate_subscription`, `update_subscription_status`, `cancel_subscription_db`, `handle_payment_failed`, `cancel_active_subscription_for_student` |
| `backend/src/subscription/router.py` | `GET /subscription/status`, `POST /subscription/checkout`, `POST /subscription/webhook`, `DELETE /subscription` |
| `backend/tests/test_subscription.py` | 12 tests: free status, active status, checkout 503/200, webhook signature, dedup, checkout.session.completed, invoice.payment_failed grace, subscription.deleted cancel, cancel 404/200, auth guard |
| `mobile/src/api/subscription_client.py` | Async HTTP client: `get_subscription_status`, `get_checkout_url`, `cancel_subscription` |
| `mobile/src/ui/SubscriptionScreen.py` | Full paywall UI: two plan cards (Monthly/Annual), Stripe checkout via system browser, refresh/restore button, back navigation |

### Files Modified

| File | Change |
|---|---|
| `backend/config.py` | Added `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_MONTHLY_ID`, `STRIPE_PRICE_ANNUAL_ID` (all Optional[str]) |
| `backend/main.py` | Registered `subscription_router` at `/api/v1` |
| `backend/src/auth/router.py` | `DELETE /auth/account` now cancels active Stripe subscription before GDPR erasure (best-effort, never blocks deletion) |

### Key Decisions

- **Stripe webhook signature always verified first** — `stripe.Webhook.construct_event()` called before any processing; 400 on failure (CLAUDE.md rule #8). Webhook does NOT use `get_current_student` — authenticated by signature only.
- **Idempotent webhook handler** — `stripe_events` table with `stripe_event_id` as PRIMARY KEY; `already_processed()` checked before dispatch; `log_stripe_event()` uses `ON CONFLICT DO NOTHING` (CLAUDE.md rule #9).
- **3-day grace period on payment failure** — `invoice.payment_failed` sets `status='past_due'` and `grace_period_end = NOW() + 3 days`. Student retains content access until grace period ends (no code changes needed — entitlement service reads `valid_until` which is updated to `grace_period_end`).
- **Entitlement cache expired on every state change** — `expire_entitlement_cache(redis, student_id)` called in `activate_subscription`, `update_subscription_status`, `cancel_subscription_db`, and `handle_payment_failed`. Next content request re-fetches entitlement from DB.
- **Mobile never holds Stripe keys** — `mobile/src/api/subscription_client.py` calls backend REST only; checkout URL is fetched from backend and opened in system browser (`webbrowser.open()`).
- **Stripe subscription cancelled before GDPR erasure** — `DELETE /auth/account` attempts Stripe cancellation first; on failure it logs a warning and proceeds with GDPR deletion (subscription is orphaned in Stripe, acceptable trade-off vs blocking deletion).
- **Webhook returns 200 after signature verification even on handler errors** — processing errors are logged to `stripe_events` with `outcome='error'` but HTTP response is still 200 so Stripe does not retry unnecessarily.

### New Endpoints (Phase 5)

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/v1/subscription/status` | Student JWT | Current plan + status + valid_until + lessons_accessed |
| `POST` | `/api/v1/subscription/checkout` | Student JWT | Create Stripe Checkout Session; returns `checkout_url` |
| `POST` | `/api/v1/subscription/webhook` | Stripe signature | Handles: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed` |
| `DELETE` | `/api/v1/subscription` | Student JWT | Cancel at period end; student retains access until `current_period_end` |

### Stripe Webhook Setup (production)

```bash
stripe listen --forward-to localhost:8000/api/v1/subscription/webhook

STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_MONTHLY_ID=price_...
STRIPE_PRICE_ANNUAL_ID=price_...
```

*Last updated: 2026-03-25*

---

## Phase 6 Implementation Log — Experiment Visualization

**Date:** 2026-03-25
**Branch:** main (snapshot branch created after commit)
**Tests after phase:** 100 passed, 0 failed (+1 new)

### Files Created

| File | Purpose |
|---|---|
| `mobile/src/ui/ExperimentScreen.py` | Step-by-step lab experiment screen: title, materials, safety notes, numbered steps with expected observations, reflection questions, conclusion prompt |

### Files Modified

| File | Change |
|---|---|
| `mobile/src/api/content_client.py` | Added `get_experiment(unit_id, token)` — returns experiment JSON; raises 404 for non-lab units |
| `mobile/src/ui/SubjectScreen.py` | Added "🔬 Experiment" button (hidden by default); `_probe_experiment_async()` called on enter; experiment JSON cached in LocalCache; `experiment_viewed` flag passed in analytics on leave |
| `backend/tests/test_content.py` | Added `test_experiment_returns_200_for_lab_unit` |

### Key Decisions

- **Experiment button hidden until probed** — `_probe_experiment_async()` fires on `on_enter`. Button only becomes visible (`opacity=1`, `disabled=False`) after backend returns HTTP 200. HTTP 404 silently keeps button hidden (non-lab unit).
- **Experiment cached in `LocalCache`** — uses `content_type="experiment"`, same keying as lesson (`unit_id, curriculum_id, lang, content_version`). Cache is checked before network call; populated after first successful fetch.
- **`experiment_viewed` flag in analytics** — `SubjectScreen.on_leave` passes `experiment_viewed=self._experiment_viewed` into the EventQueue payload. Flag is set to `True` only when user actually taps the button and navigates to `ExperimentScreen`.
- **No new Alembic migration** — experiment endpoint was already implemented in Phase 2. Phase 6 is purely mobile UI + client probe logic + caching.
- **Backend unchanged** — `GET /content/{unit_id}/experiment` was already complete with entitlement gating, L2 Redis caching, and published/block guards.
- **ExperimentScreen receives data from SubjectScreen** — never makes its own network call; data is passed via `set_experiment(data)` before navigation, keeping the experiment screen fast and cache-aware.

*Last updated: 2026-03-25*
