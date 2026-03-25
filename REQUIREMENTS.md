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
| FR-AUTH-001 | Students and teachers authenticate via external provider (Auth0); backend issues internal JWT after token exchange | P0 | 1 | implemented |
| FR-AUTH-002 | `POST /auth/exchange` accepts Auth0 `id_token`, verifies against JWKS (L1-cached), upserts student record, returns internal JWT + refresh token | P0 | 1 | implemented |
| FR-AUTH-003 | `POST /auth/teacher/exchange` — same pattern; issues `{teacher_id, school_id, role}` JWT | P0 | 1 | implemented |
| FR-AUTH-004 | Internal JWT payload: `{student_id, grade, locale, role, exp}` (student) or `{teacher_id, school_id, role, exp}` (teacher); locale always from DB record, never from Auth0 token | P0 | 1 | implemented |
| FR-AUTH-005 | `POST /auth/refresh` exchanges refresh token (Redis, 30-day TTL) for new internal JWT; no Auth0 call required | P0 | 1 | implemented |
| FR-AUTH-006 | `POST /auth/forgot-password` calls Auth0 Management API to trigger password reset email; always returns 200; no reset tokens stored in our DB for external-auth users | P0 | 1 | implemented |
| FR-AUTH-007 | Email verification and brute-force protection delegated to Auth0; no local implementation required for students/teachers | P0 | 1 | implemented |
| FR-AUTH-008 | Internal product team (developer, tester, product_admin, super_admin) authenticate via `POST /admin/auth/login` with bcrypt-verified local credentials; `ADMIN_JWT_SECRET` is separate from student/teacher secrets | P0 | 1 | implemented |
| FR-AUTH-009 | `POST /admin/auth/forgot-password` stores one-time reset token in Redis (TTL 1 hr); `POST /admin/auth/reset-password` validates and sets new bcrypt hash | P0 | 1 | implemented |
| FR-AUTH-010 | No password hash stored for students or teachers; `password_hash` column absent from `students` and `teachers` tables | P0 | 1 | implemented |
| FR-AUTH-011 | Auth0 JWKS cached in L1 TTLCache (24-hr TTL); JWKS refresh triggered on `kid` mismatch; never fetched on every request | P1 | 1 | implemented |
| FR-AUTH-012 | Parental consent flow for students under 13 (COPPA): Auth0 blocks signup under-13; consent collected externally before account activation | P0 | 1 | implemented |
| FR-AUTH-013 | `DELETE /auth/account` deletes student PII and queues Celery task to delete Auth0 user record (GDPR) | P0 | 5 | implemented |
| FR-AUTH-014 | Student profile update: `PATCH /student/profile` (name, locale, grade) updates our DB record; locale change invalidates cached JWT content | P1 | 3 | implemented |

### FR-ACCTMGMT: Account Management

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-ACCTMGMT-001 | All user types (student, teacher, school) carry `account_status`: `pending \| active \| suspended \| deleted` | P0 | 1 | implemented |
| FR-ACCTMGMT-002 | Auth middleware checks Redis `suspended:{id}` set after JWT signature verification; suspended accounts receive `403 Forbidden`; zero DB queries on hot path | P0 | 1 | implemented |
| FR-ACCTMGMT-003 | On suspension, `id` added to Redis `suspended:{id}` with **no TTL**; key is explicitly deleted only on reactivation — a TTL would silently restore access without admin action | P0 | 1 | implemented |
| FR-ACCTMGMT-004 | On suspension, Celery task calls Auth0 Management API to block the external user (`{"blocked": true}`), preventing re-login; our `account_status` is authoritative | P1 | 1 | implemented |
| FR-ACCTMGMT-005 | `PATCH /admin/accounts/students/{id}/status` and equivalent teacher/school endpoints; accessible only by `product_admin` or `super_admin` JWT | P0 | 1 | implemented |
| FR-ACCTMGMT-006 | Suspending a school cascades: all teachers and students of that school suspended via Celery task; each added to Redis `suspended:{id}` set individually | P1 | 1 | implemented |
| FR-ACCTMGMT-007 | Schools and teachers (school_admin role) can suspend/reactivate their own students and teachers via `PATCH /schools/{school_id}/students/{id}/status` and `PATCH /schools/{school_id}/teachers/{id}/status`; cannot suspend students belonging to other schools | P1 | 8 | proposed |
| FR-ACCTMGMT-008 | `DELETE /admin/accounts/students/{id}` soft-deletes the record; Celery task anonymises PII within 30 days per GDPR | P0 | 5 | proposed |
| FR-ACCTMGMT-009 | Admin account management list endpoints support pagination, `status` filter, `school_id` filter, and free-text `q` search on name/email | P1 | 1 | proposed |
| FR-ACCTMGMT-010 | Internal admin account lockout: 5 failed login attempts → 15-minute Redis-backed cooldown (same pattern as Auth0 brute-force, applied locally for admin users) | P1 | 1 | implemented |

---

### FR-RBAC: Role-Based Access Control

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-RBAC-001 | Seven distinct roles exist: `student`, `teacher`, `school_admin`, `product_admin`, `super_admin`, `developer`, `tester`; role is encoded in every JWT `role` claim | P0 | 1 | proposed |
| FR-RBAC-002 | Permissions are defined as a static in-code mapping (`ROLE_PERMISSIONS` dict in `core/permissions.py`); no DB lookup is performed on the hot path for permission checks | P0 | 1 | proposed |
| FR-RBAC-003 | A `require_permission(permission)` FastAPI dependency is used on every admin and review endpoint; raises HTTP 403 if the role lacks the permission | P0 | 1 | proposed |
| FR-RBAC-004 | School-scoped roles (`teacher`, `school_admin`) can only act on resources where `school_id` matches their JWT `school_id` claim; scope is enforced inside each endpoint handler | P0 | 1 | proposed |
| FR-RBAC-005 | `teacher` role may annotate and rate content but not approve or publish; `school_admin` may additionally approve; `product_admin` / `super_admin` may publish and rollback | P0 | 7 | proposed |
| FR-RBAC-006 | `developer` and `tester` roles have `review:read`, `review:rate`, and (tester only) `review:annotate` permissions for staging use; they cannot approve, publish, or block | P1 | 7 | proposed |
| FR-RBAC-007 | Changing a role's permissions requires a code change and deployment; no dynamic role/permission management UI is required in Phases 1–11 | P2 | deferred | proposed |

---

### FR-CURR: Curriculum

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-CURR-001 | `GET /curriculum` returns list of available grades | P0 | 1 | implemented |
| FR-CURR-002 | `GET /curriculum/{grade}` returns full subject + unit tree for grade | P0 | 1 | implemented |
| FR-CURR-003 | Backend discovers grades from `data/grade*_stem.json` filenames (no hardcoded grade list) | P0 | 1 | implemented |

---

### FR-CONTENT: Content Delivery

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-CONT-001 | `GET /content/{unit_id}/lesson` returns lesson JSON in student locale | P0 | 2 | implemented |
| FR-CONT-002 | `GET /content/{unit_id}/lesson/audio` returns signed URL or stream for MP3 | P0 | 4 | accepted |
| FR-CONT-003 | `GET /content/{unit_id}/quiz` returns one quiz set (8 questions, rotated) in student locale | P0 | 2 | implemented |
| FR-CONT-004 | `GET /content/{unit_id}/tutorial` returns remediation content in student locale | P1 | 2 | implemented |
| FR-CONT-005 | `GET /content/{unit_id}/experiment` returns experiment JSON or HTTP 404 if no lab | P1 | 6 | implemented |
| FR-CONT-006 | `GET /content/{unit_id}/practice` returns practice test set in student locale | P1 | 2 | implemented |
| FR-CONT-007 | Entitlement middleware: HTTP 402 after 2 distinct lesson views for free-tier students | P0 | 2 | implemented |
| FR-CONT-008 | Content version check: mobile re-downloads if `content_version` in cache differs from backend | P1 | 4 | accepted |
| FR-CONT-009 | Fallback to `en` if requested locale file is missing from Content Store | P1 | 2 | implemented |
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
| FR-PIPE-001 | `build_grade.py --grade N --lang en` generates all content for a grade in one language | P0 | 2 | implemented |
| FR-PIPE-002 | Pipeline generates lesson JSON, 3 quiz sets, tutorial, practice set per unit | P0 | 2 | implemented |
| FR-PIPE-003 | Pipeline generates `experiment_{lang}.json` only for units with `assessments.labs` | P1 | 6 | implemented |
| FR-PIPE-004 | TTS worker generates `lesson_{lang}.mp3` after lesson JSON is produced | P1 | 4 | implemented |
| FR-PIPE-005 | `meta.json` written per unit with `generated_at`, `model`, `content_version`, `langs_built` | P0 | 2 | implemented |
| FR-PIPE-006 | Pipeline is idempotent: skip units already built at current `content_version` unless `--force` | P0 | 2 | implemented |
| FR-PIPE-007 | Pipeline validates output JSON schema before writing to Content Store | P0 | 2 | implemented |
| FR-PIPE-008 | Pipeline emits structured log per unit: model, tokens used, cost estimate, duration | P1 | 2 | implemented |
| FR-PIPE-009 | `build_unit.py --unit G8-MATH-001 --lang en` regenerates a single unit | P1 | 2 | implemented |
| FR-PIPE-010 | Pipeline pins to a specific Claude model ID (no implicit latest) | P1 | 2 | implemented |
| FR-PIPE-011 | Pipeline dry-run mode (`--dry-run`) that validates prompts without writing content | P2 | 2 | implemented |
| FR-PIPE-012 | Spend cap: pipeline aborts if estimated cost exceeds `MAX_PIPELINE_COST_USD` config | P1 | 2 | implemented |

---

### FR-ALEX: Automated Content Analysis

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-ALEX-001 | AlexJS runs against every Claude-generated text asset (lesson synopsis, quiz questions, tutorial text) during the pipeline, after schema validation | P0 | 2 | implemented |
| FR-ALEX-002 | AlexJS output written to `alex_report.json` in the unit's Content Store directory; warning count aggregated into `content_subject_versions.alex_warnings_count` | P0 | 2 | implemented |
| FR-ALEX-003 | Units with AlexJS warnings get `content_subject_version.status = needs_review`; units with zero warnings get `ready_for_review`; pipeline does not abort on warnings | P0 | 2 | implemented |
| FR-ALEX-004 | AlexJS invoked via subprocess (`npx alex --stdin --reporter json`); Node.js and alex installed in the pipeline environment; timeout of 30 seconds per unit | P1 | 2 | implemented |
| FR-ALEX-005 | `REVIEW_AUTO_APPROVE=true` env var marks new subject versions as `published` immediately; for dev/test environments only; absent or `false` in production | P1 | 2 | implemented |

---

### FR-CREV: Content Review & Approval

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-CREV-001 | No content is accessible to students until a `content_subject_version` with `status = published` exists for that `(curriculum_id, subject)`; content endpoint raises HTTP 404 otherwise | P0 | 2 | implemented |
| FR-CREV-002 | Review queue (`GET /admin/content/review/queue`) returns subject versions filterable by `curriculum_id`, `subject`, `status`, and paginated | P0 | 7 | proposed |
| FR-CREV-003 | A reviewer opens a session (`POST /admin/content/review/{version_id}/open`) that creates a `content_reviews` record; multiple reviewers may open sessions on the same version | P1 | 7 | proposed |
| FR-CREV-004 | Reviewers can annotate specific words or phrases (`start_offset`, `end_offset`, `original_text`) and supply a replacement suggestion; annotation is linked to a unit + content_type + lang | P0 | 7 | proposed |
| FR-CREV-005 | `GET /admin/content/dictionary?word=` returns definitions (Merriam-Webster API) and synonyms (Datamuse API); result stored in `content_annotations.dictionary_options` | P1 | 7 | proposed |
| FR-CREV-006 | Reviewer submits `language_rating` (1–5) and `content_rating` (1–5) per version; both required before approval | P0 | 7 | proposed |
| FR-CREV-007 | Reviewer approves (`POST .../approve`) or rejects with optional regeneration request (`POST .../reject` with `regenerate: bool`) | P0 | 7 | proposed |
| FR-CREV-008 | Rejection with `regenerate=true` dispatches a Celery task to re-run the pipeline for flagged units; version status → `changes_requested`; new version created after re-run | P1 | 7 | proposed |
| FR-CREV-009 | Students can submit marked-text feedback from the mobile UI (`POST /content/{unit_id}/feedback/marked`); submission stores offset + marked text + comment; does not block content access | P1 | 7 | implemented |
| FR-CREV-010 | Student marked-text feedback is surfaced in the admin review queue alongside content annotations | P2 | 7 | proposed |

---

### FR-CVER: Content Versioning

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| FR-CVER-001 | Versioning is at subject level within a curriculum; a version encompasses all units for that subject | P0 | 2 | implemented |
| FR-CVER-002 | Version numbers are sequential integers per `(curriculum_id, subject)`, starting at 1; assigned by the backend when a pipeline run creates the version record | P0 | 2 | implemented |
| FR-CVER-003 | Only one version may have `status = published` per `(curriculum_id, subject)` at any time; publishing a new version atomically archives the previously published version | P0 | 7 | proposed |
| FR-CVER-004 | Version history is accessible via `GET /admin/content/versions?curriculum_id=&subject=` | P1 | 7 | proposed |
| FR-CVER-005 | Rollback (`POST /admin/content/versions/{version_id}/rollback`) sets target version (must be `archived`) to `published` and atomically archives the current published version | P1 | 7 | proposed |
| FR-CVER-006 | Both publish and rollback invalidate L2 Redis cache keys for the affected subject and issue a CloudFront CDN invalidation | P0 | 7 | proposed |
| FR-CVER-007 | A content block (`POST /admin/content/block`) overrides published status; any active block causes the content endpoint to return HTTP 403; multiple blocks can coexist | P0 | 7 | proposed |
| FR-CVER-008 | Blocks can be school-scoped (`school_id` set) or platform-wide (`school_id = null`); school-scoped blocks only affect students of that school | P0 | 7 | proposed |
| FR-CVER-009 | Block can be lifted at any time by the same or higher-privilege user (`DELETE /admin/content/block/{block_id}`) | P0 | 7 | proposed |
| FR-CVER-010 | Block and publish actions are written to the `audit_log` table | P1 | 7 | proposed |

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
| NFR-COMP-004 | Data retention policy: progress records retained for account lifetime then anonymised (strip `student_id`) after deletion | P1 | 5 | proposed |
| NFR-COMP-005 | PII minimisation: collect only name, email, grade, locale; no location, no device ID, no behavioural fingerprinting | P1 | 1 | proposed |
| NFR-COMP-006 | Stripe handles all payment data; backend never stores card numbers or CVV | P0 | 5 | accepted |
| NFR-COMP-007 | Password reset tokens expire after 1 hour and are single-use | P0 | 1 | proposed |
| NFR-COMP-008 | FERPA: quiz scores, lesson-view history, session records, and progress data are educational records; access is scoped to the owning student or their school's teachers/admins | P0 | 1 | proposed |
| NFR-COMP-009 | FERPA: student educational records must not be shared with third-party analytics or advertising services | P0 | 1 | proposed |
| NFR-COMP-010 | FERPA: `student_id` is never accepted as a query or body parameter on student-facing endpoints; always derived from the verified JWT | P0 | 1 | proposed |
| NFR-COMP-011 | FERPA: all teacher and admin access to student records is written to `audit_log` to satisfy access tracking requirements | P0 | 1 | proposed |
| NFR-COMP-012 | FERPA: `GET /student/export` provides a full JSON download of all records for the student, for parental inspection requests | P1 | 7 | proposed |
| NFR-COMP-013 | No real student PII in development or test environments; use synthetic data only; CI must not connect to production databases | P0 | 1 | proposed |

---

### NFR-A11Y: Accessibility

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-A11Y-001 | All student-facing UI (mobile app and any web views) must target WCAG 2.1 Level AA | P0 | 3 | proposed |
| NFR-A11Y-002 | Minimum color contrast ratio: 4.5:1 for normal text, 3:1 for large text (≥ 18pt or ≥ 14pt bold) | P0 | 3 | proposed |
| NFR-A11Y-003 | Text must be resizable up to 200% without loss of content or functionality | P1 | 3 | proposed |
| NFR-A11Y-004 | All interactive elements must have a minimum touch target of 44 × 44 dp on Android | P1 | 3 | proposed |
| NFR-A11Y-005 | All images, icons, and non-text content must have `contentDescription` (Android) or `aria-label` (web) | P0 | 3 | proposed |
| NFR-A11Y-006 | Quiz answer options must be announced by screen readers with option letter/number and full text | P0 | 3 | proposed |
| NFR-A11Y-007 | Progress indicators (streak, completion %) must be announced as text, not only rendered visually | P1 | 3 | proposed |
| NFR-A11Y-008 | Lesson audio (TTS) must have a visible text alternative — synopsis text is always shown alongside the audio player | P0 | 4 | proposed |
| NFR-A11Y-009 | Student-facing error messages must use plain language; no HTTP status codes, stack traces, or internal identifiers visible to students | P0 | 1 | proposed |
| NFR-A11Y-010 | Error messages must be announced automatically by screen readers (`aria-live="assertive"` on web; `AccessibilityEvent` on Android) | P1 | 3 | proposed |
| NFR-A11Y-011 | Information must not be conveyed by color alone; icons, labels, or patterns must accompany color coding | P1 | 3 | proposed |
| NFR-A11Y-012 | Android implementation uses Jetpack Compose accessibility modifiers; tested with TalkBack before each phase release | P1 | 3 | proposed |
| NFR-A11Y-013 | No time limits on quiz sessions without an option to extend or disable (WCAG 2.2.1) | P1 | 3 | proposed |

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
| NFR-OBS-001 | All backend log entries are structured JSON (stdout); aggregated to Loki / CloudWatch / ELK | P0 | 1 | proposed |
| NFR-OBS-002 | Every log entry carries `correlation_id` (UUID per request), `event_category`, `event_type`, `component` | P0 | 1 | proposed |
| NFR-OBS-003 | `correlation_id` injected by FastAPI middleware; stored in `contextvars.ContextVar`; returned as `X-Correlation-Id` response header | P1 | 1 | proposed |
| NFR-OBS-004 | All event emissions use `emit_event(category, event_type, **ctx)` from `core/events.py`; no ad-hoc `log.info()` calls for business events | P1 | 1 | proposed |
| NFR-OBS-005 | `GET /metrics` endpoint in Prometheus exposition format; protected by `METRICS_TOKEN` bearer token; blocked by nginx for external requests | P0 | 2 | proposed |
| NFR-OBS-006 | HTTP metrics auto-instrumented via `prometheus-fastapi-instrumentator`: `http_requests_total`, `http_request_duration_seconds` | P0 | 2 | proposed |
| NFR-OBS-007 | All custom business metrics defined in `core/observability.py` (see ARCHITECTURE.md Metrics table); no metric definitions scattered across service files | P1 | 2 | proposed |
| NFR-OBS-008 | Grafana provisioned with six dashboards (Platform Overview, Content & Cache, Student Activity, Backend Health, Security, SLO Burn Rate) | P1 | 2 | proposed |
| NFR-OBS-009 | Alertmanager configured with rules file `docs/prometheus/alerts.yml`; routes critical→PagerDuty, high→Slack `#studybuddy-alerts`, payments→email, pipeline→email | P0 | 2 | proposed |
| NFR-OBS-010 | Sentry integrated in backend (`sentry-sdk[fastapi]`) and mobile; PII stripped via `before_send`; `correlation_id` attached to every Sentry event | P0 | 1 | proposed |
| NFR-OBS-011 | Sentry alert rules: new issue → Slack, regression → Slack, >10 events/5 min → Slack `#studybuddy-incidents` + email on-call | P1 | 1 | proposed |
| NFR-OBS-012 | Security-critical events (account status changes, GDPR deletions, admin logins, rate-limit exceeded) written to `audit_log` table in PostgreSQL in addition to the log stream | P0 | 1 | proposed |
| NFR-OBS-013 | `audit_log` rows are never deleted; PII in `metadata` is anonymised on GDPR schedule when account is deleted | P0 | 5 | proposed |
| NFR-OBS-014 | `GET /health` deep check covers DB, Redis, Content Store, Auth0 JWKS cache; returns 503 on any failure; used as ECS/K8s readiness probe | P0 | 1 | proposed |
| NFR-OBS-015 | Uptime monitor pings `/health` every 60 seconds from an external location; alert on 3 consecutive failures | P0 | 1 | proposed |
| NFR-OBS-016 | Celery queue depth (`sb_celery_queue_depth`) polled every 30 s; alert when `pipeline` queue > 50 for > 10 min | P1 | 2 | proposed |
| NFR-OBS-017 | DB pool state (`sb_db_pool_connections`) polled by middleware; alert when `waiting > 5` for > 2 min | P1 | 2 | proposed |
| NFR-OBS-018 | Redis eviction rate monitored; alert when > 100 keys/min evicted (indicates memory pressure) | P1 | 2 | proposed |
| NFR-OBS-019 | Pipeline run report emitted as structured event on completion; if `failed > 0`, email sent to `PIPELINE_ALERT_EMAIL` via Celery + SES | P1 | 2 | proposed |
| NFR-OBS-020 | CDN cache hit rate monitored via CloudFront metrics; alert if rate drops below 80% | P2 | 4 | proposed |
| NFR-OBS-021 | Mobile crash reports uploaded to Sentry on next network-connected launch; offline queue capped at 50 events | P1 | 2 | proposed |
| NFR-OBS-022 | SLO burn rate dashboard; engineering team reviews weekly; p95 breach fires `ContentLatencySLOBreach` alert | P1 | 2 | proposed |

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

### NFR-DR: Disaster Recovery

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-DR-001 | PostgreSQL primary backed up daily (snapshot or `pg_dump`) at 02:00 UTC; 30-day retention; stored in a separate bucket/path from production | P0 | 1 | proposed |
| NFR-DR-002 | Continuous WAL archiving enabled; WAL segments retained for 7 days | P0 | 1 | proposed |
| NFR-DR-003 | Replica promotion runbook documented and tested; RTO ≤ 15 minutes; RPO ≤ 5 minutes (WAL streaming lag) | P0 | 1 | proposed |
| NFR-DR-004 | Point-in-time restore tested monthly to a staging instance; result recorded in the incident log | P1 | 1 | proposed |
| NFR-DR-005 | Redis configured with AOF persistence (`appendonly yes`, `appendfsync everysec`); RDB snapshot daily | P0 | 1 | proposed |
| NFR-DR-006 | Content Store backed up via daily incremental snapshot at 03:00 UTC; 30-day retention; cross-region copy at Scale tier | P1 | 4 | proposed |
| NFR-DR-007 | Full service RTO ≤ 4 hours; RPO ≤ 24 hours in catastrophic failure scenario | P1 | 1 | proposed |
| NFR-DR-008 | Secrets Manager versioning enabled; full version history retained | P0 | 1 | proposed |
| NFR-DR-009 | Staging and dev environments must never contain real student PII; CI must never connect to production databases | P0 | 1 | proposed |
| NFR-DR-010 | SLA target: 99.9% uptime (≤ 8.7 hours unplanned downtime per year) | P1 | 1 | proposed |

---

### NFR-SCALE: Scalability & Load Testing

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-SCALE-001 | Hot read path (lesson, quiz) must serve ≤ 50 ms p95 at 10,000 concurrent students with cache-warm L2 Redis | P0 | 2 | proposed |
| NFR-SCALE-002 | Database connection pooling via PgBouncer (transaction mode); `max_connections ≥ 200`; asyncpg pool `min=5, max=20` per worker | P0 | 1 | proposed |
| NFR-SCALE-003 | Pre-release load test required for any change to the content or auth hot path; gate blocks deploy on SLO breach | P0 | 2 | proposed |
| NFR-SCALE-004 | Load test tool: k6; minimum scenarios: baseline (steady state), content spike, auth storm, progress write storm | P1 | 2 | proposed |
| NFR-SCALE-005 | `progress_sessions`, `lesson_views`, `progress_answers`, `audit_log` tables range-partitioned by month when rows exceed 50 M; `CREATE INDEX CONCURRENTLY` for all index additions | P1 | 5 | proposed |
| NFR-SCALE-006 | Redis Sentinel used at < 100k students; Redis Cluster (3 primary shards + 3 replicas) introduced when total keys exceed 5 M or dataset exceeds 4 GB | P1 | 5 | proposed |
| NFR-SCALE-007 | Multi-region deployment (US primary + EU standby) introduced at Scale tier (100k+ students); GeoDNS routing; EU student PII must not leave EU region | P2 | 5 | proposed |
| NFR-SCALE-008 | Pipeline quota enforced: 3 runs/school/day; 200 units/run; 5 platform-wide concurrent pipeline jobs; $50 per-run spend cap | P1 | 8 | proposed |
| NFR-SCALE-009 | CDN cache hit rate ≥ 80%; alert if rate drops below threshold (monitored via CDN metrics) | P1 | 4 | proposed |
| NFR-SCALE-010 | API worker count scales horizontally; stateless design ensures any worker can handle any request | P0 | 1 | proposed |
| NFR-SCALE-011 | Soak test (8-hour sustained load at 60% peak) run monthly; results reviewed by engineering team | P2 | 3 | proposed |

---

### NFR-PUSH: Push Notifications

| ID | Requirement | Priority | Phase | Status |
|---|---|---|---|---|
| NFR-PUSH-001 | Push notifications delivered via FCM (Android); iOS support added when mobile client ships to iOS | P1 | 4 | proposed |
| NFR-PUSH-002 | FCM tokens stored in `push_tokens` table; token registered on login; deleted on logout or explicit unregister | P1 | 4 | proposed |
| NFR-PUSH-003 | Notification preferences stored per student; all notification types default-on except quiz nudges (default-off) | P2 | 4 | proposed |
| NFR-PUSH-004 | Maximum 2 push notifications per student per day (global rate limit across all event types) | P1 | 4 | proposed |
| NFR-PUSH-005 | Quiet hours respected; no notifications sent between `quiet_hours_start` and `quiet_hours_end` in student's locale timezone | P1 | 4 | proposed |
| NFR-PUSH-006 | Notification types: streak at risk (daily), new content available (on publish), quiz nudge (48 h after lesson view without quiz start), weekly summary (Sunday) | P2 | 4 | proposed |
| NFR-PUSH-007 | Push notification tasks run via Celery Beat queue `io`; delivery failures retried up to 3 times with 60-second backoff | P1 | 4 | proposed |
| NFR-PUSH-008 | Stale FCM tokens (no `last_seen_at` update in 90 days) purged by Celery Beat weekly | P2 | 4 | proposed |

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
| ADR-022 | External auth (Auth0) for students and teachers; backend issues internal JWT after token exchange | Removes password storage and brute-force defence from our codebase; Auth0 handles MFA, social login, COPPA age-gate, and password reset email delivery; provider is swappable (Cognito, Firebase) by changing the JWKS URL and exchange logic | 2026-03-23 | proposed |
| ADR-023 | Local bcrypt auth for internal product team; separate `ADMIN_JWT_SECRET` | Internal tooling must not depend on external provider availability; admin credentials are provisioned manually and tightly controlled; separate secret ensures a compromised student JWT cannot be replayed on admin endpoints | 2026-03-23 | proposed |
| ADR-024 | Suspension check via Redis set in auth middleware; Auth0 block is best-effort sync | Checking `account_status` in PostgreSQL on every request would add a DB query to the hot path; Redis set (no TTL, explicitly deleted on reactivation) gives instant revocation with zero DB impact; Auth0 block prevents re-login after JWT expires | 2026-03-23 | proposed |
| ADR-025 | RBAC via static in-code `ROLE_PERMISSIONS` map, not a DB-backed permission table | Seven roles with fixed permissions are sufficient for Phases 1–11; a DB-backed system adds migration overhead and a DB query per request for no benefit during this phase; the map can be replaced with a DB-backed system later without changing the `require_permission()` call sites | 2026-03-24 | proposed |
| ADR-026 | AlexJS for automated content language analysis in the pipeline | Detects insensitive language (profanity, gendered phrasing, ableist terms) appropriate for educational content; Node.js tool invoked via subprocess from Python pipeline; warnings feed into the review queue rather than blocking generation, preserving pipeline throughput | 2026-03-24 | proposed |
| ADR-027 | Content versioning at subject level, not unit level | Approving one unit in isolation could expose partially reviewed content; the subject version is the atomic publication unit; simpler for reviewers (one approval covers all units in a subject); rollback is always a coherent subject snapshot | 2026-03-24 | proposed |

---

## Change Log

| Date | Version | Author | Change |
|---|---|---|---|
| 2026-03-23 | 0.1.0 | — | Initial requirements document created from architecture review |
| 2026-03-23 | 0.2.0 | — | Added school/teacher management, custom curriculum, student association, extended analytics, and feedback requirements (Phases 8–10) |
| 2026-03-23 | 0.3.0 | — | Added BACKEND_ARCHITECTURE.md; updated NFR-PERF, NFR-REL, NFR-OBS to align with SLOs and architecture decisions; added ADR-015 through ADR-018 |
| 2026-03-23 | 0.4.0 | — | Added Teacher & School Reporting system: 6 report types, CSV export, threshold alerts, weekly digest, 3 new materialized views, Phase 11 |
