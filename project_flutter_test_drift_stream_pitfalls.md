---
name: project-flutter-test-drift-stream-pitfalls
description: Known flutter_test hazards in this repo when a widget/controller touches a live Drift .watch() stream — pumpAndSettle() hangs/times out; use getAll() one-shot reads instead
metadata:
  type: project
  originSessionId: 2026-08-26-scan-report-classification
  modified: 2026-08-26T01:25:31.238Z
---

This Vitaly repo has a reproducible `flutter_test` environment hazard,
documented independently in three places now (`app_router_flow_test.dart`,
`reading_detail_screen_test.dart`, and this session's
`settings_controller.dart`/`settings_screen_test.dart` fix):

**The hazard:** combining a live Drift `.watch()` stream with a
Navigator/dialog transition, or with `pumpAndSettle()`, inside the same
`testWidgets` body reproducibly hangs or times out. Specifically found
this session: `SettingsController.deleteAccount()` called
`savedReportRepository.watchAll().first` to grab a one-time snapshot —
this opens a *brand-new* stream subscription. In the widget test's
`FakeAsync`-driven clock, that first emission needed more fake-clock time
than a short fixed `tester.pump(duration)` gave it, and switching that
same call site to `pumpAndSettle()` made it worse (hit the 10s
`pumpAndSettle timed out` assertion), because the account-deletion flow
also opens a dialog (Navigator route transition) — the exact
stream+Navigator combination the other two files already flagged as
hazardous.

**The fix, in order of preference:**
1. If the call site only needs a one-time snapshot (not reactivity), add
   and use a plain `Future`-returning `getAll()` method on the repository
   instead of `.watchAll().first` — mirrors the pre-existing
   `ReminderRepository.getAll()` pattern. This avoids opening a new stream
   subscription entirely, and a short fixed `tester.pump(duration)` (e.g.
   200-300ms) becomes sufficient again. This was the actual fix applied to
   `SettingsController.deleteAccount` + `SavedReportRepository.getAll()`
   this session.
2. If a live stream is unavoidable in the widget under test, do NOT use
   `pumpAndSettle()` — use fixed-duration `tester.pump()` calls instead,
   same as the `_settle()` helper in `app_router_flow_test.dart`.
3. If the interaction is inherently a dialog/Navigator transition over a
   widget watching a live stream (e.g. confirm-delete flows), the
   established fallback in this repo is to skip the full interactive
   widget test for that specific combination, rely on a
   `ProviderContainer`-level unit test of the controller logic (no widget
   tree, no Navigator) for correctness, and note that the interactive UX
   is verified manually on-device instead — see
   `reading_detail_screen_test.dart`'s own comment for the precedent.

**How to apply:** Before writing a widget test that taps through a
confirm/delete/dialog flow whose underlying controller method awaits
*any* repository stream (even indirectly), check whether that stream read
can be a one-shot `getAll()`-style call instead. If a hang or timeout
shows up in exactly this shape (works fine at 4-5 tests in, hangs on the
one with a dialog + async repository work, `pumpAndSettle timed out` or
a silent multi-minute hang with near-zero CPU), this is very likely the
same root cause — try fix #1 first before assuming it's a real deadlock.
