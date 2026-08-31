---
name: project-google-sign-in
description: Real Google Sign-In wired up 2026-08-27; broke for Play-Store users until the Play App Signing DEPLOYMENT cert SHA-1 was registered 2026-08-31 (commit 77d6e63)
metadata: 
  node_type: memory
  type: project
  modified: 2026-08-31T05:35:39.749Z
  originSessionId: 1bb5e4d1-301a-4e29-b105-92b8a30f093d
---

**2026-08-31 — real bug: Google Sign-In failed for everyone who
installed from the Play Store.** Debug builds and upload-key-signed
sideloads worked; anyone on a Play-distributed build got a developer
error. Root cause: Play App Signing does **not** use one certificate. Its
"App signing key" download (Play Console → Setup → App signing → download
certificates → `certificates.zip`) contains **three** Google-issued
`.der` certs, and only one of them signs the APKs delivered to devices:

| cert file in the zip | SHA-1 | role |
| --- | --- | --- |
| `deployment_cert.der` | `d26cd40e323c676bd2faa12b5d08f5cb964496d6` | **the one that signs delivered APKs — the one that matters** |
| `hybrid_classical_cert.der` | `111c2de2fd3a660a11fa717ddf1d29a5be3f12de` | classical half of the hybrid/PQC signing scheme |
| `hybrid_pqc_cert.der` | `4eb29214e4063ca455a3214ecdbd963d7d79a795` | post-quantum half |

Earlier in the same session the `111c2de2…` (hybrid_classical) hash had
been registered in Firebase, on the mistaken belief it *was* the Play
App Signing cert — it is not. `deployment_cert.der`'s SHA-1 was never
registered, so Google rejected every sign-in from a Play build.

**Fix:** added `deployment_cert.der`'s SHA-1 **and** SHA-256
(`8F:EE:46:F8:06:E3:9C:60:3D:7D:2D:B8:BC:4B:32:2F:DD:24:FE:13:63:EB:D9:F2:78:C5:C3:2D:3A:0B:65:E5`),
plus the PQC cert, in **Firebase Console → Project Settings → Android app
→ Add fingerprint**, re-downloaded `google-services.json`, committed as
`77d6e63`. The updated JSON now carries OAuth clients for all five
signing identities: deployment `d26cd40e…`, PQC `4eb29214…`,
hybrid-classical `111c2de2…`, upload `eaadf0528…`, debug `a250a61a…`,
plus the type-3 web client `749553349568-27m2of1t…`.

**No new app build was needed** — the extra `client_type:1` entries in
`google-services.json` are pure server-side registrations; the
`google-services` Gradle plugin does not compile Android OAuth client
hashes into app resources (only the web client → `default_web_client_id`,
sender id, api key, app id — none of which changed). The committed JSON
produces a byte-identical AAB. The fix takes effect once Firebase
propagates the fingerprint; the build already on Play works as-is.

`minSdk` is `maxOf(flutter.minSdkVersion, 26)` (raised past
`google_sign_in_android`'s 24 floor by `superwallkit_flutter`). keytool
passwords for the local keystores come from `android/key.properties`
(gitignored).

**How to apply — Play App Signing SHA-1 registration:** never assume Play
App Signing is a single cert. Download `certificates.zip` from Play
Console → Setup → App signing and register the SHA-1 **and SHA-256** of
`deployment_cert.der` (the delivered-APK signer) in Firebase; register
the hybrid/PQC certs too. `keytool -printcert -file deployment_cert.der`
prints both hashes. Reading a single SHA-1 off the App-signing page can
easily be the wrong one of the three.

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
without needing to run the app at all. If it fails **only for Play-Store
installs** while debug/sideload works, it's the Play App Signing
deployment cert — see the 2026-08-31 section above.
