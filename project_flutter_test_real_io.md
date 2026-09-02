---
name: project-flutter-test-real-io
description: "Under `flutter test`, real dart:io file I/O completions are not delivered between tester.pump() calls — a real File.copy() inside an awaited controller path hangs it in AsyncLoading forever; stub/inject all filesystem I/O in widget tests"
metadata: 
  node_type: memory
  type: project
  originSessionId: 247eb89e-4bcc-4af7-b068-92b57033f42c
  modified: 2026-09-02T03:21:28.685Z
---

Discovered 2026-09-02 while trying to run `integration_test/
scan_report_pipeline_test.dart` (real `ReportDocumentStorage`) as a
`flutter test` widget test instead of on-device.

**The rule this repo already follows implicitly:** widget/unit tests
must **not** do real filesystem I/O. `review_extracted_screen_test.dart`
fakes `ReportDocumentStorage`; `data_export_service_test.dart` injects a
file-reader; etc. That is not just tidiness — real I/O actively breaks
`flutter test`:

- **Real `await File(src).copy(dst)` inside an awaited path never
  completes.** `flutter test`'s `TestWidgetsFlutterBinding` drives time
  with `tester.pump()`, and real `dart:io` async completions are not
  delivered between pumps. Concretely: `ConfirmReportController
  .confirmAndSave` awaits `storage.saveLocalPages`; with a real
  `File.copy` there, the controller stays in `AsyncLoading` for the whole
  test (the Confirm button just spins), `context.go` never fires, and the
  test fails on the missing next screen — no error, no hang message, just
  a stuck future.
- **Mocking `PathProviderPlatform.instance` (with
  `MockPlatformInterfaceMixin`) so the *real* `ReportDocumentStorage`
  runs against real temp dirs is worse — it hard-hangs** the test to the
  10-minute `testWidgets` timeout (`dart:isolate
  _RawReceivePort._handleMessage` on the stack), and `tearDown`'s
  `deleteSync` then fails with "file used by another process".

**What to do instead:** in `flutter test`, override
`reportDocumentStorageProvider` (or equivalent) with a pass-through /
in-memory fake — no `File`, no `path_provider`. Real
`ReportDocumentStorage.saveLocalPages` / real `path_provider` can only be
exercised in `integration_test/` (real event loop). On this machine that
path is itself blocked — see [[project-integration-test-ondevice-stall]]
(the on-device run freezes in the Drift writes right after Confirm).

**Net coverage for the scan pipeline:** `test/features/reports/
presentation/scan_report_pipeline_test.dart` (Vitaly commit c4d1d14)
covers OCR-text -> extract -> review -> confirm -> real Drift writes ->
navigate, headless and green, with storage stubbed. The real page-file
copy is only in the (blocked) integration test.

Related: [[project-flutter-test-drift-stream-pitfalls]],
[[feedback-test-fakes-resolve-too-fast]].
