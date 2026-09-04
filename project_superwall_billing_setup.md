---
name: project-superwall-billing-setup
description: "Superwall paywall integration + Play Console subscription product/base-plan IDs for Vitaly's premium gating"
metadata: 
  node_type: memory
  type: project
  originSessionId: d9366805-0d36-441c-9593-cc8eb49dfc41
  modified: 2026-09-04T02:00:17.826Z
---

**Feature Gating gotcha (2026-08-31)** — symptom reported: paywall shows,
user swipes back without buying, and still gets the premium feature.
This is NOT a code bug. `SuperwallPaywallService.gateFeature` calls
`Superwall.shared.registerPlacement(placement, feature: () { onAccessGranted(); })`,
and whether that `feature` closure runs on paywall **dismissal** is
controlled entirely by the paywall's **Feature Gating** setting in the
Superwall dashboard:

| Feature Gating | On dismiss without purchase |
| --- | --- |
| **Non Gated** (Superwall's default) | `feature` still runs → feature granted free |
| **Gated** | `feature` runs only with an active entitlement → dismiss leaves it locked |

Fix: in the Superwall dashboard set each paywall attached to
`scan_report` / `upload_pdf_report` / `export_report_data` to **Gated**
(paywall settings / campaign paywall config). No app rebuild — effective
on next paywall fetch. Also verify each placement is actually attached to
a paywall in a live campaign; an unattached placement likewise falls
through to `feature()` and runs free. "Show close button" / swipe-to-
dismiss is a separate paywall-design option — Gated is what enforces the
lock, set that first.

---

Superwall paywall integration shipped 2026-08-30 (commit `7759bcf`, pushed to `origin/master`). Gates three premium actions via `core/paywall/` (`PaywallService` interface + `SuperwallPaywallService` + `NoOpPaywallService` default, same pattern as `AnalyticsService`):

- `scan_report` — Scan BP Report (camera)
- `upload_pdf_report` — Upload/import PDF report
- `export_report_data` — Export Report/Data

**4th placement added later (post-onboarding promo, not a gate):**
`onboarding_complete` — fired from `main.dart`. Its `onAccessGranted` is a
no-op (the router already routes to the dashboard regardless). Must be
**Not Gated** in the dashboard (opposite of the three above), with a
non-subscriber audience rule on the campaign — Superwall itself is what
limits it to non-subscribers, not app code.

*Behavior changed 2026-09-04 (commit `553f480`).* Originally fired **at
most once** — on the `AuthGateNeedsOnboarding → AuthGateReady` transition
only, via a session-latch class `OnboardingCompletionDetector`. Symptom
that prompted the change: a fresh test user saw it once right after
onboarding, dismissed without buying, and never saw it again on later
opens — that was **the designed behavior at the time**, not a bug, but
the user wanted a recurring nudge instead. Now registers on every
`AuthGateReady` (every cold start with an existing session) **and** every
foreground resume while signed in — `OnboardingCompletionDetector` is
gone (deleted, was unused after the change). No dashboard change was
needed since the placement string didn't change, but **frequency
capping** on that campaign (e.g. "at most once per day") is worth setting
in the dashboard now that the app can register it many times per
session — that's config-only, no code change.

API key loads via `--dart-define-from-file=config/superwall.json` (gitignored; `config/superwall.json.example` is the checked-in template). Always pass that flag when building — omitting it silently falls back to the no-op service (paywall never shows, all premium actions run free), not an error.

**Play Console products** (set up 2026-08-30):
- Subscription base plan: `vitaly_premium_annual` / base plan ID `annual`
- Subscription base plan: `vitaly_premium_monthly` / base plan ID `monthly`
- `vitaly_premium_lifetime` / base name `lifetime` — confirmed created under Play Console's **In-app products** (one-time, non-consumable), not Subscriptions. Link it in Superwall's Products screen as a one-time product, not a subscription.

**Why:** [[project-superwall-billing-setup]] the app code never references these product/base-plan IDs directly — Superwall's own purchase controller (StoreKit2/Play Billing v6) talks to Play Billing, and which product a paywall sells is entirely Superwall-dashboard config (Products screen, linked to a paywall, which is in turn attached to one of the three placements above). RevenueCat was deliberately NOT introduced — no prior purchase infra existed in the repo, so Superwall's built-in purchase handling was used instead of adding a second subscription system.

**How to apply:** when discussing subscription pricing, paywall copy, or "why doesn't the paywall show the right price," remember the mapping lives in the Superwall dashboard (Products → linked to a paywall → attached to a placement), not in Dart code. Android minSdk was raised 24→26 and `androidx.activity` pinned to `1.9.3` in `android/build.gradle.kts` specifically because of `superwallkit_flutter`'s own packaging (see commit `7759bcf` message) — don't "fix" those thinking they're stray/accidental.
