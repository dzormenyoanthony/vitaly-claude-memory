---
name: project-deferred-dashboard-slots
description: Dashboard educational-content and reminders slots (PROJECT_SPEC.md §10) are both built; BP classification/thresholds (the item this memory used to flag as deferred) was implemented 2026-08-26 — see [[project-status]]
metadata:
  type: project
  originSessionId: cc450c07-d367-45ae-8c65-710f50a0bf4d
  modified: 2026-08-26T01:25:55.305Z
---

**Update 2026-08-26:** the BP classification/thresholds gap this memory
used to flag as the one remaining deferred item is now closed —
`BPClassificationService` (v1 reference framework from PROJECT_SPEC.md
§19) is implemented and integrated across Dashboard/History/Reading
detail/Trends/PDF export. See [[project-status]] for full detail. The
content below is kept for historical context on the reminders/education
slots, which haven't changed.

Vitaly's dashboard (PROJECT_SPEC.md §10) lists "next reminder" and
"educational content" as components. Both are now built.

Reminders slot: shipped 2026-08-22 (`NextReminderCalculator` +
`_NextReminderCard`).

Educational content slot: shipped 2026-08-23. User explicitly approved the
American Heart Association (heart.org) as the authoritative source
(previously this was blocked — CLAUDE.md forbids inventing medical
content, and §15 requires an approved source). Built as
`lib/features/education/` (`Article`/`ArticleRepository` with 7 static
articles across Basics/Measuring well/Working with your clinician,
`EducationScreen` list + `ArticleDetailScreen`), routed at `/education` and
`/education/:id`, and surfaced on the dashboard via `_EducationCard`
(featured article + "Browse all articles"). Content deliberately excludes
any numeric BP classification thresholds (normal/elevated/stage 1/stage
2/crisis) — enforced by a repository test
(`article_repository_test.dart`) that asserts no article mentions staging
language. Verified on-device on a Pixel 8 emulator end-to-end: dashboard
card → article detail → full library list, all rendering correctly and
matching `design_references/Learn.png` and `Measure well.png` closely.

**Why kept as a memory at all:** the BP-classification/staging content
remains a separate, not-yet-approved decision gated by §14 — don't confuse
"AHA approved as source for general education" with "thresholds/staging
approved." That gate is unrelated to source approval and needs its own
explicit sign-off if ever pursued.

**How to apply:** This dashboard-slots decision is now closed for both
components. Don't reintroduce a "deferred" framing for reminders or
educational content. If asked to add BP category thresholds/classification
anywhere (dashboard, education content, history/trends), treat that as the
separate §14 gate, not as an extension of this already-approved education
feature.
