---
name: project-integration-test-ondevice-stall
description: on-device integration-test freezes on this machine are emulator SQLite CONTENTION, not a Dart deadlock — short tests slip through (scan_report_pipeline_test.dart passed 3x, 2026-09-02, after pm-uninstall + am kill-all + warm emulator), long ones still stall (offline_sync_test.dart froze a 4th time even after a fresh reboot, always right after recordReading in the sync step).
metadata:
  type: project
  originSessionId: bc2f376b-e0e8-4af1-9c36-1db1ddb1e34f
  modified: 2026-09-02T04:23:43.329Z
---

`integration_test/offline_sync_test.dart` (added in `e51be08`, see
[[project-spec-gap-audit]]) is statically verified — `flutter analyze`,
the hardcoded-string guardrail, and the full unit/widget suite all pass,
and every provider/API it references checks out against current code —
but its actual on-device run has never completed successfully.

**What happened, twice, on a real Pixel_8 emulator (API 37) on this
machine:** `flutter test integration_test/offline_sync_test.dart -d
emulator-5554` builds, installs, and launches the app fine. The very
first test's `recordReading()` helper genuinely succeeds — confirmed via
`adb shell screencap`, the Dashboard shows the just-saved 128/82 reading.
Then, ~8 minutes later, the app process is still alive (not crashed, no
ANR) but produces no further logcat output at all, and the flutter-test
process on the host shows near-zero CPU growth. This is a full stall,
not a slow-but-progressing run — confirmed by comparing two separate
attempts, both freezing at the same logical point (right after
`recordReading()` returns, during pure host-side-executing-on-device
Dart code: `_remoteReadings()` / `SyncCoordinator.syncAll()`, which touch
a freshly-constructed `NativeDatabase.memory()` Drift db and
`FakeFirebaseFirestore`).

**Leading suspect, NOT confirmed:** Drift's native (FFI) `sqlite3`
backend may spawn a background isolate for a fresh in-memory database,
and that spawn may stall under this machine's known resource pressure
(see [[project-agp9-firebase-kotlin-conflict]] for prior RAM-exhaustion
history — free RAM was ~344MB during the first attempt, ~2GB during the
second, so it recurred even with more headroom, which weakens the pure
memory-pressure theory somewhat). Could not confirm via VM service
introspection in-session (no websocket JSON-RPC tooling readily
available; only HTTP GET to the VM service's static page was tried,
which succeeded and proves the service isolate is alive but says nothing
about the main isolate's await state).

**What was NOT the cause:** not a code logic bug as far as static
analysis can tell — `SyncCoordinator.syncAll` (`lib/core/sync/
sync_coordinator.dart`) only does Drift queries and `FakeFirebaseFirestore`
calls, no real network, nothing that should block indefinitely by
inspection.

**Third occurrence (2026-09-02), a different file:**
`integration_test/scan_report_pipeline_test.dart` (added this session to
verify the Scan BP Report pipeline on-device — canned OCR text ->
`ReviewExtractedScreen` -> confirm -> real Drift save -> History; only the
ML Kit recognizer stubbed). Same machine, same Pixel_8 emulator. `flutter
test integration_test/scan_report_pipeline_test.dart -d emulator-5554`
built, installed, and launched the app (logcat: MainActivity Displayed,
Dart VM service listening at 01:22:37) — then produced **zero** host
stdout and **zero** further `flutter :` logcat lines for 17+ min before
being killed. Notably this stalled *before any test output at all* (not
even "00:00 +0: loading"), earlier than offline_sync_test's stall point,
which shifts suspicion toward the flutter_test<->device driver handshake
on this machine rather than (or in addition to) the Drift native isolate.
Statically clean: `flutter analyze` on the new file = "No issues found",
and its review-screen logic mirrors the passing widget test
`test/features/reports/presentation/review_extracted_screen_test.dart`.

**Retry, same file (2026-09-02, later):** re-run with `--reporter
expanded`, output to a file (not a `tail` pipe). This time it got
further — build + install fine, then printed `00:00 +0: canned OCR text
flows through review -> confirm -> real Drift save -> History` (the test
body actually started executing on-device), then froze there for ~12 min
with no further output. Crucially, `android layout` (the a11y tree —
`adb screencap` is black for this app) showed the on-device UI had
reached the **rendered ReviewExtractedScreen with the parsed reading
card**: "136/84 mmHg / Needs review / Pulse 72 bpm · Aug 22, 2026",
checkbox checked. So on a real device the stubbed-OCR-text ->
`BpValueExtractor.extract` -> review-screen-render half of the pipeline
demonstrably works; the freeze is *after* the "Confirm and save" tap,
during `ConfirmReportController.confirmAndSave`'s real Drift writes
(`DriftSavedReportRepository.add` + `DriftBloodPressureRepository
.addReading` on a fresh `NativeDatabase.memory()`) + real
`ReportDocumentStorage.saveLocalPages` path_provider file copy — screen
never advanced to the "SAVED REPORTS STUB" route. logcat during the
freeze showed heavy unrelated SQLite/GMS contention on the emulator
("Slow dispatch took 16815ms ... SQLiteConnectionPool"). This sharpens
the earlier lead: the stall is in **Drift native writes on this
resource-constrained emulator**, not the flutter_test<->device handshake
(which clearly worked — the reporter advanced and the widget tree ran).

**Tried running scan_report_pipeline_test.dart as a `flutter test`
widget test instead (2026-09-02):** doesn't rescue the real-storage
coverage — real `dart:io` file I/O can't be pumped under `flutter test`
(details in [[project-flutter-test-real-io]]). Settled on a headless
widget test with storage stubbed (`test/features/reports/presentation/
scan_report_pipeline_test.dart`, Vitaly c4d1d14) that covers OCR-text ->
extract -> review -> confirm -> real Drift writes -> navigate. (The real
page-file copy is covered by the integration test — which, see below,
went on to pass on-device the same day.)

**RESOLVED for scan_report_pipeline_test.dart (2026-09-02):** it now
**passes on-device, 3 runs in a row** (~6-12s test time each), fully
end-to-end — real `BpValueExtractor`, real `ReviewExtractedScreen`, real
`ConfirmReportController`, **real Drift writes**, **real
`ReportDocumentStorage.saveLocalPages`** (real path_provider + file copy
on-device), DB-row assertions, and the real `HistoryScreen` rendering the
imported reading. So the earlier freezes were **resource contention, not
a deadlock** — consistent with the "Slow dispatch 16815ms /
SQLiteConnectionPool" logcat noise seen during a freeze.

What made it pass, after ~4 prior freezes:
1. `adb -s emulator-5554 shell pm uninstall com.vitality.app.vitality`
   first — a stale test-harness APK left installed seems to be part of it
   (cf. [[project-integration-test-apk-anr-pitfall]]).
2. `adb shell am kill-all` to shed background app load.
3. Emulator already **warm** — earlier freezes were all while the AVD was
   cold-booting *and* Gradle was doing a full `assembleDebug` at the same
   time (RAM + IO + CPU all contended).

**offline_sync_test.dart retried the same way (2026-09-02) — STILL
STALLS (4th time).** Ran it twice: once on the warm emulator (which had
by then accumulated GMS ANRs + a bluetooth SIGABRT), then again after a
full `adb reboot` + settle. Both froze at the **same point every prior
time**: `recordReading('128','82')` genuinely succeeds (Dashboard shows
"LATEST READING 128/82 mmHg" via `android layout`), then the first test
hangs in the sync step (`syncAll` / `_remoteReadings`, Drift <->
FakeFirebaseFirestore). After ~1 min the framework's timeout fired
(`00:59 +0: loading ...` reappears, "Test finished."-finder spam), then
wedged again. logcat both times showed extreme emulator SQLite
contention — `Slow delivery took 206446ms / 288312ms ...
SQLiteConnectionPool$IdleConnectionHandler` in `com.google.android.gms`.

**Why scan_report passes but offline_sync doesn't, same machine/day:**
scan_report is one ~12s test — short enough to slip through before the
emulator's SQLite subsystem starves Drift's native query thread.
offline_sync's first test runs much longer (full `VitalyApp` boot +
splash + driving the Add-reading form + a Drift/Firestore sync
reconcile), so it's still running when the contention bites. Not a code
deadlock — a duration-vs-emulator-contention race.

**How to apply:** to run an on-device integration test on this machine:
`pm uninstall` the app, `am kill-all`, ensure the emulator is booted and
*idle* (not mid-cold-boot during Gradle). Short tests (scan_report) pass.
Long ones (offline_sync) still don't — for those, use a beefier machine /
give the AVD more RAM/CPU, or split the test so no single `testWidgets`
runs long enough to get caught. A no-output hang here is emulator SQLite
contention, not a Dart deadlock — don't chase it in the app code.
