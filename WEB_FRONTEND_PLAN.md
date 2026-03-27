# StudyBuddy OnDemand — Web Frontend Plan

Version 0.1.0 | 2026-03-27

---

## Overview

The web frontend is a **React / Next.js** application that consumes the existing
backend REST API (`/api/v1/`). It runs independently of the Kivy mobile app but
shares the same backend, the same Auth0 authentication flow, and the same content
store.

Three distinct portals share one codebase with role-based routing:

| Portal | Audience | Auth path |
|---|---|---|
| **Student Portal** | Students (Grades 5–12), parents | Auth0 `id_token` → `/auth/exchange` |
| **School Portal** | Teachers, school admins | Auth0 `id_token` → `/auth/teacher/exchange` |
| **Admin Console** | Internal team (developer, tester, product_admin, super_admin) | bcrypt → `/admin/auth/login` |

---

## Page Inventory

### 1. Public Pages (no auth required)

| # | Page | Route | Description |
|---|---|---|---|
| P-01 | Landing / Marketing | `/` | Hero, features, pricing CTA, testimonials, sign-up link |
| P-02 | Pricing | `/pricing` | B2C plan tiers, B2B school plan, FAQ |
| P-03 | Student Sign-Up | `/signup` | Auth0 Universal Login redirect; collects grade + locale |
| P-04 | Student Sign-In | `/login` | Auth0 redirect; redirects to `/dashboard` on success |
| P-05 | Teacher / School Sign-In | `/school/login` | Auth0 redirect for teacher accounts |
| P-06 | COPPA Parental Consent | `/consent` | Required before under-13 accounts activate |
| P-07 | Password Reset | `/reset-password` | Email-based reset link (token from backend) |
| P-08 | Terms of Service | `/terms` | Static legal page |
| P-09 | Privacy Policy | `/privacy` | Static legal page; COPPA / GDPR / FERPA sections |
| P-10 | Contact / Support | `/contact` | Support form + FAQ |

---

### 2. Student Portal (role: student)

| # | Page | Route | Description |
|---|---|---|---|
| S-01 | Student Dashboard | `/dashboard` | Streak counter, recent activity, quick-resume, announcements |
| S-02 | Subject Browser | `/subjects` | Grade-filtered grid of subjects → units |
| S-03 | Curriculum Map | `/curriculum` | Visual map of units, completion status, locked/unlocked |
| S-04 | Lesson View | `/lesson/:unit_id` | Lesson text, audio player (CDN URL), progress auto-save |
| S-05 | Quiz | `/quiz/:unit_id` | Multiple-choice questions, immediate feedback, score screen |
| S-06 | Tutorial View | `/tutorial/:unit_id` | Step-by-step tutorial content |
| S-07 | Experiment View | `/experiment/:unit_id` | Lab-bearing units: step guide + materials list |
| S-08 | Progress History | `/progress` | Session history, per-unit pass/fail timeline |
| S-09 | Stats & Streaks | `/stats` | Study time, accuracy trends, subject breakdown, streak calendar |
| S-10 | Subscription / Paywall | `/subscribe` | Stripe checkout embed; plan selection; trial status |
| S-11 | Subscription Management | `/account/subscription` | Current plan, billing portal link, cancellation |
| S-12 | Account Settings | `/account/settings` | Display name, locale/language, notification preferences |
| S-13 | Feedback | `/feedback` | Per-unit thumbs up/down + optional text; rate-limited |
| S-14 | Paywall Gate | `/paywall` | Shown on 402 from any content endpoint; upsell CTA |
| S-15 | Enrolment Confirm | `/enrol/:token` | Student accepts school enrolment invite link |

---

### 3. School Portal (roles: teacher, school_admin)

| # | Page | Route | Description |
|---|---|---|---|
| T-01 | Teacher Dashboard | `/school/dashboard` | Class summary cards, alert badges, quick links |
| T-02 | Class Overview | `/school/class/:class_id` | Student list, completion rates, avg scores per unit |
| T-03 | Student Detail | `/school/student/:student_id` | Per-student progress, quiz history, feedback submitted |
| T-04 | Overview Report | `/school/reports/overview` | Completion %, avg score, time-on-topic across class |
| T-05 | Trends Report | `/school/reports/trends` | Week-over-week lesson views and quiz scores (chart) |
| T-06 | At-Risk Report | `/school/reports/at-risk` | Students below pass threshold; drill-down to unit |
| T-07 | Unit Performance Report | `/school/reports/units` | Per-unit pass rate, avg attempts, avg score |
| T-08 | Engagement Report | `/school/reports/engagement` | Active students, streak leaders, dropout risk |
| T-09 | Feedback Report | `/school/reports/feedback` | Unreviewed student feedback; mark-reviewed action |
| T-10 | CSV Export | `/school/reports/export` | Select report + date range → download CSV |
| T-11 | Curriculum Management | `/school/curriculum` | View active curriculum, upload XLSX, trigger pipeline |
| T-12 | Pipeline Status | `/school/curriculum/pipeline/:job_id` | Poll job status; error summary on failure |
| T-13 | Student Roster | `/school/students` | Enrolled students; invite link generator; remove student |
| T-14 | Teacher Management | `/school/teachers` | school_admin only; invite teacher; role assignment |
| T-15 | School Settings | `/school/settings` | school_admin only; school name, logo, billing |
| T-16 | Alert Inbox | `/school/alerts` | Threshold breach alerts; dismiss / mark-actioned |
| T-17 | Weekly Digest Archive | `/school/digest` | View past weekly digest emails in-browser |

---

### 4. Admin Console (roles: developer, tester, product_admin, super_admin)

| # | Page | Route | Description |
|---|---|---|---|
| A-01 | Admin Login | `/admin/login` | Local bcrypt login; separate from Auth0 |
| A-02 | Platform Dashboard | `/admin/dashboard` | DAU/WAU/MAU, revenue, lesson views, error rate |
| A-03 | Platform Analytics | `/admin/analytics` | Full funnel: signups → trial → paid; churn; cohort |
| A-04 | Content Pipeline | `/admin/pipeline` | Per-grade pipeline status, last run, success/fail counts |
| A-05 | Pipeline Trigger | `/admin/pipeline/trigger` | Trigger build for grade + language; view job_id |
| A-06 | Pipeline Job Detail | `/admin/pipeline/:job_id` | Unit-level status, error messages, token spend |
| A-07 | Content Review Queue | `/admin/content-review` | Units pending review; filter by grade/subject/lang |
| A-08 | Content Review Detail | `/admin/content-review/:unit_id` | Full lesson/quiz/tutorial preview + AlexJS results; annotate |
| A-09 | Approve / Publish | (modal on A-08) | Approve, flag, rollback, or block a content version |
| A-10 | Curriculum List | `/admin/curricula` | All curricula (default + school); activation status |
| A-11 | Curriculum Detail | `/admin/curricula/:curriculum_id` | Units, languages built, content versions, pipeline history |
| A-12 | User Management | `/admin/users` | Search by email/ID; view role, status, subscription |
| A-13 | User Detail | `/admin/users/:user_id` | Full profile; suspend/activate; subscription override |
| A-14 | School Management | `/admin/schools` | All registered schools; status, subscription plan |
| A-15 | School Detail | `/admin/schools/:school_id` | Teachers, students, curriculum, billing |
| A-16 | Subscription Management | `/admin/subscriptions` | All active/cancelled subscriptions; revenue summary |
| A-17 | Stripe Event Log | `/admin/stripe-events` | Webhook event log; reprocess failed events |
| A-18 | Audit Log | `/admin/audit` | Filterable audit trail: user, action, timestamp, payload |
| A-19 | Feedback Queue | `/admin/feedback` | All student feedback; review + mark-reviewed |
| A-20 | Admin User Management | `/admin/team` | super_admin only; create/deactivate internal accounts; role assignment |
| A-21 | System Health | `/admin/health` | Live: API latency, Redis, DB pool, Celery queue depth |

---

## Page Count Summary

| Portal | Pages |
|---|---|
| Public | 10 |
| Student Portal | 15 |
| School Portal | 17 |
| Admin Console | 21 |
| **Total** | **63** |

---

## Implementation Plan

### Technology Stack

| Layer | Choice | Rationale |
|---|---|---|
| Framework | **Next.js 15** (App Router) | SSR for public/SEO pages; RSC for fast initial load; file-based routing aligns with portal structure |
| Language | **TypeScript** | Type-safe API contracts match backend Pydantic schemas |
| UI Components | **shadcn/ui** + Tailwind CSS | Accessible, composable, no runtime CSS-in-JS |
| State / Data | **TanStack Query v5** | Cache + background refresh for content; mutation hooks for progress writes |
| Auth | **Auth0 Next.js SDK** (`@auth0/nextjs-auth0`) | Matches backend's Auth0 integration; handles token refresh |
| Charts | **Recharts** | Lightweight; sufficient for trends, funnels, cohort charts |
| Forms | **React Hook Form** + **Zod** | Validation mirrors backend Pydantic schemas |
| Audio | Native `<audio>` element | Plays pre-signed S3/CloudFront URLs; no library needed |
| i18n | **next-intl** | en / fr / es; UI strings only (AI content already in correct language) |
| HTTP client | **axios** with interceptors | Attaches JWT, handles 401/402/403 globally |
| Export | **papaparse** | CSV generation for report downloads |
| Testing | **Vitest** + **React Testing Library** + **Playwright** | Unit + E2E; mirrors backend's mock-everything approach |
| Linting | **ESLint** + **Prettier** | Enforce consistent style |

---

### Phase Structure

Build portals in dependency order: public shell → student (largest audience) → school → admin.

---

### Phase W1 — Project Shell + Public Pages

**Goal:** Deployable Next.js app with all public pages, Auth0 wired up, CI passing.

**Deliverables:**

| # | Task |
|---|---|
| W1-1 | Initialise Next.js 15 app with TypeScript, Tailwind, shadcn/ui, ESLint, Prettier |
| W1-2 | Configure `next-intl` for en/fr/es; port string keys from `mobile/i18n/*.json` |
| W1-3 | Implement Auth0 Next.js SDK; `/login`, `/school/login`, `/signup` routes |
| W1-4 | Global layout: nav bar (logo, sign-in CTA), footer, skip-to-content link |
| W1-5 | Landing page (P-01): hero, features grid, testimonial carousel, pricing CTA |
| W1-6 | Pricing page (P-02): plan cards, FAQ accordion |
| W1-7 | COPPA consent page (P-06): form → `POST /auth/consent` |
| W1-8 | Password reset flow (P-07): request form + reset form |
| W1-9 | Terms (P-08), Privacy (P-09), Contact (P-10): static MDX pages |
| W1-10 | Axios instance: base URL from env, JWT interceptor, global 401 redirect |
| W1-11 | Vitest setup; React Testing Library; Playwright config |
| W1-12 | GitHub Actions CI: lint → type-check → unit tests → Playwright smoke |

**Exit criteria:** All public pages render; Auth0 login/logout cycle completes; CI green.

---

### Phase W2 — Student Portal (Core Learning Flow)

**Goal:** Students can log in, browse curriculum, take lessons and quizzes, and see their dashboard.

**Deliverables:**

| # | Task |
|---|---|
| W2-1 | Student layout: sidebar nav (Dashboard, Subjects, Curriculum Map, Progress, Stats, Account) |
| W2-2 | Dashboard (S-01): streak widget, recent sessions list, quick-resume card |
| W2-3 | Subject Browser (S-02): grade-filtered grid from `GET /curriculum/tree` |
| W2-4 | Curriculum Map (S-03): unit grid with completion badges; locked overlay for unpaid |
| W2-5 | Lesson View (S-04): rich text renderer, audio player (pre-signed URL), auto-start session on mount |
| W2-6 | Quiz (S-05): question-by-question flow, `POST /progress/answer`, score screen |
| W2-7 | Tutorial View (S-06): step-by-step content renderer |
| W2-8 | Experiment View (S-07): lab-step renderer, materials list, safety callout |
| W2-9 | Paywall Gate (S-14): intercept 402; render upsell CTA with plan options |
| W2-10 | Progress History (S-08): session list from `GET /progress/history` |
| W2-11 | Stats & Streaks (S-09): accuracy trend chart, streak calendar heatmap |
| W2-12 | Feedback widget (S-13): thumbs up/down overlay on Lesson/Quiz/Experiment pages |
| W2-13 | Offline notice banner: detect navigator.onLine; warn on disconnect |
| W2-14 | Unit tests: lesson renderer, quiz state machine, paywall intercept |

**Exit criteria:** Full lesson → quiz → score flow works end-to-end against staging backend; 402 shows paywall; progress recorded.

---

### Phase W3 — Student Portal (Account + Subscription)

**Goal:** Students can subscribe, manage their plan, and update account settings.

**Deliverables:**

| # | Task |
|---|---|
| W3-1 | Subscription page (S-10): plan cards → `POST /subscription/checkout` → Stripe redirect |
| W3-2 | Subscription management (S-11): current plan badge, billing portal link, cancel flow |
| W3-3 | Account settings (S-12): locale switcher (triggers `next-intl` reload), notification toggles |
| W3-4 | Enrolment confirm page (S-15): validate token → `POST /school/enrol/confirm` |
| W3-5 | Trial countdown banner: days remaining; CTA to subscribe |
| W3-6 | Post-payment success page: confirmation + redirect to dashboard |
| W3-7 | Unit tests: subscription state transitions, locale switcher |

**Exit criteria:** Stripe checkout round-trip completes in staging; subscription status reflected in entitlement on next lesson access.

---

### Phase W4 — School Portal (Teacher Reporting)

**Goal:** Teachers can view class performance, run all 6 report types, and export CSV.

**Deliverables:**

| # | Task |
|---|---|
| W4-1 | School layout: sidebar (Dashboard, Reports, Curriculum, Students, Teachers, Alerts, Settings) |
| W4-2 | Teacher Dashboard (T-01): class summary cards from `GET /reports/overview`; alert badge count |
| W4-3 | Class Overview (T-02): student completion table, sortable columns |
| W4-4 | Student Detail (T-03): per-student session timeline, quiz scores per unit |
| W4-5 | Overview Report (T-04): summary table + KPI cards |
| W4-6 | Trends Report (T-05): line chart (week-over-week lesson views + scores) using Recharts |
| W4-7 | At-Risk Report (T-06): red-flagged student table; link to Student Detail |
| W4-8 | Unit Performance Report (T-07): per-unit pass rate bar chart |
| W4-9 | Engagement Report (T-08): active student count, streak leaderboard, dropout risk list |
| W4-10 | Feedback Report (T-09): unreviewed feedback cards; mark-reviewed action |
| W4-11 | CSV Export (T-10): report selector + date range → download via papaparse |
| W4-12 | Alert Inbox (T-16): list from `GET /reports/alerts`; dismiss action |
| W4-13 | Weekly Digest Archive (T-17): list + in-browser render of past digests |
| W4-14 | Unit tests: chart rendering, CSV generation, alert dismiss |

**Exit criteria:** All 6 report types render with real data from staging; CSV exports correct columns and encoding.

---

### Phase W5 — School Portal (Admin + Curriculum)

**Goal:** School admins can manage teachers, students, curriculum, and school settings.

**Deliverables:**

| # | Task |
|---|---|
| W5-1 | Curriculum Management (T-11): active curriculum card; XLSX upload form → `POST /curriculum/upload` |
| W5-2 | Pipeline Status poller (T-12): poll `GET /curriculum/pipeline/:job_id/status` every 5 s; progress bar |
| W5-3 | Student Roster (T-13): enrolled students table; generate invite link; remove student |
| W5-4 | Teacher Management (T-14): invite teacher form; role assignment dropdown; deactivate |
| W5-5 | School Settings (T-15): name, logo upload, billing portal link |
| W5-6 | XLSX upload error display: per-row structured error list (matches backend 400 response) |
| W5-7 | Unit tests: invite link generation, XLSX error rendering, pipeline poller |

**Exit criteria:** school_admin can upload XLSX, see pipeline run, and manage roster on staging.

---

### Phase W6 — Admin Console

**Goal:** Internal team can monitor platform, review content, manage users, and view audit logs.

**Deliverables:**

| # | Task |
|---|---|
| W6-1 | Admin login (A-01): email + password form → `/admin/auth/login`; separate JWT store |
| W6-2 | Admin layout: sidebar with RBAC-filtered nav items (role from admin JWT) |
| W6-3 | Platform Dashboard (A-02): DAU/WAU/MAU KPI cards, revenue widget, lesson view chart |
| W6-4 | Platform Analytics (A-03): funnel chart (signup → trial → paid), churn table, cohort heatmap |
| W6-5 | Content Pipeline list (A-04): per-grade status table; last run time; success/fail counts |
| W6-6 | Pipeline Trigger (A-05): grade + language multi-select → `POST /curriculum/pipeline/trigger`; display job_id |
| W6-7 | Pipeline Job Detail (A-06): unit-level accordion; error messages; token spend |
| W6-8 | Content Review Queue (A-07): filterable table; grade / subject / language / status filters |
| W6-9 | Content Review Detail (A-08): lesson/quiz/tutorial tabs; AlexJS badge; annotation textarea |
| W6-10 | Review actions (A-09): approve / flag / rollback / block buttons → respective API calls |
| W6-11 | Curriculum List (A-10) + Detail (A-11): all curricula; unit tree; pipeline history |
| W6-12 | User Management (A-12) + Detail (A-13): search, suspend/activate, subscription override |
| W6-13 | School Management (A-14) + Detail (A-15): school table; teacher + student counts |
| W6-14 | Subscription Management (A-16): Stripe subscription table; revenue summary |
| W6-15 | Stripe Event Log (A-17): webhook event list; reprocess action |
| W6-16 | Audit Log (A-18): filterable table; user / action / date range filters |
| W6-17 | Feedback Queue (A-19): all student feedback; mark-reviewed; export |
| W6-18 | Admin User Management (A-20): super_admin only; create internal user; role assignment |
| W6-19 | System Health (A-21): live-polling health endpoint; Redis, DB pool, queue depth meters |
| W6-20 | Unit tests: RBAC nav filtering, review action dispatch, audit log filter |

**Exit criteria:** All admin pages render; content review approve/reject cycle completes; audit log records the action.

---

### Phase W7 — Quality, Accessibility, Performance

**Goal:** Production-ready: WCAG 2.1 AA, Core Web Vitals green, Playwright E2E suite complete.

**Deliverables:**

| # | Task |
|---|---|
| W7-1 | Axe-core accessibility audit on all pages; fix violations |
| W7-2 | Keyboard navigation review: focus traps in modals, skip links, ARIA labels |
| W7-3 | Next.js Image optimisation for all media; lazy-load below-fold content |
| W7-4 | Route-based code splitting review; eliminate unnecessary bundle weight |
| W7-5 | Lighthouse CI: enforce LCP < 2.5 s, CLS < 0.1, FID < 100 ms |
| W7-6 | Playwright E2E suite: student full flow, teacher report flow, admin content review flow |
| W7-7 | Error boundary components on all page sections; Sentry integration |
| W7-8 | CSP headers via `next.config.js`; remove inline scripts |
| W7-9 | Rate-limit feedback on auth pages: lock form after 10 failed attempts (mirrors backend) |
| W7-10 | Load test: 500 concurrent students browsing lessons (k6 script) |

**Exit criteria:** Lighthouse scores ≥ 90 on all portals; Playwright suite green; zero axe critical violations.

---

## Folder Structure

```
web/
  app/
    (public)/               ← P-01 to P-10  (no auth)
    (student)/              ← S-01 to S-15  (student JWT)
    (school)/               ← T-01 to T-17  (teacher/school_admin JWT)
    (admin)/                ← A-01 to A-21  (admin JWT)
  components/
    ui/                     ← shadcn/ui primitives
    charts/                 ← Recharts wrappers
    content/                ← Lesson, Quiz, Tutorial, Experiment renderers
    layout/                 ← Navbars, sidebars, footers per portal
    feedback/               ← Thumbs widget, feedback form
  lib/
    api/                    ← axios instance + typed API functions per backend module
    auth/                   ← Auth0 helpers + admin JWT helpers
    hooks/                  ← TanStack Query hooks (useLesson, useProgress, useReports…)
    utils/                  ← date formatting, CSV, streak calc
  i18n/
    en.json / fr.json / es.json   ← ported from mobile/i18n/
  public/
    fonts/ images/ icons/
  tests/
    unit/                   ← Vitest + RTL
    e2e/                    ← Playwright
  next.config.js
  tailwind.config.js
  tsconfig.json
```

---

## API Surface Used

Every page calls the existing `/api/v1/` backend. No new backend endpoints are required
for the web frontend. The complete mapping:

| Backend module | Used by pages |
|---|---|
| `/auth/*` | P-03, P-04, P-05, P-06, P-07, S-12 |
| `/curriculum/*` | S-02, S-03, T-11, T-12, A-10, A-11, A-05, A-06 |
| `/content/*` | S-04, S-05, S-06, S-07 |
| `/progress/*` | S-05, S-08, S-09, T-02, T-03 |
| `/subscription/*` | S-10, S-11, S-14, A-16, A-17 |
| `/analytics/*` | S-04, S-09, A-02, A-03 |
| `/feedback/*` | S-13, T-09, A-19 |
| `/school/*` | S-15, T-01–T-17 |
| `/reports/*` | T-04–T-10, T-16, T-17 |
| `/admin/*` | A-01–A-21 |
| `GET /health`, `GET /metrics` | A-21 |

---

## Environment Variables

```
NEXT_PUBLIC_API_URL=https://api.studybuddy.com/api/v1
NEXT_PUBLIC_AUTH0_DOMAIN=…
NEXT_PUBLIC_AUTH0_CLIENT_ID=…
AUTH0_CLIENT_SECRET=…           # server-side only
AUTH0_SECRET=…                  # session encryption key
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=…
NEXT_PUBLIC_CDN_URL=https://cdn.studybuddy.com
SENTRY_DSN=…
```

No Anthropic key. No Stripe secret key. Mirrors the mobile app's principle: all secrets
stay on the backend.

---

## Phase Summary

| Phase | Scope | Pages | Key milestone |
|---|---|---|---|
| W1 | Shell + Public | 10 | Auth0 working; CI green |
| W2 | Student core learning | 9 | Full lesson → quiz → score flow |
| W3 | Student subscription | 6 | Stripe checkout round-trip |
| W4 | School reporting | 14 | All 6 report types + CSV |
| W5 | School admin + curriculum | 3 | XLSX upload + pipeline poller |
| W6 | Admin console | 21 | Content review cycle; audit log |
| W7 | Quality + E2E | — | Lighthouse ≥ 90; Playwright green |
| **Total** | | **63** | |

Estimated sequential build order: **W1 → W2 → W3 → W4 → W5 → W6 → W7**
W4 and W5 can be parallelised across two engineers. W6 can start in parallel with W4/W5.
