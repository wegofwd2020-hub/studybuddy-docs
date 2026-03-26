# StudyBuddy OnDemand — Testing Setup Guide

How to set up the application on a local machine and run the full test suite.

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Python | 3.11+ | `python3 --version` |
| Podman **or** Docker | Any recent | Used to run PostgreSQL + Redis locally |
| Git | Any | To clone the repo |
| Auth0 account | Free tier | Required for student/teacher login flows only — not needed to run the automated test suite |

---

## 1. Clone the Repository

```bash
git clone <repo-url> StudyBuddy_OnDemand
cd StudyBuddy_OnDemand
```

---

## 2. Run the Automated Test Suite (Quickest Path)

The `dev_start.sh` script handles everything — virtualenv, containers, migrations, and test execution.

```bash
./dev_start.sh test
```

What it does:

1. Creates `venv/` and installs all Python dependencies from `backend/requirements.txt`
2. Starts a dedicated test PostgreSQL container (`sb-test-db`) on port **5433** with fixed credentials — completely separate from the dev database so tests never touch real data
3. Runs `alembic upgrade head` to apply all 10 schema migrations to the test DB
4. Executes the full `backend/tests/` suite via `pytest`

**Expected output:**

```
...............................................................................
215 passed in ~45s
```

To also run the mobile logic tests:

```bash
source venv/bin/activate
python -m pytest mobile/tests/ -q
# 50 passed
```

---

## 3. Start the Full Development Stack

To run the API server locally (for manual testing or frontend integration):

```bash
./dev_start.sh
```

What it starts:

| Service | Port | Notes |
|---|---|---|
| PostgreSQL | 5432 | `sb-db` container with persistent volume |
| Redis | 6379 | `sb-redis` with AOF persistence, 256 MB limit |
| FastAPI (uvicorn, hot-reload) | 8000 | Stops when you Ctrl-C |

On first run, `dev_start.sh` auto-generates a `.env` file at the project root with random secrets for `JWT_SECRET`, `ADMIN_JWT_SECRET`, `METRICS_TOKEN`, and the database/Redis passwords. Auth0 fields are left as placeholders (see Section 5 below).

After the server starts:

| URL | Purpose |
|---|---|
| `http://localhost:8000/health` | Should return `{"db":"ok","redis":"ok","version":"0.1.0"}` |
| `http://localhost:8000/api/docs` | Swagger UI (all endpoints) |
| `http://localhost:8000/api/redoc` | ReDoc alternative |

---

## 4. Environment Variables

The `.env` file at the project root is loaded by both `dev_start.sh` and Docker Compose. A template with all variables is at `backend/.env.example`. Copy and fill it in:

```bash
cp backend/.env.example .env
```

### Required for all tests (auto-generated on first run)

| Variable | Example | Purpose |
|---|---|---|
| `DATABASE_URL` | `postgresql://studybuddy:pw@localhost:5432/studybuddy` | Dev DB connection |
| `REDIS_URL` | `redis://:pw@localhost:6379/0` | Redis connection |
| `JWT_SECRET` | (random 40-char string) | Signs student/teacher JWTs |
| `ADMIN_JWT_SECRET` | (different random string) | Signs internal admin JWTs |
| `METRICS_TOKEN` | (random string) | Protects `GET /metrics` |

### Required for Auth0 login flows only (not needed for automated tests)

| Variable | Where to find it |
|---|---|
| `AUTH0_DOMAIN` | Auth0 dashboard → Settings → Domain |
| `AUTH0_JWKS_URL` | `https://<domain>/.well-known/jwks.json` |
| `AUTH0_STUDENT_CLIENT_ID` | Auth0 → Applications → Student app |
| `AUTH0_TEACHER_CLIENT_ID` | Auth0 → Applications → Teacher app |
| `AUTH0_MGMT_CLIENT_ID` | Auth0 → Applications → M2M app |
| `AUTH0_MGMT_CLIENT_SECRET` | Auth0 → Applications → M2M app |

The automated test suite mocks Auth0 JWKS verification, so **Auth0 credentials are not required to run tests**.

### Optional

| Variable | Default | Purpose |
|---|---|---|
| `STRIPE_SECRET_KEY` | (blank) | Stripe payments — mocked in tests |
| `STRIPE_WEBHOOK_SECRET` | (blank) | Stripe webhook verification |
| `SENDGRID_API_KEY` | (blank) | Email delivery — tasks log only if blank |
| `SENTRY_DSN` | (blank) | Error tracking — disabled if blank |
| `CONTENT_STORE_PATH` | `./data/content_store` | Where pipeline writes lesson JSON/MP3 |

---

## 5. Auth0 Setup (for manual login testing)

Skip this section if you only need to run the automated test suite.

1. Sign up at [auth0.com](https://auth0.com) — free tier is sufficient
2. Create a **Regular Web Application** for students → note the Client ID → set as `AUTH0_STUDENT_CLIENT_ID`
3. Create a **Regular Web Application** for teachers → note the Client ID → set as `AUTH0_TEACHER_CLIENT_ID`
4. Create a **Machine-to-Machine Application**, authorise it against the Auth0 Management API with scopes `read:users update:users delete:users` → set as `AUTH0_MGMT_CLIENT_ID` + `AUTH0_MGMT_CLIENT_SECRET`
5. Add your Auth0 domain to `AUTH0_DOMAIN` and `AUTH0_JWKS_URL`
6. Restart the API server

---

## 6. Creating an Admin User

Admin users (developer / tester / product_admin / super_admin) use local bcrypt auth — they are not created through Auth0.

```bash
# Connect to the dev database
psql postgresql://studybuddy:<POSTGRES_PASSWORD>@localhost:5432/studybuddy

# Insert an admin user (bcrypt hash for 'changeme123')
INSERT INTO admin_users (admin_user_id, email, password_hash, role, account_status)
VALUES (
    gen_random_uuid(),
    'admin@example.com',
    '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewdBPj4J/8XyV.2m',
    'product_admin',
    'active'
);
```

Or generate a proper hash:

```bash
source venv/bin/activate
python3 -c "import bcrypt; print(bcrypt.hashpw(b'yourpassword', bcrypt.gensalt()).decode())"
```

Then log in:

```bash
curl -X POST http://localhost:8000/api/v1/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "changeme123"}'
```

---

## 7. Running a Subset of Tests

```bash
source venv/bin/activate
cd backend

# All tests
DATABASE_URL="postgresql://studybuddy:testpassword@localhost:5433/studybuddy_test" \
  python -m pytest tests/ -q

# Single module
DATABASE_URL="postgresql://studybuddy:testpassword@localhost:5433/studybuddy_test" \
  python -m pytest tests/test_auth.py -v

# Keyword filter
DATABASE_URL="postgresql://studybuddy:testpassword@localhost:5433/studybuddy_test" \
  python -m pytest tests/ -k "subscription" -v
```

The test DB must be running first:

```bash
./dev_start.sh test   # starts test DB + runs full suite
# OR start the test DB only:
podman start sb-test-db  # if already created
```

---

## 8. Pipeline Testing (Content Generation)

The content pipeline (Phases 2–6) requires `ANTHROPIC_API_KEY`. It is fully mocked in the test suite, so no API key is needed for `pytest`.

To run the pipeline manually against real Claude:

```bash
# Set pipeline env vars
export ANTHROPIC_API_KEY=sk-ant-...
export CLAUDE_MODEL=claude-sonnet-4-6
export CONTENT_STORE_PATH=./data/content_store

# Seed default curricula into the dev database
python pipeline/seed_default.py --year 2026

# Generate content for grade 8 in English
python pipeline/build_grade.py --grade 8 --lang en

# Regenerate a single unit
python pipeline/build_unit.py --curriculum-id default-2026-g8 --unit G8-MATH-001 --lang en --force

# Dry run (shows what would be generated, no API calls made)
python pipeline/build_grade.py --grade 8 --lang en --dry-run
```

---

## 9. Resetting the Dev Environment

```bash
# Stop all containers (data preserved)
./dev_start.sh stop

# Delete all containers AND database volumes (fresh start)
./dev_start.sh reset
```

---

## 10. Test Architecture Notes

- **No live external calls in tests.** PostgreSQL and Redis are real local containers. Auth0, Stripe, SendGrid, Anthropic, and FCM are all mocked via `pytest` fixtures in `backend/tests/conftest.py`.
- **Each test is isolated.** The `db_conn` fixture in `conftest.py` wraps every test in a transaction that is rolled back on completion — no test data persists between tests.
- **Test DB is fixed-credential.** `sb-test-db` always uses `studybuddy / testpassword` on port 5433 so test commands are reproducible without sourcing `.env`.
- **Mobile tests need no containers.** `mobile/tests/` only exercises logic and i18n — SQLite is used in-memory per test via `tmp_path`.
