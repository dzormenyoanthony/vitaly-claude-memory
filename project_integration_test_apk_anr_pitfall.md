---
name: project-integration-test-apk-anr-pitfall
description: launching the app from the emulator home screen after a flutter test integration_test/... run causes a real ANR — it's the leftover test-harness APK, not an app bug
metadata:
  type: project
  originSessionId: bc2f376b-e0e8-4af1-9c36-1db1ddb1e34f
  modified: 2026-08-30T04:49:45.992Z
---

`flutter test integration_test/<file>.dart -d <device>` installs a
special instrumentation APK whose entry point is the test harness
itself, not `lib/main.dart` — it's built to be driven by the
`flutter test`/`flutter drive` process, not launched standalone.

If you (or the user) tap the app icon on the emulator's home screen
afterward instead of relaunching via `flutter run`/`flutter install`,
the installed test APK has no driver to talk to and just sits there —
Android's ActivityManager eventually gives up waiting for it to draw a
first frame and shows a real ANR ("<app> isn't responding" / "Close
app" / "Wait"), which can persist for many minutes if the user taps
"Wait" repeatedly. `adb shell dumpsys package <id>` confirms this: the
installed package shows `DEBUGGABLE` and a `lastUpdateTime` matching
the integration-test run, not a normal `flutter run`.

This looks exactly like an app startup hang (indistinguishable from a
real "stuck at splash screen" bug from a screenshot alone) but isn't
one — confirmed 2026-08-30 by uninstalling and doing a clean `flutter
run --debug`, which launched normally with zero code changes.

**How to apply:** after any `flutter test integration_test/...` run on
a device/emulator, uninstall the app (`adb uninstall <applicationId>`)
before manually launching it from the home screen, or just relaunch via
`flutter run`/`flutter install` (which overwrites it with a normal
build). If a "stuck at splash" or ANR report ever comes in right after
an integration-test session touched the same device, check
`dumpsys package <id>` for `DEBUGGABLE` + a suspicious `lastUpdateTime`
before assuming it's a real regression — see also
[[project-integration-test-ondevice-stall]] for the separate (still
unresolved) issue of the integration test itself stalling when run
properly.
