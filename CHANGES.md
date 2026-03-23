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

---

---

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

*Last updated: 2026-03-23*
