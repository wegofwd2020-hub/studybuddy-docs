# Diagram 3 — Service Dependencies

> Sync (HTTP) and async (Celery) call graph with per-path SLA targets.
> Audience: Developers, SRE.
> Last updated: 2026-04-05.

---

## Call Graph

```mermaid
graph TD
    %% ── Clients ──────────────────────────────────────────────────────────
    subgraph CLIENTS["Clients"]
        MOB["Mobile App\n(Kivy / Android)"]
        WEB["Web Browser\n(Teacher / Admin)"]
    end

    %% ── Edge ─────────────────────────────────────────────────────────────
    subgraph EDGE["Edge Layer"]
        CDN["CloudFront CDN\naudio MP3 · lesson JSON\nCache-Control: 1 hr JSON / 24 hr MP3"]
        GW["nginx\nTLS termination\nRate limiting\nIP allowlist for /metrics + webhooks"]
    end

    %% ── Application ──────────────────────────────────────────────────────
    subgraph APP["Application Layer  (stateless · N workers)"]
        API["FastAPI Workers\nuvicorn × N"]
    end

    %% ── Cache ────────────────────────────────────────────────────────────
    subgraph CACHE["Cache Layer"]
        REDIS["Redis\nL2 cache · Celery broker\nRate-limit counters · Job status"]
    end

    %% ── Data ─────────────────────────────────────────────────────────────
    subgraph DATA["Data Layer"]
        PGPRI[("PostgreSQL Primary\nWrites · strong reads")]
        PGREP[("PostgreSQL Read Replica\nAnalytics · reports")]
        S3["S3 Bucket\nPre-generated content\nlesson / quiz / tutorial / experiment / MP3"]
    end

    %% ── Workers ──────────────────────────────────────────────────────────
    subgraph WORKERS["Worker Layer"]
        CWP["Celery Pipeline Worker\ncontent build · TTS"]
        CWI["Celery IO Worker\nemail · Stripe callbacks · Auth0 sync"]
        BEAT["Celery Beat\nscheduled tasks"]
    end

    %% ── External ─────────────────────────────────────────────────────────
    subgraph EXT["External Services"]
        AUTH0["Auth0\nJWKS endpoint\n(cached 1 hr in L1)"]
        STRIPE["Stripe\ncheckout · webhook · subscription"]
        ANT["Anthropic API\nContent generation\n(pipeline only)"]
        TTS["Amazon Polly\n/ Google TTS\n(pipeline only)"]
        SES["AWS SES / SendGrid\ntransactional email"]
    end

    %% ── SYNC PATHS ───────────────────────────────────────────────────────
    MOB & WEB -->|"HTTPS — audio / static JSON\n(CDN cache hit · p95 <50 ms)"| CDN
    MOB & WEB -->|"HTTPS — API requests\np95 <100 ms content\np95 <50 ms auth"| GW

    CDN -->|"S3 origin fetch on cache miss\n(not on hot path)"| S3

    GW -->|"HTTP/1.1 — round-robin\nno health-check bypass"| API

    API -->|"aioredis pool — L2 cache\np99 <2 ms"| REDIS
    API -->|"asyncpg pool via PgBouncer\np95 <10 ms point queries"| PGPRI
    API -->|"asyncpg pool (direct)\np95 <20 ms analytics"| PGREP
    API -->|"boto3 presigned URL\n<50 ms — client fetches S3 directly"| S3

    API -.->|"JWKS fetch on L1 miss\n(L1 TTL = 1 hr; ≈0 network hops at steady state)"| AUTH0

    %% ── ASYNC PATHS (Celery) ─────────────────────────────────────────────
    API ==>|"Celery task dispatch\nfire-and-forget\n(returns 200 before task runs)"| REDIS
    BEAT ==>|"enqueue scheduled tasks\nanalytics agg · freshness check · digest emails"| REDIS
    REDIS ==>|"task queue — at-least-once delivery"| CWP & CWI

    CWP -->|"Claude API call\n8192 tokens · up to 30 s per unit"| ANT
    CWP -->|"TTS synthesis — MP3 generation"| TTS
    CWP -->|"write generated content"| S3
    CWP -->|"upsert pipeline_jobs · curriculum_units"| PGPRI

    CWI -->|"transactional email"| SES
    CWI -->|"Stripe subscription events\nidempotent by stripe_event_id"| STRIPE
    CWI -->|"suspend/unsuspend user\nasync Auth0 Management API"| AUTH0

    %% ── STRIPE WEBHOOK (inbound sync) ────────────────────────────────────
    STRIPE -->|"POST /subscription/webhook\nStripe IPs only (nginx allowlist)\nsignature verified before processing"| GW
```

---

## SLA Reference

| Path | p50 | p95 | p99 | Notes |
|---|---|---|---|---|
| Client → CDN (audio / JSON cache hit) | <5 ms | <50 ms | <100 ms | CloudFront PoP |
| Client → API (content, cache-warm) | <20 ms | <100 ms | <200 ms | L1/L2 hit; zero DB |
| Client → API (content, cold) | <30 ms | <150 ms | <300 ms | One DB query + Redis SET |
| Client → API (auth token exchange) | <40 ms | <150 ms | <300 ms | Auth0 JWKS L1 hit |
| API → Redis | <0.5 ms | <2 ms | <5 ms | aioredis pool |
| API → PostgreSQL (point query) | <2 ms | <10 ms | <20 ms | PgBouncer + index hit |
| API → PostgreSQL (analytics) | <5 ms | <20 ms | <50 ms | Read replica |
| Celery task enqueue | <1 ms | <3 ms | <10 ms | Redis LPUSH |
| Pipeline: unit generation (Claude) | — | — | <30 s/unit | Best-effort; not user-facing |
| Stripe webhook round-trip | — | <500 ms | <2 s | Stripe retry on non-2xx |

---

## Key Rules

- **FastAPI never calls Anthropic or TTS APIs.** Only Celery pipeline workers make those calls.
- **Auth0 JWKS is never fetched per-request.** L1 TTLCache holds the key set for 1 hour; verified entirely in-process.
- **`POST /subscription/webhook` is not rate-limited** but is allowlisted to Stripe CIDR ranges only in nginx. Signature verification (`construct_event`) is the first line of the handler.
- **Progress and analytics writes are fire-and-forget.** `POST /progress/answer` and `POST /analytics/lesson/end` dispatch a Celery task and return `200 OK` without waiting for a DB write.
- **Audio bytes are never proxied through FastAPI.** The API returns a presigned S3 URL; the client fetches MP3 bytes directly from S3/CDN.
- **Celery delivers at-least-once.** All task handlers must be idempotent (progress events deduplicated by `event_id`, Stripe events by `stripe_event_id`).
