---
name: project-spec-gap-audit
description: 2026-08-29 audit of PROJECT_SPEC.md vs. the codebase — §36 i18n and §26 analytics are now resolved; §37 integration tests is the sole remaining real gap
metadata: 
  node_type: memory
  type: project
  originSessionId: 9b40e65e-c3a7-4aaf-ae18-ab12e9a491fa
  modified: 2026-08-30T04:13:37.931Z
---

On 2026-08-29 a full re-read of `PROJECT_SPEC.md` against `lib/` was done
(the user picked "Spec gap audit" from a menu). Method: read the whole
spec, then cross-check pubspec + every feature folder. `flutter analyze`
was clean and the suite green (254 tests) at audit time. Two of the three
gaps have since been closed.

## Gaps found

**1. §36 Internationalization — RESOLVED (2026-08-29).**
Was the widest divergence: no `flutter_localizations`/`intl`/`gen_l10n`,
every UI string an inline literal, 8 hardcoded English month/weekday
tables, no locale-aware date/number formatting. Fixed across 13 commits
(`3170c9e..22bb0f3`, pushed) — see [[project-i18n-externalization]] for
what shipped and what was deliberately left English.

**2. §26 Analytics — RESOLVED (2026-08-30), commit `68b8742` (pushed).**
Added a swappable `AnalyticsService` abstraction (`lib/core/analytics/`)
with a `FirebaseAnalyticsService` impl (`firebase_analytics` 12.5.0 — no
toolchain change, repo already on the AGP-8.13 pin) and a
`NoOpAnalyticsService` default overridden in `main.dart`. The five §26
funnel events fire at natural chokepoints: `app_opened` (VitalyApp
initState + resume), `onboarding_completed` (OnboardingController + the
pre-auth carousel path in SignUpController), `bp_reading_recorded
{source: manual|imported_report}` (RecordBpController new-only +
ConfirmReportController per confirmed reading), `reminder_created`
(ReminderController new-only), `educational_content_opened {article_id}`
(ArticleDetailScreen initState). Privacy (§26): no method takes a BP
value; a test proves a 180/110 save logs neither number. Fake +
`analytics_events_test.dart` cover it.

**3. §37 Testing Strategy — RESOLVED (2026-08-30, statically), commit
`e51be08` (pushed).** Added `integration_test/offline_sync_test.dart`
(3 tests: offline-save-then-sync via `SyncCoordinator.syncAll`,
remote-only-reading pulled into History on sync, online save reaching
Firestore without an explicit sync) plus the `integration_test`/
`fake_cloud_firestore` pubspec dev-dependencies. Drives the real
`VitalyApp` widget tree against an in-memory Drift db and
`FakeFirebaseFirestore`. `flutter analyze`, the hardcoded-string
guardrail, and the full unit/widget suite (274 tests) all pass, and
every provider/repository signature the test references was checked
against current code. **Caveat: the on-device run itself is NOT
confirmed** — see [[project-integration-test-ondevice-stall]]. Commit
was made on the strength of static verification per explicit user
choice, not a green on-device run.

## Borderline / by-design (noted, not fixed)

- **§22 sync-on-reconnect** — no connectivity listener; resync fires only
  on app-resume and sign-in (`main.dart` `didChangeAppLifecycleState` /
  `_handleAuthGateChange`). Misses "connectivity returns while app stays
  foregrounded". Deliberate, documented in the code.
- **§32 Record BP in-progress state** — no `RestorationMixin`/draft save.
  Flutter `State` natively survives rotation + brief backgrounding (the
  spec's literal examples), so covered in practice; only a full process
  death mid-entry loses the form.
- **§27 Monetization** — no freemium tiering. Spec defers pricing
  ("determined separately") and all "potential premium" features (reports,
  PDF/CSV export) currently ship free. Not actionable without approved
  pricing.

## Confirmed conformant

§5–14 (BP model, recording, validation ranges in `ReadingValidator`,
history, edit/delete-with-confirm, dashboard, trends incl. pulse chart,
non-diagnostic trend language, classification), §15–16 (7 AHA-sourced
articles + disclaimers), §17/§23 (reminders + permission-on-enable),
§19/§30 (onboarding + nav gating), §20/§21 (Firebase auth + Firestore/
Storage), §26 (analytics — see above), §28 (Trends PDF + full ZIP export,
plus the new Export data screen `f085120` that adds a date range / format
choice / column toggles on top of §28), §29 (design system), §35
(textScaler not overridden → respects system scaling; chart Semantics
labels present), and the full Phase 1 + Phase 2 feature spec. All
`design_references/*.png` mockups are implemented.

## How to apply

Use this to answer "what's left per the spec?" without re-auditing. §36,
§26, and §37 are all done in code and pushed — do not re-flag them as
missing. §37's on-device execution is still unverified, though — see
[[project-integration-test-ondevice-stall]] before assuming it actually
passes on a real device/emulator. Re-verify against current `git log`/
code before acting, since this snapshot will age.
