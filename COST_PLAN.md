# StudyBuddy OnDemand — Hosting Cost & Revenue Plan

Cloud infrastructure cost projections, revenue model, and break-even analysis for a public-facing, revenue-generating deployment.

---

## Revenue Model

The application supports two subscription tracks already built into the codebase:

### Student Direct (B2C)

| Plan | Price | Notes |
|---|---|---|
| Free tier | $0 | 2 lessons unlocked; then paywall |
| Monthly | $9.99 / month | Full curriculum access, all grades |
| Annual | $99.99 / year | ~$8.33/month — 17% saving over monthly |

### School / Institutional (B2B)

| Plan | Price | Notes |
|---|---|---|
| School Starter | $299 / month | Up to 100 students, 1 curriculum upload |
| School Pro | $599 / month | Up to 500 students, unlimited uploads |
| District | $1,499 / month | Unlimited students, dedicated support |

> B2B pricing is not currently enforced in code — the subscription module supports per-student plans only. School-level billing would require a billing wrapper or Stripe metered billing per school. This is the recommended next development phase.

---

## Hosting Tiers

Three deployment tiers are defined based on user scale. All use AWS as the cloud provider (see `PRODUCTION_DEPLOYMENT.md` for provisioning steps).

---

### Tier 1 — Launch (0–500 students, 1–3 schools)

**Target:** MVP launch, pilot schools, early B2C subscribers.

| Component | Spec | Monthly Cost (USD) |
|---|---|---|
| **RDS PostgreSQL** | db.t3.small, single-AZ, 20 GB gp3 | $35 |
| **ElastiCache Redis** | cache.t3.micro, 1 node | $15 |
| **ECS Fargate — API** | 1–2 tasks, 0.5 vCPU / 1 GB each | $20 |
| **ECS Fargate — Celery worker** | 1 task, 0.25 vCPU / 512 MB | $8 |
| **ECS Fargate — Celery Beat** | 1 task, 0.25 vCPU / 512 MB | $8 |
| **PgBouncer** | 1 task, 0.25 vCPU / 256 MB | $5 |
| **ALB** | 1 load balancer, low LCU | $18 |
| **S3** | 10 GB content store | $1 |
| **CloudFront** | 50 GB transfer/month | $5 |
| **ECR** | Image storage | $2 |
| **Route 53** | 1 hosted zone | $1 |
| **CloudWatch Logs** | Low volume | $3 |
| **Auth0** | Free tier (≤7,500 MAUs) | $0 |
| **SendGrid** | Free tier (≤100 emails/day) | $0 |
| **Sentry** | Developer free tier | $0 |
| **Stripe** | 2.9% + $0.30 per transaction | Variable |
| **Domain name** | Annual / 12 | $1 |
| **TOTAL (fixed)** | | **~$122/month** |

> **Note:** No Multi-AZ or read replica at this tier. Acceptable for a pilot — add before scaling to paying schools.

---

### Tier 2 — Growth (500–5,000 students, 5–20 schools)

**Target:** Paying school contracts coming in, B2C growing, reliability matters.

| Component | Spec | Monthly Cost (USD) |
|---|---|---|
| **RDS PostgreSQL** | db.t3.medium, Multi-AZ, 100 GB gp3 | $140 |
| **RDS Read Replica** | db.t3.small (for report queries) | $50 |
| **ElastiCache Redis** | cache.t3.medium, 2 nodes (primary + replica) | $80 |
| **ECS Fargate — API** | 2–4 tasks, 0.5 vCPU / 1 GB, auto-scaling | $55 |
| **ECS Fargate — Celery worker** | 2 tasks, 0.5 vCPU / 1 GB | $25 |
| **ECS Fargate — Celery Beat** | 1 task, 0.25 vCPU / 512 MB | $8 |
| **PgBouncer** | 1 task, 0.5 vCPU / 512 MB | $10 |
| **ALB** | 1 load balancer, moderate LCU | $25 |
| **S3** | 50 GB content store | $2 |
| **CloudFront** | 500 GB transfer/month | $45 |
| **ECR** | Image storage | $3 |
| **Route 53** | 1 hosted zone + health checks | $5 |
| **CloudWatch Logs + Alarms** | Moderate volume + 10 alarms | $20 |
| **Auth0** | Essential ($23/month, 10,000 MAUs) | $23 |
| **SendGrid** | Essentials ($20/month, 50,000 emails) | $20 |
| **Sentry** | Team ($26/month) | $26 |
| **Stripe** | 2.9% + $0.30 per transaction | Variable |
| **Domain + SSL** | Route 53 + ACM (free) | $1 |
| **TOTAL (fixed)** | | **~$538/month** |

---

### Tier 3 — Scale (5,000–50,000 students, 20+ schools)

**Target:** Established SaaS product, predictable revenue, SLA commitments to schools.

| Component | Spec | Monthly Cost (USD) |
|---|---|---|
| **RDS PostgreSQL** | db.r6g.large, Multi-AZ, 500 GB gp3, 7-day PITR | $580 |
| **RDS Read Replica × 2** | db.r6g.medium (analytics + reports) | $280 |
| **ElastiCache Redis** | cache.r6g.large, cluster mode (3 shards) | $350 |
| **ECS Fargate — API** | 4–10 tasks, 1 vCPU / 2 GB, auto-scaling | $180 |
| **ECS Fargate — Celery worker** | 3–6 tasks, 0.5 vCPU / 1 GB | $80 |
| **ECS Fargate — Celery Beat** | 1 task, 0.25 vCPU / 512 MB | $8 |
| **PgBouncer** | 2 tasks (HA), 0.5 vCPU / 512 MB each | $20 |
| **ALB** | 1 load balancer, high LCU | $40 |
| **S3** | 250 GB content store | $6 |
| **CloudFront** | 5 TB transfer/month | $420 |
| **ECR** | Image storage | $5 |
| **Route 53** | 2 hosted zones + health checks | $10 |
| **CloudWatch Logs + Alarms + Dashboards** | High volume | $60 |
| **Auth0** | Professional (custom, ~$240/month, 100K MAUs) | $240 |
| **SendGrid** | Pro ($90/month, 100,000 emails) | $90 |
| **Sentry** | Business ($80/month) | $80 |
| **Stripe** | 2.9% + $0.30 (volume discounts available at this tier) | Variable |
| **AWS WAF** | API Gateway protection, ~1M requests/month | $25 |
| **AWS Secrets Manager** | ~20 secrets, monthly rotation | $8 |
| **Domain + CDN cert** | Route 53 + ACM (free) | $1 |
| **TOTAL (fixed)** | | **~$2,483/month** |

---

## Cost Summary

| Tier | Students | Schools | Fixed Infra/Month |
|---|---|---|---|
| Launch | 0–500 | 1–3 | ~$122 |
| Growth | 500–5,000 | 5–20 | ~$538 |
| Scale | 5,000–50,000 | 20+ | ~$2,483 |

---

## Break-Even Analysis

### Tier 1 — Launch (~$122/month fixed costs)

| Revenue Source | Price | Units Needed to Break Even |
|---|---|---|
| B2C Monthly subscribers | $9.99/month | **13 subscribers** |
| B2C Annual subscribers | $99.99/year | 2 annual subscribers (≈$17/month amortised) |
| School Starter contract | $299/month | **1 school contract** |

A single school contract at $299/month covers infrastructure costs **2.4×** at this tier.

### Tier 2 — Growth (~$538/month fixed costs)

| Revenue Source | Price | Units Needed to Break Even |
|---|---|---|
| B2C Monthly subscribers | $9.99/month | **54 subscribers** |
| School Starter contracts | $299/month | **2 contracts** |
| School Pro contracts | $599/month | **1 contract** |
| Mixed (2 school + 20 B2C) | — | $598 + $200 = $798/month → **profitable** |

### Tier 3 — Scale (~$2,483/month fixed costs)

| Revenue Source | Price | Units Needed to Break Even |
|---|---|---|
| B2C Monthly subscribers | $9.99/month | 249 subscribers |
| School Pro contracts | $599/month | 5 contracts |
| District contracts | $1,499/month | **2 contracts** |
| Mixed (5 school pro + 100 B2C) | — | $2,995 + $999 = $3,994/month → **profitable** |

---

## Revenue Projections

Assumes 20% monthly growth in subscribers from a standing start, combined B2C + B2B.

| Month | Students | B2C Subs | School Contracts | Gross Revenue | Infra Cost | Net |
|---|---|---|---|---|---|---|
| 1 | 50 | 10 | 0 | $100 | $122 | -$22 |
| 2 | 120 | 25 | 0 | $250 | $122 | +$128 |
| 3 | 250 | 50 | 1 (Starter) | $799 | $122 | +$677 |
| 4 | 400 | 80 | 1 | $1,099 | $122 | +$977 |
| 6 | 800 | 150 | 3 | $2,396 | $538 | +$1,858 |
| 9 | 2,000 | 350 | 6 | $5,144 | $538 | +$4,606 |
| 12 | 5,000 | 700 | 10 | $9,983 | $2,483 | +$7,500 |

> These are illustrative projections. Actual conversion from free tier (2-lesson limit) to paid depends on onboarding quality and school sales cycle.

---

## Stripe Transaction Costs

Stripe charges are variable — not included in the fixed infrastructure cost above.

| Monthly Revenue | Stripe Fee (2.9% + $0.30/tx) | Net After Stripe |
|---|---|---|
| $500 (50 × $9.99) | ~$29.50 | ~$470 |
| $2,000 | ~$87 | ~$1,913 |
| $10,000 | ~$320 | ~$9,680 |

For school invoices above $1,000/month, consider Stripe invoicing (ACH/bank transfer at 0.8%, capped at $5) to reduce payment processing fees.

---

## Anthropic API Costs (Content Pipeline)

The pipeline runs **offline** — not part of the always-on API. Cost is incurred only when generating or regenerating content.

| Grade | Units | ~Tokens per Unit | Cost per Grade (claude-sonnet-4-6) |
|---|---|---|---|
| Grade 5 | 18 units | ~8,000 tokens | ~$0.72 |
| Grades 5–12 (full pipeline) | ~151 units × 3 langs | ~8,000 tokens | ~$18 |
| Annual refresh (all grades, all languages) | Full rebuild | — | **~$18 one-time** |
| Per-school custom curriculum (20 units) | 20 units × 3 langs | ~8,000 tokens | **~$2.40** |

At current claude-sonnet-4-6 pricing (~$3/M input + $15/M output tokens), generating the entire default curriculum costs approximately **$18**. This is negligible relative to revenue.

---

## Cost Optimisation Recommendations

### Immediate (Tier 1)

1. **Use Savings Plans** — commit to 1-year Compute Savings Plan on ECS Fargate for ~30% discount on compute.
2. **RDS Reserved Instance** — 1-year reserved db.t3.small reduces RDS cost from $35 to ~$22/month.
3. **CloudFront caching** — ensure `Cache-Control: max-age=86400` headers are set on all lesson JSON responses (already implemented in the content pipeline). High cache hit rate eliminates most data transfer costs.
4. **Free tier services** — Auth0 (7,500 MAUs), SendGrid (100 emails/day), and Sentry (5,000 errors/month) are free at launch scale.

### At Growth (Tier 2)

5. **Read replica for reports** — route all teacher report queries (`/reports/*`) to the read replica to reduce load on the primary. This is an architectural change but prevents expensive queries from affecting student response times.
6. **S3 Intelligent-Tiering** — automatically moves infrequently accessed content to cheaper storage classes. Content for older school years is rarely accessed after ~3 months.
7. **Celery worker spot instances** — pipeline and non-time-critical Celery tasks can run on Spot/Fargate Spot at ~70% discount.

### At Scale (Tier 3)

8. **CloudFront origin shield** — adds a regional cache layer between edge locations and S3, reducing origin requests and S3 data transfer costs.
9. **RDS Aurora Serverless v2** — for workloads with variable traffic (school term vs summer break), Aurora auto-scales ACUs and can be significantly cheaper than a fixed instance during low periods.
10. **Auth0 volume discount** — negotiate directly with Auth0 at 50,000+ MAUs; enterprise contracts offer 40–60% discount vs list price.
11. **Stripe volume pricing** — contact Stripe for custom pricing at >$50K/month in transaction volume; interchange-plus pricing typically reduces effective rate to ~1.5–2%.

---

## Three-Year Total Cost of Ownership

| Year | Avg Monthly Infra | Annual Infra | Avg Monthly Revenue | Annual Revenue | Annual Profit |
|---|---|---|---|---|---|
| Year 1 | $300 | $3,600 | $2,000 | $24,000 | **+$20,400** |
| Year 2 | $800 | $9,600 | $8,000 | $96,000 | **+$86,400** |
| Year 3 | $2,000 | $24,000 | $20,000 | $240,000 | **+$216,000** |

> Year 1 includes an initial pipeline run cost of ~$18 and assumes 3 months of pre-revenue onboarding. Infrastructure costs are weighted averages across tier transitions.

---

## Recommended Launch Sequence

| Phase | Action | Infra Tier | Target |
|---|---|---|---|
| **Now** | Deploy Tier 1 on AWS | Tier 1 (~$122/mo) | 1 pilot school, free access |
| **Month 1–2** | Onboard pilot school, gather feedback | Tier 1 | Validate product-market fit |
| **Month 3** | Enable Stripe (live keys), B2C paywall | Tier 1 | First paying subscribers |
| **Month 3–4** | Sign first paid school contract ($299/mo) | Tier 1 | Cash flow positive |
| **Month 6** | 3+ school contracts, 100+ B2C subscribers | Tier 2 | Upgrade to Growth tier |
| **Month 12** | 10+ schools, 700+ B2C subscribers | Tier 2→3 | Consider Series A / growth capital |
