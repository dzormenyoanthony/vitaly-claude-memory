---
name: project-integration-test-ondevice-stall
description: integration_test/offline_sync_test.dart freezes identically twice on-device on this machine, past the recordReading step, no crash/error — root cause unconfirmed
metadata:
  type: project
  originSessionId: bc2f376b-e0e8-4af1-9c36-1db1ddb1e34f
  modified: 2026-08-30T04:13:56.994Z
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

**Decision made:** given repeated identical stalls and diminishing
returns from continued retries, the user chose (via AskUserQuestion) to
commit and push on the strength of static verification alone, rather
than keep burning session time or hold the work back. This is a
deliberate, informed choice — not an oversight.

**How to apply:** Before trusting that §37's integration test actually
passes end-to-end, run it fresh (ideally on a less resource-constrained
machine, or with `flutter drive`/explicit VM-service debugging attached
to see exactly what the isolate is awaiting when it stalls). If it stalls
again at the same point, treat "Drift native isolate spawn on a
memory-constrained Android emulator" as a real lead worth testing
directly (e.g., try `NativeDatabase.memory()` without the background
isolate, or profile isolate spawn time in isolation). Do not assume a
past `flutter analyze`/unit-test-green report means the on-device
integration test itself works — those are necessary, not sufficient,
verification for this specific file.
