# StudyBuddy OnDemand — Glossary of Acronyms & Terms

Quick reference for all abbreviations used across the project documentation.
Entries are grouped by domain, then sorted alphabetically within each group.

---

## Compliance & Legal

| Acronym / Term | Full Form | Notes |
|---|---|---|
| **COPPA** | Children's Online Privacy Protection Act | US federal law; applies to students under 13. Requires verifiable parental consent before data collection. |
| **FERPA** | Family Educational Rights and Privacy Act | US federal law; applies to schools receiving federal funding. Student progress records, quiz scores, and lesson-view history are educational records under FERPA. |
| **GDPR** | General Data Protection Regulation | EU data privacy law. Governs data minimisation, right to erasure (30-day anonymisation schedule), and storage limitation. |
| **PCI** | Payment Card Industry | PCI DSS (Data Security Standard) defines card-data handling requirements. Using Stripe hosted checkout keeps StudyBuddy out of PCI scope. |
| **PII** | Personally Identifiable Information | Data that can identify an individual. Collected data is limited to name, email, grade, locale. Never logged, never in dev/test environments. |
| **WCAG** | Web Content Accessibility Guidelines | Published by W3C. Target: WCAG 2.1 Level AA. Requires 4.5:1 colour contrast, 44×44 dp touch targets, screen reader support, and audio text alternatives. |

---

## Authentication & Security

| Acronym / Term | Full Form | Notes |
|---|---|---|
| **bcrypt** | — (proper noun) | Password hashing algorithm. Used for internal admin accounts. Always run in a thread-pool executor — never on the async event loop. |
| **CORS** | Cross-Origin Resource Sharing | Browser security mechanism. Configured via `ALLOWED_ORIGINS` env var in the backend. |
| **JWKS** | JSON Web Key Set | Public key set published by Auth0 at `/.well-known/jwks.json`. Backend caches it in L1 TTLCache (24-hr TTL) to verify student/teacher JWTs. |
| **JWT** | JSON Web Token | Signed token used for stateless authentication. Students and teachers use HS256 internal JWTs issued after Auth0 exchange. Admin team uses a separate ADMIN_JWT_SECRET. |
| **M2M** | Machine-to-Machine | Auth0 application type used by the backend to call the Auth0 Management API (for suspension sync and GDPR deletion). |
| **MFA** | Multi-Factor Authentication | Admin accounts may require MFA (TOTP) in Phase 7+. Not enforced in Phase 1. |
| **PKCE** | Proof Key for Code Exchange | OAuth 2.0 extension for public (mobile) clients. Prevents authorisation code interception. Used in the Kivy mobile Auth0 login flow. |
| **RBAC** | Role-Based Access Control | Access control model. Implemented as a static `ROLE_PERMISSIONS` dict in `core/permissions.py`. Seven roles; 15 permissions; enforced via `require_permission()` FastAPI dependency. |
| **SHA** | Secure Hash Algorithm | SHA-256 used to hash refresh tokens before storing in Redis (`refresh:{sha256(token)}`). |
| **TLS** | Transport Layer Security | Successor to SSL. All production traffic requires TLS 1.2 or 1.3. nginx handles termination. |
| **TOTP** | Time-Based One-Time Password | Algorithm (RFC 6238) for generating rotating 6-digit codes. Used for admin MFA (Phase 7+). |

---

## Technology Stack

| Acronym / Term | Full Form | Notes |
|---|---|---|
| **aioredis** | Async Redis (Python library) | Async Redis client. Pool stored on `app.state.redis`; initialised in FastAPI lifespan. |
| **asyncpg** | Async PostgreSQL (Python library) | Fastest async PostgreSQL driver for Python. Pool stored on `app.state.pool`. Used instead of an ORM on hot paths. |
| **API** | Application Programming Interface | The set of HTTP endpoints the backend exposes. All routes prefixed `/api/v1/`. |
| **Celery** | — (proper noun) | Distributed task queue. Used for fire-and-forget progress writes, pipeline jobs, Auth0 sync, GDPR deletion, and scheduled Celery Beat tasks. |
| **FCM** | Firebase Cloud Messaging | Google push notification service. Used to deliver streak reminders, new content alerts, quiz nudges, and weekly summaries to Android devices. |
| **httpx** | — (proper noun) | Async HTTP client library for Python. Used for Auth0 Management API calls and external service requests. |
| **JSON** | JavaScript Object Notation | Data interchange format. All API responses and content files are JSON. |
| **Kivy** | — (proper noun) | Python GUI framework for cross-platform mobile apps. Used for the student mobile app (Android and Chrome targets). |
| **MP3** | MPEG Audio Layer 3 | Audio format used for pre-generated TTS lesson audio files. |
| **ORM** | Object-Relational Mapper | Abstraction layer over SQL. The project uses asyncpg directly on hot paths (no ORM); Alembic (SQLAlchemy-based) is used for migrations only. |
| **REST** | Representational State Transfer | Architectural style for HTTP APIs. All StudyBuddy backend endpoints follow REST conventions. |
| **SDK** | Software Development Kit | Pre-built libraries for interacting with a service. e.g. Stripe SDK, Sentry SDK, Auth0 SDK. |
| **SQL** | Structured Query Language | Language for relational database operations. PostgreSQL is the primary database. |
| **SQLite** | — (proper noun) | Embedded, file-based SQL database. Used on the mobile app for the offline progress event queue and local content cache. |
| **TTS** | Text-To-Speech | Technology that synthesises spoken audio from text. Pre-generated at pipeline time via Amazon Polly or Google Cloud TTS; stored as MP3 files. |
| **UUID** | Universally Unique Identifier | 128-bit identifier. All primary keys use UUID v4 (`gen_random_uuid()`). Used as `event_id` for offline event deduplication. |
| **XLSX** | Excel Open XML Spreadsheet | File format used by teachers to upload custom curriculum definitions (`POST /curriculum/upload`). |

---

## Infrastructure & Cloud

| Acronym / Term | Full Form | Notes |
|---|---|---|
| **AOF** | Append-Only File | Redis persistence mode. Every write is appended to a log file. `appendfsync everysec` is mandatory in production — without it, a Redis restart logs out all students. |
| **AWS** | Amazon Web Services | Cloud provider referenced in the architecture for S3, CloudFront, SES, Polly, and RDS. Architecture is cloud-agnostic; AWS is the reference implementation. |
| **CDN** | Content Delivery Network | Global edge caching network. CloudFront or Cloudflare serves audio MP3s and large JSON files. Audio is never proxied through the API server. |
| **CI** | Continuous Integration | Automated build-and-test pipeline. GitHub Actions runs pytest on every push with PostgreSQL and Redis service containers. |
| **DNS** | Domain Name System | System that translates domain names to IP addresses. GeoDNS is used in the multi-region (US + EU) topology to route students to the nearest region. |
| **ECS** | Elastic Container Service | AWS container orchestration service. One of the production deployment targets (alternative to Kubernetes). |
| **nginx** | — (proper noun, always lowercase) | Web server and reverse proxy. Handles TLS termination, HTTP→HTTPS redirect, rate limiting, gzip, and blocks `/metrics` from public access. |
| **PgBouncer** | PostgreSQL Connection Pooler | Connection pooler running in transaction-pooling mode in front of PostgreSQL. Prevents connection exhaustion when multiple uvicorn workers are running. |
| **RDB** | Redis Database (file) | Redis point-in-time snapshot format (`.rdb` file). Created daily alongside AOF for backup purposes. |
| **RDS** | Relational Database Service | AWS managed PostgreSQL service. Used in the replica promotion runbook in OPERATIONS.md. |
| **S3** | Simple Storage Service | AWS object storage. Used as the Content Store from Phase 4 onwards (Phase 1–3 use local filesystem). |
| **SES** | Simple Email Service | AWS transactional email service. Used for pipeline alert emails, weekly teacher digests, and parental consent emails. |

---

## Architecture & Patterns

| Acronym / Term | Full Form | Notes |
|---|---|---|
| **AlexJS** | alex (Node.js library) | Automated content language analysis tool. Detects profanity, gendered phrasing, and ableist language. Run as a subprocess from the Python pipeline after JSON schema validation. |
| **DDL** | Data Definition Language | SQL statements that define schema (`CREATE TABLE`, `ALTER TABLE`, etc.). All DDL is managed through Alembic migrations. |
| **L1 / L2 / L3** | Cache Level 1 / 2 / 3 | Three-tier caching hierarchy: L1 = per-worker in-process TTLCache (cachetools); L2 = shared Redis; L3 = CloudFront CDN. Hot read path must serve zero DB queries when cache-warm. |
| **LRU** | Least Recently Used | Cache eviction policy. Used in L1 TTLCache and for mobile SQLite content cache when approaching `MAX_CACHE_MB`. |
| **OTEL** | OpenTelemetry | Observability framework for distributed tracing. Referenced in `backend/.env.example`; planned for Phase 2+ as an optional integration. |
| **TTL** | Time-To-Live | Duration after which a cached item or token expires. Examples: JWKS cache 24 hr; entitlement cache 300 s; refresh token 30 days; admin reset token 1 hr. |

---

## Operations & Reliability

| Acronym / Term | Full Form | Notes |
|---|---|---|
| **DR** | Disaster Recovery | The set of procedures to restore service after a failure. Covered in OPERATIONS.md. |
| **RPO** | Recovery Point Objective | Maximum acceptable data loss in a recovery scenario. e.g. PostgreSQL replica promotion: RPO 5 minutes (WAL streaming lag). |
| **RTO** | Recovery Time Objective | Maximum acceptable downtime before service is restored. e.g. PostgreSQL replica promotion: RTO 15 minutes. |
| **SEV** | Severity (incident level) | Incident classification: SEV-1 (full outage, immediate response), SEV-2 (critical feature down, 15 min), SEV-3 (degraded, 1 hr), SEV-4 (minor, next business day). |
| **SLA** | Service Level Agreement | Contractual uptime commitment. Target: 99.9% uptime (≤ 8.7 hours unplanned downtime per year). |
| **SLO** | Service Level Objective | Internal performance target. e.g. content endpoint p95 latency ≤ 200 ms; availability ≥ 99.9%. |
| **WAL** | Write-Ahead Log | PostgreSQL durability mechanism. Every change is written to the WAL before being applied. WAL archiving enables point-in-time recovery; retained for 7 days. |

---

## Testing & Quality

| Acronym / Term | Full Form | Notes |
|---|---|---|
| **ADR** | Architectural Decision Record | A log entry capturing a significant design choice, its rationale, and status. Stored in REQUIREMENTS.md `## Architectural Decisions Log`. |
| **FR** | Functional Requirement | A requirement describing what the system must do. Prefixed `FR-` in REQUIREMENTS.md (e.g. `FR-AUTH-001`). |
| **NFR** | Non-Functional Requirement | A requirement describing how well the system must do it (performance, security, compliance). Prefixed `NFR-` in REQUIREMENTS.md (e.g. `NFR-PERF-001`). |

---

## Business & Product

| Acronym / Term | Full Form | Notes |
|---|---|---|
| **ARR** | Annual Recurring Revenue | Total contracted recurring revenue over a 12-month period. Used in financial projections. |
| **ARPU** | Average Revenue Per User | Total revenue divided by number of active users. Blended ARPU target: ~$7.20/student/month. |
| **MRR** | Monthly Recurring Revenue | Total recurring revenue for a given month. Tracked in subscription analytics (Phase 7). |
| **SAM** | Serviceable Addressable Market | The portion of the TAM that StudyBuddy can realistically reach with its current model. Estimated $28B–$55B. |
| **SaaS** | Software as a Service | Cloud-delivered software subscription model. The primary business model for individual student and school subscriptions. |
| **SOM** | Serviceable Obtainable Market | The portion of the SAM that can realistically be captured in the first 5–7 years. Estimated $800M–$2.5B. |
| **STEM** | Science, Technology, Engineering, and Mathematics | The academic subject domain of the platform. Grades 5–12. |
| **TAM** | Total Addressable Market | The total global market revenue opportunity. Global K-12 EdTech: $340B (2025) → $700B+ by 2032. |
| **VC** | Venture Capital | Private equity funding from investors in exchange for equity. Referenced in the investor deck. |

---

*Last updated: 2026-03-24. Add new acronyms to the appropriate section; keep entries sorted alphabetically within each group.*
