---
name: project-status
description: Current build phase status for Vitaly — what's done, what's next per PROJECT_SPEC.md, repo/git state
metadata:
  type: project
  originSessionId: cc450c07-d367-45ae-8c65-710f50a0bf4d
  modified: 2026-08-26T03:32:55.370Z
---

As of 2026-08-26, both halves of PROJECT_SPEC.md's "BP Report Scanning +
BP Reading Status Classification" section are implemented, on top of the
full MVP core (§4) and visual-design pass documented previously (see
[[project-design-references]]). `flutter analyze` is clean and the full
`flutter test` suite is green at 215 tests (was 156 as of the last
snapshot, then 197 after Phase 1 alone).

**Phase 1 — Scan BP Report & Saved Reports** (new `lib/features/reports/`):
camera scan via `google_mlkit_document_scanner` (Android-only, isolated
behind `DocumentScannerService`) or import an image/PDF via `file_picker`
(PDFs rasterized through the existing `printing` dependency, no new PDF
package); on-device OCR via `google_mlkit_text_recognition` (isolated
behind `TextRecognitionService`) — no cloud OCR, so no external-OCR privacy
review was needed; `BpValueExtractor` (pure, deterministic, unit-tested)
parses recognized text into candidate readings, always marked
`needsReview` per spec §14; `ReviewExtractedScreen` lets the user edit/
delete/add candidates and choose which to add to BP History; `SavedReport`
rows persist in a new Drift `SavedReports` table (schema v5) with
JSON-encoded extracted/confirmed reading lists, synced to Firestore
(metadata only) via a new `SyncCoordinator._syncReports` pass; original
page files live in local app-support storage
(`ReportDocumentStorage.saveLocalPages`, keyed by a random folder token,
not the DB id) and best-effort upload to Firebase Storage at
`users/{uid}/reports/{reportId}/...`. `SavedReportsScreen` (list/rename/
delete) and `ReportViewerScreen` (PageView + InteractiveViewer) round out
the feature. `Readings` gained `source` (`manual`/`importedReport`,
default manual) and `sourceReportId` columns (still schema v5) — confirming
a scanned reading calls the same `BloodPressureRepository.addReading` used
by manual entry, tagged with its origin; History and Reading detail show
an "Imported Report" tag when applicable.

**Phase 2 — BP Reading Status & Range Classification**
(`lib/features/blood_pressure/domain/bp_reference_ranges.dart`,
`bp_classification.dart`, `bp_classification_service.dart`): a single
centralized `BPClassificationService.classify(systolic, diastolic)` —
categorizes systolic and diastolic independently against
`BPReferenceRanges` (versioned, v1, exactly the four categories/thresholds
proposed in spec §19) and returns the higher-severity category (never
averages the two values). Verified against all 8 exact boundary values
spec §32 mandates, plus mixed-category/deterministic/average-pair cases.
A new `BPStatusColors` theme extension (`app_colors.dart`/`app_theme.dart`)
and shared `BPStatusBadge` widget
(`lib/features/blood_pressure/presentation/bp_status_badge.dart` —
deliberately NOT under `lib/core/widgets/`, since it depends on the
blood_pressure feature's domain model and core/ stays feature-agnostic
elsewhere in this codebase) render the status everywhere spec §35 requires:
Dashboard latest-reading card, History list rows, Reading detail (with a
"Why am I seeing this?" bottom sheet generated from the classification's
own data), Trends (a new `_AverageStatusCard` classifying the period
average, labeled "Average of N recorded readings" per §23, plus a
category-movement sentence from `trend_summary_lines.dart`'s
`categoryMovementLine` when the average's category actually changed vs.
the prior period), and the Trends PDF export (same shared
`trendSummaryLines` source, so the on-screen card and PDF can't drift
apart — though today only the PDF surfaces this text; the on-screen Trends
UI uses the stat-tile/badge layout instead).

**Why this was the next feature:** §14 previously blocked any BP
status/category display until this exact spec section was approved (it
was added mid-session on 2026-08-23, see the prior snapshot); this session
implemented it in the spec's mandated order (§36: Phase 1 fully
tested before Phase 2 began).

**Repo:** unchanged — `https://github.com/dzormenyoanthony/Vitality`
(private), Firebase project `vitality-23966`. New this session:
`storage.rules` (repo root) for the Firebase Storage bucket, and an added
`reports` subcollection block in `firestore.rules` — **both need manual
publish via the Firebase console** (no Firebase CLI/deploy access from
this session, same constraint as the existing `firestore.rules` workflow).
Cloud Storage itself may also need one-time enabling in the Firebase
console if it hasn't been activated for this project yet — unverified from
this session, flag to the user before relying on cloud sync of scanned
documents.

**Data model note:** Drift schema is now at v5. `Readings` gained `source`
(text, default `'manual'`) and `sourceReportId` (nullable int). New table
`SavedReports` (id, title, documentType, reportDate, pageCount, ocrStatus,
extractedReadingsJson, confirmedReadingsJson, source, localPagePaths,
storagePagePaths, createdAt/updatedAt, remoteId, deletedAt). Migration is
additive-only per the established pattern — no dedicated raw-SQL
migration-path test was written (none existed for v2-v4 either); coverage
relies on `flutter analyze`/codegen success plus CRUD tests against the
fresh (onCreate) schema.

**New dependencies added this session:** `google_mlkit_document_scanner`,
`google_mlkit_text_recognition` (pulls in `google_mlkit_commons`),
`file_picker`, `firebase_storage`, `path` (promoted from transitive to
direct since `report_document_storage.dart` calls it directly).

**Known environment pitfall found and fixed this session:** any code path
that opens a *new* Drift `.watch()` stream subscription for the first time
inside a widget-test interaction (e.g. `SettingsController.deleteAccount`
was calling `savedReportRepository.watchAll().first`) needs the
`FakeAsync`-driven test clock to advance before that stream's first
emission fires, so a short fixed `tester.pump(duration)` — or worse,
`pumpAndSettle()` — can time out or read stale state. Fix: use a one-shot
`Future`-returning `getAll()` method instead of `.watchAll().first` for
any one-time snapshot read (mirrors the pre-existing
`ReminderRepository.getAll()` pattern) — added `SavedReportRepository
.getAll()` for this reason. This is a variant of the already-documented
"Drift stream + Navigator/dialog transition hangs `pumpAndSettle()`"
pitfall noted in `app_router_flow_test.dart` and
`reading_detail_screen_test.dart` — same family of hazard, worth checking
for on any future controller method that both awaits a fresh stream
subscription AND is exercised via a widget-level (not just
`ProviderContainer`-level) test.

**Firestore/Storage rules: deployed.** The user had the Firebase CLI
available after all (contradicts the earlier "no CLI access" note — that
constraint no longer holds, re-check CLI availability fresh each session
rather than trusting this). Added `.firebaserc` (default project
`vitality-23966`) and a `firestore`/`storage` block in `firebase.json`
pointing at `firestore.rules`/`storage.rules`. Cloud Storage had never
been activated on the project — user enabled it via the console
("Get Started" at the Storage tab) mid-session. `firebase deploy --only
storage:rules` failed with "Could not find rules for the following
storage targets: rules" (a targeting-syntax quirk in this CLI
version, 15.28.1) — `firebase deploy --only storage` (no `:rules`
suffix) is the working form. Both rule sets are now live.

**On-device verification: performed this session, full pass, on a real
emulator (Pixel_8, cold-started clean this session).** This surfaced a
real Android-toolchain blocker unrelated to any app-code mistake: this
project's Gradle/AGP were already pinned to the bleeding-edge AGP 9.1.0 /
Gradle 9.3.1 (with AGP 9's new "built-in Kotlin" compiler) before this
session touched anything. Adding `firebase_storage` forced `firebase_core`
up to 4.14.0 (a hard lower-bound from `firebase_storage`'s own pubspec),
which exposed a confirmed, currently-open upstream bug: `firebase_auth`'s
(and five other Firebase Flutter plugins', including `firebase_storage`
itself) Android build still applies the classic `kotlin-android` Gradle
plugin, which conflicts with AGP 9's built-in Kotlin
(flutterfire#17987, closed as fixed 2026-03-03 in the monorepo, but no
version incorporating the fix has been published to pub.dev as of
2026-08-26 — `firebase_storage: 13.5.0`'s `android/build.gradle` line 49
still has `apply plugin: 'kotlin-android'`). User approved downgrading the
Android toolchain as the workaround. Chased through a real chain of
version floors before landing on a working combination — **the actual
constraint conflict** is that this Flutter SDK version (3.47.1) also
enforces ITS OWN minimum-version floors on the Gradle wrapper (>=8.14.0)
and Kotlin Gradle Plugin (>=2.2.20) even when targeting AGP 8.x, separate
from AGP's own stated minimums:

- AGP `8.13.0` (latest pre-built-in-Kotlin release, supports compileSdk up
  to 36.1 — Flutter 3.47.1's default `compileSdkVersion` is 36, so an
  older AGP 8.x wouldn't have supported it)
- Gradle `8.14.3` (Flutter's floor; AGP 8.13 itself only requires 8.13, so
  the higher requirement came from Flutter's own gradle-plugin-loader, not AGP)
- Kotlin Gradle Plugin `2.2.20` (same story — Flutter's floor, not AGP's)
- Explicit `id("org.jetbrains.kotlin.android")` added back to
  `android/app/build.gradle.kts`'s plugins block (needed under AGP 8.x
  since `MainActivity.kt` exists and built-in Kotlin isn't available pre-AGP-9)
- **A second, independent blocker**: the only JDK on this machine was a
  JBR 25.0.2 (Android Studio's bundled runtime), which Gradle 8.x cannot
  run under at all (needs Gradle >=9.1.0 to launch on JDK 25 — this is
  about the Gradle *daemon's own* JVM, unrelated to the project's Java
  17 source/target compatibility). Installed Eclipse Temurin 17 via
  `winget install --id EclipseAdoptium.Temurin.17.JDK` and pointed Flutter
  at it with `flutter config --jdk-dir="C:\Program Files\Eclipse
  Adoptium\jdk-17.0.20.101-hotspot"` — this is now the persistent default
  for this machine's Flutter tooling, not just this project.
- Also hit a real, unrelated resource-exhaustion problem while iterating
  through failed Gradle/JDK combos: each failed attempt left its own
  Gradle daemon alive (`org.gradle.jvmargs=-Xmx8G` per daemon on a 16GB
  machine), and the machine dropped to ~2GB free RAM, which made
  subsequent builds crawl or appear hung. Lowered
  `android/gradle.properties`'s `org.gradle.jvmargs` to `-Xmx3G` (with
  smaller metaspace/code-cache too) — if Gradle builds on this machine
  seem to hang again, check `Get-Process java` and free RAM before
  assuming a real build problem, and don't force-kill all `java`
  processes mid-build (killed one genuinely-succeeding build this way).

**Flutter build/run pattern for this project, now proven working**:
`flutter build apk --debug` (NOT `flutter run` — never tried, the
build+`android install --apks <path> --device emulator-5554`+
`adb shell am start -n com.vitality.app.vitality/.MainActivity` sequence
is what was actually used and verified) → `android screen capture` for
screenshots → `android layout -p` for exact tap coordinates (much more
reliable than eyeballing screenshot pixel positions — every misclick
during this session's manual testing came from skipping this step).
`am force-stop` + relaunch signs the session out (expected — "Keep me
signed in" was left unchecked on the test account). A `KEYCODE_BACK` from
the Saved Reports screen (reached via `context.go`, which replaces the
nav stack) exits the whole app rather than returning to Dashboard — use
the bottom-nav Home tab instead, not system back, when navigating away
from a `context.go`-reached screen.

**Bug found and fixed via this on-device pass**: `_ExtractedReadingCard`
in `review_extracted_screen.dart` had a real `RenderFlex` right-overflow
(visible as Flutter's yellow/black debug banner on-device, NOT caught by
any widget test — font-metric differences between the test harness and
real device rendering meant the overflow only manifested at actual device
width/font). The systolic/diastolic value `Text` + "Needs review" badge
`Container` were in a plain `Row` with no `Expanded`/`Flexible`; fixed by
switching that inner `Row` to a `Wrap` (`spacing`/`runSpacing`) so the
badge drops to its own line under tight width instead of overflowing.
Rebuilt, reinstalled, reverified visually — confirmed fixed, badge now
wraps cleanly. This is the only functional bug this pass found — the
entire scan → OCR → review → confirm → Saved Reports → Report Viewer →
BP History → Dashboard/History/Trends classification-badge pipeline
otherwise worked correctly end-to-end, verified visually at every step,
including the real Google Play Services on-demand "GmsDocumentScanning"
module download and the live-camera edge-detection scan UI (both fired
for real, not mocked).

**Latest commits:** still not committed as of this snapshot — all of
Phase 1, Phase 2, the Firebase rules deploy config, and the Android
toolchain downgrade are sitting in the working tree only; commit/push
were not requested this session (only on explicit ask, per CLAUDE.md
§16-17 and standing repo practice).

**How to apply:** At the start of a new session on this project, use this
to quickly reorient instead of re-deriving phase history from git log —
but verify against current `git log`/code state since this snapshot will
go stale. Do NOT assume "no Firebase CLI access" going forward — check
fresh. Do NOT assume the Android toolchain is still AGP 9/Gradle 9 if
`flutter pub upgrade` or a fresh `flutter create` template touches
`android/` again — re-check `android/settings.gradle.kts` before adding
any Firebase-family plugin, since the AGP-9-vs-classic-Kotlin-plugin
conflict will resurface for ANY of the six affected Firebase plugins
(`firebase_storage`, `firebase_analytics`, `firebase_performance`,
`firebase_remote_config`, `firebase_database`, `cloud_functions`) until
pub.dev ships versions with the fix already merged upstream.
