---
name: project-preauth-light-theme-lock
description: "Onboarding, sign up, sign in, forgot password, and name-collection screens are permanently locked to light theme, regardless of system/app dark mode — only Dashboard-onward follows theme preference"
metadata: 
  node_type: memory
  type: project
  modified: 2026-08-28T03:20:05.496Z
  originSessionId: 41ef1397-50e4-4ce4-9ff0-2d6578bd9f49
---

Explicit product direction (2026-08-28): the entire pre-auth flow must
render in light theme at all times, even when the device/app is set to
dark mode. Only screens reached after auth (Dashboard, History, Trends,
Settings, etc.) should follow the user's actual theme preference
(`themeModeProvider` — System/Light/Dark).

**Implementation:** a `_lightLocked(Widget child) => Theme(data:
AppTheme.light(), child: child)` helper in
`lib/core/router/app_router.dart`, wrapped around the `builder:` for
exactly these routes: `AppRoutes.signIn`, `signUp`, `forgotPassword`,
`onboarding` (the pre-auth carousel, which internally also hosts the name
step), and `onboardingProfile` (`OnboardingCompleteProfileScreen`, the
post-signup name-completion screen for Google sign-in users). This is a
router-level fix, not per-widget color patches — it forces
`Theme.of(context)` (including the `AppAccentColors`/`BPStatusColors`
extensions) to resolve to the light theme throughout each screen's whole
subtree, so no individual widget in those screens needs its own
light/dark branching.

**Why this approach over patching individual colors:** initially started
auditing `sign_in_screen.dart` for specific dark-mode contrast bugs
(e.g. the Google button's fixed-white background against
`theme.colorScheme.onSurface` text turning invisible in dark mode, and a
hardcoded `AppColors.onboardingChipUnselected` pale-mint field fill not
inverting). The user then redirected mid-task: rather than fixing colors
screen-by-screen, lock the whole pre-auth flow to light unconditionally.
This is both simpler and more robust — it also fixes latent bugs that
hadn't been individually diagnosed yet (e.g. text inside onboarding's
fixed-light background screens that would have inherited dark-mode
`textTheme` colors).

**Verified on-device**: with the emulator's system dark mode forced on
(`adb shell cmd uimode night yes`), Sign In, Sign Up, and the name-entry
screen all rendered correctly in light theme; Dashboard and Settings (set
to "Dark" in-app) correctly rendered in dark theme, confirming the split
works as intended.

**How to apply:** Any new pre-auth screen added to the auth/onboarding
flow in the future should also be wrapped in `_lightLocked(...)` at its
route registration in `app_router.dart`, consistent with this decision.
