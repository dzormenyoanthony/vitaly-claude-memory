---
name: project-i18n-externalization
description: "PROJECT_SPEC.md §36 fully implemented 2026-08-29 — gen_l10n infra, all UI strings externalized to app_en.arb, locale-aware formatters, medical-safety wording guard, hard-coded-string lint"
metadata: 
  node_type: memory
  type: project
  originSessionId: 9b40e65e-c3a7-4aaf-ae18-ab12e9a491fa
  modified: 2026-08-29T03:44:20.517Z
---

Delivered 2026-08-29 in 13 commits on `master` (`3170c9e..22bb0f3`, pushed
to origin). Motivated by the [[project-spec-gap-audit]] which flagged §36
as the widest divergence. Scope was locked via AskUserQuestion: mechanical
externalization English-only now (translation later), feature-by-feature
commits on master, CI grep guardrail, article bodies stay in the data
layer.

## What shipped

- **Infra:** `flutter_localizations` + `intl` (direct), `generate: true`,
  `l10n.yaml`, `lib/l10n/app_en.arb` (~345 keyed messages with `@`
  metadata, ICU plurals + `select` where needed). Generated
  `app_localizations*.dart` are committed. Delegates wired into both
  `MaterialApp.router` and the startup-failure `MaterialApp`.
- **Formatting:** `lib/core/i18n/formatters.dart` wraps `DateFormat`/
  `NumberFormat` skeletons resolved against `Localizations.localeOf`. All
  8 hardcoded English month/weekday tables deleted. 24-hour clock kept
  (`DateFormat.Hm`) to match the mockups; date field order now follows
  locale (en_US shows "Aug 22" / "August 21, 2026", a visible change from
  the old British-style ordering — expected, needed for real l10n).
- **Strings:** every screen cluster externalized — splash/nav/errors/
  Settings, auth, onboarding, dashboard, blood_pressure (record/history/
  detail/trends), reminders, reports, education chrome. Enum-label
  extensions (`MeasurementContext`/`BodyPosition`/`CuffArm`,
  `HistoryFilter`, `ReportCategory`, `ArticleCategory`, `BPCategory`) now
  take an `AppLocalizations`. Form validators (`ReadingValidator`,
  `CredentialsValidator`) take `l10n`; call sites pass
  `(v) => Validator.validateX(l10n, v)`.
- **Medical-safety wording:** `bp_classification`, `trend_summary_lines`,
  `trend_pdf_export`, `logging_insight` (now carries a `kind` + day
  counts, never a reading value) all externalized verbatim.
  `test/l10n/medical_safety_wording_test.dart` pins the exact English of
  §12–14/§21/§24/§27/§29 strings so §36 can't silently reword them;
  changing them still needs the §37 review.
- **Guardrail:** `tool/check_hardcoded_strings.dart` fails on a bare
  user-facing literal (`Text` / `TextSpan.text` / `labelText` / `hintText`
  / `tooltip` / `semanticLabel` / `Semantics.label`) in `lib/` outside a
  small allowlist. Currently passes clean. Documented in CLAUDE.md §15 +
  `tool/README.md`. **No CI pipeline exists** — it's a manual `dart run`
  step for now.
- **Tests:** widget tests get the delegates via
  `test/support/pump_app.dart` (`pumpApp`, `localizationWrappers`,
  `loadAppLocalizations`). Suite is 264 tests, green.

## Deliberately left English (allowlisted, each with an in-code comment)

- `bp_readings_csv.dart` — the export CSV is an interchange format with
  §28-pinned English headers, not localized UI.
- `article_repository.dart` — the 7 full articles are data-layer content
  (still need approved-source review per §15/§37 before any change).
- `failure.dart` + `firebase_auth_repository.dart` /
  `fake_auth_repository.dart` — backend/auth error strings mapped from
  provider error codes. **Follow-up:** localizing these cleanly needs an
  error-code→key refactor at the ~15 `friendlyMessage(error)` display
  sites; not done in this pass.
- `formatFileSize` in `saved_reports_screen.dart` keeps literal "KB"/"MB"
  (units are constant across locales, §36).

## How to apply

Adding a real locale is now just `lib/l10n/app_xx.arb` + a `Locale` in
`supportedLocales`. Any new user-facing copy must be an ARB key accessed
via `AppLocalizations.of(context)` (or `l10n` threaded into a domain
helper); run `flutter gen-l10n` after editing the ARB and
`dart run tool/check_hardcoded_strings.dart` before calling a task done.
The `formatters_test.dart` en_GB case proves the formatter layer follows
the active locale.
