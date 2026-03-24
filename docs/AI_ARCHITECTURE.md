# AI Architecture — StudyBuddy OnDemand

> **One-liner for technical discussions:**
> "We use AI as a content factory, not a content server."

---

## The Core Decision

Most AI tutoring products call a model on every student request. We don't.

```
Traditional AI tutoring
  Student request → AI model → response
  Problem: slow, expensive per-student, unpredictable output quality

StudyBuddy OnDemand
  Pipeline run (offline) → AI model → Content Store
  Student request        → Content Store → response
  Result: instant, cheap to serve, consistent quality
```

Once a unit is generated, serving it to 1 student or 10,000 students costs the same. The AI cost is **per unit of content**, not **per student interaction**.

---

## AI Components

### 1. Lesson & Quiz Generation — Claude (Anthropic)

| Property | Value |
|---|---|
| Model | `claude-sonnet-4-6` (pinned — never implicit "latest") |
| Trigger | Operator runs `pipeline/build_grade.py` or `pipeline/build_unit.py` |
| Runtime | Offline CLI / Celery — never on the student request path |
| Output | Structured JSON validated against a schema before storage |

**What gets generated per curriculum unit:**

| Artifact | Description |
|---|---|
| `lesson_{lang}.json` | Sections, key concepts, worked examples |
| `quiz_set_{n}_{lang}.json` | Multiple choice + short answer with explanations |
| `tutorial_{lang}.json` | Step-by-step problem walkthroughs |
| `experiment_{lang}.json` | Lab instructions (science units only, `has_lab: true`) |
| `meta.json` | `generated_at`, `model`, `content_version`, `langs_built` |

### 2. Audio Generation — AWS Polly / Google TTS

- Each generated lesson text is passed through TTS to produce an MP3
- Runs as part of the same pipeline, after Claude generation
- Audio is stored in the Content Store and served via CDN pre-signed URL
- The API server **never proxies audio bytes** — it only returns the URL

### 3. What the Student Experiences

From the student's perspective there is no visible AI. Content loads instantly because it was pre-built. Latency is CDN + database, not model inference.

---

## Pipeline Safeguards

| Safeguard | Detail |
|---|---|
| **Pinned model ID** | `CLAUDE_MODEL = "claude-sonnet-4-6"` in `pipeline/config.py`. Upgrading models is a deliberate, reviewed act. Different model = potentially different content format. |
| **JSON schema validation** | Every Claude response is validated before writing to the Content Store. On `ValidationError`: retry up to 3×, then mark unit failed and continue. Never write malformed content. |
| **Idempotency** | Pipeline checks `meta.json` content_version at unit start. Skips if already built. Use `--force` to override. Safe to re-run at any time. |
| **Spend cap** | Pipeline aborts if `tokens_used × TOKEN_COST_USD > MAX_PIPELINE_COST_USD` (default $50). Logs and alerts. |
| **Async pipeline trigger** | `POST /curriculum/pipeline/trigger` returns `{job_id}` immediately (202). Status polled via `GET /curriculum/pipeline/{job_id}/status`. UI never blocks on generation. |

---

## Content Store Layout

All content keyed by `curriculum_id` and `unit_id`. Default curricula follow `default-{year}-g{grade}`. School uploads use UUIDs.

```
{CONTENT_STORE_PATH}/
  curricula/
    default-2026-g8/
      G8-MATH-001/
        lesson_en.json
        lesson_fr.json
        lesson_en.mp3
        quiz_set_1_en.json
        quiz_set_2_en.json
        quiz_set_3_en.json
        tutorial_en.json
        experiment_en.json   ← science units only
        meta.json
```

---

## Three Runtime Contexts — AI Only Touches One

```
1. Content Pipeline  ← AI lives here
   CLI/Celery → Claude API + TTS → Content Store
   Runs offline, operator-triggered, never student-facing

2. Backend API       ← no AI
   FastAPI → PostgreSQL + Redis + Content Store
   Serves pre-generated content; enforces auth and entitlement

3. Mobile App        ← no AI, no API keys
   Kivy → local SQLite cache + backend REST
   NEVER calls Claude or any AI service directly
```

The mobile app has no Anthropic API key. It cannot call Claude even if a student tried.

---

## Multi-language Strategy

AI-generated content is created in the target language by Claude directly — not translated after the fact.

```bash
# Generate Grade 8 content in English, French, and Spanish in one run
python pipeline/build_grade.py --grade 8 --lang en,fr,es
```

- French content was generated in French by Claude — never run through a translation layer
- UI strings (buttons, labels) use `mobile/i18n/{lang}.json` — entirely separate from AI content
- Locale is authoritative from the student's JWT — content endpoints never accept a `?lang=` parameter

---

## Unit Economics

| Cost driver | Value |
|---|---|
| Generate one full curriculum unit (all content types, one language) | ~$0.05–0.10 |
| Serve that unit to one student | < $0.001 |
| Serve that unit to 10,000 students | < $0.001 (same CDN cost) |
| Marginal AI cost per student session | **$0** after content is generated |

Gross margin improves as the student base grows — AI cost is fixed per content version, not variable per user.

---

## What AI Does Not Do Here

- No RAG / vector search / embeddings
- No real-time tutoring, chat, or Q&A
- No per-student personalisation at inference time
- No AI on the mobile app
- No AI on the backend API server

Personalisation (Phase 3+) feeds student performance data back into future **pipeline runs** — adjusting which content gets regenerated or prioritised — but the inference itself remains offline.

---

## Common Questions

**Q: What if Claude generates wrong content?**
Every response is schema-validated before storage. Phase 7 adds a human content review queue with approve / rollback / block controls before content reaches students.

**Q: What happens if Claude is down?**
Students are unaffected — they're reading from the Content Store, not calling Claude. Pipeline runs fail gracefully (unit marked failed, run continues) and can be retried.

**Q: How do you handle model upgrades?**
The model ID is pinned. Upgrading is a deliberate pipeline re-run with `--force` on the affected grades. New content goes through the Phase 7 review queue before publication.

**Q: Can schools use their own content?**
Yes (Phase 8). Schools upload a curriculum via XLSX. The same pipeline generates AI content for their custom units, stored under a school-specific `curriculum_id` UUID. Everything else — serving, entitlement, caching — is identical.
