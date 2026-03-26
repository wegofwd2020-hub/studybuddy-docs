# StudyBuddy OnDemand — Production Deployment Guide

How to deploy the application to a cloud environment for production use.

**Reference docs:** `OPERATIONS.md` (runbooks) · `SCALABILITY.md` (capacity planning) · `BACKEND_ARCHITECTURE.md` (caching, SLOs)

---

## Architecture Overview

```
Internet
    │
    ▼
CloudFront CDN  ──────────────────────────────────────► S3 / Content Store
    │                                                    (lesson JSON, MP3)
    ▼
Application Load Balancer (HTTPS :443)
    │
    ├──► API containers (ECS / GKE)  ──► PgBouncer ──► RDS PostgreSQL (primary)
    │         │                                              │
    │         │                                         RDS Read Replica
    │         ▼
    │       ElastiCache Redis (cluster mode)
    │
    └──► Celery Worker containers (ECS / GKE)
              ▼
            Celery Beat (scheduler — single instance)
```

**Stateless API workers** — scale horizontally without coordination.
**All state** lives in PostgreSQL and Redis — no local disk state on API containers.
**Audio/large JSON** served directly from CloudFront — never proxied through the API.

---

## Supported Cloud Platforms

The stack is cloud-agnostic (Docker containers + standard managed services). The examples below use AWS. GCP and Azure equivalents are noted where relevant.

| Component | AWS | GCP equivalent | Azure equivalent |
|---|---|---|---|
| Container orchestration | ECS Fargate | Cloud Run / GKE | ACI / AKS |
| PostgreSQL | RDS PostgreSQL 16 | Cloud SQL | Azure Database for PostgreSQL |
| Redis | ElastiCache (Valkey/Redis) | Memorystore | Azure Cache for Redis |
| Object storage | S3 | GCS | Azure Blob Storage |
| CDN | CloudFront | Cloud CDN | Azure Front Door |
| Container registry | ECR | Artifact Registry | ACR |
| Secrets | Secrets Manager | Secret Manager | Key Vault |
| Load balancer | ALB | Cloud Load Balancing | Azure App Gateway |
| Email | SendGrid (provider-agnostic) | SendGrid | SendGrid |

---

## Step 1 — Prerequisites

- AWS CLI configured (`aws configure`) with a deployment IAM role
- Docker installed locally for building images
- Domain name with DNS control (for HTTPS certificate)
- Auth0 tenant (production tier recommended)
- Stripe account (live keys)
- SendGrid account (for teacher digest emails)
- Anthropic API key (for the content pipeline — not needed by the API at runtime)

---

## Step 2 — Build and Push the Docker Image

```bash
# Authenticate to ECR
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Create the ECR repository (first time only)
aws ecr create-repository --repository-name studybuddy-api --region us-east-1

# Build and tag
docker build -t studybuddy-api:latest ./backend
docker tag studybuddy-api:latest \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com/studybuddy-api:latest

# Push
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/studybuddy-api:latest
```

The same image is used for both the API and Celery worker containers — the entrypoint command differs (set in the ECS task definition).

---

## Step 3 — Provision Managed Services

### PostgreSQL (RDS)

```bash
aws rds create-db-instance \
  --db-instance-identifier studybuddy-prod \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 16.3 \
  --master-username studybuddy \
  --master-user-password "<strong-password>" \
  --db-name studybuddy \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --backup-retention-period 30 \
  --multi-az \
  --deletion-protection \
  --no-publicly-accessible
```

**Required settings:**
- Multi-AZ: **yes** (automatic failover)
- Deletion protection: **yes**
- Automated backups: 30-day retention
- Storage encryption: **yes**
- Public access: **no** (API containers connect via VPC)

Enable WAL archiving for point-in-time recovery:

```bash
aws rds modify-db-instance \
  --db-instance-identifier studybuddy-prod \
  --enable-cloudwatch-logs-exports '["postgresql","upgrade"]' \
  --backup-retention-period 30
```

### Redis (ElastiCache)

```bash
aws elasticache create-replication-group \
  --replication-group-id studybuddy-prod-redis \
  --description "StudyBuddy production Redis" \
  --node-type cache.t3.medium \
  --num-cache-clusters 2 \
  --engine redis \
  --engine-version 7.1 \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "<strong-redis-password>" \
  --snapshot-retention-limit 7
```

**Required settings:**
- AOF persistence: set `appendonly yes` and `appendfsync everysec` — without it a Redis restart logs out all students and resets rate-limit counters
- At-rest encryption: **yes**
- In-transit encryption (TLS): **yes**
- Auth token: **yes**

### PgBouncer

Run PgBouncer as a sidecar container or dedicated ECS service between the API containers and RDS. Use **transaction pooling mode**:

```
pool_mode = transaction
max_client_conn = 200
default_pool_size = 50
```

This is critical — `asyncpg` opens many short-lived connections; without PgBouncer, RDS will hit `max_connections` under load.

### S3 + CloudFront (Content Store)

```bash
# Create the content bucket (private — CloudFront accesses it via OAC)
aws s3 mb s3://studybuddy-content-prod --region us-east-1

# Block all public access
aws s3api put-public-access-block \
  --bucket studybuddy-content-prod \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Create CloudFront distribution (Origin Access Control)
# → point to s3://studybuddy-content-prod
# → enable HTTPS only
# → set default TTL 3600s, max TTL 86400s
# → enable compression
```

Set `CONTENT_STORE_PATH=s3://studybuddy-content-prod` and configure the API with AWS credentials or an IAM role.

---

## Step 4 — Secrets Management

Store all secrets in AWS Secrets Manager. **Never pass secrets as plain environment variables in ECS task definitions.**

```bash
# Create secrets
aws secretsmanager create-secret \
  --name studybuddy/prod/jwt-secret \
  --secret-string "$(python3 -c 'import secrets; print(secrets.token_urlsafe(48))')"

aws secretsmanager create-secret \
  --name studybuddy/prod/admin-jwt-secret \
  --secret-string "$(python3 -c 'import secrets; print(secrets.token_urlsafe(48))')"

# Repeat for: postgres-password, redis-password, stripe-secret-key,
# stripe-webhook-secret, sendgrid-api-key, auth0-mgmt-client-secret,
# sentry-dsn, metrics-token
```

Reference secrets in your ECS task definition using `valueFrom`:

```json
{
  "name": "JWT_SECRET",
  "valueFrom": "arn:aws:secretsmanager:us-east-1:<account>:secret:studybuddy/prod/jwt-secret"
}
```

The ECS task execution role must have `secretsmanager:GetSecretValue` permission for these ARNs.

---

## Step 5 — Environment Variables (Production)

Set these on the ECS task definition (non-sensitive values only — secrets go via Secrets Manager):

```env
APP_ENV=production
APP_VERSION=<git-tag>
LOG_LEVEL=INFO

DATABASE_URL=postgresql://studybuddy:<password>@pgbouncer:6432/studybuddy
REDIS_URL=rediss://:@<elasticache-endpoint>:6380/0

DATABASE_POOL_MIN=5
DATABASE_POOL_MAX=20
REDIS_MAX_CONNECTIONS=10

AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_JWKS_URL=https://your-tenant.auth0.com/.well-known/jwks.json
AUTH0_STUDENT_CLIENT_ID=<client-id>
AUTH0_TEACHER_CLIENT_ID=<client-id>
AUTH0_MGMT_API_URL=https://your-tenant.auth0.com/api/v2
AUTH0_MGMT_CLIENT_ID=<client-id>

CONTENT_STORE_PATH=s3://studybuddy-content-prod
ALLOWED_ORIGINS=https://app.yourdomain.com
EMAIL_FROM=no-reply@yourdomain.com

JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=15
JWT_REFRESH_TOKEN_EXPIRE_DAYS=30
ADMIN_JWT_EXPIRE_MINUTES=60
```

> **Note:** `APP_ENV=production` disables `/api/docs` and `/api/redoc` — Swagger UI is not exposed in production.

---

## Step 6 — Run Database Migrations

Run migrations once before deploying new containers. Use a one-off ECS task:

```bash
aws ecs run-task \
  --cluster studybuddy-prod \
  --task-definition studybuddy-migrate \
  --overrides '{"containerOverrides":[{"name":"api","command":["alembic","upgrade","head"]}]}' \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<subnet-id>],securityGroups=[<sg-id>]}"
```

Or run locally against the production DB (temporarily allow access via a bastion host):

```bash
cd backend
DATABASE_URL="postgresql://studybuddy:<password>@<rds-endpoint>:5432/studybuddy" \
  alembic upgrade head
```

Always run migrations **before** deploying new API container versions to avoid schema mismatch during rolling deployment.

---

## Step 7 — ECS Task Definitions

### API Service

```json
{
  "family": "studybuddy-api",
  "cpu": "512",
  "memory": "1024",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "containerDefinitions": [{
    "name": "api",
    "image": "<account>.dkr.ecr.us-east-1.amazonaws.com/studybuddy-api:latest",
    "portMappings": [{"containerPort": 8000}],
    "command": [
      "gunicorn", "main:app",
      "-w", "4",
      "-k", "uvicorn.workers.UvicornWorker",
      "--bind", "0.0.0.0:8000",
      "--timeout", "30",
      "--access-logfile", "-",
      "--error-logfile", "-"
    ],
    "environment": [/* non-sensitive vars */],
    "secrets": [/* Secrets Manager ARNs */],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/studybuddy-api",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "api"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 15
    }
  }]
}
```

**Gunicorn workers:** set `-w` to `2 × vCPU + 1`. For a 0.5 vCPU Fargate task use `-w 2`; for 1 vCPU use `-w 4`.

### Celery Worker Service

Same image, different command:

```json
"command": [
  "celery", "-A", "src.auth.tasks.celery_app",
  "worker", "-Q", "io,default",
  "--concurrency=4",
  "--loglevel=info"
]
```

### Celery Beat (Scheduler)

Run as a **single instance** — multiple Beat instances will fire duplicate scheduled tasks:

```json
"command": [
  "celery", "-A", "src.auth.tasks.celery_app",
  "beat", "--loglevel=info"
]
```

Set the ECS service desired count to **1** and disable auto-scaling for Beat.

---

## Step 8 — Auto Scaling

### API Service

Scale on CPU utilisation:

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/studybuddy-prod/studybuddy-api \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

Minimum: **2 tasks** (availability), Maximum: **10 tasks** (cost cap).

### Celery Workers

Scale on SQS queue depth (if using SQS as Celery broker) or on CPU. Keep a minimum of **1 worker** running at all times.

---

## Step 9 — Load Balancer + HTTPS

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name studybuddy-prod-alb \
  --subnets <public-subnet-a> <public-subnet-b> \
  --security-groups <alb-sg>

# Create HTTPS listener (requires ACM certificate)
aws elbv2 create-listener \
  --load-balancer-arn <alb-arn> \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=<acm-cert-arn> \
  --default-actions Type=forward,TargetGroupArn=<api-tg-arn>

# Redirect HTTP → HTTPS
aws elbv2 create-listener \
  --load-balancer-arn <alb-arn> \
  --protocol HTTP \
  --port 80 \
  --default-actions \
    Type=redirect,RedirectConfig="{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}"
```

**ACM certificate:** request a public certificate for `api.yourdomain.com` via the AWS Console → Certificate Manager → Request certificate → DNS validation.

---

## Step 10 — DNS

Point `api.yourdomain.com` to the ALB:

```bash
# In Route 53 (or your DNS provider)
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.yourdomain.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "<alb-hosted-zone-id>",
          "DNSName": "<alb-dns-name>",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

Point `cdn.yourdomain.com` (or use the CloudFront domain directly) for the content store.

---

## Step 11 — Celery Beat Scheduled Tasks

The following tasks are registered in `backend/src/auth/tasks.py` and fire automatically once Celery Beat is running:

| Task | Schedule | Purpose |
|---|---|---|
| `update_student_streak_task` | Daily 01:00 UTC | Update streak counters in Redis |
| `promote_student_grades` | Daily 00:05 UTC | Promote students on `GRADE_PROMOTION_DATE` |
| `refresh_report_views_task` | Daily 02:00 UTC | Refresh teacher report materialized views |
| `evaluate_report_alerts_task` | Daily 06:00 UTC | Check pass-rate thresholds; insert alerts |
| `send_weekly_digest_task` | Monday 08:00 UTC | Email digest to subscribed teachers |

No additional configuration is required — these are baked into the Beat schedule in code.

---

## Step 12 — Content Pipeline

The pipeline runs **offline** (operator-run, not part of the always-on API). Run it from a machine with `ANTHROPIC_API_KEY`:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export CLAUDE_MODEL=claude-sonnet-4-6
export CONTENT_STORE_PATH=s3://studybuddy-content-prod
export TTS_API_KEY=...

# Seed default curricula into the production database
python pipeline/seed_default.py --year 2026

# Build all default content for grade 8 (English, French, Spanish)
python pipeline/build_grade.py --grade 8 --lang en,fr,es

# Regenerate a single unit (e.g., after a content fix)
python pipeline/build_unit.py \
  --curriculum-id default-2026-g8 \
  --unit G8-MATH-001 \
  --lang en \
  --force
```

After the pipeline writes new content, invalidate CloudFront so students receive the updated files:

```bash
aws cloudfront create-invalidation \
  --distribution-id <distribution-id> \
  --paths "/curricula/default-2026-g8/G8-MATH-001/*"
```

---

## Step 13 — Observability

### Health Check

```bash
curl https://api.yourdomain.com/health
# {"db": "ok", "redis": "ok", "version": "0.1.0"}
```

### Prometheus Metrics

`GET /metrics` is protected by the `METRICS_TOKEN` header:

```bash
curl -H "Authorization: Bearer <METRICS_TOKEN>" https://api.yourdomain.com/metrics
```

Configure Prometheus to scrape this endpoint. Key metrics to alert on:

| Metric | Alert threshold |
|---|---|
| `http_request_duration_seconds_p99` | > 500ms |
| `http_requests_total{status="5xx"}` | > 1% error rate over 5 min |
| `db_pool_size` approaching `DATABASE_POOL_MAX` | > 80% utilisation |
| Redis memory utilisation | > 80% of `maxmemory` |

### Sentry

Set `SENTRY_DSN` to your Sentry project DSN. The app sends 5% of traces (`traces_sample_rate=0.05`) and strips PII (email, tokens) before sending.

### CloudWatch Logs

All application logs go to stdout in structured JSON format and are captured by the ECS `awslogs` log driver. Create a CloudWatch Insights query for errors:

```
fields @timestamp, level, message, correlation_id
| filter level = "error"
| sort @timestamp desc
| limit 50
```

---

## Step 14 — Security Checklist

Before going live, verify:

- [ ] `APP_ENV=production` (disables Swagger UI)
- [ ] All secrets stored in Secrets Manager — not in environment variables or code
- [ ] `JWT_SECRET` ≠ `ADMIN_JWT_SECRET` (they must be different)
- [ ] RDS is not publicly accessible (VPC only)
- [ ] ElastiCache is not publicly accessible (VPC only)
- [ ] S3 bucket has all public access blocked; CloudFront uses Origin Access Control
- [ ] HTTPS enforced on ALB (HTTP → HTTPS redirect in place)
- [ ] ACM certificate covers `api.yourdomain.com`
- [ ] Stripe webhook endpoint registered at `https://api.yourdomain.com/api/v1/subscription/webhook`
- [ ] Stripe webhook secret stored in Secrets Manager as `STRIPE_WEBHOOK_SECRET`
- [ ] `ALLOWED_ORIGINS` contains only your production frontend domain(s)
- [ ] Redis AOF persistence enabled (`appendonly yes`, `appendfsync everysec`)
- [ ] Database deletion protection enabled on RDS
- [ ] Automated RDS backups enabled (30-day retention)
- [ ] Sentry DSN configured and error alerts set up
- [ ] Rate limits tested: auth (10 req/min/IP), content (100 req/min/student), feedback (5/hr/student)

---

## Step 15 — Deployment Workflow (Zero-Downtime)

```bash
# 1. Run migrations (before code deploy)
aws ecs run-task --cluster studybuddy-prod --task-definition studybuddy-migrate ...

# 2. Build and push new image
docker build -t studybuddy-api:<git-tag> ./backend
docker push <ecr-url>/studybuddy-api:<git-tag>

# 3. Update ECS service (rolling deployment — ALB health checks gate traffic)
aws ecs update-service \
  --cluster studybuddy-prod \
  --service studybuddy-api \
  --task-definition studybuddy-api:<new-revision> \
  --force-new-deployment

# 4. Monitor rollout
aws ecs wait services-stable --cluster studybuddy-prod --services studybuddy-api

# 5. Verify health
curl https://api.yourdomain.com/health
```

ECS performs a rolling replacement — new tasks become healthy (pass the ALB health check) before old tasks are terminated, ensuring zero downtime.

---

## Estimated Monthly Cloud Cost (AWS, us-east-1)

| Component | Spec | Est. cost/month |
|---|---|---|
| RDS PostgreSQL | db.t3.medium, Multi-AZ, 100 GB gp3 | ~$120 |
| ElastiCache Redis | cache.t3.medium, 2 nodes | ~$80 |
| ECS Fargate — API | 2–4 tasks, 0.5 vCPU / 1 GB each | ~$40–80 |
| ECS Fargate — Celery | 1–2 tasks, 0.25 vCPU / 512 MB each | ~$15 |
| ALB | 1 load balancer | ~$20 |
| CloudFront + S3 | 50 GB transfer, 10 GB storage | ~$10 |
| ECR | Image storage | ~$5 |
| CloudWatch Logs | Moderate log volume | ~$10 |
| **Total (small school)** | | **~$300–340/month** |

Costs scale primarily with RDS and ECS task count. For a single-school pilot, a single-AZ RDS instance (`db.t3.small`) and 2 Fargate tasks reduce costs to ~$150/month.
