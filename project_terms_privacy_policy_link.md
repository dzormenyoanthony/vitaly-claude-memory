---
name: project-terms-privacy-policy-link
description: "Sign-up screen and Settings both now link to https://vitalyprivacy.netlify.app/ via url_launcher — shipped 2026-09-05, commits 2bc573e + fea1e0a"
metadata: 
  node_type: memory
  type: project
  originSessionId: 11254c09-6c91-4b58-a00e-5dfce1edeb53
  modified: 2026-09-05T01:56:46.105Z
---

The sign-up screen's "Terms" and "Privacy Policy" agreement text
(`sign_up_screen.dart`) was previously plain, non-tappable copy — Vitaly
had no published policy document to point to. The user supplied
`https://vitalyprivacy.netlify.app/` (a single combined Terms/Privacy
document) and both spans now open it via `url_launcher`
(`LaunchMode.externalApplication`), with a snackbar fallback
(`signUpLinkOpenError`) if launching fails.

Settings (`settings_screen.dart`) gained a matching **LEGAL** section
(between Subscription and Account) with a "Terms & Privacy Policy" row
doing the same, so the policy is reachable without going through
sign-up. New constant: `lib/core/constants/legal_links.dart` →
`LegalLinks.termsAndPrivacyUrl`, shared by both screens.

**New dependency:** `url_launcher` (dependencies) +
`url_launcher_platform_interface` (dev, for a `FakeUrlLauncher` at
`test/support/fake_url_launcher.dart` shared by both screens' widget
tests — avoids opening a real browser in tests).

**Verified live on the Pixel_8 emulator, 2026-09-05:** tapped both the
Settings row and the sign-up "Terms" span; Chrome opened to
`vitalyprivacy.netlify.app` showing the actual privacy policy page both
times. `flutter analyze` clean, hard-coded-string guardrail clean, full
suite 345/345 (was 344 before this).

**How to apply:** if the user ever changes the policy URL, it's a
one-line edit in `legal_links.dart` — both screens pick it up
automatically. See [[project-play-store-release-prep]] for the release
build (versionCode 15/1.0.4) that ships this.
