# StudyBuddy OnDemand — Backend Architecture

**Version:** 1.0.0
**Last updated:** 2026-03-23
**Companion docs:** [ARCHITECTURE.md](ARCHITECTURE.md) · [REQUIREMENTS.md](REQUIREMENTS.md)

---

## Design Principle

The backend has two fundamentally different traffic patterns that must be optimised independently:

| Path | Characteristics | Target |
|---|---|---|
| **Hot read path** | Student fetches lesson/quiz content | p95 < 50 ms (cache hit), < 200 ms (miss) |
| **Write path** | Progress events, analytics events | Fire-and-forget; async; never blocks the student |
| **Auth path** | Login, register, token refresh | p95 < 500 ms (bcrypt-bound) |
| **Admin / pipeline** | Content generation, analytics | Latency-insensitive; async jobs |

Every architectural decision below derives from this separation. The hot read path must be optimised end-to-end. Writes must never block reads.

### Logical Architecture

The backend is structured as six horizontal layers. Dependencies flow downward only — no layer imports from a layer above it.

```mermaid
graph TB
    subgraph EDGE_L["Edge Layer"]
        CDN_L["CloudFront CDN\nAudio MP3s + static JSON files\nGlobal edge caching · signed URLs"]
        NGX_L["nginx Reverse Proxy\nTLS termination · gzip compression\nRate limiting · keep-alive pooling\nHTTP→HTTPS redirect"]
    end

    subgraph APP_L["Application Layer  —  Stateless · Horizontally Scalable"]
        subgraph WP1["Worker Process (uvicorn) × N"]
            L1_L["L1 In-Process Cache\ncachetools TTLCache\nJWT keys · curriculum trees · config\n(ephemeral, per-process)"]
            FAPI_L["FastAPI Routers + Middleware\nAuth · Content · Progress · Subscription\nSchool · Analytics · Feedback · Admin"]
        end
        GUN_L["gunicorn\nProcess manager\n2×vCPU+1 workers"]
    end

    subgraph CACHE_L["Cache Layer  —  Shared Across All Workers"]
        RD_L["Redis Cluster\nEntitlement · Curriculum resolver\nContent JSON · Rate limiters\nJWT refresh tokens · Job state\nCelery broker + result backend"]
    end

    subgraph WORKER_L["Worker Layer  —  Async Background Jobs"]
        CQ_L["Celery Queues\npipeline  ·  io  ·  default"]
        CW_L["Celery Workers\nPipeline content generation\nEmail · Progress flush\nCache invalidation"]
        CB_L["Celery Beat\nNightly: materialized view refresh\nHourly: content freshness check\nEvery 5min: stuck event flush"]
    end

    subgraph DB_L["Database Layer"]
        PGB_L["PgBouncer\nTransaction-mode pooling\npool_size=50"]
        PGP_L[("PostgreSQL Primary\nWrites + strong-consistency reads")]
        PGR_L[("PostgreSQL Read Replica\nAnalytics queries · Reports")]
        MV_L["Materialized Views\nmv_unit_struggle\nRefreshed nightly"]
    end

    subgraph CONTENT_L["Content Layer"]
        S3_L["S3 Bucket\ncurricula/{curriculum_id}/{unit_id}/\nlesson_{lang}.json · quiz_set_{n}_{lang}.json\ntutorial_{lang}.json · lesson_{lang}.mp3"]
    end

    EDGE_L -->|"Routed requests"| APP_L
    APP_L -->|"L2 cache reads/writes\nTask dispatch"| CACHE_L
    APP_L -->|"Strong-read/write queries"| DB_L
    APP_L -->|"Presigned URL generation"| CONTENT_L
    CACHE_L -->|"Task queue"| WORKER_L
    WORKER_L -->|"Pipeline writes\nAnalytics aggregation"| DB_L
    WORKER_L -->|"Write generated content"| CONTENT_L
    CDN_L -->|"Direct content fetch\n(bypasses App Layer)"| CONTENT_L
    PGP_L -->|"Streaming replication"| PGR_L
    PGR_L --> MV_L

    style EDGE_L fill:#2a1a4a,color:#e2e8f0
    style APP_L fill:#1e3a5f,color:#e2e8f0
    style CACHE_L fill:#1a3a1a,color:#e2e8f0
    style WORKER_L fill:#3a2a1a,color:#e2e8f0
    style DB_L fill:#1a2a4a,color:#e2e8f0
    style CONTENT_L fill:#3a1a1a,color:#e2e8f0
```

---

## System Topology

```mermaid
graph TD
    subgraph CLIENTS["Clients"]
        MOB["Mobile App\n(Kivy)"]
        TEACH["Teacher Dashboard\n(Web Browser)"]
        ADMIND["Admin Dashboard\n(Web Browser)"]
    end

    subgraph EDGE["Edge / Ingress"]
        CDN["CDN\nCloudFront / Cloudflare\nStatic content + audio\nCache-Control headers"]
        GW["API Gateway + Reverse Proxy\nnginx / Caddy\nTLS termination\nRate limiting\nRequest routing"]
    end

    subgraph APP["Application Tier (Stateless)"]
        API1["FastAPI Worker 1\nuvicorn"]
        API2["FastAPI Worker 2\nuvicorn"]
        APIN["FastAPI Worker N\nuvicorn"]
    end

    subgraph CACHE["Cache Tier"]
        REDIS["Redis Cluster\nL2 Cache\nEntitlement · JWT tokens\nCurriculum resolver\nRate limiters\nJob status\nSession state"]
    end

    subgraph DB["Database Tier"]
        PGPRIMARY[("PostgreSQL Primary\nWrites + strong-read queries")]
        PGREPLICA[("PostgreSQL Read Replica\nAnalytics queries\nReporting")]
    end

    subgraph CONTENT["Content Tier"]
        S3["S3 Bucket\nPre-generated content\n lesson.json · quiz_set_N.json\ntutorial.json · experiment.json\nlesson.mp3"]
    end

    subgraph WORKERS["Background Worker Tier"]
        CW1["Celery Worker\nPipeline jobs\n(CPU+IO bound)"]
        CW2["Celery Worker\nEmail · webhooks\n(IO bound)"]
        BEAT["Celery Beat\nScheduled: analytics agg\ncontent freshness check"]
        BROKER["Redis\n(Celery broker + result backend)"]
    end

    subgraph EXTERNAL["External Services"]
        STRIPE["Stripe"]
        ANT["Anthropic API"]
        TTS["Amazon Polly\nGoogle Cloud TTS"]
        SES["AWS SES / SendGrid\nTransactional email"]
    end

    MOB & TEACH & ADMIND -->|HTTPS| CDN
    CDN -->|Cache miss| GW
    CDN -->|Signed URLs for MP3 + JSON| S3

    GW -->|Load balance| API1 & API2 & APIN

    API1 & API2 & APIN -->|aioredis pool| REDIS
    API1 & API2 & APIN -->|asyncpg pool| PGPRIMARY
    API1 & API2 & APIN -->|asyncpg pool| PGREPLICA
    API1 & API2 & APIN -->|boto3 async / presigned URLs| S3
    API1 & API2 & APIN -->|Celery task dispatch| BROKER

    CW1 -->|API calls| ANT & TTS
    CW1 -->|Write| S3
    CW1 -->|Write| PGPRIMARY
    CW2 -->|Send| SES
    CW2 -->|Webhook calls| STRIPE
    BEAT -->|Enqueue| BROKER
    BROKER -->|Task queue| CW1 & CW2
```

---

## Hot Read Path — End-to-End

This is the most critical path: a student opens a lesson or starts a quiz.

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant GW as nginx
    participant API as FastAPI Worker
    participant L1 as In-Process Cache
    participant L2 as Redis
    participant DB as PostgreSQL
    participant S3 as S3 / CDN

    App->>GW: GET /content/G8-MATH-001/lesson<br/>Authorization: Bearer <jwt>
    GW->>GW: TLS termination + rate limit check (Redis counter)
    GW->>API: Forward request

    API->>API: Verify JWT signature (in-memory key, ~0.1ms)
    API->>L1: curriculum_id for (school_id, grade, year)?
    alt L1 hit (~0.1ms)
        L1-->>API: curriculum_id
    else L1 miss
        API->>L2: GET curriculum:{student_id}
        alt L2 hit (~1ms)
            L2-->>API: curriculum_id
        else L2 miss (~5ms)
            API->>DB: SELECT active curriculum
            DB-->>API: curriculum_id
            API->>L2: SET curriculum:{student_id} TTL=300s
            API->>L1: store in local LRU
        end
    end

    API->>L2: GET entitlement:{student_id} (~1ms)
    alt Cache hit
        L2-->>API: {plan, lessons_accessed}
    else Cache miss
        API->>DB: SELECT subscription + lesson count
        DB-->>API: entitlement data
        API->>L2: SET entitlement:{student_id} TTL=300s
    end

    alt Entitlement denied
        API-->>App: 402 SUBSCRIPTION_REQUIRED
    end

    API->>L2: GET content:{curriculum_id}:{unit_id}:lesson:{locale}
    alt Redis hit (~1ms)
        L2-->>API: lesson JSON
    else Redis miss
        API->>S3: GetObject curriculum_id/unit_id/lesson_en.json (~30-80ms)
        S3-->>API: lesson JSON
        API->>L2: SET content:… TTL=3600s (1hr)
    end

    API-->>App: 200 lesson JSON (~5-100ms total)
```

**Key optimisation:** the common case (enrolled student, valid subscription, lesson in Redis) executes zero database queries.

---

## Caching Architecture

### Three Cache Levels

| Level | Store | Scope | Purpose |
|---|---|---|---|
| L1 | In-process (per worker) | Per-process, ephemeral | JWT keys, curriculum tree, grade JSON |
| L2 | Redis | Shared across all workers | Entitlement, curriculum resolver, content, rate limits, sessions |
| L3 | CDN (CloudFront) | Global edge | Audio MP3s, large static JSON files |

### Cache Read Flow

```mermaid
flowchart LR
    REQ_C["Request arrives\nat FastAPI worker"]

    subgraph L1BOX["L1 — In-Process TTLCache\n(per worker, ~0ms)"]
        L1_KEYS["JWT signing keys  TTL 1hr\nCurriculum trees  TTL 5min\nApp config / thresholds  TTL 1min"]
    end

    subgraph L2BOX["L2 — Redis Cluster\n(shared, ~1ms)"]
        L2_ENT["ent:{student_id}  TTL 5min\nEntitlement + plan"]
        L2_CUR["cur:{student_id}  TTL 5min\nCurriculum resolver result"]
        L2_CON["content:{curriculum}:{unit}:{type}:{lang}\nLesson/quiz JSON  TTL 1hr"]
        L2_RL["rl:auth:{ip} / rl:content:{student}\nRate limit counters  TTL 60s"]
        L2_JWT["jwt_refresh:{token_hash}\nRefresh tokens  TTL = token expiry"]
        L2_JOB["job:{job_id}\nPipeline job state  TTL 24hr"]
    end

    subgraph L3BOX["L3 — CloudFront CDN\n(global edge, ~5-30ms)"]
        L3_JSON["lesson/quiz JSON\nCache-Control: max-age=3600"]
        L3_MP3["lesson MP3 audio\nCache-Control: max-age=86400"]
    end

    subgraph ORIGIN["Origin (cache miss only)"]
        PG_O[("PostgreSQL\n~5-20ms")]
        S3_O[("S3\n~30-80ms")]
    end

    REQ_C --> L1BOX
    L1BOX -->|"Miss"| L2BOX
    L2BOX -->|"Miss (content)"| L3BOX
    L2BOX -->|"Miss (entitlement/curriculum)"| PG_O
    L3BOX -->|"Miss"| S3_O
    PG_O -->|"Populate L2"| L2BOX
    S3_O -->|"Populate L2 + L3"| L2BOX
```

### L1 — In-Process Cache

Implemented with `cachetools.TTLCache` (thread-safe LRU+TTL). Stored in a module-level singleton — survives within a worker process but is lost on restart.

```python
# backend/src/core/cache.py
from cachetools import TTLCache
import threading

_lock = threading.Lock()

L1_CURRICULUM_TREE = TTLCache(maxsize=50, ttl=300)     # grade trees rarely change
L1_JWT_KEYS        = TTLCache(maxsize=10, ttl=3600)    # signing key by kid
L1_APP_CONFIG      = TTLCache(maxsize=100, ttl=60)     # feature flags, thresholds
```

Never store per-student data in L1 — the LRU would fill up and evict useful entries.

### L2 — Redis Key Schema

All keys are namespaced to avoid collisions. TTLs are set at write time.

| Key pattern | Value | TTL | Invalidation trigger |
|---|---|---|---|
| `ent:{student_id}` | `{plan, lessons_accessed, valid_until}` JSON | 300 s | Subscription change, lesson view |
| `cur:{student_id}` | `curriculum_id` string | 300 s | School transfer, curriculum activation |
| `content:{curriculum_id}:{unit_id}:{type}:{locale}` | Lesson/quiz JSON string | 3600 s | Content version bump |
| `jwt_refresh:{token_hash}` | `student_id` | Token TTL | Logout, rotation |
| `rl:auth:{ip}` | Request counter | 60 s | Rolling window |
| `rl:content:{student_id}` | Request counter | 60 s | Rolling window |
| `rl:feedback:{student_id}` | Submission counter | 3600 s | Rolling window |
| `job:{job_id}` | `{status, progress, errors}` JSON | 86400 s | Pipeline completion |
| `quiz_set:{student_id}:{unit_id}` | `1\|2\|3` (last set used) | 30 days | — |

**Redis configuration for production:**

```
maxmemory 2gb
maxmemory-policy allkeys-lru
appendonly yes                  # AOF persistence — survive restart
appendfsync everysec
requirepass <strong-password>
```

### L3 — CDN

CloudFront (or Cloudflare) sits in front of S3. The mobile app fetches audio and large JSON directly from the CDN URL — not through the FastAPI servers — reducing backend load.

| Asset type | Cache-Control | CDN TTL | Invalidation |
|---|---|---|---|
| `lesson_{lang}.json` | `public, max-age=3600` | 1 hour | `content_version` bump → CDN invalidation |
| `quiz_set_{N}_{lang}.json` | `public, max-age=3600` | 1 hour | Same |
| `lesson_{lang}.mp3` | `public, max-age=86400` | 24 hours | Rarely changes |
| Signed URLs (private) | `private, no-store` | N/A | Expiry in URL |

**Content delivery flow for audio:**
1. `GET /content/{unit_id}/lesson/audio` → API generates a pre-signed S3 URL (TTL 1 hour)
2. Mobile app fetches MP3 directly from CloudFront using the signed URL
3. FastAPI server never proxies audio bytes — no memory or bandwidth cost

---

## Application Server

### Process Model

```
gunicorn (process manager)
  ├── uvicorn worker 1  (async event loop)
  ├── uvicorn worker 2
  ├── uvicorn worker 3
  └── uvicorn worker N   (N = 2 × vCPU + 1)
```

Each uvicorn worker is a single-threaded async event loop. All I/O (DB, Redis, S3) is non-blocking. CPU-bound work (bcrypt, JWT sign) is offloaded to a thread pool executor.

```python
# backend/main.py
import asyncio
from contextlib import asynccontextmanager
import asyncpg
import aioredis
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Create connection pools once per worker process at startup
    app.state.db_pool = await asyncpg.create_pool(
        dsn=settings.DATABASE_URL,
        min_size=5,
        max_size=20,
        command_timeout=10,
        server_settings={"application_name": "studybuddy-api"},
    )
    app.state.redis = aioredis.from_url(
        settings.REDIS_URL,
        encoding="utf-8",
        decode_responses=True,
        max_connections=10,
    )
    yield
    await app.state.db_pool.close()
    await app.state.redis.close()

app = FastAPI(lifespan=lifespan)
```

### Connection Pool Sizing

| Resource | Pool size per worker | Workers | Max total connections |
|---|---|---|---|
| PostgreSQL primary | min 5, max 20 | 4 | 80 |
| PostgreSQL replica | min 2, max 10 | 4 | 40 |
| Redis | max 10 | 4 | 40 |

PostgreSQL `max_connections` should be set to ≥ 200 (leaving headroom for migrations, admin queries, and Celery workers).

Use **PgBouncer** in transaction-pooling mode in front of PostgreSQL to reduce connection overhead at scale.

```
                  ┌─────────────────┐
  API workers ───►│   PgBouncer     │───► PostgreSQL
  Celery workers ►│  pool_size=50   │     max_connections=200
                  └─────────────────┘
```

### Async Throughout

Every database operation uses `asyncpg` (async-native). Every Redis operation uses `aioredis`. External HTTP calls (Stripe, Anthropic, TTS) use `httpx.AsyncClient`. bcrypt runs in `asyncio.get_event_loop().run_in_executor(None, ...)`.

Nothing blocks the event loop.

---

## Database Architecture

### Primary vs Read Replica

| Query type | Route to | Reason |
|---|---|---|
| Student auth, writes (INSERT/UPDATE) | Primary | Consistency required |
| Content entitlement check (SELECT) | Primary | Must be current |
| Analytics aggregations (`GET /analytics/…`) | Replica | Expensive; eventual consistency acceptable |
| Admin reporting | Replica | Same reason |
| Progress history `GET /progress/student` | Replica | Acceptable 1–2s lag |

The routing is encapsulated in a dependency:

```python
async def get_db(request: Request):
    return request.app.state.db_pool          # primary

async def get_read_db(request: Request):
    return request.app.state.db_replica_pool  # replica
```

### Critical Indexes

These are the indexes that make hot-path queries fast. All must be created in the initial migration.

```sql
-- Curriculum resolver: (school, grade, year) → active curriculum_id
CREATE INDEX idx_curricula_resolver
    ON curricula (school_id, grade, year, status)
    WHERE status = 'active';

-- Entitlement: count distinct lesson views per free-tier student
CREATE INDEX idx_lesson_views_student
    ON lesson_views (student_id, curriculum_id);

-- Subscription lookup by student
CREATE INDEX idx_subscriptions_student
    ON subscriptions (student_id)
    WHERE status = 'active';

-- Attempt number: count prior attempts for a student × unit × curriculum
CREATE INDEX idx_sessions_attempt_count
    ON sessions (student_id, unit_id, curriculum_id);

-- Progress history ordered by time
CREATE INDEX idx_sessions_student_time
    ON sessions (student_id, started_at DESC);

-- Enrolment lookup on student registration (email match)
CREATE INDEX idx_enrolments_email
    ON school_enrolments (student_email)
    WHERE status = 'pending';

-- Stripe event deduplication
CREATE UNIQUE INDEX idx_stripe_events_id
    ON stripe_events (stripe_event_id);

-- Content reports by unit (for admin review queue)
CREATE INDEX idx_content_reports_unit
    ON content_reports (unit_id, reviewed)
    WHERE reviewed = false;

-- Feedback by category + reviewed
CREATE INDEX idx_feedback_admin
    ON feedback (category, reviewed, submitted_at DESC)
    WHERE reviewed = false;
```

### Materialized Views for Analytics

Heavy analytics queries run on the replica against pre-aggregated materialized views, refreshed by Celery Beat nightly.

```sql
-- Struggle analysis: units with low pass rate or high attempt count
CREATE MATERIALIZED VIEW mv_unit_struggle AS
SELECT
    unit_id,
    curriculum_id,
    COUNT(*)                                         AS total_attempts,
    COUNT(*) FILTER (WHERE attempt_number = 1)       AS first_attempts,
    COUNT(*) FILTER (WHERE passed AND attempt_number = 1) AS first_attempt_passes,
    AVG(attempt_number) FILTER (WHERE passed)        AS avg_attempts_to_pass,
    AVG(score::float / total_questions)              AS avg_score,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE passed AND attempt_number = 1)
        / NULLIF(COUNT(*) FILTER (WHERE attempt_number = 1), 0), 1
    )                                                AS first_attempt_pass_rate_pct
FROM sessions
WHERE completed = true
GROUP BY unit_id, curriculum_id;

CREATE UNIQUE INDEX ON mv_unit_struggle (unit_id, curriculum_id);

-- Refresh nightly at 02:00 UTC (Celery Beat task)
```

### Database Tier Component Diagram

```mermaid
graph TB
    subgraph APP_TIER["Application Tier"]
        API_POOL["FastAPI Workers\nasyncpg pool\nmin=5 max=20 per worker"]
        CEL_POOL["Celery Workers\nasyncpg pool\nmin=2 max=10 per worker"]
    end

    subgraph PGB_TIER["PgBouncer  (transaction-pooling mode)"]
        PGB_CFG["pool_size = 50\nserver_idle_timeout = 600s\npool_mode = transaction\nmax_client_conn = 200"]
    end

    subgraph PRIMARY["PostgreSQL Primary  (writes + strong reads)"]
        HOT["Hot read tables\nstudents · subscriptions · curricula\nschool_enrolments"]
        WRITE["Write tables\nsessions · progress_answers\nlesson_views · feedback\nstripe_events"]
        IDX_P["Indexes\nidx_curricula_resolver (school, grade, year, status)\nidx_subscriptions_student (student_id) WHERE active\nidx_sessions_attempt_count (student, unit, curriculum)\nidx_enrolments_email (student_email) WHERE pending\nidx_stripe_events_id UNIQUE (stripe_event_id)"]
    end

    subgraph REPLICA["PostgreSQL Read Replica  (analytics + reports)"]
        ANLQ["Analytics queries\nGET /analytics/school/{id}/class\nGET /analytics/unit/{id}/summary"]
        MV_D["Materialized View\nmv_unit_struggle\n(pass rate · avg attempts · avg score)\nRefreshed nightly 02:00 UTC"]
        IDX_R["Indexes\nidx_lesson_views_student (student_id, curriculum_id)\nidx_feedback_admin (category, reviewed, submitted_at)"]
    end

    API_POOL -->|"Write + strong reads"| PGB_CFG
    CEL_POOL -->|"Pipeline writes"| PGB_CFG
    PGB_CFG --> HOT & WRITE
    API_POOL -->|"Analytics (direct, no PgBouncer)"| REPLICA
    ANLQ --> MV_D
    PRIMARY -->|"Streaming replication"| REPLICA

    style PRIMARY fill:#1e3a5f,color:#e2e8f0
    style REPLICA fill:#1a3a1a,color:#e2e8f0
    style PGB_TIER fill:#2a2a1a,color:#e2e8f0
```

---

## Background Worker Architecture

### Celery Topology

```
Redis (broker)
  ├── Queue: pipeline      → CeleryWorker[pipeline] (2-4 workers, high CPU/IO)
  ├── Queue: email         → CeleryWorker[io]       (2 workers, light IO)
  ├── Queue: analytics     → CeleryWorker[io]       (shared with email)
  └── Queue: default       → CeleryWorker[io]       (misc tasks)

Celery Beat (scheduler)
  ├── Every night 02:00 UTC: refresh_materialized_views
  ├── Every hour:            check_content_freshness
  └── Every 5 min:           flush_stuck_progress_events  (safety net)
```

```mermaid
graph TB
    subgraph DISPATCH["Task Dispatch — FastAPI Workers"]
        API_W2["FastAPI Worker\n.send_task(name, args, queue=...)"]
    end

    subgraph BROKER_W["Redis — Celery Broker"]
        Q_PIPE["Queue: pipeline\nContent generation jobs"]
        Q_IO["Queue: io\nEmail · Stripe callbacks"]
        Q_DEF["Queue: default\nProgress flush · cache invalidation"]
    end

    subgraph PIPE_WORKERS["Pipeline Workers  (2–4, CPU+IO bound)"]
        PW["celery worker -Q pipeline\n--concurrency=2"]
        subgraph CHAIN["Task Chain per Curriculum"]
            TC1["validate_curriculum"]
            TC2["build_units.chunks\n5 units in parallel"]
            TC3["build_unit\nlesson → quizzes (×3)\n→ tutorial → experiment\n→ TTS (MP3)"]
            TC4["finalize_curriculum\nstatus=active · notify teacher"]
        end
        TC1 --> TC2 --> TC3 --> TC4
    end

    subgraph IO_WORKERS["IO Workers  (2, lightweight)"]
        IW["celery worker -Q io,default\n--concurrency=4"]
        IT1["send_email\n(SES / SendGrid)"]
        IT2["flush_progress_events\nbulk INSERT ON CONFLICT DO NOTHING"]
        IT3["invalidate_cache\nRedis DEL ent: / cur: keys"]
        IT4["refresh_materialized_views\nCONCURRENTLY on replica"]
    end

    subgraph BEAT_W["Celery Beat — Scheduler"]
        SCH1["02:00 UTC daily → refresh_materialized_views"]
        SCH2["Every 1hr → check_content_freshness"]
        SCH3["Every 5min → flush_stuck_progress_events"]
    end

    API_W2 --> Q_PIPE & Q_IO & Q_DEF
    Q_PIPE --> PW
    Q_IO & Q_DEF --> IW
    IW --> IT1 & IT2 & IT3 & IT4
    BEAT_W --> Q_IO & Q_DEF

    style PIPE_WORKERS fill:#1e3a5f,color:#e2e8f0
    style IO_WORKERS fill:#1a3a1a,color:#e2e8f0
    style BEAT_W fill:#3a2a1a,color:#e2e8f0
```

### Pipeline Task Design

The pipeline for a custom curriculum is an async Celery chain:

```
validate_curriculum(curriculum_id)
  → build_units.chunks(unit_list, chunk_size=5)   # 5 units in parallel
      → [build_unit(unit_id, lang) for each unit]
          → generate_lesson → validate → write
          → generate_quizzes (3) → validate → write
          → generate_tutorial → validate → write
          → generate_experiment (if lab) → validate → write
          → generate_tts → write mp3
  → finalize_curriculum(curriculum_id)             # set status=active, notify teacher
```

Parallelism within a grade: process 5 units concurrently (limited by Anthropic API rate limits). Each unit's assets are generated sequentially within that unit (lesson → quiz → tutorial → experiment → TTS) because each depends on the lesson content.

### Progress Event Flush

Progress events arrive from mobile clients in the offline queue. A lightweight Celery task deduplicates and bulk-inserts them:

```python
@app.task(queue="default", acks_late=True)
async def flush_progress_events(student_id: str, events: list[dict]):
    """
    Bulk-insert progress events. Deduplicate by event_id.
    acks_late=True: message stays in queue until task completes.
    """
    async with db_pool.acquire() as conn:
        await conn.executemany(
            """
            INSERT INTO progress_answers (event_id, session_id, ...)
            VALUES ($1, $2, ...)
            ON CONFLICT (event_id) DO NOTHING
            """,
            [(e["event_id"], ...) for e in events],
        )
```

---

## API Gateway (nginx)

nginx sits in front of all FastAPI workers, handling:
- TLS termination (Let's Encrypt / ACM certificate)
- HTTP → HTTPS redirect
- Rate limiting (leaky bucket, backed by shared memory zone)
- Request routing (API vs static assets vs webhooks)
- Compression (gzip for JSON responses > 1 KB)
- Keep-alive connections to upstream workers

```nginx
upstream studybuddy_api {
    least_conn;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
    server 127.0.0.1:8004;
    keepalive 32;
}

limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=10r/m;
limit_req_zone $http_authorization  zone=api_limit:20m  rate=100r/m;

server {
    listen 443 ssl http2;
    ssl_certificate     /etc/letsencrypt/live/api.studybuddy.app/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.studybuddy.app/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    gzip on;
    gzip_types application/json text/plain;
    gzip_min_length 1024;

    # Auth endpoints — strict rate limit
    location ~ ^/auth/(login|register|forgot-password) {
        limit_req zone=auth_limit burst=5 nodelay;
        proxy_pass http://studybuddy_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # Stripe webhook — no rate limit, IP allowlist
    location /subscription/webhook {
        allow 54.187.174.169;   # Stripe IP range (maintain full list)
        deny all;
        proxy_pass http://studybuddy_api;
    }

    # General API
    location / {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://studybuddy_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 30s;
        proxy_connect_timeout 5s;
    }
}

server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

---

## Write Path Optimisation

Progress events and analytics events are high-frequency writes that must not slow down the student experience.

### Fire-and-Forget Pattern

`POST /progress/answer` and `POST /analytics/lesson/end` return `200 OK` immediately after:
1. Validating the JWT (< 1 ms)
2. Validating the request body (Pydantic, < 1 ms)
3. Dispatching a Celery task (Redis publish, < 2 ms)

The actual database write happens asynchronously in a Celery worker. The student is never waiting for a database round-trip.

```python
@router.post("/progress/answer", status_code=200)
async def record_answer(
    body: AnswerEvent,
    student: Student = Depends(get_current_student),
    celery: Celery = Depends(get_celery),
):
    # Validate event_id format only — no DB touch
    celery.send_task(
        "tasks.flush_progress_events",
        args=[student.id, [body.dict()]],
        queue="default",
    )
    return {"ok": True}
```

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant GW as nginx
    participant API as FastAPI Worker
    participant RD as Redis (Celery Broker)
    participant CW as Celery Worker
    participant DB as PostgreSQL

    App->>GW: POST /progress/answer {event_id, session_id, answer, correct, ms_taken}
    GW->>API: Forward request
    API->>API: Verify JWT (~0.1ms)
    API->>API: Pydantic body validation (~0.5ms)
    API->>RD: LPUSH queue:default {task: flush_progress_events, payload} (~1ms)
    API-->>App: 200 OK {ok: true}   ← Returns immediately (total ~3ms)

    Note over RD,DB: Asynchronous — happens within seconds, invisible to student

    RD->>CW: Dequeue task
    CW->>DB: INSERT INTO progress_answers (event_id, …)<br/>ON CONFLICT (event_id) DO NOTHING
    DB-->>CW: OK (idempotent — safe to retry)
    CW->>RD: ACK (task complete)
```

### Bulk Writes

Events from the same student's offline queue flush arrive together. The Celery task receives a list and bulk-inserts with `executemany`, then deduplicates with `ON CONFLICT (event_id) DO NOTHING`.

---

## Circuit Breakers & Resilience

External service calls (Stripe, Anthropic, TTS, SES) are wrapped in circuit breakers to prevent cascading failures.

```python
# backend/src/core/circuit_breaker.py
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30, expected_exception=httpx.HTTPError)
async def call_stripe(method, **kwargs):
    ...

@circuit(failure_threshold=3, recovery_timeout=60, expected_exception=Exception)
async def call_anthropic(prompt, **kwargs):
    ...
```

If Stripe is down, `POST /subscription/checkout` returns HTTP 503 — the mobile app shows a "Try again later" message. The student's lesson access is unaffected (entitlement is cached in Redis).

If Anthropic is down during a pipeline run, the circuit opens after 3 failures. The pipeline task catches `CircuitBreakerError`, marks the unit as failed, and continues to the next unit. Failed units are reported to the teacher.

---

## Deployment Topologies

### Development (Docker Compose)

Everything on one machine. Good for Phase 1–3 development.

```yaml
# docker-compose.yml (abbreviated)
services:
  api:
    build: ./backend
    command: uvicorn main:app --reload --host 0.0.0.0 --port 8000
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@db/studybuddy
      REDIS_URL: redis://redis:6379/0
    depends_on: [db, redis]

  worker:
    build: ./backend
    command: celery -A tasks worker -Q pipeline,default --concurrency=2
    depends_on: [db, redis]

  beat:
    build: ./backend
    command: celery -A tasks beat --scheduler django_celery_beat.schedulers:DatabaseScheduler
    depends_on: [db, redis]

  db:
    image: postgres:16
    volumes: [postgres_data:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes

  nginx:
    image: nginx:alpine
    volumes: [./nginx.conf:/etc/nginx/nginx.conf:ro]
    ports: ["443:443", "80:80"]
```

### Production — Small Scale (Phases 2–6, up to ~10k students)

| Component | Service | Sizing |
|---|---|---|
| API servers | 2× EC2 `t3.medium` (2 vCPU, 4 GB) or equivalent | 4 uvicorn workers each |
| Database | AWS RDS PostgreSQL `db.t3.medium` Multi-AZ | 1 primary + 1 replica |
| Cache | AWS ElastiCache Redis `cache.t3.micro` | Single node + AOF |
| PgBouncer | Runs on API server instances | transaction pooling |
| Content Store | S3 + CloudFront distribution | Standard |
| Workers | 1× EC2 `t3.small` running Celery | 2 pipeline + 2 IO workers |
| Load balancer | AWS ALB | Health check on `/health` |
| Email | AWS SES or SendGrid | Per-use |

```mermaid
graph TB
    subgraph INTERNET_D["Internet"]
        MOB_D["Mobile App\nTeacher Dashboard\nAdmin Dashboard"]
    end

    subgraph AWS_D["AWS Cloud (us-east-1)"]
        subgraph EDGE_D["Edge"]
            CF_D["CloudFront Distribution\nOrigin: S3 bucket\nCache: audio 24hr · JSON 1hr\nGlobal PoPs"]
            ACM_D["ACM Certificate\n*.studybuddy.app"]
        end

        subgraph COMPUTE_D["Compute  (Multi-AZ)"]
            ALB_D["Application Load Balancer\nHealth check: GET /health\nSSL via ACM · access logs → S3"]

            subgraph ASG_D["Auto-Scaling Group  (2–8 × EC2 t3.large)"]
                EC2A_D["EC2 t3.large — AZ-a\nnginx + gunicorn (4 workers)\nPgBouncer sidecar\nSentry · CloudWatch agent"]
                EC2B_D["EC2 t3.large — AZ-b\nnginx + gunicorn (4 workers)\nPgBouncer sidecar\nSentry · CloudWatch agent"]
            end

            EC2W_D["EC2 t3.medium (Worker host)\nCelery pipeline workers ×2\nCelery IO workers ×2\nCelery Beat ×1"]
        end

        subgraph DATA_D["Data  (Multi-AZ)"]
            RDSP_D[("RDS PostgreSQL db.r6g.large\nPrimary — AZ-a\nMulti-AZ standby")]
            RDSR_D[("RDS PostgreSQL db.r6g.large\nRead Replica — AZ-b\nAnalytics + app reads")]
            ECC_D["ElastiCache Redis\ncache.r6g.large  3-node cluster\nAOF persistence enabled"]
        end

        subgraph STORAGE_D["Storage"]
            S3B_D["S3 Bucket: sb-content-prod\nVersioning enabled\nServer-side encryption (AES-256)\nLifecycle: archive after 1yr"]
        end

        subgraph OPS_D["Observability"]
            CWL_D["CloudWatch\nLogs · Metrics · Alarms\nDashboard: latency · error rate"]
            SENT_D["Sentry\nBackend + Mobile error tracking"]
        end
    end

    MOB_D -->|"Audio + static JSON\n(HTTPS)"| CF_D
    MOB_D -->|"API requests (HTTPS)"| ALB_D
    CF_D --> S3B_D
    ALB_D --> EC2A_D & EC2B_D
    EC2A_D & EC2B_D -->|"aioredis"| ECC_D
    EC2A_D & EC2B_D -->|"asyncpg → PgBouncer"| RDSP_D
    EC2A_D & EC2B_D -->|"asyncpg (direct)"| RDSR_D
    EC2A_D & EC2B_D -->|"presigned URLs"| S3B_D
    EC2W_D --> ECC_D & RDSP_D & S3B_D
    RDSP_D -->|"streaming replication"| RDSR_D
    EC2A_D & EC2B_D & EC2W_D --> CWL_D & SENT_D

    style ALB_D fill:#1e3a5f,color:#e2e8f0
    style ECC_D fill:#1a3a1a,color:#e2e8f0
    style RDSP_D fill:#1a2a4a,color:#e2e8f0
    style RDSR_D fill:#1a2a4a,color:#e2e8f0
```

### Production — Scaled (Phase 7+, >50k students)

Auto-scaling group for API servers behind ALB. ElastiCache cluster (3-node). RDS Multi-AZ with 2 read replicas (one for analytics, one for app reads). S3 + CloudFront with multi-region edge.

| Component | Service | Notes |
|---|---|---|
| API tier | Auto-Scaling Group: 2–8× `t3.large` | Scale on CPU >60% or request latency >300ms |
| Database | RDS `db.r6g.xlarge` Multi-AZ + 2 replicas | One replica for app, one for analytics |
| Cache | ElastiCache Redis Cluster 3 nodes | Sharding by key prefix |
| PgBouncer | Sidecar on each API instance | Transaction pooling |
| Workers | Auto-Scaling Group: 2–4× `t3.medium` | Scale on Celery queue depth |
| CDN | CloudFront with S3 origin | Cache content files at edge globally |

---

## Performance Targets (SLOs)

| Endpoint | p50 | p95 | p99 | Notes |
|---|---|---|---|---|
| `GET /content/{unit_id}/lesson` (cache hit) | < 10 ms | < 50 ms | < 100 ms | Redis L2 hit |
| `GET /content/{unit_id}/lesson` (cache miss) | < 50 ms | < 200 ms | < 400 ms | S3 read |
| `GET /content/{unit_id}/quiz` | < 10 ms | < 50 ms | < 100 ms | Same as lesson |
| `POST /auth/login` | < 200 ms | < 500 ms | < 800 ms | bcrypt bound |
| `POST /auth/register` | < 200 ms | < 500 ms | < 800 ms | bcrypt bound |
| `POST /progress/answer` | < 5 ms | < 20 ms | < 50 ms | Fire-and-forget |
| `POST /analytics/lesson/end` | < 5 ms | < 20 ms | < 50 ms | Fire-and-forget |
| `GET /curriculum/{grade}` | < 10 ms | < 50 ms | < 100 ms | L1 cache hit |
| `GET /analytics/school/{id}/class` | < 200 ms | < 1 s | < 3 s | Replica + mat. view |

Uptime target: **99.9%** (≤ 8.7 hours downtime per year).

---

## Technology Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| ASGI framework | FastAPI + uvicorn | Async-native; Pydantic validation; OpenAPI docs |
| Process manager | gunicorn | Manages uvicorn worker processes; graceful reload |
| Reverse proxy | nginx | TLS termination; rate limiting; keep-alive pooling |
| Async PostgreSQL | asyncpg | Fastest Python PostgreSQL driver; native async |
| Connection pooler | PgBouncer | Transaction-mode pooling; reduces DB connection overhead |
| Async Redis | aioredis / redis-py async | Non-blocking; matches FastAPI async model |
| Background tasks | Celery + Redis broker | Mature; retry/backoff; task routing; Beat scheduler |
| In-process cache | cachetools TTLCache | Zero-cost LRU+TTL; no network hop |
| Circuit breaker | `circuitbreaker` library | Wrap all external API calls |
| Migrations | Alembic | Schema versioning; automated apply on deploy |
| Content delivery | S3 + CloudFront | Pre-signed URLs; global CDN; no API server bandwidth |
| Metrics | `prometheus_client` + `prometheus-fastapi-instrumentator` | Auto HTTP metrics + custom business/infra metrics; `GET /metrics` scrape endpoint |
| Alerting | Prometheus Alertmanager | Alert rules in `docs/prometheus/alerts.yml`; routes to PagerDuty / Slack / SES |
| Dashboards | Grafana | Six dashboards (Platform Overview, Content & Cache, Student Activity, Backend Health, Security, SLO Burn Rate) |
| Distributed tracing | OpenTelemetry SDK + `opentelemetry-instrumentation-fastapi` | Request traces with span context; correlation ID propagation |
| Error tracking | Sentry (`sentry-sdk[fastapi]`) | Unhandled exceptions with `correlation_id` attached; PII stripped via `before_send` |
| Event logging | `core/events.py` — `emit_event()` | Structured JSON events + Prometheus counter increment in one call |
| Audit log | PostgreSQL `audit_log` table | Tamper-resistant record of security-critical actions |
| Deployment | Docker + Docker Compose (dev) / ECS or K8s (prod) | Reproducible; horizontal scaling |

---

## Key Rules (Do Not Break)

1. **The hot read path never touches PostgreSQL when the student has a valid cached entitlement.** If entitlement is in Redis, that is the only source of truth for that request.
2. **The FastAPI event loop never blocks.** bcrypt → thread pool. S3 → async SDK. DB → asyncpg. Any synchronous library call must be wrapped in `run_in_executor`.
3. **Audio files are never proxied through the API server.** The API issues a pre-signed URL; the client downloads directly from CloudFront.
4. **Progress writes never delay the client response.** Use the fire-and-forget Celery task pattern. The 200 OK is returned before the DB write occurs.
5. **Redis must survive a restart.** AOF persistence is mandatory in production. A Redis restart without persistence logs out every student and resets all rate limit counters.
6. **PgBouncer sits in front of PostgreSQL.** API workers create many short-lived async connections; without pooling, PostgreSQL will hit `max_connections` under moderate load.
7. **Content version bumps invalidate both Redis L2 and CDN.** Stale content in either layer means students see outdated material. On pipeline completion, call CloudFront `create_invalidation` for the affected `curriculum_id/*` paths.
