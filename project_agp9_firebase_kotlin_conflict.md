---
name: project-agp9-firebase-kotlin-conflict
description: AGP 9's built-in Kotlin conflicts with 6 Firebase Flutter plugins that still apply the classic kotlin-android plugin — unfixed on pub.dev as of 2026-08-26; this repo works around it by pinning AGP 8.13/Gradle 8.14.3/Kotlin 2.2.20 + a Java 17 JDK
metadata:
  type: project
  originSessionId: 2026-08-26-scan-report-classification
  modified: 2026-08-26T03:33:31.044Z
---

**The bug:** Android Gradle Plugin 9.0+ ships "built-in Kotlin" (no
separate Kotlin Gradle Plugin needed). Six Firebase Flutter plugins'
Android builds still explicitly `apply plugin: 'kotlin-android'`, which
conflicts with AGP 9's built-in compiler and produces confusing,
seemingly-unrelated compile errors (in this repo's case: a
`checker-qual`/`UnknownInitialization` "type annotation ... is
inaccessible" error inside `firebase_auth`'s
`IdTokenChannelStreamHandler.kt`, triggered merely by ALSO having
`firebase_storage` in the dependency graph — `firebase_storage` is one of
the six offenders and its presence alone was enough to break the shared
build, even though the error surfaced in a different plugin's file).

**Confirmed affected plugins** (per flutterfire maintainer comment on
issue #17987, 2026-03-03): `firebase_analytics`, `firebase_performance`,
`firebase_remote_config`, `firebase_database`, `cloud_functions` (applies
the plugin twice — a separate bug), and `firebase_storage`.

**Status as of 2026-08-26:** the fix (removing the explicit
`kotlin-android` application from all six) was merged and issue #17987
closed on 2026-03-03, but no pub.dev release of `firebase_storage` (still
at 13.5.0) or the others carries it yet — `firebase_storage-13.5.0`'s
`android/build.gradle` line 49 still has `apply plugin:
'kotlin-android'`. **Before re-attempting AGP 9 in this repo, check
whether a newer `firebase_storage`/`firebase_auth`/etc. release has
shipped the fix** (`flutter pub outdated`, or check the package's
`android/build.gradle` directly in `~/.pub-cache` for the `apply plugin:
'kotlin-android'` line) — if it's gone, AGP 9 should work again and this
whole workaround can be reverted.

**The workaround applied in this repo** (all under `android/`):
- `settings.gradle.kts`: `com.android.application` pinned to `8.13.0`
  (latest pre-built-in-Kotlin AGP release; needed for compileSdk 36
  support — earlier 8.x releases cap out lower) and
  `org.jetbrains.kotlin.android` pinned to `2.2.20`
- `app/build.gradle.kts`: added `id("org.jetbrains.kotlin.android")`
  explicitly to the `plugins {}` block (needed under AGP 8.x since
  `MainActivity.kt` exists and built-in Kotlin isn't available pre-AGP-9)
- `gradle/wrapper/gradle-wrapper.properties`: Gradle pinned to `8.14.3`
- `gradle.properties`: the `android.builtInKotlin`/`android.newDsl` flags
  (AGP-9-specific) were removed, but Flutter's own tooling re-adds them
  automatically on every `flutter build`/`flutter run` (harmless no-ops
  under AGP 8.x — don't fight this, it's expected)

**Why these exact versions, not just "any AGP 8.x + any Gradle 8.x"**:
Flutter SDK 3.47.1 (the version in use) enforces its OWN minimum-version
floors on the Gradle wrapper (>=8.14.0) and Kotlin Gradle Plugin
(>=2.2.20) via `dev.flutter.flutter-gradle-plugin`, independent of
whatever AGP itself requires — going below either floor fails with a
Flutter-tooling error (not a Gradle/AGP error) even though AGP 8.13 alone
would be satisfied by much older Gradle/Kotlin. Confirm current floors
with `flutter build apk --debug` — it prints the exact required minimum
in the error message if a floor isn't met, so re-derive fresh rather than
assuming these numbers stay correct after a Flutter upgrade.

**A second, independent prerequisite**: Gradle 8.x cannot launch its own
daemon under a Java 25 JDK (needs Gradle >=9.1.0 for that specifically —
this is about the Gradle daemon's host JVM, unrelated to the project's
Java 17 source/target compatibility setting). If `flutter build apk`
fails with "Gradle build failed due to Java/Gradle incompatibility" citing
a JDK version in the 20s, a JDK 17 or 21 install is needed alongside the
AGP downgrade, not instead of it. See [[project-status]] for how that was
installed on this machine (Temurin 17 via winget, then `flutter config
--jdk-dir=...`) — that `flutter config` setting is global to this
machine's Flutter installation, not per-project, so it persists across
projects too.

**How to apply:** If a future session hits a Kotlin/checker-qual/guava
compile error immediately after adding or upgrading any Firebase Flutter
plugin, check this memory before spending time on guava/checker-qual
version-forcing — it's very likely this exact AGP-9-vs-kotlin-android
conflict, not a dependency version clash, and the fix is either (a) pin
the toolchain down as described here, or (b) check if pub.dev has caught
up and just needed a version bump.
