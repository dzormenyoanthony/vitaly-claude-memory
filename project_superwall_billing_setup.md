---
name: project-superwall-billing-setup
description: "Superwall paywall integration + Play Console subscription product/base-plan IDs for Vitaly's premium gating"
metadata: 
  node_type: memory
  type: project
  originSessionId: d9366805-0d36-441c-9593-cc8eb49dfc41
  modified: 2026-08-30T20:51:51.860Z
---

Superwall paywall integration shipped 2026-08-30 (commit `7759bcf`, pushed to `origin/master`). Gates three premium actions via `core/paywall/` (`PaywallService` interface + `SuperwallPaywallService` + `NoOpPaywallService` default, same pattern as `AnalyticsService`):

- `scan_report` — Scan BP Report (camera)
- `upload_pdf_report` — Upload/import PDF report
- `export_report_data` — Export Report/Data

API key loads via `--dart-define-from-file=config/superwall.json` (gitignored; `config/superwall.json.example` is the checked-in template). Always pass that flag when building — omitting it silently falls back to the no-op service (paywall never shows, all premium actions run free), not an error.

**Play Console products** (set up 2026-08-30):
- Subscription base plan: `vitaly_premium_annual` / base plan ID `annual`
- Subscription base plan: `vitaly_premium_monthly` / base plan ID `monthly`
- `vitaly_premium_lifetime` / base name `lifetime` — confirmed created under Play Console's **In-app products** (one-time, non-consumable), not Subscriptions. Link it in Superwall's Products screen as a one-time product, not a subscription.

**Why:** [[project-superwall-billing-setup]] the app code never references these product/base-plan IDs directly — Superwall's own purchase controller (StoreKit2/Play Billing v6) talks to Play Billing, and which product a paywall sells is entirely Superwall-dashboard config (Products screen, linked to a paywall, which is in turn attached to one of the three placements above). RevenueCat was deliberately NOT introduced — no prior purchase infra existed in the repo, so Superwall's built-in purchase handling was used instead of adding a second subscription system.

**How to apply:** when discussing subscription pricing, paywall copy, or "why doesn't the paywall show the right price," remember the mapping lives in the Superwall dashboard (Products → linked to a paywall → attached to a placement), not in Dart code. Android minSdk was raised 24→26 and `androidx.activity` pinned to `1.9.3` in `android/build.gradle.kts` specifically because of `superwallkit_flutter`'s own packaging (see commit `7759bcf` message) — don't "fix" those thinking they're stray/accidental.
