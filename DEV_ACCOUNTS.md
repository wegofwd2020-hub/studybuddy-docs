# StudyBuddy OnDemand — Developer Test Accounts

Local development bypass for testing all four roles without Auth0 configured.

---

## Quick Start

```bash
# 1. Start the stack
docker compose up

# 2. Seed all dev accounts + test data (run once after first docker compose up)
docker compose exec api python scripts/setup_dev.py

# 3. Open the dev login page
open http://localhost:3000/dev-login
```

To wipe all dev data and start fresh:

```bash
docker compose exec api python scripts/setup_dev.py --reset
```

---

## Accounts

| Role | Email | Password / Method | Portal |
|---|---|---|---|
| **Student** | `dev.student@studybuddy.dev` | `/dev-login` → Student | `http://localhost:3000/dashboard` |
| **Teacher** | `dev.teacher@studybuddy.dev` | `/dev-login` → Teacher | `http://localhost:3000/school/dashboard` |
| **School Admin** | `dev.schooladmin@studybuddy.dev` | `/dev-login` → School Admin | `http://localhost:3000/school/dashboard` |
| **Super Admin** | `dev.admin@studybuddy.dev` | Password below | `http://localhost:3000/admin/login` |

**Super Admin password:** `DevAdmin1234!`

---

## What Gets Seeded

`setup_dev.py` is fully idempotent — safe to run multiple times. On each run it creates (or skips if already present):

| Item | Details |
|---|---|
| Dev School | `school_id = 00000000-0000-0000-0000-000000000001` · enrolment code `DEVSCHOOL01` |
| Super Admin user | bcrypt login for the admin portal |
| School Admin teacher | Role `school_admin` on Dev School |
| Teacher | Role `teacher` on Dev School |
| Student | Grade 8 · enrolled in Dev School · premium entitlement (no paywall) |
| Grade 8 curriculum | All units from `data/grade8_stem.json` seeded to DB + content store |
| Progress data | 5 passed sessions (5-day streak) · 3 older passed sessions · 2 failed sessions · 1 open session |
| Lesson views | 8 lesson view records with realistic durations |

---

## Test Data Overview — Student Portal

After setup, the student (`dev.student@studybuddy.dev`) will show:

- **Streak:** 5 days
- **Units completed:** 8 (passed)
- **Units needing retry:** 2
- **Units in progress:** 1
- **Subject breakdown:** across all Grade 8 subjects

This gives the Curriculum page, Progress screen, and student analytics all meaningful data to display.

---

## Test Data Overview — Teacher Portal

After setup, the teacher/school admin will see:

- **Class Overview:** 1 student in roster with real activity metrics
- **Reports → Overview:** active student count, pass rates, lesson views
- **Reports → Trends:** week-over-week data
- **Reports → Curriculum Health:** unit health tiers based on pass rates
- **Reports → Student:** full report card for Dev Student

---

## How It Works

### Backend — `POST /api/v1/auth/dev-login`

- Accepts `{"role": "student" | "teacher" | "school_admin"}`
- Only registered when `APP_ENV=development` — returns HTTP 403 in any other environment
- Creates the school/teacher/student records on first call (idempotent)
- Returns a **7-day JWT** + name + email
- Source: `backend/src/auth/dev_router.py`

### Web — `/dev-login` page

- Source: `web/app/(public)/dev-login/page.tsx`
- On login:
  1. Calls `/api/dev-login` (Next.js proxy — avoids CORS)
  2. Stores token in `localStorage` (`sb_token` for student, `sb_teacher_token` for teacher/school_admin)
  3. Sets a `sb_dev_session` cookie (base64 JSON `{name, email}`) for server-side layouts
  4. Redirects to the portal

### Web — Server layout fallback

- Sources: `web/app/(student)/layout.tsx`, `web/app/(school)/layout.tsx`
- Both layouts do: `session = auth0.getSession() ?? getDevSession()`
- Auth0 takes priority. Without Auth0, the `sb_dev_session` cookie is used as fallback.
- Helper: `web/lib/dev-session.ts`

### `setup_dev.py` — Master Setup Script

Idempotent script that creates all four accounts + seeds curriculum + seeds realistic
progress/analytics data. Designed to be run once after `docker compose up` on a fresh
environment, or with `--reset` to wipe and re-seed.

```bash
# First-time setup
docker compose exec api python scripts/setup_dev.py

# Reset and re-seed everything
docker compose exec api python scripts/setup_dev.py --reset
```

---

## Security Notes

- The `/auth/dev-login` endpoint returns HTTP 403 if `APP_ENV != "development"` — it cannot be called in production even if the code were deployed
- `setup_dev.py` connects directly to `db:5432`, bypassing PgBouncer (required for asyncpg prepared statements)
- The `sb_dev_session` cookie is not signed — it is display-only (name/email for the portal header). All API authorization uses the JWT in `localStorage`, verified server-side
- Tokens expire after 7 days; revisit `/dev-login` to refresh
- Dev accounts use `auth_provider = 'dev'` so they are distinguishable from real Auth0 accounts in the database

---

## Files

| File | Purpose |
|---|---|
| `backend/scripts/setup_dev.py` | Master idempotent setup script — accounts + curriculum + test data |
| `backend/scripts/seed_dev_content.py` | Content-only seed (called by migrate service on startup) |
| `backend/src/auth/dev_router.py` | `/auth/dev-login` endpoint (student / teacher / school_admin) |
| `web/app/(public)/dev-login/page.tsx` | Dev login UI with all 3 role buttons |
| `web/app/api/dev-login/route.ts` | Next.js proxy to backend (avoids CORS) |
| `web/lib/dev-session.ts` | Server-side dev session cookie reader |
