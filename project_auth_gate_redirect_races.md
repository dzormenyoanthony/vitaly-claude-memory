---
name: project-auth-gate-redirect-races
description: Four related router/auth-gate bugs found and fixed 2026-08-27 — Splash/name-screen/error-screen flashing during sign-up and sign-in, plus a stale-listener bug causing inconsistent "please try again" errors on repeat sign-ins
metadata: 
  node_type: memory
  type: project
  modified: 2026-08-27T03:19:27.383Z
  originSessionId: 1bb5e4d1-301a-4e29-b105-92b8a30f093d
---

On 2026-08-27, live on-device testing (by the user, driving the emulator
directly after I handed off) surfaced three related bugs in
`lib/core/router/app_router.dart` and `lib/core/router/auth_gate_provider.dart`,
all the same underlying class of problem: the router's `redirect:` reacted
too eagerly to **transient** states of `authGateProvider`, which briefly
passes through `AuthGateLoading` (and sometimes a spurious `AuthGateError`)
every time a *new* Firebase Auth uid starts being watched — i.e. on every
sign-up and every sign-in, not just at cold boot.

**Bug 1 — Splash never actually visible.** `authStateChanges()` can emit its
first "signed out" event before Flutter draws its first frame on a fresh
install, so the redirect carried the user straight past Splash into
Onboarding with ~0 visible duration. Fixed with
`lib/core/router/splash_min_duration_provider.dart` — a `Timer`-backed
`splashMinDurationElapsedProvider` that the redirect checks before allowing
any move off `/`. **Landed on 2 seconds** after the user asked to escalate
it 3s→5s→7s live; pushed back with the industry-standard splash-duration
range (~1.5–2.5s, citing Android's own guidance) instead of continuing to
comply with an ungrounded escalation — the user accepted 2s without further
objection. See [[feedback-grounded-pushback-on-ux-numbers]].

**Bug 2 — Sign-up flash.** `SignUpController.signUp()` creates the Firebase
Auth user first, then writes the Firestore profile as a separate awaited
step. In that gap `authGateProvider` legitimately reports `Loading` then
`NeedsOnboarding` (profile not written yet), and the redirect bounced the
user through Splash and the name-collection screen while they were still
sitting on `/sign-up` waiting on that same write. Fixed by checking
`pendingProfileNameProvider` (non-null only during that exact window,
cleared right after the write succeeds) and returning `null` (stay put)
for those two gate states while it's set.

**Bug 3 — Sign-in flash (two layers).**
- 3a: Firestore's `users/{uid}` rule requires `request.auth.uid == userId`.
  Right after sign-in, attaching a fresh profile listener can hit a
  transient `permission-denied` before the Firestore SDK's own auth token
  catches up with the just-completed sign-in — a known Firebase race, not a
  real failure. Riverpod's `StreamProvider` doesn't tear down its
  subscription after one error, so the same stream typically self-heals
  moments later. Fixed with an 800ms grace window
  (`profileErrorGraceElapsedProvider` in `auth_gate_provider.dart`): an
  error only escalates to user-facing `AuthGateError` if it's still present
  after the window elapses.
- 3b: even without any error, watching a brand-new profile stream is
  *always* momentarily `AuthGateLoading` (Riverpod's `StreamProvider`
  starts `Loading` synchronously on first watch), and the redirect was
  unconditionally forcing navigation to Splash for `AuthGateLoading`
  whenever the current location wasn't already Splash — hijacking away from
  `/sign-in` every single sign-in, not just during a race. Fixed by
  **simplifying `AuthGateLoading`'s case to always `return null`** (never
  force navigation for a transient loading tick) rather than adding another
  one-off guard. This is safe/sufficient because the min-duration check
  above already pins the user to Splash for the first ~2s at boot
  (`initialLocation` is `/`, and the gate starts `Loading` there); post-boot,
  a transient reload should just let whatever screen the user is on show its
  own loading state instead. This fix also made the sign-up-specific guard
  for `AuthGateLoading` (bug 2's fix) redundant, so it was removed — kept
  only for `NeedsOnboarding`, which still needs it.

**Bug 4 — the real explanation for "works for some existing accounts, not
others" (found after bug 3's grace-period fix still wasn't enough for some
real accounts).** `userProfileStreamProvider` (a `StreamProvider.family`,
not `autoDispose`) was never torn down on sign-out. Confirmed via logcat:
`Firestore: Listen for ... users/{uid} ... failed: PERMISSION_DENIED`
firing ~0.8s *after* `FirebaseAuth: Notifying ... about a sign-out event` —
the old user's listener kept running in the background after sign-out,
correctly got denied once `request.auth` no longer matched, and Riverpod
cached that error **permanently** on the (never-disposed) provider
instance. Signing back into that *same* account later in the session
reused the stale, already-errored instance instead of a fresh subscription
— explaining exactly why it was account-dependent (accounts touched
earlier in the session broke; untouched ones worked). Bug 3's 800ms grace
period couldn't help here because there was no self-heal coming — the
listener was genuinely dead. Fixed by marking both
`userProfileStreamProvider` and `profileErrorGraceElapsedProvider`
`autoDispose`, so each is cleanly cancelled/reset the moment nothing
watches it (right after sign-out) and rebuilt fresh on the next sign-in.
Regression test: `test/features/onboarding/data/user_profile_providers_test.dart`
watches a uid, unwatches (simulated sign-out), watches again (simulated
re-sign-in), and asserts the repository's `watchProfile` was called twice
— i.e. a genuine fresh subscription, not a cached one. Confirmed it fails
(only 1 call) without `autoDispose` and passes with it.

**Lesson for any `.family` provider keyed by uid/session-scoped data in this
codebase:** default to `autoDispose` unless there's a specific reason the
provider needs to outlive its last watcher. A plain `.family` provider is
effectively permanent once created — Riverpod won't recreate it just
because the underlying resource (a Firestore listener, in this case) has
become logically invalid.

**How to apply:** if a *new* class of "screen flashes during auth
transition" bug turns up (e.g. around Google sign-in, forgot-password, or
account deletion), suspect the same root pattern first: some transient
`authGateProvider` state is being reacted to as if it were a settled one.
Check whether the redirect is forcing navigation away from wherever the
user currently is, rather than letting that screen's own loading/error UI
carry the moment.

**Testing this class of bug:** [[feedback-test-fakes-resolve-too-fast]] —
the in-memory fakes resolve synchronously, so a naive regression test can
pass whether or not the fix is present. Verify any new test in this area
actually discriminates by deliberately reverting the fix and confirming the
test fails, the way this session did for both the profile-error grace test
and the sign-in Loading-redirect test (the latter needed a
`_DelayedProfileRepository` wrapper with an artificial 30ms delay before
the fake's own stream would even manifest an observable `Loading` window).

All four fixes verified via `flutter analyze` (clean) and `flutter test`
(223 passing, up from 220 — three new regression tests added) each round,
then confirmed by the user directly on-device (`emulator-5554`, debug APK)
after each fix, including creating a real throwaway Firebase test account
and cycling sign-in/sign-out across multiple existing real accounts.
User's final confirmation: "good everything works now." As of this
snapshot the fixes are uncommitted (only committed on explicit user ask,
per CLAUDE.md §16-17 and standing repo practice) — check `git log`/`git
status` fresh in any later session rather than assuming.
