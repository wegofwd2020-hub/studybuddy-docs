# StudyBuddy OnDemand — Phase 1 Setup Guide

Everything a developer needs to stand up Phase 1 from scratch: Auth0 configuration,
environment variables, schema notes, infrastructure templates, mobile integration,
and testing setup.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Auth0 Tenant Setup](#2-auth0-tenant-setup)
3. [Environment Variables (.env.example)](#3-environment-variables)
4. [PostgreSQL Schema — Phase 1 Tables](#4-postgresql-schema)
5. [PgBouncer Configuration](#5-pgbouncer-configuration)
6. [nginx Configuration](#6-nginx-configuration)
7. [Mobile Auth0 Integration (Kivy/Python)](#7-mobile-auth0-integration)
8. [Testing Setup](#8-testing-setup)
9. [Standard API Contracts](#9-standard-api-contracts)
10. [Design Clarifications](#10-design-clarifications)

---

## 1. Prerequisites

Software required before starting:

| Tool | Version | Purpose |
|---|---|---|
| Python | 3.11+ | Backend + pipeline |
| PostgreSQL | 16 | Primary database |
| Redis | 7 | Cache + Celery broker |
| Docker + Docker Compose | 24+ | Local dev orchestration |
| PgBouncer | 1.22+ | Connection pooler |
| nginx | 1.25+ | Reverse proxy |
| Auth0 account | — | External auth provider |
| Sentry account | — | Error tracking |

---

## 2. Auth0 Tenant Setup

### Step 1 — Create Tenant

1. Sign up at [auth0.com](https://auth0.com) and create a new tenant.
2. Choose region closest to your production deployment (e.g., `US`, `EU`, `AU`).
3. Note your **domain**: `your-tenant.auth0.com` → this becomes `AUTH0_DOMAIN`.

### Step 2 — Create Mobile Application (Students)

1. In Auth0 Dashboard → **Applications** → **Create Application**.
2. Name: `StudyBuddy Student App` | Type: **Native**.
3. Under **Settings**:
   - Allowed Callback URLs: `studybuddy://callback`
   - Allowed Logout URLs: `studybuddy://logout`
   - Allowed Origins (CORS): leave blank (mobile app, not browser)
4. Note **Client ID** → `AUTH0_STUDENT_CLIENT_ID`.
5. Under **Advanced Settings → OAuth**:
   - Enable **OIDC Conformant**
   - Grant Types: `Authorization Code`, `Refresh Token`

### Step 3 — Create Web Application (Teachers / Admin Dashboard)

1. **Applications** → **Create Application**.
2. Name: `StudyBuddy Teacher Dashboard` | Type: **Regular Web Application**.
3. Allowed Callback URLs: `https://yourdomain.com/auth/callback` (dev: `http://localhost:3000/auth/callback`)
4. Note **Client ID** → `AUTH0_TEACHER_CLIENT_ID`.

### Step 4 — Enable Management API Access

1. **Applications** → **APIs** → **Auth0 Management API**.
2. Under **Machine to Machine Applications** → authorize **your tenant's backend service account**.
3. Create a new M2M application: `StudyBuddy Backend Service` | Type: **Machine to Machine**.
4. Authorize it against **Auth0 Management API** with scopes:
   - `update:users` (block/unblock on suspension)
   - `delete:users` (GDPR erasure)
   - `read:users` (lookup by email)
5. Note:
   - **Client ID** → `AUTH0_MGMT_CLIENT_ID`
   - **Client Secret** → `AUTH0_MGMT_CLIENT_SECRET`
   - Management API URL: `https://{AUTH0_DOMAIN}/api/v2` → `AUTH0_MGMT_API_URL`

### Step 5 — COPPA Age Verification (Under-13)

Auth0 does not enforce age natively. Implement as a **post-registration Action**:

1. **Actions** → **Flows** → **Post User Registration**.
2. Add a custom action that checks the `birthdate` claim (collected during registration).
3. If age < 13: set `app_metadata.requires_parental_consent = true`.
4. The backend checks this claim during token exchange and sets `account_status = pending`.
5. Student remains `pending` until the `parental_consents` record is updated to `granted`.

### Step 6 — JWKS Endpoint

Your JWKS URL (used by the backend to verify student/teacher JWTs) is:

```
https://{AUTH0_DOMAIN}/.well-known/jwks.json
```

This is fetched once and cached in L1 TTLCache (24-hr TTL). The backend env var
`AUTH0_JWKS_URL` should point to this URL.

### Step 7 — Auth0 Log Streaming (Optional but recommended)

1. **Monitoring** → **Log Streams** → **Create Stream**.
2. Choose **Datadog**, **Splunk**, or **AWS EventBridge** to forward Auth0 login events
   to your log aggregation platform alongside backend logs.

---

## 3. Environment Variables

Copy this file to `backend/.env` and fill in all values. **Never commit `.env` to git.**

```bash
# ─────────────────────────────────────────────
# backend/.env.example
# ─────────────────────────────────────────────

# ── Application ──────────────────────────────
APP_ENV=development              # development | staging | production
APP_VERSION=0.1.0
DEBUG=false                      # set true only in development
LOG_LEVEL=INFO                   # DEBUG | INFO | WARNING | ERROR

# ── Database ─────────────────────────────────
DATABASE_URL=postgresql://studybuddy:password@localhost:5432/studybuddy
# Via PgBouncer in dev:
# DATABASE_URL=postgresql://studybuddy:password@localhost:6432/studybuddy
DATABASE_POOL_MIN=5
DATABASE_POOL_MAX=20

# ── Redis ─────────────────────────────────────
REDIS_URL=redis://:yourpassword@localhost:6379/0
REDIS_MAX_CONNECTIONS=10

# ── JWT ───────────────────────────────────────
JWT_SECRET=change-me-minimum-32-chars-random-string
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=15
JWT_REFRESH_TOKEN_EXPIRE_DAYS=30

ADMIN_JWT_SECRET=different-secret-minimum-32-chars   # MUST differ from JWT_SECRET
ADMIN_JWT_EXPIRE_MINUTES=60

# ── Auth0 (External Auth — Students & Teachers) ──
AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_JWKS_URL=https://your-tenant.auth0.com/.well-known/jwks.json
AUTH0_STUDENT_CLIENT_ID=your-auth0-student-app-client-id
AUTH0_TEACHER_CLIENT_ID=your-auth0-teacher-app-client-id

# Auth0 Management API (for forgot-password + suspension sync)
AUTH0_MGMT_CLIENT_ID=your-m2m-client-id
AUTH0_MGMT_CLIENT_SECRET=your-m2m-client-secret
AUTH0_MGMT_API_URL=https://your-tenant.auth0.com/api/v2

# ── CORS ──────────────────────────────────────
# Comma-separated list of allowed origins
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8080

# ── Sentry ────────────────────────────────────
SENTRY_DSN=https://your-key@sentry.io/your-project-id
# Leave blank to disable Sentry (e.g., local dev)

# ── Observability ─────────────────────────────
METRICS_TOKEN=random-token-for-prometheus-scrape-endpoint
# OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317   # Phase 2+

# ── Content Store ─────────────────────────────
CONTENT_STORE_PATH=/data/content   # local filesystem (Phase 1-3)
# AWS_S3_BUCKET=studybuddy-content  # Phase 4+
# AWS_REGION=us-east-1

# ── Stripe (Phase 5 — leave blank in Phase 1) ─
# STRIPE_SECRET_KEY=sk_test_...
# STRIPE_WEBHOOK_SECRET=whsec_...

# ── Pipeline (pipeline/.env — not backend) ───
# ANTHROPIC_API_KEY=sk-ant-...
# CLAUDE_MODEL=claude-sonnet-4-6
# TTS_API_KEY=...
# TTS_PROVIDER=polly              # polly | google
# MAX_PIPELINE_COST_USD=50
# PIPELINE_ALERT_EMAIL=content-team@example.com

# ── Email (SES — Phase 2+) ───────────────────
# AWS_SES_REGION=us-east-1
# SES_FROM_ADDRESS=no-reply@yourdomain.com

# ── Admin Account Seed ────────────────────────
# First super_admin account created by seed script:
# python backend/scripts/create_admin.py --email admin@example.com --role super_admin
# (prompted for password; never set via env var)
```

### Mobile app config (`mobile/config.py`)

```python
# mobile/config.py  — no secrets; all non-sensitive
import os

BACKEND_URL = os.getenv("STUDYBUDDY_BACKEND_URL", "http://localhost:8000")

# Auth0 public config (not secret — client IDs are safe to embed in mobile apps)
AUTH0_DOMAIN = "your-tenant.auth0.com"
AUTH0_CLIENT_ID = "your-auth0-student-app-client-id"
AUTH0_REDIRECT_URI = "studybuddy://callback"

# Local storage
SQLITE_DB_PATH = None          # set at runtime from App.user_data_dir
JWT_STORAGE_FILENAME = "jwt.token"
REFRESH_TOKEN_FILENAME = "refresh.token"
MAX_CACHE_MB = 200
```

---

## 4. PostgreSQL Schema

### Phase 1 Tables

All DDL is managed by Alembic. Initialize:

```bash
cd backend
alembic init alembic
# Edit alembic/env.py to use DATABASE_URL from config
alembic revision --autogenerate -m "phase1_initial_schema"
alembic upgrade head
```

#### Type conventions

| PostgreSQL type | Python / Pydantic type | Notes |
|---|---|---|
| `uuid` | `UUID` | `DEFAULT gen_random_uuid()` |
| `timestamptz` | `datetime` (UTC) | Always `DEFAULT NOW()` for `created_at` |
| `text` | `str` | Prefer over `varchar(N)` unless length constraint needed |
| `account_status` | `AccountStatus` enum | Create as PostgreSQL ENUM for integrity |

#### Phase 1 ENUM types

```sql
CREATE TYPE account_status AS ENUM ('pending', 'active', 'suspended', 'deleted');
CREATE TYPE admin_role     AS ENUM ('developer', 'tester', 'product_admin', 'super_admin');
CREATE TYPE consent_status AS ENUM ('pending', 'granted', 'denied');
```

#### `students` table

```sql
CREATE TABLE students (
    student_id       uuid          PRIMARY KEY DEFAULT gen_random_uuid(),
    external_auth_id text          UNIQUE NOT NULL,   -- Auth0 sub claim
    auth_provider    text          NOT NULL DEFAULT 'auth0',
    name             text          NOT NULL,
    email            text          UNIQUE NOT NULL,
    grade            smallint      NOT NULL CHECK (grade BETWEEN 5 AND 12),
    locale           text          NOT NULL DEFAULT 'en',
    account_status   account_status NOT NULL DEFAULT 'pending',
    school_id        uuid          REFERENCES schools(school_id) ON DELETE SET NULL,
    enrolled_at      timestamptz,
    lessons_accessed int           NOT NULL DEFAULT 0,
    created_at       timestamptz   NOT NULL DEFAULT NOW()
);
CREATE INDEX ix_students_external_auth_id ON students(external_auth_id);
CREATE INDEX ix_students_school_id        ON students(school_id);
CREATE INDEX ix_students_account_status   ON students(account_status);
```

#### `teachers` table

```sql
CREATE TABLE teachers (
    teacher_id       uuid          PRIMARY KEY DEFAULT gen_random_uuid(),
    school_id        uuid          NOT NULL REFERENCES schools(school_id),
    external_auth_id text          UNIQUE NOT NULL,
    auth_provider    text          NOT NULL DEFAULT 'auth0',
    name             text          NOT NULL,
    email            text          UNIQUE NOT NULL,
    role             text          NOT NULL DEFAULT 'teacher',  -- teacher | school_admin
    account_status   account_status NOT NULL DEFAULT 'pending',
    created_at       timestamptz   NOT NULL DEFAULT NOW()
);
CREATE INDEX ix_teachers_school_id ON teachers(school_id);
```

#### `admin_users` table

```sql
CREATE TABLE admin_users (
    admin_user_id uuid       PRIMARY KEY DEFAULT gen_random_uuid(),
    email         text       UNIQUE NOT NULL,
    password_hash text       NOT NULL,   -- bcrypt
    role          admin_role NOT NULL DEFAULT 'developer',
    account_status account_status NOT NULL DEFAULT 'active',
    totp_secret   text,                  -- NULL until Phase 7+ MFA rollout
    last_login_at timestamptz,
    created_at    timestamptz NOT NULL DEFAULT NOW()
);
-- Admin login lockout tracked in Redis (key: login_attempts:{email}, TTL 15 min)
-- NOT stored in this table — avoids DB write on every failed attempt
```

#### `parental_consents` table

Required for COPPA compliance. Created in Phase 1 alongside `students`.

```sql
CREATE TABLE parental_consents (
    consent_id       uuid          PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id       uuid          NOT NULL REFERENCES students(student_id) ON DELETE CASCADE,
    guardian_email   text          NOT NULL,
    status           consent_status NOT NULL DEFAULT 'pending',
    consent_token    text          UNIQUE,   -- one-use token sent to guardian (also in Redis with TTL 72h)
    token_expires_at timestamptz,
    consented_at     timestamptz,
    ip_address       inet,
    created_at       timestamptz   NOT NULL DEFAULT NOW()
);
CREATE INDEX ix_parental_consents_student_id ON parental_consents(student_id);
```

#### `audit_log` table

```sql
CREATE TABLE audit_log (
    id             bigserial     PRIMARY KEY,
    timestamp      timestamptz   NOT NULL DEFAULT NOW(),
    event_type     text          NOT NULL,
    actor_type     text          NOT NULL,
    actor_id       uuid,
    target_type    text,
    target_id      uuid,
    metadata       jsonb,
    ip_address     inet,
    correlation_id text
);
CREATE INDEX ix_audit_log_timestamp ON audit_log(timestamp DESC);
CREATE INDEX ix_audit_log_target    ON audit_log(target_type, target_id);
CREATE INDEX ix_audit_log_actor     ON audit_log(actor_type, actor_id);
```

#### `schools` table (stub for Phase 1 curriculum resolver)

```sql
CREATE TABLE schools (
    school_id      uuid          PRIMARY KEY DEFAULT gen_random_uuid(),
    name           text          NOT NULL,
    contact_email  text          NOT NULL,
    country        text          NOT NULL DEFAULT 'CA',
    enrolment_code text          UNIQUE,
    status         account_status NOT NULL DEFAULT 'active',
    created_at     timestamptz   NOT NULL DEFAULT NOW()
);
```

### Admin Account Creation (first run)

```bash
python backend/scripts/create_admin.py --email admin@example.com --role super_admin
# Prompts for password interactively; hashes with bcrypt; inserts into admin_users
# Never pass password via CLI argument or env var
```

---

## 5. PgBouncer Configuration

Create `infra/pgbouncer/pgbouncer.ini`:

```ini
[databases]
studybuddy = host=db port=5432 dbname=studybuddy

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction          ; transaction pooling — stateless; works with asyncpg
max_client_conn = 200            ; max connections from API workers
default_pool_size = 50           ; connections PgBouncer maintains to PostgreSQL

; Timeouts
server_idle_timeout = 60
client_idle_timeout = 120
query_timeout = 30

; Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

; Admin
admin_users = pgbouncer_admin
```

`infra/pgbouncer/userlist.txt` (bcrypt-hash the password — use `pg_md5` tool):
```
"studybuddy" "md5<hashed_password>"
```

Docker Compose service:
```yaml
pgbouncer:
  image: bitnami/pgbouncer:1.22
  environment:
    POSTGRESQL_HOST: db
    POSTGRESQL_PORT: 5432
    POSTGRESQL_DATABASE: studybuddy
    POSTGRESQL_USERNAME: studybuddy
    POSTGRESQL_PASSWORD: ${POSTGRES_PASSWORD}
    PGBOUNCER_POOL_MODE: transaction
    PGBOUNCER_MAX_CLIENT_CONN: 200
    PGBOUNCER_DEFAULT_POOL_SIZE: 50
  ports:
    - "6432:6432"
  depends_on:
    db:
      condition: service_healthy
```

---

## 6. nginx Configuration

Create `infra/nginx/nginx.conf`:

```nginx
worker_processes auto;
events { worker_connections 1024; }

http {
    # ── Upstream ─────────────────────────────────
    upstream api {
        server api:8000;
        keepalive 32;                    # reuse connections to gunicorn workers
    }

    # ── Rate Limit Zones ─────────────────────────
    limit_req_zone $binary_remote_addr zone=auth:10m     rate=10r/m;
    limit_req_zone $http_authorization  zone=content:10m rate=100r/m;

    # ── Gzip ─────────────────────────────────────
    gzip on;
    gzip_types application/json text/plain;
    gzip_min_length 1024;

    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name yourdomain.com;

        ssl_certificate     /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;
        ssl_protocols       TLSv1.2 TLSv1.3;

        # ── Block /metrics from public ────────────
        location /metrics {
            allow 10.0.0.0/8;            # Prometheus internal IP range
            deny  all;
            proxy_pass http://api;
        }

        # ── Auth endpoints — strict rate limit ───
        location ~ ^/auth/ {
            limit_req zone=auth burst=5 nodelay;
            proxy_pass http://api;
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # ── API — standard rate limit ─────────────
        location / {
            limit_req zone=content burst=20 nodelay;
            proxy_pass         http://api;
            proxy_http_version 1.1;
            proxy_set_header   Connection      "";   # keep-alive to upstream
            proxy_set_header   Host            $host;
            proxy_set_header   X-Real-IP       $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_read_timeout 30s;
        }
    }
}
```

> **Dev (no TLS):** replace the `listen 443 ssl` block with a plain `listen 80` block and remove SSL directives.

---

## 7. Mobile Auth0 Integration (Kivy / Python)

There is no official Auth0 SDK for Kivy/Python. Use the **Authorization Code + PKCE** flow manually via `httpx`.

### Flow

```
1. App generates code_verifier (random 43-128 chars) and code_challenge (SHA256 base64url)
2. App opens Auth0 Universal Login in device browser:
   https://{AUTH0_DOMAIN}/authorize?
     response_type=code
     &client_id={AUTH0_STUDENT_CLIENT_ID}
     &redirect_uri=studybuddy://callback
     &scope=openid profile email
     &code_challenge={code_challenge}
     &code_challenge_method=S256
3. User authenticates in browser; Auth0 redirects to:
   studybuddy://callback?code={authorization_code}
4. App intercepts deep link, extracts `code`
5. App calls Auth0 token endpoint:
   POST https://{AUTH0_DOMAIN}/oauth/token
   {client_id, code, redirect_uri, code_verifier, grant_type: authorization_code}
   → returns {id_token, access_token, refresh_token}
6. App calls backend:
   POST /auth/exchange  {id_token}
   → returns {token, refresh_token, student_id}
7. App stores internal JWT and refresh_token securely
```

### Deep Link Handling in Kivy

```python
# mobile/src/auth/auth0_client.py
import hashlib, base64, os, httpx
from kivy.utils import platform

AUTH0_DOMAIN = "your-tenant.auth0.com"
AUTH0_CLIENT_ID = "your-client-id"
REDIRECT_URI = "studybuddy://callback"

def generate_pkce_pair():
    verifier = base64.urlsafe_b64encode(os.urandom(40)).rstrip(b"=").decode()
    challenge = base64.urlsafe_b64encode(
        hashlib.sha256(verifier.encode()).digest()
    ).rstrip(b"=").decode()
    return verifier, challenge

def get_auth0_login_url(code_challenge: str) -> str:
    return (
        f"https://{AUTH0_DOMAIN}/authorize"
        f"?response_type=code"
        f"&client_id={AUTH0_CLIENT_ID}"
        f"&redirect_uri={REDIRECT_URI}"
        f"&scope=openid+profile+email"
        f"&code_challenge={code_challenge}"
        f"&code_challenge_method=S256"
    )

async def exchange_code_for_tokens(code: str, code_verifier: str) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"https://{AUTH0_DOMAIN}/oauth/token",
            json={
                "grant_type": "authorization_code",
                "client_id": AUTH0_CLIENT_ID,
                "code": code,
                "redirect_uri": REDIRECT_URI,
                "code_verifier": code_verifier,
            },
        )
        resp.raise_for_status()
        return resp.json()   # contains id_token
```

### JWT Storage (Cross-Platform)

Store JWT files in the app's private data directory (`App.user_data_dir`):

```python
# mobile/src/auth/token_store.py
import os, stat
from kivy.app import App

def _path(filename: str) -> str:
    return os.path.join(App.get_running_app().user_data_dir, filename)

def save_token(filename: str, token: str):
    path = _path(filename)
    with open(path, "w") as f:
        f.write(token)
    os.chmod(path, stat.S_IRUSR | stat.S_IWUSR)  # 0600 — owner read/write only

def load_token(filename: str) -> str | None:
    path = _path(filename)
    if not os.path.exists(path):
        return None
    with open(path, "r") as f:
        return f.read().strip()

def delete_token(filename: str):
    path = _path(filename)
    if os.path.exists(path):
        os.remove(path)
```

- **Android:** `user_data_dir` maps to the app's private internal storage (not accessible to other apps without root)
- **iOS:** maps to the app sandbox Documents directory
- **Linux/dev:** maps to `~/.studybuddy/`

### Opening Browser from Kivy

```python
import webbrowser
from kivy.utils import platform

def open_auth0_login(url: str):
    if platform in ("android", "ios"):
        webbrowser.open(url)         # opens system default browser with custom tab
    else:
        webbrowser.open_new(url)     # desktop dev: opens default browser
```

### Handling the Callback Deep Link

On Android, register the intent filter in `buildozer.spec`:
```ini
android.manifest.intent_filters = studybuddy_intent_filter.xml
```

`studybuddy_intent_filter.xml`:
```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="studybuddy" android:host="callback" />
</intent-filter>
```

On iOS, add to `Info.plist`:
```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array><string>studybuddy</string></array>
  </dict>
</array>
```

---

## 8. Testing Setup

### conftest.py

```python
# backend/tests/conftest.py
import asyncio, pytest, fakeredis.aioredis
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from alembic import command
from alembic.config import Config

from main import app
from config import settings

TEST_DATABASE_URL = "postgresql+asyncpg://studybuddy:password@localhost:5432/studybuddy_test"

@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="session", autouse=True)
def run_migrations():
    """Apply Alembic migrations to test DB before any test."""
    cfg = Config("alembic.ini")
    cfg.set_main_option("sqlalchemy.url", TEST_DATABASE_URL.replace("+asyncpg", ""))
    command.upgrade(cfg, "head")
    yield
    command.downgrade(cfg, "base")

@pytest.fixture
async def db_session():
    engine = create_async_engine(TEST_DATABASE_URL)
    async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
    async with async_session() as session:
        yield session
        await session.rollback()

@pytest.fixture
def fake_redis():
    return fakeredis.aioredis.FakeRedis()

@pytest.fixture
async def client(fake_redis):
    app.state.redis = fake_redis
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as c:
        yield c

@pytest.fixture
def mock_auth0_jwks():
    """Return a fake JWKS; patch verify_auth0_token to bypass signature check."""
    return {
        "keys": [{
            "kty": "RSA", "kid": "test-key-id",
            "n": "...", "e": "AQAB"
        }]
    }

@pytest.fixture
def valid_student_id_token():
    """Pre-signed fake Auth0 id_token for testing (signed with test key)."""
    # Use python-jose to sign a test JWT with a test RSA key
    # See tests/helpers/token_factory.py
    ...
```

### Mocking Celery in Tests

```python
# In conftest.py or individual test files
from unittest.mock import patch

@pytest.fixture(autouse=True)
def mock_celery_tasks():
    """Run Celery tasks synchronously in tests."""
    with patch("src.core.events.write_audit_log") as mock_audit, \
         patch("src.auth.tasks.sync_auth0_suspension") as mock_auth0:
        mock_audit.return_value = None
        mock_auth0.return_value = None
        yield
```

### Test DB Strategy

- **CI:** Use a PostgreSQL service container in GitHub Actions (not SQLite)
- **Local dev:** Use Docker Compose `db` service; create a separate `studybuddy_test` database

```yaml
# .github/workflows/test.yml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_DB: studybuddy_test
      POSTGRES_USER: studybuddy
      POSTGRES_PASSWORD: testpassword
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
  redis:
    image: redis:7-alpine
```

### Environment for Tests

```bash
# backend/.env.test
DATABASE_URL=postgresql+asyncpg://studybuddy:testpassword@localhost:5432/studybuddy_test
REDIS_URL=redis://localhost:6379/1      # DB index 1 to avoid dev data collision
JWT_SECRET=test-secret-do-not-use-in-production
ADMIN_JWT_SECRET=test-admin-secret
AUTH0_DOMAIN=test.auth0.com
AUTH0_JWKS_URL=http://localhost:9999/.well-known/jwks.json   # mock JWKS server
SENTRY_DSN=                             # blank = Sentry disabled in tests
METRICS_TOKEN=test-metrics-token
CONTENT_STORE_PATH=/tmp/studybuddy-test-content
```

---

## 9. Standard API Contracts

### Success Response

All successful responses return the resource directly (no wrapper envelope):

```json
// POST /auth/exchange → 200
{
  "token": "eyJ...",
  "refresh_token": "a1b2c3...",
  "student_id": "uuid",
  "student": {
    "student_id": "uuid",
    "name": "Alice",
    "grade": 8,
    "locale": "en",
    "account_status": "active"
  }
}
```

### Error Response Envelope

All 4xx and 5xx responses use this format:

```json
{
  "error": "invalid_token",
  "detail": "JWT signature verification failed",
  "correlation_id": "a1b2c3d4-e5f6-..."
}
```

| HTTP Status | `error` value | When |
|---|---|---|
| 400 | `bad_request` | Malformed body, missing required field |
| 401 | `unauthenticated` | Missing or expired JWT |
| 401 | `invalid_token` | JWT signature invalid |
| 403 | `forbidden` | Valid JWT but insufficient role |
| 403 | `account_suspended` | Account status is `suspended` |
| 403 | `account_pending` | Account awaiting verification |
| 404 | `not_found` | Resource does not exist |
| 409 | `conflict` | Duplicate resource (e.g., email already registered) |
| 422 | `validation_error` | Pydantic schema validation failure (FastAPI default) |
| 429 | `rate_limited` | Rate limit exceeded; includes `Retry-After` header |
| 500 | `internal_error` | Unhandled exception; safe message only |
| 503 | `service_unavailable` | Health check failed; dependency unreachable |

### Pagination Shape (Admin List Endpoints)

```json
// GET /admin/accounts/students?status=active&page=1&per_page=50
{
  "items": [ { "student_id": "...", "name": "...", ... } ],
  "total": 1234,
  "page": 1,
  "per_page": 50,
  "pages": 25
}
```

Default `per_page = 50`. Maximum `per_page = 200`. Sort order: `created_at DESC`.

### `GET /curriculum` Response

```json
[
  { "grade": 5, "subject_count": 4, "unit_count": 32 },
  { "grade": 6, "subject_count": 4, "unit_count": 34 },
  { "grade": 7, "subject_count": 5, "unit_count": 40 },
  { "grade": 8, "subject_count": 5, "unit_count": 40 },
  ...
]
```

### `GET /health` Endpoint Name

The health endpoint is `GET /health` (not `/health/live` or `/health/ready`).
Use it as both the liveness and readiness probe in Phase 1.

---

## 10. Design Clarifications

### 10.1 Account Suspension — TTL Fix

**Original design had a bug:** `suspended:{id}` in Redis with TTL = 15 min (JWT lifetime)
meant the key would expire naturally and the student would silently regain access without admin action.

**Correct design:**

```
- On suspension:   SET suspended:{id} 1  (no TTL — permanent until explicitly removed)
- On reactivation: DEL suspended:{id}
- Auth middleware: EXISTS suspended:{id}  → 403 if found
```

The Redis key has **no TTL**. It is only removed when an admin explicitly reactivates the account
or when the account is deleted. Auth0 block (15-min sync window) catches any gap where a student's
JWT is still valid but the Redis key has been set.

### 10.2 Logout Endpoint

Add `POST /auth/logout` to invalidate the refresh token:

```
POST /auth/logout
Authorization: Bearer <access_token>
Body: { "refresh_token": "..." }
Response: 200 {}
```

Action: delete `refresh:{token_hash}` key from Redis. The access JWT expires naturally
within 15 minutes — no server-side revocation needed for the short-lived access token.

### 10.3 JWT Signing Algorithm

- **Phase 1:** HS256 (symmetric HMAC-SHA256) with `JWT_SECRET` / `ADMIN_JWT_SECRET`
- **Phase 2+:** Migrate to RS256 (asymmetric RSA) for zero-downtime key rotation via `kid` header
- All JWTs carry `exp` (expiry), `iat` (issued at), `jti` (JWT ID for replay detection)

### 10.4 Account Status Transition Rules

| From → To | Allowed by | Notes |
|---|---|---|
| `pending` → `active` | System (on verification) | Triggered by email verification or parental consent |
| `active` → `suspended` | product_admin, super_admin, school_admin (own school only) | Adds to Redis suspended set |
| `suspended` → `active` | product_admin, super_admin, school_admin (own school only) | Removes from Redis suspended set |
| `active` → `deleted` | product_admin, super_admin | Soft-delete; schedules GDPR anonymisation |
| `suspended` → `deleted` | product_admin, super_admin | Same |
| `deleted` → * | Nobody | Terminal state |
| `pending` → `suspended` | product_admin, super_admin | Edge case (abuse during signup) |

### 10.5 Admin Login Lockout

Admin lockout is tracked **in Redis only** (not in the `admin_users` table):

```
Key:   login_attempts:{email}
Value: integer (incremented on each failure)
TTL:   900 seconds (15 minutes)
```

On 5th failure: return 423 Locked + `Retry-After: 900` header.
Auto-resets when TTL expires. No manual unlock required for standard lockout.

### 10.6 School Cascade on Suspension — Reactivation

Suspending a school suspends all its teachers and students.
**Reactivating a school does NOT automatically reactivate its members** — each must be
explicitly reactivated. This prevents accidentally un-suspending a student who was
individually suspended for a different reason.
