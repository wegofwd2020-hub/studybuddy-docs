# StudyBuddy OnDemand — Requirements

**Version:** 0.1.0
**Last updated:** 2026-03-23

---

## How to Use

- Each requirement has a stable **ID** that never changes, even if the description is refined.
- **Priority:** `P0` = must-have for launch / legal blocker · `P1` = important · `P2` = nice-to-have
- **Status:** `proposed` → `accepted` → `in-progress` → `done` | `deferred`
- **Phase** refers to the phase in [ARCHITECTURE.md](ARCHITECTURE.md#phased-implementation-plan) where the requirement is delivered.
- When an architectural decision is required to satisfy a requirement, record it in the [ADR Log](#architectural-decisions-log) and link back here.

---

## Functional Requirements

### FR-AUTH: Authentication & Identity

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-AUTH-001 | Student registration with name, email, password, grade, locale | P0 | 1 | accepted |
| FR-AUTH-002 | Student login with email + password → JWT | P0 | 1 | accepted |
| FR-AUTH-003 | JWT issue and refresh; locale + grade embedded in payload | P0 | 1 | accepted |
| FR-AUTH-004 | Email verification on registration (confirm before first login) | P1 | 1 | proposed |
| FR-AUTH-005 | Password reset: `POST /auth/forgot-password` → email link → `POST /auth/reset-password` | P0 | 1 | proposed |
| FR-AUTH-006 | Account deletion: `DELETE /auth/account` removes all student data (GDPR) | P0 | 5 | proposed |
| FR-AUTH-007 | Student profile update: `PATCH /student/profile` (name, locale, grade) | P1 | 3 | proposed |
| FR-AUTH-008 | Account lockout after 5 consecutive failed login attempts (15-minute cooldown) | P1 | 1 | proposed |
| FR-AUTH-009 | JWT signing key rotation with `kid` header for zero-downtime key rollover | P1 | 2 | proposed |
| FR-AUTH-010 | Parental consent flow for students under 13 (COPPA) — guardian email + consent record | P0 | 1 | proposed |

---

### FR-CURR: Curriculum

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-CURR-001 | `GET /curriculum` returns list of available grades | P0 | 1 | accepted |
| FR-CURR-002 | `GET /curriculum/{grade}` returns full subject + unit tree for grade | P0 | 1 | accepted |
| FR-CURR-003 | Backend discovers grades from `data/grade*_stem.json` filenames (no hardcoded grade list) | P0 | 1 | accepted |

---

### FR-CONTENT: Content Delivery

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-CONT-001 | `GET /content/{unit_id}/lesson` returns lesson JSON in student locale | P0 | 2 | accepted |
| FR-CONT-002 | `GET /content/{unit_id}/lesson/audio` returns signed URL or stream for MP3 | P0 | 4 | accepted |
| FR-CONT-003 | `GET /content/{unit_id}/quiz` returns one quiz set (8 questions, rotated) in student locale | P0 | 2 | accepted |
| FR-CONT-004 | `GET /content/{unit_id}/tutorial` returns remediation content in student locale | P1 | 2 | accepted |
| FR-CONT-005 | `GET /content/{unit_id}/experiment` returns experiment JSON or HTTP 404 if no lab | P1 | 6 | accepted |
| FR-CONT-006 | `GET /content/{unit_id}/practice` returns practice test set in student locale | P1 | 2 | accepted |
| FR-CONT-007 | Entitlement middleware: HTTP 402 after 2 distinct lesson views for free-tier students | P0 | 2 | accepted |
| FR-CONT-008 | Content version check: mobile re-downloads if `content_version` in cache differs from backend | P1 | 4 | accepted |
| FR-CONT-009 | Fallback to `en` if requested locale file is missing from Content Store | P1 | 2 | accepted |
| FR-CONT-010 | Signed URL expiry for audio files (S3 phase); short TTL (≤ 1 hour) | P1 | 4 | proposed |

---

### FR-PROGRESS: Progress Tracking

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-PROG-001 | `POST /progress/session` opens a quiz session | P0 | 3 | accepted |
| FR-PROG-002 | `POST /progress/answer` records a single answer event with idempotency key (`event_id`) | P0 | 3 | accepted |
| FR-PROG-003 | `POST /progress/session/end` closes session, stores score | P0 | 3 | accepted |
| FR-PROG-004 | `GET /progress/student` returns full answer history and scores | P0 | 3 | accepted |
| FR-PROG-005 | `GET /progress/unit/{unit_id}` returns attempts and best score for a unit | P1 | 3 | accepted |
| FR-PROG-006 | Backend deduplicates progress events by `event_id` (idempotent POST) | P0 | 3 | accepted |
| FR-PROG-007 | Progress data retained after subscription lapse (students keep history on downgrade) | P1 | 5 | proposed |

---

### FR-SUB: Subscription & Payments

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-SUB-001 | `GET /subscription/status` returns plan, valid_until, lessons_accessed | P0 | 5 | accepted |
| FR-SUB-002 | `POST /subscription/checkout` creates Stripe Checkout Session → checkout_url | P0 | 5 | accepted |
| FR-SUB-003 | `POST /subscription/webhook` handles Stripe lifecycle events | P0 | 5 | accepted |
| FR-SUB-004 | `DELETE /subscription` cancels at period end | P1 | 5 | accepted |
| FR-SUB-005 | Stripe webhook signature verification (reject without valid signature) | P0 | 5 | accepted |
| FR-SUB-006 | Webhook event log table (`stripe_events`) for dedup and audit | P1 | 5 | proposed |
| FR-SUB-007 | Handle `invoice.payment_failed` with 3-day grace period before locking content | P1 | 5 | proposed |
| FR-SUB-008 | Entitlement cache in Redis (5-minute TTL per student) | P1 | 5 | accepted |
| FR-SUB-009 | Force-expire entitlement cache on subscription cancellation / downgrade | P1 | 5 | proposed |
| FR-SUB-010 | Promo/coupon code support (Stripe coupon integration) | P2 | 5 | proposed |

---

### FR-PIPELINE: Content Generation Pipeline

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-PIPE-001 | `build_grade.py --grade N --lang en` generates all content for a grade in one language | P0 | 2 | accepted |
| FR-PIPE-002 | Pipeline generates lesson JSON, 3 quiz sets, tutorial, practice set per unit | P0 | 2 | accepted |
| FR-PIPE-003 | Pipeline generates `experiment_{lang}.json` only for units with `assessments.labs` | P1 | 6 | accepted |
| FR-PIPE-004 | TTS worker generates `lesson_{lang}.mp3` after lesson JSON is produced | P1 | 4 | accepted |
| FR-PIPE-005 | `meta.json` written per unit with `generated_at`, `model`, `content_version`, `langs_built` | P0 | 2 | accepted |
| FR-PIPE-006 | Pipeline is idempotent: skip units already built at current `content_version` unless `--force` | P0 | 2 | proposed |
| FR-PIPE-007 | Pipeline validates output JSON schema before writing to Content Store | P0 | 2 | proposed |
| FR-PIPE-008 | Pipeline emits structured log per unit: model, tokens used, cost estimate, duration | P1 | 2 | proposed |
| FR-PIPE-009 | `build_unit.py --unit G8-MATH-001 --lang en` regenerates a single unit | P1 | 2 | accepted |
| FR-PIPE-010 | Pipeline pins to a specific Claude model ID (no implicit latest) | P1 | 2 | proposed |
| FR-PIPE-011 | Pipeline dry-run mode (`--dry-run`) that validates prompts without writing content | P2 | 2 | proposed |
| FR-PIPE-012 | Spend cap: pipeline aborts if estimated cost exceeds `MAX_PIPELINE_COST_USD` config | P1 | 2 | proposed |

---

### FR-OFFLINE: Offline & Sync

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-OFF-001 | Mobile caches lesson JSON by `unit_id + content_version + lang` in local SQLite | P0 | 4 | accepted |
| FR-OFF-002 | Mobile caches quiz sets alongside lesson JSON | P0 | 4 | accepted |
| FR-OFF-003 | Mobile caches audio MP3 to local cache directory | P1 | 4 | accepted |
| FR-OFF-004 | Progress events queued locally in `event_queue` table with client-generated `event_id` | P0 | 4 | accepted |
| FR-OFF-005 | SyncManager flushes queue to backend on app foreground and network restore | P0 | 4 | accepted |
| FR-OFF-006 | Cache size limit enforced: oldest unused entries evicted when cache exceeds `MAX_CACHE_MB` | P1 | 4 | proposed |
| FR-OFF-007 | Content freshness check on app start: compare local `content_version` vs backend | P1 | 4 | accepted |

---

### FR-I18N: Multi-language

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-I18N-001 | English, French, Spanish content pre-generated at pipeline time | P0 | 4 | accepted |
| FR-I18N-002 | Locale stored in student profile and embedded in JWT | P0 | 1 | accepted |
| FR-I18N-003 | Language change in Settings clears local content cache and re-downloads | P1 | 4 | accepted |
| FR-I18N-004 | UI strings translated via static `mobile/i18n/{lang}.json` dictionaries | P1 | 4 | accepted |
| FR-I18N-005 | UI string fallback to `en` if key missing in target locale | P1 | 4 | accepted |

---

### FR-TTS: Text-to-Speech

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-TTS-001 | Lesson MP3 pre-generated in pipeline (not at request time) | P0 | 4 | accepted |
| FR-TTS-002 | TTS provider configurable via `TTS_PROVIDER` env var (Polly / Google TTS) | P1 | 4 | accepted |
| FR-TTS-003 | Audio cached on device for offline playback | P1 | 4 | accepted |
| FR-TTS-004 | `GET /content/{unit_id}/lesson/audio` returns signed URL or stream | P0 | 4 | accepted |

---

### FR-EXP: Experiment Visualization

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-EXP-001 | Experiment guide generated only for units with `assessments.labs` | P1 | 6 | accepted |
| FR-EXP-002 | `GET /content/{unit_id}/experiment` returns 200 + JSON or 404 | P1 | 6 | accepted |
| FR-EXP-003 | Mobile `ExperimentScreen` renders step-by-step cards with diagram hints | P1 | 6 | accepted |
| FR-EXP-004 | "🔬 Experiment" button shown on SubjectScreen only when endpoint returns 200 | P1 | 6 | accepted |
| FR-EXP-005 | Experiment JSON cached alongside lesson JSON | P1 | 6 | accepted |

---

### FR-ADMIN: Admin & Analytics

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-ADM-001 | Admin API: content build status per unit and language | P1 | 7 | accepted |
| FR-ADM-002 | Admin API: trigger regeneration for a single unit or full grade × language | P1 | 7 | accepted |
| FR-ADM-003 | Admin API: list students and view aggregate scores | P1 | 7 | accepted |
| FR-ADM-004 | Subscription analytics: MRR, churn, free-to-paid conversion rate | P1 | 7 | accepted |
| FR-ADM-005 | Struggle analytics: questions with >60% wrong rate across all students | P1 | 7 | accepted |
| FR-ADM-006 | Simple web dashboard (Jinja2 or React) | P2 | 7 | accepted |
| FR-ADM-007 | `GET /admin/pipeline/status` returns last run time, units built, units missing, errors | P1 | 7 | proposed |
| FR-ADM-008 | Content quality flag: surface units where student report count exceeds threshold | P2 | 7 | proposed |
| FR-ADM-009 | Audit log for all admin actions (who triggered what regeneration, when) | P1 | 7 | proposed |
| FR-ADM-010 | Admin endpoints protected by separate admin JWT role (not student JWT) | P0 | 7 | proposed |

---

### FR-MOBILE: Mobile UX

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-MOB-001 | All screens defined: Login, Register, Curriculum, GradeContent, UnitList, Subject, Quiz, Result, Subscription, Experiment, Progress, Settings | P0 | 1–6 | accepted |
| FR-MOB-002 | Quiz set rotation: never repeat same set twice in a row (1→2→3→1) | P1 | 2 | accepted |
| FR-MOB-003 | "Report a problem" button on quiz/lesson screens → `POST /content/{unit_id}/report` | P2 | 7 | proposed |
| FR-MOB-004 | Minimum app version check on launch: `GET /app/version` forces upgrade if below minimum | P1 | 3 | proposed |
| FR-MOB-005 | Graceful offline mode: show cached content + offline indicator when network is unavailable | P1 | 4 | accepted |

---

### FR-SCHOOL: School & Teacher Management

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-SCH-001 | School registration: `POST /schools/register` with school name, contact email, country | P0 | 8 | proposed |
| FR-SCH-002 | School status field: `pending → active → suspended`; auto-approve in Phase 8 | P1 | 8 | proposed |
| FR-SCH-003 | Teacher account created automatically on school registration (role = school_admin) | P0 | 8 | proposed |
| FR-SCH-004 | Additional teachers invited by school_admin via email invite + one-time token | P1 | 8 | proposed |
| FR-SCH-005 | Teacher JWT: payload includes `{teacher_id, school_id, role, exp}` | P0 | 8 | proposed |
| FR-SCH-006 | School enrolment code: short human-readable code per school for student-led joins | P1 | 9 | proposed |
| FR-SCH-007 | Teacher can view enrolled student list with enrolment status | P1 | 9 | proposed |
| FR-SCH-008 | Teacher can remove a student from the school roster | P1 | 9 | proposed |
| FR-SCH-009 | Platform admin can list, approve, or suspend schools | P1 | 8 | proposed |
| FR-SCH-010 | Teacher password reset uses same flow as student (forgot-password → token → reset) | P0 | 8 | proposed |

---

### FR-CURRIC: Curriculum Management

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-CUR-001 | XLSX template download: `GET /curriculum/template` returns a pre-formatted workbook | P0 | 8 | proposed |
| FR-CUR-002 | XLSX upload: `POST /curriculum/upload` parses, validates, returns row-level errors on failure | P0 | 8 | proposed |
| FR-CUR-003 | UI form upload: same endpoint accepts JSON body with grade + units array | P1 | 8 | proposed |
| FR-CUR-004 | Curriculum validation: required columns present, no duplicate unit codes, objectives ≥ 2 per unit | P0 | 8 | proposed |
| FR-CUR-005 | Curriculum pipeline trigger: `POST /curriculum/pipeline/trigger` — generates content per unit × lang | P0 | 8 | proposed |
| FR-CUR-006 | Pipeline status polling: `GET /curriculum/pipeline/{job_id}/status` | P1 | 8 | proposed |
| FR-CUR-007 | Teacher notified by email when pipeline completes (or partially fails) | P1 | 8 | proposed |
| FR-CUR-008 | Curriculum versioning: one `active` curriculum per (school_id, grade, year); activating new one archives old | P0 | 9 | proposed |
| FR-CUR-009 | `restrict_access` flag: when enabled, only enrolled students can access the curriculum | P1 | 9 | proposed |
| FR-CUR-010 | Default curriculum seeded per year from `data/grade*_stem.json` files via `seed_default.py` | P0 | 8 | proposed |
| FR-CUR-011 | Default curriculum ID pattern: `default-{year}-g{grade}` (e.g. `default-2026-g8`) | P0 | 8 | proposed |
| FR-CUR-012 | Curriculum resolver: content requests resolve `curriculum_id` from student's school + grade + year; fall back to default | P0 | 9 | proposed |

---

### FR-ASSOC: Student–School Association

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-ASC-001 | Teacher-led enrolment: teacher uploads CSV or enters list of student emails | P0 | 9 | proposed |
| FR-ASC-002 | Student-led enrolment: optional `enrolment_code` field at registration | P1 | 9 | proposed |
| FR-ASC-003 | On student registration: if email matches pending enrolment record, auto-link to school | P0 | 9 | proposed |
| FR-ASC-004 | A student can belong to at most one school at a time | P0 | 9 | proposed |
| FR-ASC-005 | Unenrolled students fall back to default curriculum for their grade and the current year | P0 | 9 | proposed |
| FR-ASC-006 | Student can leave a school from Settings screen; reverts to default curriculum | P1 | 9 | proposed |
| FR-ASC-007 | Progress history is preserved when a student transfers schools | P0 | 9 | proposed |

---

### FR-ANALYTICS: Extended Analytics

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-ANL-001 | Lesson view start/end events recorded with duration, audio_played, experiment_viewed flags | P0 | 10 | proposed |
| FR-ANL-002 | Quiz sessions record `attempt_number` (1-based per student × unit × curriculum) | P0 | 10 | proposed |
| FR-ANL-003 | Quiz sessions record `passed` flag (score ≥ passing threshold, default 70%) | P0 | 10 | proposed |
| FR-ANL-004 | `GET /analytics/student/me` — student's own metrics: scores, time, attempt counts, improvement | P1 | 10 | proposed |
| FR-ANL-005 | `GET /analytics/school/{school_id}/class` — teacher's class aggregate per unit | P1 | 10 | proposed |
| FR-ANL-006 | `GET /analytics/unit/{unit_id}/summary` — platform-wide metrics per unit (admin only) | P2 | 10 | proposed |
| FR-ANL-007 | Struggle flag: surface units where mean-attempts-to-pass > 2 OR first-attempt pass rate < 50% | P1 | 10 | proposed |
| FR-ANL-008 | Mobile: Progress screen shows student's own analytics dashboard | P2 | 10 | proposed |

---

### FR-FEEDBACK: Student Feedback

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-FDB-001 | `POST /feedback` accepts category (content/ux/general), optional unit_id, message (max 500 chars), optional rating (1–5) | P1 | 10 | proposed |
| FR-FDB-002 | "Give Feedback" button accessible from lesson screen, result screen, and Settings | P1 | 10 | proposed |
| FR-FDB-003 | Feedback rate-limited: max 5 submissions per student per hour | P1 | 10 | proposed |
| FR-FDB-004 | `GET /admin/feedback` returns paginated feedback with filters (category, unit, date, reviewed) | P1 | 10 | proposed |
| FR-FDB-005 | Admin can mark feedback as reviewed and add an internal note | P2 | 10 | proposed |
| FR-FDB-006 | Teacher can view content feedback for their school's curriculum units | P2 | 10 | proposed |

---

### FR-REPORTS: Teacher & School Reporting

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-RPT-001 | Class Overview report: `GET /reports/school/{id}/overview` returns active students, lesson views, quiz attempts, avg score, pass rate, audio play rate, struggling units, unreviewed feedback count for a configurable period (7d/30d/term) | P0 | 11 | proposed |
| FR-RPT-002 | Unit Performance report: `GET /reports/school/{id}/unit/{unit_id}` returns engagement metrics, score distribution, attempt distribution, struggle flag, and feedback summary | P0 | 11 | proposed |
| FR-RPT-003 | Student Progress report: `GET /reports/school/{id}/student/{student_id}` returns per-student completion, scores, time, attempts, strongest and weakest subjects | P1 | 11 | proposed |
| FR-RPT-004 | Curriculum Health report: `GET /reports/school/{id}/curriculum-health` returns all units ranked into Healthy / Watch / Struggling / No Activity tiers with recommended action | P0 | 11 | proposed |
| FR-RPT-005 | Feedback report: `GET /reports/school/{id}/feedback` returns all student feedback grouped by unit, with category breakdown, avg rating, trending flag, and paginated message list | P0 | 11 | proposed |
| FR-RPT-006 | Trend report: `GET /reports/school/{id}/trends` returns week-over-week data for active students, lessons viewed, quiz attempts, avg score, pass rate, audio play rate, feedback volume | P1 | 11 | proposed |
| FR-RPT-007 | CSV export: `POST /reports/school/{id}/export` accepts `{report_type, filters}`, generates CSV asynchronously (Celery), returns a time-limited download URL | P1 | 11 | proposed |
| FR-RPT-008 | Teachers can mark individual feedback items as reviewed directly from the Feedback report | P1 | 11 | proposed |
| FR-RPT-009 | Threshold alerts: configurable per school — unit pass rate, feedback volume, student inactivity, score drop; evaluated daily by Celery Beat; delivered by email + dashboard badge | P1 | 11 | proposed |
| FR-RPT-010 | Weekly email digest: opt-in per teacher; sent Monday 08:00 local time; covers active students, struggling units, new feedback, inactive students; configurable via `POST /reports/school/{id}/digest/subscribe` | P1 | 11 | proposed |
| FR-RPT-011 | All report data served from PostgreSQL read replica via materialized views; never touches primary for report queries | P0 | 11 | proposed |
| FR-RPT-012 | Materialized views refreshed nightly at 02:00 UTC by Celery Beat (`mv_class_summary`, `mv_student_progress`, `mv_feedback_summary`); `mv_feedback_summary` refreshed hourly | P0 | 11 | proposed |
| FR-RPT-013 | On-demand report refresh: `POST /reports/school/{id}/refresh` available to school_admin role; triggers immediate Celery task to refresh materialized views | P2 | 11 | proposed |
| FR-RPT-014 | "Last updated" timestamp shown on all report pages indicating materialized view refresh time | P1 | 11 | proposed |
| FR-RPT-015 | Report endpoints accessible only to teacher/school_admin JWT; students cannot access any /reports/* endpoint | P0 | 11 | proposed |
| FR-RPT-016 | Teacher can filter Unit Performance and Feedback reports by subject, date range, and feedback category | P1 | 11 | proposed |
| FR-RPT-017 | Curriculum Health recommended_action field: `none | review_content | add_class_time | report_to_admin` based on health tier and feedback count | P2 | 11 | proposed |

### FR-SPRD: Student Progress Reports

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-SPRD-001 | `GET /student/dashboard` returns: summary card (units_completed, quizzes_passed, current_streak_days, total_time_minutes, avg_quiz_score), subject_progress[], next_unit, recent_activity[] | P0 | 3 | proposed |
| FR-SPRD-002 | `GET /student/progress` returns the full curriculum tree with per-unit status (`not_started`, `in_progress`, `needs_retry`, `completed`), best_score, attempts, and last_attempt_at | P0 | 3 | proposed |
| FR-SPRD-003 | `GET /student/stats?period=7d\|30d\|all` returns lessons_viewed, quizzes_completed, quizzes_passed, avg_quiz_score, total_time_minutes, audio_plays, streak_current_days, streak_longest_days, daily_activity[] | P1 | 3 | proposed |
| FR-SPRD-004 | Unit status `completed` requires `best_score ≥ QUIZ_PASS_THRESHOLD` (configurable, default 60 %) and at least one closed session; `needs_retry` requires ≥ 1 closed session with `best_score < threshold`; `in_progress` requires an open session; otherwise `not_started` | P0 | 3 | proposed |
| FR-SPRD-005 | Daily learning streak stored in Redis (`streak:{student_id}`); incremented by Celery task on first progress event per calendar day; never computed from raw DB rows on request | P1 | 3 | proposed |
| FR-SPRD-006 | `next_unit` in the dashboard is the first curriculum-ordered unit with status `not_started` or `needs_retry`; falls back to first `in_progress` unit; null if all units completed | P1 | 3 | proposed |
| FR-SPRD-007 | Dashboard response cached at L1 (60 s TTL, key `dashboard:{student_id}`) and L2 Redis; invalidated on `POST /progress/session/end` and on curriculum change | P1 | 3 | proposed |
| FR-SPRD-008 | Curriculum progress map backed by materialized view `mv_student_curriculum_progress` (per `student_id × unit_id`); refreshed on `POST /progress/session/end` via async Celery task | P1 | 3 | proposed |
| FR-SPRD-009 | `daily_activity` in stats served from `lesson_views` table on read replica grouped by `DATE(started_at)`; never touches primary | P1 | 3 | proposed |
| FR-SPRD-010 | Student progress endpoints (`/student/*`) must only return data belonging to the authenticated student; a student JWT must never access another student's progress — validated by comparing `student_id` from JWT against the data owner | P0 | 3 | proposed |
| FR-SPRD-011 | Mobile app provides three screens: `ProgressDashboardScreen` (summary card + next unit + recent activity), `CurriculumMapScreen` (full subject/unit list with status badges), `StatsScreen` (period picker + stat cards + daily activity chart) | P1 | 3 | proposed |

---

## Non-Functional Requirements

### NFR-SEC: Security

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-SEC-001 | Rate limiting on `POST /auth/login` and `POST /auth/register` (10 req/min per IP) | P0 | 1 | proposed |
| NFR-SEC-002 | Rate limiting on content endpoints (100 req/min per student JWT) | P1 | 2 | proposed |
| NFR-SEC-003 | Passwords stored with bcrypt (min cost factor 12) | P0 | 1 | accepted |
| NFR-SEC-004 | All secrets sourced from environment variables; never hardcoded | P0 | 1 | accepted |
| NFR-SEC-005 | Secrets managed via a secrets manager in production (AWS Secrets Manager, HashiCorp Vault, or equivalent) | P1 | 2 | proposed |
| NFR-SEC-006 | HTTPS enforced for all backend endpoints; HTTP redirects to HTTPS | P0 | 1 | proposed |
| NFR-SEC-007 | CORS policy: allow only known origins (mobile deeplink scheme + admin dashboard domain) | P1 | 1 | proposed |
| NFR-SEC-008 | JWT signing key rotation without downtime using `kid` header | P1 | 2 | proposed |
| NFR-SEC-009 | Redis stores only non-sensitive data (no raw passwords, no PII beyond student_id) | P0 | 1 | proposed |
| NFR-SEC-010 | Redis persistence enabled (`AOF` mode) to survive restarts without logging out all users | P0 | 1 | proposed |
| NFR-SEC-011 | Mobile: no Anthropic API key, no Stripe key stored in app | P0 | 1 | accepted |
| NFR-SEC-012 | Stripe webhook endpoint rejects requests with invalid signatures | P0 | 5 | accepted |
| NFR-SEC-013 | Content Store not directly accessible by mobile clients (served through Content Service only) | P0 | 2 | accepted |
| NFR-SEC-014 | SQL injection prevention: use ORM (SQLAlchemy) parameterized queries exclusively | P0 | 1 | proposed |
| NFR-SEC-015 | Input validation on all request bodies via Pydantic models | P0 | 1 | proposed |

---

### NFR-COMP: Compliance & Privacy

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-COMP-001 | COPPA compliance: collect parental consent for students under 13 before account activation | P0 | 1 | proposed |
| NFR-COMP-002 | GDPR: honour right-to-erasure request within 30 days (`DELETE /auth/account`) | P0 | 5 | proposed |
| NFR-COMP-003 | GDPR: data processing disclosure in privacy policy; consent captured at registration | P0 | 1 | proposed |
| NFR-COMP-004 | Data retention policy: progress records retained for N years then anonymised | P1 | 5 | proposed |
| NFR-COMP-005 | PII minimisation: collect only name, email, grade, locale; no location, no device ID | P1 | 1 | proposed |
| NFR-COMP-006 | Stripe handles all payment data; backend never stores card numbers or CVV | P0 | 5 | accepted |
| NFR-COMP-007 | Password reset tokens expire after 1 hour and are single-use | P0 | 1 | proposed |

---

### NFR-PERF: Performance

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-PERF-001 | Content endpoint (cache hit) p95 ≤ 50 ms | P1 | 2 | proposed |
| NFR-PERF-002 | Content endpoint (cache miss / S3 read) p95 ≤ 200 ms | P1 | 2 | proposed |
| NFR-PERF-003 | Auth endpoints (login/register) p95 ≤ 500 ms (bcrypt-bound) | P1 | 1 | proposed |
| NFR-PERF-004 | Mobile app: cached content renders in < 100 ms (no network) | P1 | 2 | proposed |
| NFR-PERF-005 | Pipeline throughput: generate one complete grade (all units, one language) within 60 minutes | P2 | 2 | proposed |
| NFR-PERF-006 | Progress write endpoints (`POST /progress/answer`) p95 ≤ 20 ms (fire-and-forget pattern) | P1 | 3 | proposed |
| NFR-PERF-007 | API server availability: 99.9% uptime (≤ 8.7 hours downtime per year) | P0 | 2 | proposed |

---

### NFR-REL: Reliability & Resilience

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-REL-001 | PostgreSQL daily automated backups with 30-day retention | P0 | 1 | proposed |
| NFR-REL-002 | Content Store backups: daily snapshot to a separate bucket/path | P1 | 2 | proposed |
| NFR-REL-003 | Redis HA: Sentinel or Cluster configuration for production | P1 | 2 | proposed |
| NFR-REL-004 | Database migrations managed with Alembic; migrations run automatically on deploy | P0 | 1 | proposed |
| NFR-REL-005 | Pipeline is idempotent: re-running never produces duplicate content or duplicate DB records | P0 | 2 | proposed |
| NFR-REL-006 | Stripe webhook delivery failures logged and alerted; manual replay available | P1 | 5 | proposed |
| NFR-REL-007 | Backend health endpoint reports DB, Redis, and Content Store reachability | P1 | 1 | proposed |
| NFR-REL-008 | Mobile: all network calls run in daemon threads; UI never blocked by network | P0 | 1 | accepted |
| NFR-REL-009 | Deployment: zero-downtime rolling deploys (containerised backend) | P1 | 2 | proposed |
| NFR-REL-010 | PgBouncer deployed in transaction-pooling mode in front of PostgreSQL to prevent connection exhaustion | P0 | 1 | proposed |
| NFR-REL-011 | Circuit breakers wrap all external API calls (Stripe, Anthropic, TTS, SES); failure threshold 5 for Stripe, 3 for Anthropic | P1 | 2 | proposed |
| NFR-REL-012 | FastAPI event loop must never be blocked; all CPU-bound operations (bcrypt) run in thread pool executor | P0 | 1 | proposed |

---

### NFR-OBS: Observability & Monitoring

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-OBS-001 | Structured JSON logs from backend (stdout) aggregated to a log platform (CloudWatch / ELK / Loki) | P1 | 1 | proposed |
| NFR-OBS-002 | Error tracking via Sentry (backend + mobile) | P1 | 1 | proposed |
| NFR-OBS-003 | Request metrics exported: request count, latency (p50/p95/p99), error rate by endpoint | P1 | 2 | proposed |
| NFR-OBS-004 | Redis cache hit/miss rate monitored | P2 | 2 | proposed |
| NFR-OBS-005 | Pipeline run report emitted on completion: units attempted, succeeded, failed, tokens used, cost estimate | P1 | 2 | proposed |
| NFR-OBS-006 | Alert when pipeline run fails (email or Slack webhook) | P1 | 2 | proposed |
| NFR-OBS-007 | Alert when Stripe webhook returns non-200 or delivery rate drops | P1 | 5 | proposed |
| NFR-OBS-008 | Alert when content freshness check fails (curriculum updated but content not regenerated) | P1 | 4 | proposed |
| NFR-OBS-009 | Dashboard: 402-rate trend (free-to-paywall conversion funnel) | P2 | 5 | proposed |
| NFR-OBS-010 | Mobile crash reports via Sentry or equivalent | P1 | 2 | proposed |
| NFR-OBS-011 | Uptime monitoring: external ping on `/health` every 60 seconds | P1 | 1 | proposed |
| NFR-OBS-012 | Celery queue depth monitored; alert when pipeline queue depth > 10 for > 5 minutes | P1 | 2 | proposed |
| NFR-OBS-013 | CDN cache hit rate monitored via CloudFront metrics; alert if hit rate drops below 80% | P2 | 4 | proposed |

---

### NFR-CQ: Content Quality

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-CQ-001 | Pipeline validates generated JSON against schema before writing to Content Store | P0 | 2 | proposed |
| NFR-CQ-002 | Quiz validation: confirm exactly 8 questions per set, each with 4 choices and one marked correct | P0 | 2 | proposed |
| NFR-CQ-003 | Pipeline retries a single unit up to 3 times on malformed output before marking as failed | P1 | 2 | proposed |
| NFR-CQ-004 | Content staged to a review path before promotion to live Content Store | P2 | 7 | proposed |
| NFR-CQ-005 | Student content report endpoint (`POST /content/{unit_id}/report`) + admin review queue | P2 | 7 | proposed |
| NFR-CQ-006 | Claude model ID pinned in `pipeline/config.py`; upgrading model version triggers selective regeneration | P1 | 2 | proposed |
| NFR-CQ-007 | Pipeline logs tokens consumed and estimated cost per unit for budget tracking | P1 | 2 | proposed |

---

## Architectural Decisions Log

| ID | Decision | Rationale | Date | Status |
|---|---|---|---|---|
| ADR-001 | Pre-generate all AI content offline in a pipeline | Eliminates per-student API latency (5–10 s → instant); removes Anthropic key from student devices | 2026-03-23 | accepted |
| ADR-002 | JWT locale is authoritative; no `?lang=` query parameter on content endpoints | Prevents students accessing languages not configured on their account; keeps locale change in profile | 2026-03-23 | accepted |
| ADR-003 | Entitlement enforced on backend only; mobile acts on HTTP status codes | Client-side gating is trivially bypassed; backend is the single source of truth | 2026-03-23 | accepted |
| ADR-004 | 3 quiz sets per unit, rotated per attempt | Prevents identical questions on retake; sets rotate 1→2→3→1 on device | 2026-03-23 | accepted |
| ADR-005 | Content Store keyed by `unit_id` (not grade/subject name) | Grade and subject names can change; unit IDs are stable identifiers | 2026-03-23 | accepted |
| ADR-006 | Progress event queue on device; sync on network restore | Supports offline quiz-taking without data loss | 2026-03-23 | accepted |
| ADR-007 | TTS audio pre-generated in pipeline; `GET /audio` serves a file only | Runtime TTS adds latency and per-request API cost; pre-generation enables offline playback | 2026-03-23 | accepted |
| ADR-008 | Content Store filesystem (Phase 1–3) → S3 (Phase 4+) | Start simple; S3 migration enables signed URLs, CDN, and horizontal scaling without code changes | 2026-03-23 | accepted |
| ADR-009 | Stripe handles all payment data; backend stores only `stripe_customer_id` / `stripe_subscription_id` | Avoids PCI scope; hosted checkout removes card handling from our infrastructure | 2026-03-23 | accepted |
| ADR-010 | Pipeline model ID pinned; model upgrades are explicit and trigger selective regeneration | Implicit latest model creates inconsistency between units built at different times | 2026-03-23 | proposed |
| ADR-011 | All content keyed by `curriculum_id` (not grade number) | Enables default and school curricula to use identical pipeline, storage, and delivery paths | 2026-03-23 | proposed |
| ADR-012 | Student can belong to at most one school | Simplifies curriculum resolution; avoids conflicting curricula per student | 2026-03-23 | proposed |
| ADR-013 | Curriculum pipeline reads units from PostgreSQL when triggered for school curricula | Allows pipeline to be triggered via API (not just CLI); same pipeline code handles both default and custom curricula | 2026-03-23 | proposed |
| ADR-014 | Lesson view events sent from mobile on open/close; backend records duration | More accurate than client-computed duration; handles offline by queuing alongside progress events | 2026-03-23 | proposed |
| ADR-015 | Three-level cache (L1 in-process TTLCache, L2 Redis, L3 CDN) | Hot read path must serve zero DB queries on cache-warm requests; each level serves a different latency/scope tradeoff | 2026-03-23 | accepted |
| ADR-016 | Progress writes are fire-and-forget via Celery | Student response time must not be gated on DB write latency; writes can tolerate seconds of delay with dedup guarantees | 2026-03-23 | accepted |
| ADR-017 | Audio files served via CloudFront pre-signed URLs; never proxied through API servers | Proxying audio through FastAPI would consume worker memory and bandwidth for no benefit; CDN handles caching and global delivery | 2026-03-23 | accepted |
| ADR-018 | PgBouncer in transaction-pooling mode in front of PostgreSQL | Each uvicorn worker holds an asyncpg pool; without PgBouncer, total connections = workers × pool_max, easily exceeding PostgreSQL max_connections | 2026-03-23 | accepted |
| ADR-019 | All report queries route to PostgreSQL read replica via materialized views | Report aggregations are expensive; running them on the primary would compete with write-path queries and entitlement lookups that students depend on | 2026-03-23 | proposed |
| ADR-020 | Materialized views refreshed nightly (not real-time) with 24-hour stale tolerance for most reports | Real-time aggregation would require expensive incremental computation on every event; nightly refresh is accurate enough for teacher planning and avoids write amplification | 2026-03-23 | proposed |
| ADR-021 | Feedback summary refreshed hourly (not nightly) | Feedback is time-sensitive — a content error reported by students should be visible to a teacher within the hour, not the next morning | 2026-03-23 | proposed |

---

## Change Log

| Date | Version | Author | Change |
|---|---|---|---|
| 2026-03-23 | 0.1.0 | — | Initial requirements document created from architecture review |
| 2026-03-23 | 0.2.0 | — | Added school/teacher management, custom curriculum, student association, extended analytics, and feedback requirements (Phases 8–10) |
| 2026-03-23 | 0.3.0 | — | Added BACKEND_ARCHITECTURE.md; updated NFR-PERF, NFR-REL, NFR-OBS to align with SLOs and architecture decisions; added ADR-015 through ADR-018 |
| 2026-03-23 | 0.4.0 | — | Added Teacher & School Reporting system: 6 report types, CSV export, threshold alerts, weekly digest, 3 new materialized views, Phase 11 |
