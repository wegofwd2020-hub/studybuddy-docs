# UX Goals by Persona

> One-sentence north stars for each persona. Use these as a filter when prioritising
> features, sequencing work, and making UI decisions. If a proposed change does not
> move the needle on the relevant goal statement, it is not the next thing to build.
>
> Last updated: 2026-04-05.

---

## How to Use This Document

Every feature decision, screen layout, and prioritisation call should start with
the question: **does this get the target persona closer to their goal, or further away?**

If a proposed feature is not on the critical path to the goal, it either belongs in
a later phase or should be cut entirely. This document is not a wish list — it is a
filter.

---

## Persona 1 — Student

### North Star Goal

> **"Get from confusion to confidence on a concept in under 3 minutes, with no dead ends."**

### Job to Be Done

A student opens the app because they don't understand something from class, or they
want to prepare for a test. They are on a deadline, probably on a phone, and have low
tolerance for friction. Every extra tap, loading spinner, or confusing menu is a
moment where they close the app instead.

### UX Principles

1. **The learning path is one tap deep, always.**
   Lesson → Quiz → Result → Tutorial (if needed) is a straight line. No navigation
   required between steps. A student who finishes a quiz should never wonder what to
   do next.

2. **A wrong answer is a teaching moment, not a failure state.**
   The result screen shows the correct answer with an explanation immediately — before
   the student sees their score. The tutorial is surfaced as "want to go deeper?" not
   as a punishment redirect.

3. **Offline is invisible.**
   If content is cached, the student should not know or care that they are offline.
   No banners, no disabled buttons. The only offline signal is a subtle indicator when
   actively syncing.

4. **Progress is encouraging, not pressuring.**
   Streak counters and scores are shown, but never in a way that makes skipping a
   day feel like failure. The dashboard celebrates what the student has done, not what
   they haven't.

5. **Audio is first-class, not a bonus feature.**
   The listen button appears at the same visual weight as the lesson text. A student
   who learns better by listening should find that path immediately, not buried in a
   settings menu.

### Anti-Goals

- Do not optimise for session length. A student who understood the concept in 90
  seconds succeeded. More time is not better.
- Do not surface subscription prompts inside the learning flow. Paywall appears once,
  at the point of access denial, with a clear path forward. Never between a lesson
  and a quiz.
- Do not show grade-level labels on content (e.g. "Grade 6 reading level"). Content
  is written at 1–2 levels below the student's grade for accessibility — this should
  be invisible to them.

### Priority Filter

When two student-facing features compete for the same sprint:
**choose the one that removes a step from the learning path.**

Current critical gaps (apply this filter):
- `QuizScreen` (missing) blocks the entire learning loop — highest priority
- `LoginScreen` (missing) blocks the app from being used at all
- `TutorialScreen` (missing) means failed quiz has no recovery path
- After those: audio caching improvement, experiment screen polish

---

## Persona 2 — Teacher

### North Star Goal

> **"Know which students need help and with what, before the student gives up."**

### Job to Be Done

A teacher has 25–30 students across multiple subjects. They cannot monitor individual
progress manually. They open the app to answer one question: *"Is anyone falling
behind, and who?"* — and then act on the answer. They do not want to run reports;
they want the insight to come to them.

### UX Principles

1. **Alerts come to the teacher, not the other way around.**
   The weekly digest email is more valuable than the reports dashboard for most
   teachers. The primary data delivery channel is the inbox, not a login session.
   Design every reporting feature with the question: "should this be a proactive
   alert instead?"

2. **Class-level first, individual second.**
   The default view is the whole class. Drilling into a single student is one tap
   from the class overview. The reports that show every student as a separate row
   are secondary — the primary signal is which units are red for the class.

3. **At-risk is a queue, not a chart.**
   The at-risk report is a list of students to act on, not a visualisation. Each
   row should show the student, the problem, and a suggested action — not just a
   number.

4. **Teacher time is 10 minutes a week, maximum.**
   A teacher who logs in on Monday morning should be able to review the week's
   situation, act on any alerts, and leave in under 10 minutes. If the UI requires
   more time than that to extract useful signal, the design has failed.

5. **Never show data the teacher cannot act on.**
   Platform-wide averages, pipeline status, and admin-only metrics are hidden from
   teacher views. Every number visible to a teacher should have an implicit next
   action attached to it.

### Anti-Goals

- Do not build more report types until the existing reports surface actionable
  insight proactively. More charts are not the answer.
- Do not require teachers to configure thresholds before alerts work. Sensible
  defaults (pass rate <60%, no activity >7 days) must work out of the box.
- Do not cross school boundaries. A teacher's view of students is strictly scoped
  to their own school — never show aggregate platform data.

### Priority Filter

When two teacher-facing features compete:
**choose the one that gets information to the teacher without requiring a login.**

Current state assessment:
- Digest email ✅ — highest-leverage feature; already built
- Alerts system ✅ — built; threshold configuration is optional-nice-to-have
- At-risk report ✅ — exists; could be more actionable (one-tap from alert to student)
- Class overview ✅ — exists
- Report navigation — functional but secondary to proactive delivery

---

## Persona 3 — School Admin

### North Star Goal

> **"Run the school's StudyBuddy deployment end-to-end in under 10 minutes a week."**

### Job to Be Done

The school admin is also a teacher — they have a full teaching load and no dedicated
time for platform administration. They need to onboard teachers, activate curriculum,
manage enrolments, and handle subscription — all in the gaps between teaching.
Administrative friction compounds across every teacher and student in the school.

### UX Principles

1. **Onboarding a new teacher is one action.**
   Enter email, send invite. The teacher receives a link, sets a password, and is
   active. No admin approval step, no back-and-forth. The invite link is the
   authentication.

2. **Curriculum activation is one action.**
   Upload XLSX (or select from library) → validate → activate. The admin should
   never need to run a pipeline manually or wait for content to be available. If
   content is pending, show a clear estimated wait, not a spinner with no context.

3. **Enrolment errors surface immediately, row by row.**
   When a bulk enrolment upload fails for some students, the error response must
   identify exactly which rows failed and why — not just a count. The admin
   should be able to fix and re-upload without losing the successful rows.

4. **Subscription management requires no support ticket.**
   Plan upgrade, cancel, and invoice download are all self-service. The admin
   never needs to email anyone to change their plan.

5. **School admin is a superset of teacher.**
   Switching between "my class" and "whole school" views should be instant — one
   tab or toggle, not a different login or portal.

### Anti-Goals

- Do not show school admins any content pipeline internals (Celery job IDs,
  token counts, model names). Those are super-admin concerns.
- Do not require school admins to manually refresh materialized views in normal
  operation. Scheduled refresh should be invisible; manual refresh is a fallback.

### Priority Filter

When two school-admin features compete:
**choose the one that unblocks the most teachers or students downstream.**

Curriculum upload and teacher invite unblock entire cohorts; digest settings
affect only one user.

---

## Persona 4 — Super Admin / Platform Team

### North Star Goal

> **"Trust that content reaching students is accurate, age-appropriate, and reviewed
> — without the review process becoming a bottleneck."**

### Job to Be Done

The super admin runs the platform. Their primary responsibility is content quality —
ensuring that what the AI generates is correct, inclusive, and appropriate for
Grades 5–12 before any student sees it. They also handle school approvals,
escalations, and platform health. The risk they are managing is not a broken
feature — it is wrong or harmful content reaching students at scale.

### UX Principles

1. **Content review is the primary workflow, not a menu item.**
   The admin portal's main landing after login should surface the review queue, not
   a analytics dashboard. The number of pending reviews and any alex_warnings are
   the first things a super admin sees in the morning.

2. **Alex warnings are blocking, not decorative.**
   A content version with open alex_warnings should be visually distinct from clean
   versions in the review queue — different colour, a warning badge, never just
   fine print. Reviewers should not be able to approve without explicitly
   acknowledging each warning.

3. **Approve or reject in one screen.**
   A reviewer should be able to read the content, leave annotations, and
   approve/reject without navigating away from the unit viewer. Round-trips between
   the queue, the viewer, and the action buttons create friction that slows the
   review cycle.

4. **Pipeline visibility without noise.**
   The admin can see job status, progress percentage, and failures — but is not
   required to understand Celery internals. A failed job shows: which units failed,
   why (in plain English), and one button to retry.

5. **Rollback is always one action away.**
   If bad content reaches students, the time-to-rollback must be seconds. The
   rollback button is visible on every published version, never buried in a
   settings menu.

### Anti-Goals

- Do not surface student PII to admin roles that don't need it. Product admins
  see aggregate platform metrics; only super_admin sees individual student records.
- Do not make the admin portal a student-facing design. Density and efficiency
  matter more than visual warmth. Tables over cards, text over icons where
  information density is needed.
- Do not normalise approving content without reading it. UX should create light
  friction (annotation prompt, warning acknowledgment) that makes rubber-stamping
  harder than doing the job properly.

### Priority Filter

When two admin-facing features compete:
**choose the one that reduces the time between bad content being generated and
it being blocked or corrected.**

Current open issues mapped to this filter:
- **#55 Surface alex_warnings prominently** — directly serves the north star goal; highest priority
- **#54 Batch approve all subjects in a grade** — speeds up review of *clean* content; secondary
- **#56 Review assignment** — improves team coordination; tertiary

---

## Cross-Persona Principles

These apply across all three portals:

### Speed of the Core Action

Every persona has one action they perform most frequently:
- Student: open a lesson or take a quiz
- Teacher: check who is struggling
- School Admin: check enrolment / curriculum status
- Super Admin: process the review queue

The path to that action must be the shortest path from login. Navigation that buries
the core action behind two or more clicks is a design problem.

### Accessibility Is Not a Feature

Dyslexia mode (Alt+D), WCAG 2.1 AA contrast, and audio alternatives are present in
every portal. They are not opt-in features — they are the baseline. No persona's
experience is considered complete until it passes these bars.

### Empty States Teach, Not Just Inform

When a persona first logs in and nothing has happened yet — no students enrolled, no
content reviewed, no quizzes taken — the empty state tells them exactly what to do
next. "No students enrolled yet — [Invite students →]" is better than "No data
available."
