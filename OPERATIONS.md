# StudyBuddy OnDemand — Operations

Runbooks, incident response, disaster recovery, and deployment procedures.
This document is for on-call engineers and the deployment team.

**Companion docs:** [ARCHITECTURE.md](ARCHITECTURE.md) · [SCALABILITY.md](SCALABILITY.md) · [BACKEND_ARCHITECTURE.md](BACKEND_ARCHITECTURE.md)

---

## Disaster Recovery

### RTO / RPO Targets

| Component | RPO (max data loss) | RTO (max recovery time) |
|---|---|---|
| PostgreSQL — replica promotion | 5 minutes (WAL streaming lag) | 15 minutes |
| PostgreSQL — point-in-time restore | 24 hours (last backup) | 2 hours |
| Redis — AOF replay | 1 second (`appendfsync everysec`) | 5 minutes |
| Redis — full loss (AOF unavailable) | All cached state lost | 5 minutes (cold start) |
| Content Store | 24 hours (daily snapshot) | 30 minutes |
| API workers | 0 (stateless) | 5 minutes |
| **Full service — catastrophic** | 24 hours | 4 hours |

**SLA target:** 99.9% uptime = ≤ 8.7 hours unplanned downtime per year.

---

### Backup Schedule

| Asset | Method | Frequency | Retention | Location |
|---|---|---|---|---|
| PostgreSQL | Automated snapshot (RDS or pg_dump) | Daily at 02:00 UTC | 30 days | Separate bucket / path from production |
| PostgreSQL WAL | Continuous WAL archiving | Continuous | 7 days | Separate bucket / path |
| Redis | RDB snapshot + AOF | RDB: daily; AOF: continuous | 7 days | Primary host + remote replica |
| Content Store | Incremental snapshot | Daily at 03:00 UTC | 30 days | Cross-region copy |
| Schema state | Alembic revision history | Every commit | Indefinite | Git repository |
| Secrets | Secrets manager versioning | On every change | Full version history | Secrets manager |

**Backup test schedule:** restore a PostgreSQL backup to a staging instance and run `alembic current` monthly. Record the result in the incident log.

---

### PostgreSQL Restore Procedure

#### Replica Promotion (Primary Unreachable)

```bash
# 1. Confirm primary is unreachable
psql $DATABASE_URL -c "SELECT 1"

# 2. Promote the read replica
#    On RDS: use the console "Promote to standalone DB instance"
#    On self-managed: pg_ctl promote -D /var/lib/postgresql/data

# 3. Update DATABASE_URL to point to the promoted replica
#    (update secrets manager / environment variable)

# 4. Rolling restart of API workers (zero-downtime if load balancer is healthy)
docker compose up -d --no-deps --scale api=4 api

# 5. Verify
curl -f $BACKEND_URL/health   # expect {"db": "ok", ...}

# 6. Spin up a new replica from the promoted primary within 24 hours
# 7. File SEV-1 incident report
```

#### Point-in-Time Restore (Data Loss Scenario)

1. Restore the latest daily snapshot to a **new** instance (never overwrite production)
2. Apply WAL archive from snapshot time to desired recovery point
3. Validate schema: `alembic current` — must match latest revision
4. Smoke-test with a read-only test account before switching traffic
5. Update `DATABASE_URL` and rolling restart API workers
6. Announce the data-loss window to affected schools via email
7. Estimated duration: **2 hours** for a 100 GB database

---

### Redis Restore Procedure

#### Restart with AOF Intact (Common Case)

```bash
# Redis restarts and replays AOF automatically
docker restart studybuddy-redis

# Verify — all sessions and cache restored
redis-cli -h $REDIS_HOST ping        # PONG
curl -f $BACKEND_URL/health          # {"redis": "ok", ...}

# Monitor eviction rate for 30 minutes post-restart
```

Expected downtime: **< 2 minutes** for AOF replay on typical dataset.

#### Cold Start (AOF Unavailable — Data Loss)

**Impact:** all refresh tokens invalidated (students logged out), all rate-limit counters reset, entitlement L2 cache empty.

```bash
# Start Redis with empty dataset
docker start studybuddy-redis

# Warm entitlement cache from DB (runs for ~5 minutes)
celery -A tasks call core.tasks.warm_entitlement_cache

# Monitor: DB query rate will spike 10–15× for ~10 minutes while L2 warms
# Students will need to re-authenticate — this is expected; communicate via status page
```

---

### Content Store Restore Procedure

```bash
# 1. Restore daily snapshot to a staging path
aws s3 sync s3://studybuddy-backup/content/YYYY-MM-DD/ $CONTENT_STORE_PATH/

# 2. Verify file integrity for a known curriculum
python pipeline/verify_store.py --curriculum-id default-2026-g8

# 3. Issue CDN invalidation for affected paths
python pipeline/invalidate_cdn.py --prefix curricula/default-2026-g8/

# 4. Smoke-test
curl -H "Authorization: Bearer $TEST_JWT" $BACKEND_URL/content/G8-MATH-001/lesson
```

---

## Incident Management

### Severity Definitions

| Severity | Criteria | Response time | Examples |
|---|---|---|---|
| **SEV-1** | Full service unavailable; critical data loss | Immediate; wake on-call | Backend API down, PostgreSQL unreachable, Auth0 JWKS failure, mass student data loss |
| **SEV-2** | Critical feature unavailable; majority of users impacted | 15 minutes | Content not serving, Stripe payments down, all quiz sessions failing |
| **SEV-3** | Significant degradation; subset of users impacted | 1 hour | Content SLO breach, pipeline stuck > 2 hours, Redis eviction storm |
| **SEV-4** | Minor issue; non-critical feature broken; isolated impact | Next business day | Reporting dashboard slow, single unit content missing, feedback endpoint error |

### Escalation Policy

```
Alert fires (PagerDuty / Slack)
      │
      ▼
On-call engineer acknowledges within:
  SEV-1 → 5 min  |  SEV-2 → 15 min  |  SEV-3 → 60 min
      │
      ├── Resolved within 30 min (SEV-1) / 2 hr (SEV-2) → close incident
      │
      └── Not resolved →
            Escalate to senior engineer
                  │
                  └── Not resolved within 1 hr (SEV-1) / 4 hr (SEV-2) →
                        Escalate to engineering lead; notify stakeholders
```

**Communications:**
- Status page updated within 10 minutes of SEV-1/SEV-2 detection
- Affected schools notified by email for outages > 30 minutes
- All students see a maintenance banner via `GET /app/version` `maintenance_message` field
- Post-incident review: SEV-1 within 48 hours; SEV-2 within 1 week

---

### Incident Runbooks

#### RB-001 — Backend API Down

**Symptoms:** `GET /health` returns non-200; `BackendDown` Alertmanager alert.

```bash
# 1. Check container / pod status
docker ps | grep studybuddy-api
# or: kubectl get pods -n studybuddy

# 2. Check recent logs
docker logs studybuddy-api --tail 100 | grep -E "ERROR|CRITICAL|Exception"

# 3. Common causes:
#    OOM killed    → increase memory limit; restart container
#    Port conflict  → lsof -i :8000; kill conflicting process
#    Lifespan fail  → DB or Redis unreachable; fix underlying service first (see RB-002/003)
#    Bad deploy     → roll back (see Rollback Procedure)

# 4. Verify recovery
curl -f $BACKEND_URL/health
```

---

#### RB-002 — Database Connection Pool Exhausted

**Symptoms:** `"remaining connection slots are reserved"` in logs; `sb_db_pool_connections{state="waiting"}` > 5; all endpoints slow.

```bash
# 1. Check PgBouncer pool state
psql postgresql://pgbouncer:5432/pgbouncer -c "SHOW POOLS;"

# 2. Check total PostgreSQL connections
psql $DATABASE_URL -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# 3. Kill long-running idle connections
psql $DATABASE_URL -c "
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE state = 'idle'
    AND query_start < NOW() - INTERVAL '10 minutes';"

# 4. If pool is healthy but API workers are exhausted:
#    Scale API workers horizontally (add one worker container)
#    OR reduce asyncpg pool_max_size if over-provisioned
```

Root cause is almost always: a missing `await` on an asyncpg call, a long-running transaction, or a sudden traffic spike.

---

#### RB-003 — Redis Unreachable

**Symptoms:** `GET /health` returns `{"redis": "error"}`; authentication failures spike; rate limiting not enforcing.

```bash
# 1. Ping Redis
redis-cli -h $REDIS_HOST ping

# 2. Check memory pressure
redis-cli -h $REDIS_HOST info memory | grep used_memory_human

# 3. If crashed → follow Redis Restore Procedure above

# Degraded mode behaviour (Redis down but not restored):
# - L1 in-process TTLCache continues serving content (per-worker)
# - New student logins FAIL (cannot issue/verify refresh tokens)
# - Rate limiting is effectively disabled (all requests pass through)
# - Communicate expected degradation via status page
```

---

#### RB-004 — Content Not Serving (404 / 503)

**Symptoms:** `GET /content/{unit_id}/lesson` returns 404 or 503; students report blank lesson screens.

```bash
# 1. Check Content Store files exist
ls $CONTENT_STORE_PATH/curricula/{curriculum_id}/{unit_id}/lesson_en.json

# 2. Check subject version status in DB
psql $DATABASE_URL -c "
  SELECT status FROM content_subject_versions
  WHERE curriculum_id = '{curriculum_id}' AND subject = '{subject}'
  ORDER BY version_number DESC LIMIT 1;"
# Must be 'published' — if 'approved' or earlier, admin must publish

# 3. Check for active content block
psql $DATABASE_URL -c "
  SELECT * FROM content_blocks
  WHERE curriculum_id = '{curriculum_id}'
    AND lifted_at IS NULL;"

# 4. Check L2 Redis content key
redis-cli get "content:{curriculum_id}:{unit_id}:en"

# 5. If files exist but CDN returns 404 → issue CDN invalidation
python pipeline/invalidate_cdn.py --prefix curricula/{curriculum_id}/{unit_id}/
```

---

#### RB-005 — Auth0 JWKS Unreachable (Students Cannot Log In)

**Symptoms:** `POST /auth/exchange` returns 503; `AUTH0_JWKS_FAILURE` counter rising.

```bash
# 1. Check Auth0 status
# https://status.auth0.com

# 2. Check L1 JWKS cache TTL (24-hour cache — existing sessions still work)
# Existing JWTs remain valid; only NEW logins fail during Auth0 outage

# 3. If Auth0 is degraded long-term:
#    Extend JWKS_CACHE_TTL_HOURS in config temporarily
#    Restart workers after Auth0 recovers to force JWKS re-fetch

# DO NOT disable JWKS verification as a workaround — this disables all auth security
```

---

#### RB-006 — Stripe Webhook Failures

**Symptoms:** `StripeWebhookErrors` alert; subscriptions not activating after payment.

```bash
# 1. Check Stripe Dashboard → Webhooks → recent delivery attempts for error details

# 2. Check stripe_events table for recent entries
psql $DATABASE_URL -c "
  SELECT stripe_event_id, event_type, outcome, processed_at
  FROM stripe_events ORDER BY processed_at DESC LIMIT 20;"

# 3. Common causes:
#    STRIPE_WEBHOOK_SECRET mismatch → verify env var matches Stripe dashboard secret
#    Endpoint returning 500         → check app logs for the specific event type

# Stripe retries webhooks for 72 hours — fix the endpoint; Stripe replays automatically
# Do NOT manually activate subscriptions until the endpoint is verified healthy
```

---

#### RB-007 — Content Pipeline Stuck

**Symptoms:** `CeleryPipelineBacklog` alert (queue > 50 for > 10 min); `GET /admin/pipeline/status` shows a job running for > 2 hours.

```bash
# 1. Check active Celery tasks
celery -A tasks inspect active

# 2. Check for Anthropic API rate limits in logs
docker logs studybuddy-celery-pipeline | grep -i "rate.limit\|RateLimitError"

# 3. Check Anthropic status
# https://status.anthropic.com

# 4. If worker is stuck (not just rate-limited):
celery -A tasks inspect revoke {task_id} --terminate
# Restart the pipeline job from the admin dashboard

# 5. If rate-limited: pipeline backs off automatically — no manual action needed
#    Monitor: job should resume within 60 seconds of rate limit lifting
```

---

### Post-Incident Review

Every SEV-1 and SEV-2 incident requires a blameless post-incident review:

| Field | Contents |
|---|---|
| **Timeline** | Start, detection, mitigation steps, resolution (with timestamps) |
| **Root cause** | Technical cause and contributing process factors |
| **Impact** | Users affected, data affected, duration, revenue impact |
| **What went well** | Detection speed, escalation, communication |
| **What went poorly** | Gaps in monitoring, runbook missing, slow response |
| **Action items** | Owner, due date, severity of fix |

Reviews are stored in `docs/incidents/YYYY-MM-DD-title.md`.

---

## Deployment Strategy

### Environments

| Environment | Purpose | DB | Redis | Auth0 Tenant | Student PII |
|---|---|---|---|---|---|
| `dev` | Local development | Local Docker | Local Docker | Dev tenant | None — synthetic data only |
| `staging` | Pre-production testing | Separate staging DB | Separate staging Redis | Staging tenant | None — synthetic data only |
| `production` | Live service | Production DB (PgBouncer) | Production Redis Cluster | Production tenant | Real |

**Rule:** staging and dev environments must never contain real student PII.

---

### Blue/Green Deployment

```
Blue (current live)           Green (new version)
       │                             │
 100% traffic               0% traffic (idle)
       │
       ├── 1. Build and push new Docker image (green tag)
       ├── 2. Deploy green containers (same count as blue); do NOT receive traffic yet
       ├── 3. Run Alembic migrations (must be backward-compatible — both blue and green run together briefly)
       ├── 4. Run post-deploy smoke tests against green (via internal health route)
       ├── 5. Shift 100% traffic to green (load balancer update)
       ├── 6. Monitor for 10 minutes:
       │       error rate normal  → decommission blue
       │       error rate elevated → shift traffic back to blue (rollback; < 30 seconds)
       └── 7. Decommission blue containers
```

---

### Canary Release

Use for high-risk changes: authentication logic, entitlement logic, payment handling, DB schema changes.

| Step | Traffic to canary | Observation window | Abort if |
|---|---|---|---|
| 1 | 5% | 30 minutes | Error rate > 2× baseline OR p95 latency > 2× baseline |
| 2 | 25% | 30 minutes | Same |
| 3 | 50% | 30 minutes | Same |
| 4 | 100% | 10 minutes | Same |

If any step fails: revert canary traffic to 0%; file SEV-3 incident.

---

### Rollback Procedure

```bash
# Identify last known-good image tag
docker images studybuddy-backend --format "{{.Tag}}" | head -5

# Shift load balancer back to blue (or previous tag in k8s rollout)
kubectl rollout undo deployment/studybuddy-api
# or: update target group to prior task definition (ECS)

# Verify
curl -f $BACKEND_URL/health

# Alembic: backward-compatible migrations do NOT need rollback
# If a non-backward-compatible migration was applied:
alembic downgrade -1  # then redeploy previous image
```

---

### Database Migration Rules

All Alembic migrations applied during the deployment window must be **backward-compatible** — both the old and new application version must work against the migrated schema simultaneously.

| Operation | Safe? | Notes |
|---|---|---|
| Add nullable column | ✓ Safe | |
| Add new table | ✓ Safe | |
| Add index | ✓ Safe | Use `CREATE INDEX CONCURRENTLY` to avoid table lock |
| Rename a column | ✗ Breaking | Use two-phase: add new column → dual-write → drop old column |
| Drop a column | ✗ Breaking | Two-phase: stop reading in code first → deploy → then drop |
| Add NOT NULL to existing column | ✗ Breaking | Add as nullable, backfill, then add constraint |
| Change column type | ✗ Breaking | Two-phase |

---

### Pre-Deploy Checklist

- [ ] Alembic migration reviewed and backward-compatible
- [ ] Staging deployment completed successfully
- [ ] Staging smoke tests passing (auth, content, quiz, progress, health)
- [ ] Load test passed on staging (required if touching content or auth hot path)
- [ ] On-call engineer notified of deployment window
- [ ] Rollback plan confirmed (which image tag; expected downtime if rollback triggered)
- [ ] Feature flags set correctly (`REVIEW_AUTO_APPROVE=false` in production)
- [ ] `CHANGELOG` entry written

---

### Post-Deploy Smoke Tests

Run within 5 minutes of every production deployment:

```bash
# Health check
curl -f $BACKEND_URL/health

# Auth exchange (dedicated smoke-test Auth0 account)
TOKEN=$(curl -s -X POST $BACKEND_URL/auth/exchange \
  -H "Content-Type: application/json" \
  -d "{\"id_token\": \"$SMOKE_TEST_ID_TOKEN\"}" | jq -r .token)

# Curriculum list
curl -sf -H "Authorization: Bearer $TOKEN" $BACKEND_URL/curriculum | jq '.[] | .grade'

# Content fetch (known published unit)
curl -sf -H "Authorization: Bearer $TOKEN" \
  $BACKEND_URL/content/G8-MATH-001/lesson | jq '.synopsis | length'

# Metrics endpoint
curl -sf -H "Authorization: Bearer $METRICS_TOKEN" \
  $BACKEND_URL/metrics | grep -c "^sb_"
```

All commands must exit 0. Any failure → rollback immediately.
