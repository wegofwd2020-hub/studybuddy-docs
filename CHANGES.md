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
| D-06 | Per-answer progress logging | Enables adaptive difficulty and struggle analytics in Phase 5 |
| D-07 | JWT authentication | Stateless; works offline between refreshes; no session server needed |
| D-08 | Grade gating on backend | Controls which grades are visible per student; natural upgrade path |
| D-09 | Shared `prompts.py` between pipeline and Free edition | Consistent grade-language calibration; one place to improve prompts |
| D-10 | Content Store keyed by `unit_id` | Unit IDs are stable; grade/subject names can change without breaking storage |

---

## Implementation Log

*No implementation has begun. Log entries will be added here as phases are completed.*

---

## Pending

| # | Item | Phase |
|---|---|---|
| P-01 | FastAPI project skeleton | 1 |
| P-02 | PostgreSQL schema design | 1 |
| P-03 | Auth service (register / login / refresh) | 1 |
| P-04 | Curriculum service | 1 |
| P-05 | Mobile thin-client auth flow | 1 |
| P-06 | Content generation pipeline CLI | 2 |
| P-07 | Content service endpoints | 2 |
| P-08 | Mobile SQLite content cache | 2 |
| P-09 | Progress service endpoints | 3 |
| P-10 | Mobile progress event posting | 3 |
| P-11 | SQLite offline event queue + SyncManager | 4 |
| P-12 | Backend event deduplication | 4 |
| P-13 | Admin API + dashboard | 5 |

---

*Last updated: 2026-03-23*
