# Diagram 11 — API Index

> Complete endpoint inventory grouped by service, with method, path, auth role, and purpose.
> All routes are prefixed `/api/v1/`. Audience: Developers, API consumers.
> Last updated: 2026-04-05.

---

## Summary

| Service Group | Endpoints | Auth |
|---|---|---|
| Auth — Students & Teachers | 7 | Public / Student JWT / Teacher JWT |
| Auth — Admin | 4 | Public / Admin JWT |
| Admin Account Management | 8 | product_admin+ |
| Curriculum (student-facing) | 2 | Student JWT |
| Content | 8 | Student JWT |
| Content Review & Approval | 14 | review:read / review:annotate / content:publish |
| Progress | 5 | Student JWT |
| Subscription | 4 | Student JWT / Public (webhook) |
| School & Teacher | 9 | Teacher JWT / school_admin JWT |
| Curriculum Upload & Pipeline | 6 | school_admin JWT / Admin JWT |
| Analytics | 6 | Student JWT / Teacher JWT / Admin JWT |
| Feedback | 2 | Student JWT / Admin JWT |
| Reports | 12 | Teacher JWT / school_admin JWT |
| Demo | 5 | Public / Demo token |
| Observability | 3 | Public (/healthz, /readyz) / Bearer (/metrics) |
| **Total** | **95** | |

---

## Auth — Students & Teachers

Base: `/api/v1/`

```
POST   /auth/exchange                Student or Teacher
         Body:  {id_token: str}
         Returns: {token, refresh_token, student_id}
         Notes: Auth0 id_token → internal JWT; upserts student row

POST   /auth/teacher/exchange        Teacher
         Body:  {id_token: str}
         Returns: {token, refresh_token, teacher_id}

POST   /auth/refresh                 Student JWT
         Body:  {refresh_token: str}
         Returns: {token}
         Notes: Refresh token stored in Redis (TTL 30 days)

POST   /auth/logout                  Student JWT
         Body:  {refresh_token: str}
         Returns: 200
         Notes: Deletes refresh token from Redis

POST   /auth/forgot-password         Public
         Body:  {email: str}
         Returns: 200 always  ← never leaks whether email exists

PATCH  /student/profile              Student JWT
         Body:  {name?, locale?, grade?}
         Returns: {student}

DELETE /auth/account                 Student JWT
         Returns: 200
         Notes: GDPR erasure — schedules Celery task; Auth0 user deletion
```

---

## Auth — Admin

Base: `/api/v1/admin/auth/`

```
POST   /admin/auth/login             Public
         Body:  {email: str, password: str}
         Returns: {token, admin_id}
         Notes: bcrypt verify; separate ADMIN_JWT_SECRET

POST   /admin/auth/refresh           Admin JWT
         Body:  {refresh_token: str}
         Returns: {token}

POST   /admin/auth/forgot-password   Public
         Body:  {email: str}
         Returns: 200 always

POST   /admin/auth/reset-password    Public (token in body)
         Body:  {token: str, new_password: str}
         Returns: 200
         Notes: Token is single-use UUID stored in Redis (TTL 1 hr)
```

---

## Admin Account Management

Base: `/api/v1/admin/accounts/`  ·  Requires: `product_admin` or `super_admin`

```
GET    /admin/accounts/students              ?status=&school_id=&q=&page=
         Returns: paginated student list

GET    /admin/accounts/students/{id}
         Returns: student detail

PATCH  /admin/accounts/students/{id}/status
         Body:  {status: active|suspended}
         Notes: sets Redis suspended:{id}; writes audit_log

DELETE /admin/accounts/students/{id}
         Returns: 202
         Notes: schedules GDPR deletion Celery task

GET    /admin/accounts/teachers              ?status=&school_id=&page=
         Returns: paginated teacher list

PATCH  /admin/accounts/teachers/{id}/status
         Body:  {status: active|suspended}

GET    /admin/accounts/schools               ?status=&page=
         Returns: paginated school list

PATCH  /admin/accounts/schools/{id}/status
         Body:  {status: active|suspended}
         Notes: suspension cascades to all teachers + students via Celery
```

---

## Curriculum (Student-Facing)

Base: `/api/v1/curriculum/`  ·  Requires: Student JWT

```
GET    /curriculum
         Returns: [{grade, subject_count, curriculum_id}]
         Notes: L1/L2 cached; locale from JWT

GET    /curriculum/{grade}
         Returns: full subject + unit tree for grade
         Notes: curriculum_id resolved from JWT (school or default)
```

---

## Content

Base: `/api/v1/content/`  ·  Requires: Student JWT  ·  Entitlement gated

```
GET    /content/{unit_id}/lesson
         Returns: {synopsis, key_concepts, ...}
         Notes: locale from JWT; L2 cached; count against lesson limit

GET    /content/{unit_id}/lesson/audio
         Returns: {url: presigned_s3_url, expires_in_s: 3600}
         Notes: Client fetches MP3 directly from S3/CDN — NOT proxied through API

GET    /content/{unit_id}/quiz
         Returns: {questions: [8 rotated questions]}
         Notes: rotation ensures different set each attempt

GET    /content/{unit_id}/practice
         Returns: {questions: [practice set]}

GET    /content/{unit_id}/tutorial
         Returns: {sections: [{title, body, ...}]}
         Notes: rendered as tabs in UI

GET    /content/{unit_id}/experiment
         Returns: {steps, materials, diagram_hint}
         Notes: 404 if unit has_lab = false

POST   /content/{unit_id}/report
         Body:  {category: str, message?: str}
         Returns: 200

POST   /content/{unit_id}/feedback/marked
         Body:  {content_type, start_offset, end_offset, marked_text?, feedback_text}
         Returns: 200
         Notes: rate limited to 5 submissions per student per hour
```

---

## Content Review & Approval

Base: `/api/v1/admin/content/`  ·  Permission column from `permissions.py`

```
GET    /admin/content/review/queue           review:read
         ?curriculum_id=&subject=&status=&page=
         Returns: paginated version list with has_content flag

GET    /admin/content/review/{version_id}    review:read
         Returns: version detail + annotations summary

POST   /admin/content/review/{version_id}/open       review:annotate
         Returns: {review_id}

POST   /admin/content/review/{version_id}/annotate   review:annotate
         Body:  {unit_id, content_type, start_offset, end_offset,
                 original_text, annotation_type, suggestion_text?}
         Returns: {annotation_id}
         Notes: compound key {unit_id}::{content_type}::{section_id}

DELETE /admin/content/review/annotations/{annotation_id}  review:annotate
         Returns: 200

GET    /admin/content/dictionary             review:read
         ?word=
         Returns: {definitions, synonyms, alternatives}

POST   /admin/content/review/{version_id}/rate       review:rate
         Body:  {language_rating, content_rating, notes?}
         Returns: 200

POST   /admin/content/review/{version_id}/approve    review:approve
         Body:  {notes?}
         Returns: 200

POST   /admin/content/review/{version_id}/reject     review:approve
         Body:  {reason, regenerate: bool}
         Returns: 202 if regenerate=true (triggers pipeline Celery task)

GET    /admin/content/versions               review:read
         ?curriculum_id=&subject=&page=
         Returns: paginated version history

POST   /admin/content/versions/{version_id}/publish  content:publish
         Returns: 200  ← archives current live version; invalidates CDN + Redis

POST   /admin/content/versions/{version_id}/rollback content:rollback
         Returns: 200  ← archives current; restores this version

POST   /admin/content/block                  content:block
         Body:  {curriculum_id, subject, school_id?, reason}
         Returns: {block_id}

DELETE /admin/content/block/{block_id}       content:block
         Returns: 200

GET    /admin/content/blocks                 review:read
         ?curriculum_id=&school_id=&page=
         Returns: paginated active blocks

GET    /admin/content/{unit_id}/feedback/marked  review:read
         ?page=
         Returns: paginated student marked-text feedback
```

---

## Progress

Base: `/api/v1/progress/`  ·  Requires: Student JWT  ·  Writes: fire-and-forget

```
POST   /progress/session
         Body:  {unit_id, grade, subject}
         Returns: {session_id}
         Notes: attempt_number computed server-side

POST   /progress/answer
         Body:  {session_id, question_id, answer, correct, ms_taken}
         Returns: 200  ← dispatches Celery task; does NOT await DB write

POST   /progress/session/end
         Body:  {session_id, score, duration_s, completed}
         Returns: 200  ← dispatches Celery task

GET    /progress/student
         Returns: full history + scores per unit

GET    /progress/unit/{unit_id}
         Returns: {attempts: [...], best_score, last_attempt_at}
```

---

## Subscription

Base: `/api/v1/subscription/`

```
GET    /subscription/status          Student JWT
         Returns: {plan, valid_until, lessons_accessed, lessons_limit}

POST   /subscription/checkout        Student JWT
         Body:  {plan: monthly|annual}
         Returns: {checkout_url}  ← Stripe Checkout Session URL

POST   /subscription/webhook         Public (Stripe IPs only — nginx allowlist)
         Body:  Stripe event payload
         Returns: 200
         Notes: signature verified first; idempotent by stripe_event_id

DELETE /subscription                 Student JWT
         Returns: 200  ← cancel_at_period_end = true via Stripe API
```

---

## School & Teacher

Base: `/api/v1/schools/`  ·  Requires: Teacher JWT or school_admin JWT

```
POST   /schools/register                     Public
         Body:  {school_name, contact_email, country}
         Returns: {school_id, teacher_id}
         Notes: creates school + first school_admin teacher account

GET    /schools/{school_id}                  Teacher JWT
         Returns: school profile

POST   /schools/{school_id}/teachers/invite  school_admin JWT
         Body:  {email, name}
         Returns: 200  ← sends invite email via SES

POST   /schools/{school_id}/enrolment        school_admin JWT
         Body:  {students: [{email, grade?, teacher_id?}]}
         Returns: {enrolled, already_enrolled, errors: [{email, reason}]}
         Notes: grade+teacher_id optional at upload; set later via assignment endpoint

GET    /schools/{school_id}/enrolment        school_admin JWT
         Returns: roster with status, assigned_teacher_id, assigned_grade per student

GET    /schools/{school_id}/students/{student_id}/assignment  Teacher JWT
         Returns: StudentAssignmentResponse

PUT    /schools/{school_id}/students/{student_id}/assignment  school_admin JWT
         Body:  {teacher_id, grade}
         Returns: StudentAssignmentResponse
         Notes: school is sole authority on grade; student self-change blocked

POST   /schools/{school_id}/teachers/{from_teacher_id}/reassign  school_admin JWT
         Body:  {to_teacher_id, grade}
         Returns: {reassigned: int}
         Notes: moves all students in a grade from one teacher to another
```

---

## Curriculum Upload & Pipeline

Base: `/api/v1/curriculum/`  ·  Requires: school_admin JWT or Admin JWT

```
POST   /curriculum/upload            school_admin JWT
         Body:  multipart XLSX  |  {grade, units: [...]} JSON
         Returns: {curriculum_id, status, errors: [{row, field, message}]}
         Notes: errors are HTTP 400 with structured per-row error list

GET    /curriculum/{curriculum_id}   school_admin JWT
         Returns: curriculum metadata + unit list

PUT    /curriculum/{curriculum_id}/activate  school_admin JWT
         Returns: 200  ← activates; archives previous active curriculum for grade

POST   /curriculum/pipeline/trigger  school_admin JWT | Admin JWT
         Body:  {curriculum_id, lang?, force?}
         Returns: {job_id}  ← async Celery task; poll status endpoint

GET    /curriculum/pipeline/{job_id}/status  school_admin JWT | Admin JWT
         Returns: {status, built, failed, progress_pct, duration_s}

GET    /curriculum/template          school_admin JWT
         Returns: XLSX template file download
```

---

## Analytics

Base: `/api/v1/analytics/`

```
POST   /analytics/lesson/start       Student JWT  — fire-and-forget
         Body:  {unit_id, curriculum_id}
         Returns: {view_id}

POST   /analytics/lesson/end         Student JWT  — fire-and-forget
         Body:  {view_id, duration_s, audio_played, experiment_viewed}
         Returns: 200

GET    /analytics/student/me         Student JWT
         Returns: {scores, time_on_topic, attempts} per unit

GET    /analytics/student/stats      Student JWT
         Returns: {streak_days, session_dates, lessons_viewed, quizzes_completed,
                   pass_rate, avg_score, audio_sessions, subject_breakdown}

GET    /analytics/school/{school_id}/class   Teacher JWT
         ?grade=&subject=
         Returns: aggregate class metrics per unit

GET    /analytics/unit/{unit_id}/summary     Admin JWT
         Returns: platform-wide metrics (all schools)
```

---

## Feedback

```
POST   /feedback                     Student JWT
         Body:  {category, unit_id?, message, rating?}
         Returns: {feedback_id}
         Notes: rate limited to 5 submissions per student per hour

GET    /admin/feedback               Admin JWT (review:read)
         ?unit_id=&category=&reviewed=&page=
         Returns: paginated feedback list
```

---

## Reports

Base: `/api/v1/reports/school/{school_id}/`  ·  Requires: Teacher JWT or school_admin JWT
Student records scoped to `school_id` from JWT — cross-school access is blocked.

```
GET    /reports/school/{school_id}/overview
         ?period=7d|30d|term
         Returns: class overview summary

GET    /reports/school/{school_id}/unit/{unit_id}
         ?period=7d|30d|term
         Returns: unit performance deep-dive

GET    /reports/school/{school_id}/student/{student_id}
         Returns: individual student report card

GET    /reports/school/{school_id}/curriculum-health
         Returns: all units ranked by health tier (green/amber/red)

GET    /reports/school/{school_id}/feedback
         ?unit_id=&category=&reviewed=
         Returns: feedback by unit

GET    /reports/school/{school_id}/trends
         ?period=4w|12w|term
         Returns: week-over-week trend data

GET    /reports/school/{school_id}/roster
         ?grade=
         Returns: per-student rows {student_id, name, grade, completed_units,
                  avg_score_pct, last_active}

POST   /reports/school/{school_id}/export
         Body:  {report_type, filters}
         Returns: {download_url}  ← S3 presigned URL; CSV

GET    /reports/school/{school_id}/alerts
         Returns: active threshold alerts

PUT    /reports/school/{school_id}/alerts/settings
         Body:  {pass_rate_threshold, inactivity_days, ...}
         Returns: 200

POST   /reports/school/{school_id}/digest/subscribe
         Body:  {email, frequency: weekly|monthly}
         Returns: 200

POST   /reports/school/{school_id}/refresh          school_admin JWT only
         Returns: 200  ← triggers on-demand materialized view refresh (Celery)
```

---

## Demo

Base: `/api/v1/demo/`

```
POST   /demo/request                 Public
         Body:  {email, name, grade?}
         Returns: 200  ← sends magic link email

POST   /demo/verify                  Public
         Body:  {token: str}
         Returns: {demo_token}  ← short-lived JWT with demo entitlement

POST   /demo/login                   Public
         Body:  {email, password}
         Returns: {token, student_id}

POST   /demo/teacher/request         Public
         Body:  {email, name, school_name}
         Returns: 200

POST   /demo/teacher/verify          Public
         Body:  {token: str}
         Returns: {demo_teacher_token}
```

---

## Observability

```
GET    /healthz                      Public
         Returns: 200 {"status": "ok"}
         Notes: liveness probe — always 200 if process is alive
                never checks DB or Redis (fast, no deps)

GET    /readyz                       Public
         Returns: 200 {"status": "ok", "db": "ok", "redis": "ok"}
                  503 if any dependency is unavailable
         Notes: readiness probe — used by ALB health check
                checks DB connection + Redis ping

GET    /metrics                      Bearer METRICS_TOKEN
         Returns: Prometheus text format
         Notes: nginx blocks from public internet; internal scrape only
```

---

## API Versioning Policy

- All routes under `/api/v1/`. Future breaking changes use `/api/v2/`.
- **Additive changes** (new fields in response, new optional query params) are non-breaking and deployed without version bump.
- **Removing or renaming fields** requires a v2 endpoint with a 6-month deprecation notice.
- The `X-API-Version` response header echoes the active version on every response.
- Mobile app checks `GET /app/version` on startup; if `min_version > client_version`, the user is prompted to upgrade before any API call proceeds.
