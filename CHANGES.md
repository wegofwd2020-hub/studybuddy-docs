# StudyBuddy OnDemand — Change Log

Tracks all decisions, design changes, and implementation milestones.

---

## Design Decisions

| # | Decision | Rationale |
|---|---|---|
| D-01 | Backend-only Anthropic API calls | Removes student API key requirement; eliminates on-device latency |
| D-02 | Pre-generated content pipeline | Lessons and quizzes available instantly; no timeout risk on mobile |
| D-03 | 3 quiz sets per unit | Student can retake a topic without seeing identical questions |
| D-04 | Content versioned by `content_version` in `meta.json` | Allows selective regeneration without disrupting in-progress students |
| D-05 | Offline-first with SQLite event queue | Progress never lost on spotty connections; synced when online |
| D-06 | Per-answer progress logging | Enables adaptive difficulty and struggle analytics in Phase 7 |
| D-07 | JWT authentication with `locale` in payload | Stateless; works offline between refreshes; locale travels with token |
| D-08 | Grade gating on backend | Controls which grades are visible per student; natural upgrade path |
| D-09 | Shared `prompts.py` between pipeline and Free edition | Consistent grade-language calibration; one place to improve prompts |
| D-10 | Content Store keyed by `unit_id` | Unit IDs are stable; grade/subject names can change without breaking storage |
| D-11 | Freemium: 2 free lessons, then subscription | Lowers registration barrier; shows value before asking for payment; no time limit on trial |
| D-12 | Subscription via Stripe (hosted checkout) | Removes PCI scope from backend; Stripe handles card storage and compliance |
| D-13 | Content pre-generated per language (en, fr, es) | No runtime translation; content quality consistent across languages; new languages added in pipeline only |
| D-14 | Locale authoritative from JWT only | Prevents content language mismatch; students cannot request a language not on their account |
| D-15 | TTS audio pre-generated in pipeline (Amazon Polly / Google TTS) | No per-request latency; audio cached offline with lesson JSON; provider swappable |
| D-16 | Experiment visualization JSON generated only for lab units | Avoids generating empty content; mobile detects availability via HTTP 200/404 on content endpoint |

---

## Implementation Log

*No implementation has begun. Log entries will be added here as phases are completed.*

---

## Pending

| # | Item | Phase |
|---|---|---|
| P-01 | FastAPI project skeleton | 1 |
| P-02 | PostgreSQL schema design (`students`, `sessions`, `subscriptions`) | 1 |
| P-03 | Auth service (register / login / refresh) — with `locale` field | 1 |
| P-04 | Curriculum service | 1 |
| P-05 | Mobile thin-client auth flow (with language picker at registration) | 1 |
| P-06 | Content generation pipeline CLI (`build-grade --grade N --lang en`) | 2 |
| P-07 | Content service endpoints (lesson, quiz, practice, tutorial) | 2 |
| P-08 | Entitlement middleware (free-tier 2-lesson limit, HTTP 402) | 2 |
| P-09 | Mobile SQLite content cache (`unit_id + content_version + lang`) | 2 |
| P-10 | `SubscriptionScreen` stub | 2 |
| P-11 | Progress service endpoints | 3 |
| P-12 | Mobile progress event posting | 3 |
| P-13 | SQLite offline event queue + SyncManager | 4 |
| P-14 | Backend event deduplication | 4 |
| P-15 | Multi-language pipeline (`--lang fr,es`) | 4 |
| P-16 | TTS worker (`tts_worker.py`): lesson text → MP3 | 4 |
| P-17 | Audio delivery endpoint + mobile "🔊 Listen" button | 4 |
| P-18 | Language picker in mobile Settings; locale switch + cache clear | 4 |
| P-19 | Mobile i18n: static UI string dictionaries (en/fr/es) | 4 |
| P-20 | Subscription service: Stripe checkout + webhook handler | 5 |
| P-21 | `SubscriptionScreen` with real plan cards and checkout flow | 5 |
| P-22 | Entitlement cache in Redis | 5 |
| P-23 | Experiment pipeline: detect `assessments.labs`, generate `experiment_{lang}.json` | 6 |
| P-24 | Experiment content endpoint (`GET /content/{unit_id}/experiment`) | 6 |
| P-25 | `ExperimentScreen` mobile UI (step-by-step lab guide) | 6 |
| P-26 | Admin API + dashboard | 7 |
| P-27 | Subscription analytics (MRR, churn, conversion) | 7 |

---

*Last updated: 2026-03-23*
