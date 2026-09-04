---
name: project-status
description: "Current build status for Vitaly — feature-complete per PROJECT_SPEC, what's shipped, repo/git state, open items"
metadata: 
  node_type: memory
  type: project
  originSessionId: 247eb89e-4bcc-4af7-b068-92b57033f42c
  modified: 2026-09-04T02:38:41.601Z
---

**As of 2026-09-04.** Vitaly is feature-complete against `PROJECT_SPEC.md`
and everything is committed and pushed to `master`
(`github.com/dzormenyoanthony/Vitality`, private; Firebase
`vitality-23966`). `flutter analyze` clean; `flutter test` green at
**348+** (host unit/widget suite). Working tree has two pre-existing
uncommitted files (`export_data.md`, `pubspec.yaml`, modified before the
2026-09-04 session, not touched by it — still sitting uncommitted, not
yet understood). `master` level with `origin/master`. `pubspec.yaml`
(committed) version `1.0.3+14` (commit `1f9fcfd`) — a release AAB was
built and verified from this version, see
[[project-play-store-release-prep]].

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
- Trends PDF export premium-gated (was bypassing the paywall), streak
  reworked to calendar-days (current/best/at-risk), streak-at-risk +
  re-engagement local notifications, Settings notification toggles —
  shipped 2026-09-04, verified live on a Pixel 8 emulator
  ([[project-streak-and-engagement-notifications]]).
- `onboarding_complete` Superwall placement changed from one-time to a
  recurring non-subscriber nudge (every cold start + every foreground
  resume) — 2026-09-04, commit `553f480`
  ([[project-superwall-billing-setup]]).

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
repo (`vitaly-claude-memory`) — commit/push that on ask too. On
2026-09-04 the user tried a PR-based flow once (feature branch → PR →
merge → delete branch) instead of pushing straight to master; unclear if
that's now the standing preference or a one-off — ask if it's ambiguous
which flow they want next time. `gh pr merge` is blocked by Claude Code's
auto-mode permission classifier in this environment (confirmed it stays
blocked even with fast mode off) — don't retry it; tell the user to merge
via the GitHub web UI instead.

**How to apply:** reorient from this at session start instead of
re-deriving from git log, but verify against current `git log` since it
goes stale. This replaced a long 2026-08-26 brain-dump; phase-by-phase
history is in git and the linked memories.
