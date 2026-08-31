---
name: project-play-store-release-prep
description: Android release-signing, INTERNET permission, and launcher-label fixes for Play Console readiness (2026-08-30) — what's done, what's still outstanding
metadata:
  type: project
  originSessionId: bc2f376b-e0e8-4af1-9c36-1db1ddb1e34f
  modified: 2026-08-31T04:20:31.183Z
---

On 2026-08-30, at the user's request, did a Play Console release-
readiness pass on the Android build config only (no Flutter UI touched).
Used EnterPlanMode/ExitPlanMode for approval before editing, per the
user's explicit request for a plan-first workflow.

## Two real blockers found and fixed

1. **No release signing config** — `android/app/build.gradle.kts` was
   still signing release builds with the debug keystore (unedited
   Flutter template). Generated a new upload keystore
   (`android/app/upload-keystore.jks`, gitignored) via `keytool`, added
   `android/key.properties` (gitignored) and a `signingConfigs.release`
   block that loads from it, with a fallback to the debug key if
   `key.properties` is absent (so `flutter run --release` still works
   on a machine without the keystore). SHA-1:
   `EA:AD:F0:52:8D:E6:3D:B1:D5:1A:28:78:19:1F:15:8B:05:42:75:E5`,
   SHA-256:
   `7C:1C:27:B2:A2:00:40:B5:3C:54:1B:24:B3:55:67:91:A8:74:8A:91:67:D9:2F:69:2A:F4:9D:8C:F4:E3:5B:14`.
   The password (store and key, same value) was shown to the user once
   in the session transcript and is NOT saved anywhere by me — the user
   is responsible for backing up the `.jks` + password themselves; if
   lost, the app can never be updated under that Play Store listing.

2. **No `INTERNET` permission in the main manifest** — Flutter's
   template only grants it to the debug/profile source sets (for the
   dev-tooling VM service). Every Firebase call (Auth, Firestore,
   Storage, Analytics) and Google Sign-In would have silently had zero
   network access in the actual release build shipped to Play Console —
   no crash, just broken auth/sync from first launch. Added
   `<uses-permission android:name="android.permission.INTERNET"/>` to
   `android/app/src/main/AndroidManifest.xml`. Verified present in the
   real merged release manifest
   (`build/app/intermediates/packaged_manifests/release/...`).

## Also fixed

- Launcher label `android:label="vitality"` → `"Vitaly"` (matches
  in-app `AppLocalizations.appTitle`; was still the unedited Flutter
  default).
- Discovered during verification: current Flutter tooling enables R8
  minification by default for release builds even with no
  `minifyEnabled` line present (silence no longer means "off"). A first
  `flutter build appbundle --release` failed R8 with "Missing class"
  errors for `google_mlkit_text_recognition`'s optional CJK/Devanagari
  recognizer classes (not in the actual dependency tree). Per the
  user's explicit choice (asked via AskUserQuestion) to skip
  minification for this first release, added explicit
  `isMinifyEnabled = false` / `isShrinkResources = false` to the release
  buildType to force it off. Revisit post-launch with proper keep rules
  if a smaller build is wanted later.
- `.gitignore` gained `/android/key.properties`,
  `/android/app/*.jks`, `/android/app/*.keystore`.

## Verified

- `flutter analyze` — clean
- `flutter build appbundle --release` — succeeds,
  `build/app/outputs/bundle/release/app-release.aab` (86.0MB)
- `jarsigner -verify -verbose -certs` on the AAB confirms every entry is
  signed with the new upload key (`CN=Vitaly, OU=Development, O=Vitaly,
  L=Unknown, ST=Unknown, C=US`), not the debug key
- Merged release manifest inspected directly for `INTERNET` + label +
  versionCode/versionName/minSdk/targetSdk

Changes were left uncommitted at the user's option (three files:
`android/app/build.gradle.kts`, `android/app/src/main/AndroidManifest.xml`,
`.gitignore`) — check `git status` before assuming they're pushed.

## Confirmed OK, deliberately not changed

- `applicationId`/`namespace` `com.vitality.app.vitality` — unusual-
  looking (repeats "vitality") but functional and already wired to the
  existing Firebase Android app registration; changing it now would be
  destructive (new Firebase app registration, existing installs treated
  as a different app). Flagged for the user's explicit sign-off, not
  auto-changed.
- `versionCode`/`versionName` `1.0.0+1` — fine as a first release.
- AGP `8.13.0`/Kotlin `2.2.20`/Gradle `8.14.3` pin — see
  [[project-agp9-firebase-kotlin-conflict]], still the right combination.

## Outstanding — non-code, needs the user directly

1. ~~Register the upload keystore's SHA-1 in Firebase Console~~ **DONE**
   — verified 2026-08-31: `google-services.json` contains an Android
   OAuth client for the upload key's SHA-1
   (`eaadf0528de63db1d51a2878191f158b054275e5`). See
   [[project-google-sign-in]].
2. ~~Register the Play App Signing SHA-1 in Firebase~~ **DONE** —
   verified 2026-08-31: client for
   `111c2de2fd3a660a11fa717ddf1d29a5be3f12de` present and matches the
   SHA-1 shown in Play Console → Setup → App signing. All three
   fingerprints (debug, upload, Play App Signing) now confirmed.
3. **App icon** — still Flutter's default placeholder logo
   (`android/app/src/main/res/mipmap-*/ic_launcher.png`). Deliberately
   not generated by me (real design asset, out of scope per the user's
   explicit choice). Blocks a credible store listing.
4. Play Console Data Safety form + privacy-policy URL (health data).
5. Confirm Email/Password + Google auth providers are enabled on the
   production Firebase project (can't check from this session).

## How to apply

Use this to answer "what's left before Play Console submission?"
without re-auditing the Android build config from scratch. The two real
code/config blockers are fixed; everything remaining is either a manual
console step or a design asset. Re-run `git status` first since the
three changed files may still be uncommitted.
