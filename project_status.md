---
name: project-status
description: "Current build status for Vitaly — feature-complete per PROJECT_SPEC, what's shipped, repo/git state, open items"
metadata: 
  node_type: memory
  type: project
  originSessionId: 247eb89e-4bcc-4af7-b068-92b57033f42c
  modified: 2026-09-02T04:53:28.893Z
---

**As of 2026-09-02.** Vitaly is feature-complete against `PROJECT_SPEC.md`
and everything is committed and pushed to `master`
(`github.com/dzormenyoanthony/Vitality`, private; Firebase
`vitality-23966`). `flutter analyze` clean; `flutter test` green at
**312** (host unit/widget suite). Working tree clean, `master` level with
`origin/master`. `pubspec.yaml` version `1.0.1+7`.

**Shipped (see the dedicated memories for detail):**
- MVP core §4 + BP Report scanning + BP status/range classification
  (Phase 1/2, 2026-08-26).
- Visual-design pass — all `design_references/` mockups implemented
  ([[project-design-references]]).
- Auth-gate redirect-race fixes; pre-auth screens locked to light theme
  ([[project-auth-gate-redirect-races]], [[project-preauth-light-theme-lock]]).
- Google Sign-In (server-side cert fix done 2026-08-31)
  ([[project-google-sign-in]]).
- §36 i18n externalization — gen_l10n, ~345 ARB keys, medical-safety
  wording guard test, hard-coded-string lint ([[project-i18n-externalization]]).
- Data export: Trends PDF summary **now with colored vector charts**
  ([[project-trend-pdf-charts]]) + full CSV+scanned-reports ZIP
  ([[project-data-export-feature]]).
- Account deletion re-auth ordering fix ([[project-account-deletion-reauth]]).
- Play Store release prep — signing, permissions, release AAB verified
  ([[project-play-store-release-prep]]); Superwall + Play Console billing
  ([[project-superwall-billing-setup]]).
- Adaptive launcher icon + Android-12 system-splash restyle
  ([[project_launcher_and_native_splash]]).
- Spec-gap audit done 2026-08-29 — §36/§26/§37 fixed; a few minor items
  by-design ([[project-spec-gap-audit]]).

**Android toolchain:** pinned to AGP 8.13 / Gradle 8.14.3 / Kotlin 2.2.20
+ Temurin JDK 17 as a workaround for the AGP-9-vs-classic-Kotlin Firebase
plugin conflict — do NOT let a template regen bump `android/` back to AGP
9 ([[project-agp9-firebase-kotlin-conflict]]).

**Open items:**
- `integration_test/offline_sync_test.dart` still hangs on-device on this
  machine (native-SQLite contention under host+emulator load — not a code
  bug; splitting into 6 small tests didn't fix it).
  `integration_test/scan_report_pipeline_test.dart` passes on-device.
  ([[project-integration-test-ondevice-stall]])
- Keystore / final icon polish / Firebase fingerprint follow-ups from the
  Play Store prep memory.

**Working pattern:** trunk-based on `master`; commit + push only when the
user explicitly asks (they do so per chunk). Memory lives in a separate
repo (`vitaly-claude-memory`) — commit/push that on ask too.

**How to apply:** reorient from this at session start instead of
re-deriving from git log, but verify against current `git log` since it
goes stale. This replaced a long 2026-08-26 brain-dump; phase-by-phase
history is in git and the linked memories.
