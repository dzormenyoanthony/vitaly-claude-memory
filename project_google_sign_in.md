---
name: project-google-sign-in
description: Real Google Sign-In wired up and verified live 2026-08-27; all 3 signing fingerprints (debug, upload, Play App Signing) verified against google-services.json 2026-08-31
metadata: 
  node_type: memory
  type: project
  modified: 2026-08-31T04:20:13.080Z
  originSessionId: 1bb5e4d1-301a-4e29-b105-92b8a30f093d
---

**2026-08-31 re-verification** — all three Android OAuth clients in
`android/app/google-services.json` now confirmed:

| `certificate_hash` | Key | How verified |
| --- | --- | --- |
| `eaadf0528de63db1d51a2878191f158b054275e5` | `android/app/upload-keystore.jks` (alias `upload`) | `keytool -list -v` locally |
| `a250a61a659b1af9f22ac10b2b52b17c03078a93` | `~/.android/debug.keystore` (alias `androiddebugkey`) | `keytool -list -v` locally |
| `111c2de2fd3a660a11fa717ddf1d29a5be3f12de` | Play App Signing key | user read SHA-1 off Play Console → Setup → App signing |

Web client (type 3) `749553349568-27m2of1t…` present for
`default_web_client_id`. `minSdk` is now `maxOf(flutter.minSdkVersion, 26)`
(raised past 24 by `superwallkit_flutter`). The release/app-signing SHA-1s
were added in commit `5929e12`. Debug + upload paths were already proven;
Play-distributed builds are now also fully backed — the earlier
"unverified third fingerprint" caveat is cleared. keytool passwords come
from `android/key.properties` (gitignored, local).

---

Google Sign-In ("Continue with Google" on Sign In and Create Account) is
now fully working, verified live end-to-end by the user on their own real
device with a real Google account, and committed/pushed (`7cf1ee1`, merged
with an unrelated remote README commit into `0010273` on `origin/master`).

**The code was already complete before this session touched it** —
`FirebaseAuthRepository.signInWithGoogle()`, `GoogleSignInController`, the
button wiring on both screens, and routing a first-time Google user
through `AuthGateNeedsOnboarding` → `OnboardingCompleteProfileScreen` (name
collection) → Dashboard were all already correct. It didn't work purely
because of two external gaps:

1. **Firebase Console**: `android/app/google-services.json` had
   `"oauth_client": []` — completely empty. No Google sign-in provider was
   enabled in Firebase Authentication, and no Android SHA fingerprint was
   registered, so no OAuth Web Client existed for the project. Fixed by
   the user enabling Google as a provider and registering the debug
   keystore's SHA-1 (`A2:50:A6:1A:65:9B:1A:F9:F2:2A:C1:0B:2B:52:B1:7C:03:07:8A:93`)
   and SHA-256 in Project Settings, then re-downloading
   `google-services.json` (now contains an Android client type=1 matching
   that SHA-1, and an auto-created Web client type=3).
2. **`android/app/build.gradle.kts`**: `minSdk` was 23 (the `firebase_auth`
   floor); `google_sign_in_android` requires 24 for its Credential
   Manager-based flow. Bumped the `maxOf(...)` floor to 24.

No client ID needed to be hardcoded in Dart — per
`google_sign_in_android`'s own README, when using `google-services.json` +
Gradle registration, the plugin auto-reads the web client from the
generated `default_web_client_id` resource, "as long as your
`google-services.json` contains a web OAuth client entry."

**Also fixed as part of this work**: cancelling the account picker used to
show a red "Sign-in was cancelled." error banner (technically not a crash,
but a false error for a normal user action). Added `CancelledFailure` (a
new `Failure` subtype in `core/errors/failure.dart`) and had
`GoogleSignInController` reset to idle silently instead of surfacing it —
see [[project-auth-gate-redirect-races]] for the broader pattern of this
session's auth-flow UX fixes (different root causes, same spirit: don't
let a normal/transient condition read as an error to the user).

**On-device verification method worth remembering**: since a full
successful Google sign-in requires real account credentials I don't have
and won't fabricate, the cancellation path was instead verified by
deliberately backing out of the real Credential Manager flow and reading
logcat directly — `CredentialManager: finishing session with
propagateCancellation false` plus the *absence* of any `AppLogger.error`
Flutter-tagged log line confirmed the app took the intended `canceled`
code path cleanly, without needing a real account to prove the error
handling itself was correct. The actual successful-sign-in leg was left
for the user to verify on their own device/account, which they then did
directly ("EVERYTHING WORKS").

**How to apply:** if Google Sign-In ever needs re-verifying (a new
machine, a release build, a different Firebase environment), check
`google-services.json`'s `oauth_client` array first — an empty array is
the single most likely cause of a silent failure, and is diagnosable
without needing to run the app at all.
