---
name: project-streak-and-engagement-notifications
description: "Trends PDF export premium-gated, calendar-day streak rework, and streak/re-engagement local notifications — shipped 2026-09-04"
metadata: 
  node_type: memory
  type: project
  originSessionId: 4b944628-4807-4462-bb74-7262be217995
  modified: 2026-09-04T01:34:45.123Z
---

Shipped 2026-09-04 (commit `01df656` on `master`, merged via PR #1 →
`dcadc56`): four related features, all verified live on a Pixel 8
emulator (not just `flutter test`).

**1. Trends PDF export gating (bug fix).** The Trends screen's "Export
PDF" button called `buildTrendSummaryPdf` + `Printing.sharePdf` directly
with no paywall check — a real bypass, since History/Settings exports
already gated on `PaywallPlacements.exportReportData`. Now routes through
the same `paywallServiceProvider.gateFeature(...)` call.

**2. Streak rework — calendar days, not rolling windows.** The pre-existing
`computeLoggingStreak` (`lib/features/blood_pressure/domain/logging_streak.dart`)
used 24-hour windows anchored to the most recent reading. Replaced with
`computeStreakStats` → `StreakStats {currentStreak, bestStreak,
lastActivityDate, recordedToday, isAtRisk}`, built from the *set of
calendar days* with ≥1 reading — logging twice in a day no longer inflates
anything. Dashboard streak tile now also shows best streak + a 7-day dot
visual.

**3. Streak-at-risk / re-engagement local notifications.** New:
`lib/features/reminders/domain/engagement_notification_plan.dart` (pure
planner, same shape as `TrendCalculator`) decides what to schedule from
`StreakStats` + a preferred time; `lib/features/reminders/data/
engagement_notification_coordinator.dart` (`EngagementNotificationCoordinator`)
does the actual (re)scheduling via three new `NotificationScheduler`
methods (`scheduleOneOff`, `scheduleWeekly`, `cancelById`). Rescheduled
after every reading save, app resume, and sign-in — always cancel-and-
replace, never additive, so nothing duplicates. Missed-tracking fires 3
days after last reading, inactivity at 10 days (far enough apart they can
never coincide for one gap). Weekly summary is a normal recurring
Monday-09:00 schedule, only when ≥1 reading exists ever.

**4. Settings toggles.** New "NOTIFICATIONS" section, two toggles
(`streakRemindersEnabledProvider`, `reEngagementNotificationsEnabledProvider`,
both SharedPreferences-backed, **off by default**) — permission is
requested only on toggle-ON, mirroring the existing reminder-creation
permission rule (PROJECT_SPEC.md §17). Confirmed live: toggling on pops
the real Android `POST_NOTIFICATIONS` dialog.

**Why the coordinator wraps itself in try/catch:** `RecordBpController.save()`
calls `EngagementNotificationCoordinator.reschedule()` on every save, but
several existing tests (e.g. `test/core/analytics/analytics_events_test.dart`)
don't override `sharedPreferencesProvider` — reading it there throws
`UnimplementedError`. The coordinator swallows and logs any failure
internally so a notification-scheduling glitch can never break the save
flow itself; this is also just correct production behavior for a
non-critical integration (same philosophy as `NoOpPaywallService`).

**Verification method:** used the `android` CLI (`android-cli` skill) to
drive a real Pixel 8 emulator end-to-end — signed up a test account,
logged two readings on different calendar days (via the in-app date
picker, not by changing system time), confirmed the streak tile, tapped
Export PDF and got a real share sheet with `vitaly-trend-summary.pdf`,
and toggled Streak reminders on to see the live permission dialog. No
crashes in logcat. `android layout` coordinates are in real device
pixels — screenshots are scaled 1.2× smaller for display, so tap
coordinates must come from `android layout`/`screen resolve`, never
eyeballed off the displayed screenshot image.

**How to apply:** if asked to touch streak or notification logic again,
this is the current shape — don't reintroduce the old 24-hour-window
streak definition or a second paywall-gated PDF export path.
