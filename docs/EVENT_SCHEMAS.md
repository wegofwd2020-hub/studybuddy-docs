# Diagram 10 — Event Schemas

> Event catalogue, envelope format, delivery guarantees, and dead-letter queue.
> Audience: Developers, SRE.
> Last updated: 2026-04-05.

---

## Event Architecture Overview

```
                  Application Code
                        │
                        ▼
              ┌─────────────────────┐
              │  emit_event()       │  ← single call point for all events
              │  backend/src/       │    (src/core/events.py)
              │  core/events.py     │
              └──────────┬──────────┘
                         │
             ┌───────────┴────────────┐
             │                        │
             ▼                        ▼
  ┌──────────────────┐    ┌──────────────────────┐
  │  structlog       │    │  Prometheus counter   │
  │  Structured JSON │    │  sb_{category}_{type} │
  │  → stdout        │    │  _total{labels}       │
  │  → log aggregator│    │  Exposed on /metrics  │
  │  (CloudWatch /   │    │  scrape by Prometheus │
  │   Loki)          │    └──────────────────────┘
  └──────────────────┘

  For security-critical events:
                        │
                        ▼
              ┌─────────────────────┐
              │  write_audit_log()  │  ← append-only PostgreSQL table
              │  (called alongside  │    (audit_log — never updated/deleted)
              │  emit_event for     │
              │  admin + auth acts) │
              └─────────────────────┘

  For async background tasks (Celery):
  FastAPI ──LPUSH──► Redis (broker) ──BLPOP──► Celery Worker
                                                    │
                                         On failure after 3 retries:
                                                    │
                                                    ▼
                                         Redis list: celery_dlq
                                         + Prometheus: sb_celery_dlq_total
                                         + Slack alert: #studybuddy-alerts
```

---

## Structured Log Envelope

Every `emit_event()` call produces a JSON log line with this envelope:

```json
{
  "timestamp":      "2026-04-05T14:30:00.123Z",
  "level":          "INFO",
  "service":        "studybuddy-api",
  "component":      "content",
  "event_category": "content",
  "event_type":     "lesson_served",
  "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "request_id":     "f9e8d7c6-b5a4-3210-fedc-ba9876543210",
  "tenant_id":      "school-uuid-or-null",
  "actor_id":       "student-uuid",

  ...event-specific fields...
}
```

**Required fields on every event:** `timestamp`, `level`, `service`, `component`, `event_category`, `event_type`, `correlation_id`.

**Never logged:** passwords, JWT tokens, Stripe secret keys, Auth0 secrets, full email addresses (except in `auth` context at INFO level), card data, raw request bodies.

---

## Event Catalogue

### `auth` — Authentication & Identity

| Event Type | Level | Trigger | Key Fields |
|---|---|---|---|
| `exchange_success` | INFO | Student/teacher Auth0 → internal JWT exchange | `student_id` or `teacher_id`, `locale`, `grade` |
| `exchange_failure` | WARNING | Invalid/expired Auth0 id_token | `error_reason`, `ip_address` |
| `token_refresh` | DEBUG | Refresh token used to get new access token | `student_id`, `token_ttl_remaining_s` |
| `forgot_password_triggered` | INFO | `POST /auth/forgot-password` called | (no email logged — always returns 200) |
| `account_locked` | WARNING | 5 consecutive login failures | `ip_address`, `lockout_duration_min` |
| `account_suspended_access` | WARNING | Suspended student/teacher attempts access | `actor_id`, `endpoint` |
| `admin_login` | INFO | Admin bcrypt auth success | `admin_id`, `role`, `ip_address` |
| `admin_login_failed` | WARNING | Admin bcrypt auth failure | `ip_address`, `attempt_count` |

**Schema example — `exchange_success`:**
```json
{
  "event_category": "auth",
  "event_type":     "exchange_success",
  "student_id":     "d1000000-0000-0000-0000-000000000001",
  "grade":          8,
  "locale":         "en",
  "school_id":      "d3000000-0000-0000-0000-000000000001"
}
```

---

### `content` — Content Serving

| Event Type | Level | Trigger | Key Fields |
|---|---|---|---|
| `lesson_served` | INFO | `GET /content/{unit_id}/lesson` returns 200 | `unit_id`, `curriculum_id`, `locale`, `cache_layer`, `duration_ms` |
| `quiz_served` | INFO | `GET /content/{unit_id}/quiz` returns 200 | `unit_id`, `quiz_set_number`, `locale`, `cache_layer` |
| `audio_url_issued` | INFO | `GET /content/{unit_id}/lesson/audio` returns presigned URL | `unit_id`, `presigned_url_ttl_s` |
| `cache_hit` | DEBUG | L1 or L2 cache satisfied the request | `cache_layer`, `key_prefix` |
| `cache_miss` | DEBUG | Fell through to DB/Content Store | `cache_layer`, `key_prefix` |
| `content_not_found` | WARNING | Unit content files absent on disk | `unit_id`, `curriculum_id`, `locale` |
| `entitlement_denied` | INFO | Student lacks subscription to access content | `student_id`, `plan`, `lessons_accessed` |

---

### `progress` — Learning Progress

| Event Type | Level | Trigger | Key Fields |
|---|---|---|---|
| `session_open` | INFO | `POST /progress/session` | `session_id`, `student_id`, `unit_id`, `attempt_number` |
| `answer_recorded` | DEBUG | `POST /progress/answer` (fire-and-forget) | `session_id`, `question_id`, `correct`, `ms_taken` |
| `session_end` | INFO | `POST /progress/session/end` | `session_id`, `score`, `duration_s`, `completed` |
| `sync_flush_start` | INFO | Mobile sync queue upload begins | `student_id`, `queue_depth` |
| `sync_flush_complete` | INFO | Mobile sync queue flushed | `student_id`, `events_flushed`, `duration_ms` |

---

### `subscription` — Payments & Entitlements

| Event Type | Level | Trigger | Key Fields |
|---|---|---|---|
| `checkout_created` | INFO | `POST /subscription/checkout` | `student_id`, `plan`, `stripe_session_id` |
| `payment_succeeded` | INFO | Stripe `invoice.payment_succeeded` webhook | `student_id`, `plan`, `amount_usd`, `stripe_event_id` |
| `payment_failed` | WARNING | Stripe `invoice.payment_failed` webhook | `student_id`, `plan`, `stripe_event_id`, `failure_reason` |
| `subscription_cancelled` | INFO | Stripe cancel or `DELETE /subscription` | `student_id`, `cancel_at_period_end` |
| `grace_period_started` | WARNING | Payment failure, grace period begins | `student_id`, `grace_until` |
| `entitlement_granted` | INFO | Subscription active → Redis ent:{id} written | `student_id`, `plan`, `valid_until` |
| `entitlement_revoked` | INFO | Subscription expired or suspended | `student_id`, `reason` |

---

### `pipeline` — Content Generation

| Event Type | Level | Trigger | Key Fields |
|---|---|---|---|
| `build_started` | INFO | `POST /admin/pipeline/trigger` | `job_id`, `curriculum_id`, `grade`, `langs` |
| `unit_generated` | INFO | Single unit build success | `job_id`, `unit_id`, `lang`, `tokens_used`, `duration_s` |
| `unit_skipped` | DEBUG | meta.json matches content_version + files exist | `job_id`, `unit_id`, `lang` |
| `unit_failed` | ERROR | Claude response invalid or schema error after 3 retries | `job_id`, `unit_id`, `lang`, `error`, `retry_count` |
| `build_complete` | INFO | All units processed | `job_id`, `built`, `skipped`, `failed`, `duration_s`, `total_tokens`, `cost_usd` |
| `spend_cap_abort` | WARNING | `tokens_used × TOKEN_COST_USD > MAX_PIPELINE_COST_USD` | `job_id`, `spent_usd`, `cap_usd` |
| `alex_warning` | WARNING | AlexJS found potentially insensitive language | `unit_id`, `lang`, `rule_id`, `text_excerpt` |

---

### `security` — Security Events

| Event Type | Level | Trigger | Key Fields |
|---|---|---|---|
| `rate_limit_hit` | WARNING | nginx or FastAPI rate limit exceeded | `ip_address`, `endpoint`, `limit_per_min` |
| `invalid_token` | WARNING | JWT signature invalid or expired | `ip_address`, `endpoint`, `error_reason` |
| `suspended_account_access` | WARNING | Suspended user attempts any request | `actor_id`, `actor_type`, `endpoint` |
| `brute_force_detected` | WARNING | 3+ consecutive auth failures from same IP | `ip_address`, `attempt_count` |
| `account_lockout` | WARNING | Account locked after 5 failed attempts | `ip_address`, `lockout_duration_min` |
| `webhook_signature_invalid` | ERROR | Stripe webhook fails `construct_event` | `ip_address`, `stripe_event_type` |

---

### `admin` — Administrative Actions

| Event Type | Level | Trigger | Key Fields |
|---|---|---|---|
| `account_status_changed` | INFO | Admin suspends/reactivates student/teacher/school | `actor_id`, `target_type`, `target_id`, `old_status`, `new_status` |
| `curriculum_activated` | INFO | School curriculum goes live | `actor_id`, `curriculum_id`, `school_id`, `grade` |
| `gdpr_deletion_scheduled` | INFO | `DELETE /auth/account` or admin delete | `target_id`, `scheduled_at`, `delete_at` |
| `content_version_approved` | INFO | Reviewer approves a content version | `actor_id`, `version_id`, `curriculum_id`, `subject` |
| `content_version_published` | INFO | Admin publishes a content version | `actor_id`, `version_id` |
| `content_blocked` | WARNING | Unit or subject blocked for a school | `actor_id`, `curriculum_id`, `subject`, `school_id` |

---

### `system` — Infrastructure Events

| Event Type | Level | Trigger | Key Fields |
|---|---|---|---|
| `startup` | INFO | FastAPI lifespan startup | `db_pool_size`, `redis_connected`, `worker_count` |
| `shutdown` | INFO | FastAPI lifespan shutdown | — |
| `health_check_failed` | ERROR | `/readyz` dependency check fails | `failed_dependency`, `error` |
| `db_pool_exhausted` | ERROR | asyncpg connection pool at max_size | `pool_size`, `queue_depth` |
| `redis_eviction_spike` | WARNING | Redis eviction rate > threshold | `evicted_keys_per_s` |
| `circuit_breaker_open` | ERROR | Circuit breaker trips for a dependency | `dependency`, `failure_rate`, `window_s` |

---

## Delivery Guarantees

| Channel | Guarantee | Notes |
|---|---|---|
| `structlog` → stdout | Best-effort (log aggregator handles durability) | Use correlation_id to reconstruct request flow |
| Prometheus counter | Best-effort (counter lost on worker restart) | For alerting and dashboards only — not audit |
| `audit_log` PostgreSQL table | ACID — exactly-once append | Used for compliance (FERPA, SOC-2); never delete rows |
| Celery task (Redis broker) | At-least-once | Task handlers must be idempotent |
| Stripe webhook events | At-least-once (Stripe retries on non-2xx) | Deduplicated by `stripe_event_id` in `stripe_events` table |

---

## Dead-Letter Queue (DLQ)

Celery tasks that fail after `max_retries=3` are routed to the DLQ:

```
Redis list key: celery_dlq
Item format:
{
  "task_name":    "src.progress.tasks.flush_progress_event",
  "args":         [...],
  "kwargs":       {...},
  "failed_at":    "2026-04-05T14:30:00Z",
  "error":        "asyncpg.PostgresConnectionError: ...",
  "retry_count":  3,
  "correlation_id": "..."
}
```

**DLQ alerting:**
- Prometheus counter `sb_celery_dlq_total{task_name}` incremented on each DLQ write
- CloudWatch alarm: `sb_celery_dlq_total > 5 in 10 min` → SNS → PagerDuty

**DLQ replay:**
```bash
# Inspect DLQ
docker compose exec redis redis-cli LRANGE celery_dlq 0 -1

# Replay a specific failed task manually
docker compose exec api python -c "
from celery import Celery
app = Celery()
app.send_task('src.progress.tasks.flush_progress_event', args=[...])
"

# Clear DLQ (after confirmed all items are stale or replayed)
docker compose exec redis redis-cli DEL celery_dlq
```

---

## Prometheus Metrics Reference

| Metric | Type | Labels | Description |
|---|---|---|---|
| `sb_http_requests_total` | Counter | `method`, `endpoint`, `status` | All HTTP request outcomes |
| `sb_http_request_duration_seconds` | Histogram | `endpoint` | Latency distribution (p50/p95/p99) |
| `sb_cache_hits_total` | Counter | `layer`, `key_prefix` | L1/L2 cache hits |
| `sb_cache_misses_total` | Counter | `layer`, `key_prefix` | L1/L2 cache misses |
| `sb_auth_failures_total` | Counter | `reason` | Auth failure breakdown |
| `sb_celery_tasks_total` | Counter | `task_name`, `status` | Celery task outcomes |
| `sb_celery_dlq_total` | Counter | `task_name` | DLQ writes (alert on spike) |
| `sb_pipeline_units_total` | Counter | `status`, `lang` | Pipeline build outcomes |
| `sb_pipeline_tokens_used_total` | Counter | `curriculum_id` | Token consumption (cost tracking) |
| `sb_subscriptions_active` | Gauge | `plan` | Live subscription count per plan |
| `sb_db_pool_available` | Gauge | `pool` | asyncpg pool free connections |
