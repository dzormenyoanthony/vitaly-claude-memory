---
name: project-superwall-billing-setup
description: "Superwall paywall integration + Play Console subscription product/base-plan IDs for Vitaly's premium gating"
metadata: 
  node_type: memory
  type: project
  originSessionId: d9366805-0d36-441c-9593-cc8eb49dfc41
  modified: 2026-09-04T02:26:53.349Z
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
needed since the placement string didn't change.

**Confirmed in the dashboard (2026-09-04, via `claude-in-chrome`)**: the
"onboarding" campaign's "Non-subscribers" audience already had a
**Limit: up to 1 time every 1 day** set — this was already correctly
configured *before* the app-side fix, it just didn't matter because the
old app code only ever sent the registration once, ever. So the fix +
existing dashboard limit together now do the right thing (shows at most
once/day to a non-subscriber) with no further dashboard change needed.
Dashboard path: `superwall.com` → Vitaly app → **Campaigns** → "onboarding"
→ "Non-subscribers" audience → **Limit** section.

**Verified live end-to-end (2026-09-04, Pixel 8 emulator)**: cold-started
the app on an already-onboarded, non-subscribed account (past the old
one-time-only window) and the real paywall ("trial-focused", GHS pricing,
Skip button) actually rendered on screen. Confirmed in logcat too — a
real `paywall_open` event with `presented_by_event_name=onboarding_complete`,
`feature_gating="NON_GATED"`, timestamp matching the cold start exactly.
Full loop confirmed: app fires it → Superwall's dashboard rules (audience
+ limit) approve it → a real paywall renders.

**Testing gotcha that nearly produced a false result**: first pass at
this verification used a plain `flutter run -d <device> --debug` with no
API key flag — silently fell back to `NoOpPaywallService`, so nothing was
ever sent to Superwall and the test uid never appeared in Superwall's
Users list at all (checked via `claude-in-chrome` against the live
dashboard — that absence was the tell). **Any on-device verification of
paywall behavior must use
`flutter run --dart-define-from-file=config/superwall.json`** (see below)
or it silently tests the no-op stub instead of the real integration —
looks identical to a passing test until you go looking for the uid on
the dashboard and it isn't there.

API key loads via `--dart-define-from-file=config/superwall.json` (gitignored; `config/superwall.json.example` is the checked-in template). Always pass that flag when building — omitting it silently falls back to the no-op service (paywall never shows, all premium actions run free), not an error.

**Play Console products** (set up 2026-08-30):
- Subscription base plan: `vitaly_premium_annual` / base plan ID `annual`
- Subscription base plan: `vitaly_premium_monthly` / base plan ID `monthly`
- `vitaly_premium_lifetime` / base name `lifetime` — confirmed created under Play Console's **In-app products** (one-time, non-consumable), not Subscriptions. Link it in Superwall's Products screen as a one-time product, not a subscription.

**Why:** [[project-superwall-billing-setup]] the app code never references these product/base-plan IDs directly — Superwall's own purchase controller (StoreKit2/Play Billing v6) talks to Play Billing, and which product a paywall sells is entirely Superwall-dashboard config (Products screen, linked to a paywall, which is in turn attached to one of the three placements above). RevenueCat was deliberately NOT introduced — no prior purchase infra existed in the repo, so Superwall's built-in purchase handling was used instead of adding a second subscription system.

**How to apply:** when discussing subscription pricing, paywall copy, or "why doesn't the paywall show the right price," remember the mapping lives in the Superwall dashboard (Products → linked to a paywall → attached to a placement), not in Dart code. Android minSdk was raised 24→26 and `androidx.activity` pinned to `1.9.3` in `android/build.gradle.kts` specifically because of `superwallkit_flutter`'s own packaging (see commit `7759bcf` message) — don't "fix" those thinking they're stray/accidental.
