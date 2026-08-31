---
name: feedback-test-fakes-resolve-too-fast
description: "This repo's in-memory fake repositories (FakeAuthRepository, FakeUserProfileRepository) resolve synchronously, so a regression test for a timing-dependent race can pass whether or not the fix is present unless the fake is deliberately slowed"
metadata: 
  node_type: memory
  type: feedback
  modified: 2026-08-27T02:54:55.997Z
  originSessionId: 1bb5e4d1-301a-4e29-b105-92b8a30f093d
---

While fixing the auth-gate redirect races on 2026-08-27
(see [[project-auth-gate-redirect-races]]), a widget-level regression test
for "sign-in shouldn't bounce back to Splash" passed identically whether
the actual fix was in `app_router.dart` or not. Root cause:
`FakeUserProfileRepository.watchProfile` (and the auth fake's stream) yield
their data essentially synchronously, so the real-world race — a brief but
nonzero `AuthGateLoading` window while a genuine Firestore listener spins
up — never manifests as an observable state in the fake at all. Riverpod's
provider graph can jump straight past the intermediate state within a
single synchronous re-evaluation.

**Fix applied:** wrapped the fake in a thin `UserProfileRepository`
decorator that adds a real `Future.delayed(30ms)` before forwarding to the
fake's stream (see `_DelayedProfileRepository` in
`test/core/router/app_router_flow_test.dart`), specifically to make the
`Loading` window wide enough to check against across several `tester.pump()`
checkpoints. For the profile-error grace-period test, a hand-rolled
`_ManualStreamProfileRepository` (backed by a plain `StreamController`) was
used instead, so an error event and a success event could be emitted on the
*same* subscription — `FakeUserProfileRepository`'s `async*` generator
terminates its stream on the first thrown error, which doesn't match how a
real Firestore listener keeps running after a transient error.

**How to apply:** whenever writing a regression test for a bug that's
fundamentally about *timing* (a race between two async sources, a
debounce/grace-period, a minimum-duration gate), don't trust that the test
is meaningful just because it's green — deliberately revert the fix
temporarily and confirm the test actually fails. If it doesn't, the fake is
almost certainly too fast/synchronous to reproduce the real-world timing,
and needs either an artificial delay wrapper or a hand-controlled stream
(`StreamController`) instead of the shared fake.
