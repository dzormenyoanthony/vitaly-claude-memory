---
name: project-account-deletion-reauth
description: Account deletion must re-auth before touching data; ordering is constrained and there is no perfect client-only sequence
metadata: 
  node_type: memory
  type: project
  originSessionId: a29e1d86-f38d-4c37-bd48-a119fb959688
  modified: 2026-08-31T05:50:13.234Z
---

`SettingsController.deleteAccount` (Vitaly commit `89c7670`, 2026-08-31)
was reworked after a real bug: it used to wipe all local data + the
Firestore profile first and call `user.delete()` last. Firebase rejects
`user.delete()` with `requires-recent-login` for any session older than
~5 min, so the normal outcome was **data gone, Firebase Auth user still
alive** — and signing back in (esp. with Google) then landed on a
profile-less half-broken account. Reported by the user as "deleted
account in the app, then Continue with Google doesn't work". See
[[project-google-sign-in]].

**Current order (do not casually reorder):**
1. `authRepository.reauthenticate()` — re-runs the Google flow for Google
   users; password/other providers no-op here and rely on step 3
   surfacing `ReauthRequiredFailure`. Throws `CancelledFailure` if the
   user backs out. **Nothing is destroyed if this throws.**
2. delete the Firestore profile **while still authenticated** (rules
   require `request.auth.uid == uid`), then `authRepository.deleteAccount()`
   (`user.delete()`).
3. wipe local device data (reminders + notifications, readings, report
   files) — wrapped in try/catch + `AppLogger.error`, never resurfaced:
   the account is already gone, a retry is meaningless.

**Why there is no perfect sequence:** the profile doc must be deleted
before the auth user (client loses Firestore write access afterward), but
`user.delete()` can still fail post-reauth (network) leaving the profile
gone + auth user alive. Residual risk is small and self-heals
(`AuthGateNeedsOnboarding` lets them re-onboard or retry). A Cloud
Function doing the whole teardown server-side is the only fully-atomic
fix — not built.

**Related changes in the same commit:**
- `requires-recent-login` now maps to `ReauthRequiredFailure` (new
  `Failure` subtype in `core/errors/failure.dart`), not a vague
  `ValidationFailure`; added a `user-mismatch` case.
- `FirebaseAuthRepository.signOut()` and the post-deletion path now clear
  the `google_sign_in` session (`GoogleSignIn.instance.signOut()` /
  `disconnect()`), so the next "Continue with Google" starts a fresh
  consent instead of reusing stale authorization.
- Delete-account UI (`settings_screen.dart`) suppresses the error
  snackbar when the failure is a `CancelledFailure`.

**How to apply:** the regression test
`test/features/settings/presentation/settings_controller_test.dart`
→ "deleteAccount touches nothing when re-authentication is required"
was sabotage-checked (fails if step 1 is moved after step 3). Keep the
re-auth call first. `FakeAuthRepository.simulateRequiresRecentLogin`
drives that test; the fakes are synchronous so this is an
ordering/logic test, not a timing one (cf.
[[feedback-test-fakes-resolve-too-fast]]).
