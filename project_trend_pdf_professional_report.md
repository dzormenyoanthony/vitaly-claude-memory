---
name: project-trend-pdf-professional-report
description: "Trends PDF fully reworked into a clinician-shareable 'Blood Pressure Trend Summary' report — shipped 2026-09-03, commit 631b033"
metadata: 
  node_type: memory
  type: project
  originSessionId: 11254c09-6c91-4b58-a00e-5dfce1edeb53
  modified: 2026-09-05T00:13:41.472Z
---

`buildTrendSummaryPdf` (`lib/features/blood_pressure/presentation/
trend_pdf_export.dart`) was rebuilt (commit 631b033, 2026-09-03) into a
structured, professional report per a spec that was later found sitting
uncommitted in `export_data.md` (discarded 2026-09-05 as stale/duplicate
once confirmed already shipped — see [[project-trend-pdf-charts]] for the
earlier chart work this builds on).

Sections now present: branded header with reporting period, optional
patient-name block (profile name only, nothing else requested), summary
card grid, factual tracking-activity bullets (period count, previous-period
comparison, most recent reading), captioned BP + pulse charts with unit
legends, a full readings table (date/time/systolic/diastolic/pulse/source/
context/notes/linked report id) that repeats its header across page
breaks, a Supporting Documents index resolved against the Document Locker
(`SavedReport` list, only shown when a reading links to one), a "For your
healthcare professional" note, a privacy reminder, an expanded
non-diagnostic disclaimer, and a running header/footer with a generated
Report ID (`VITALY-YYYYMMDD-HHMMSS`) + page numbers.

**Wording fix:** the old on-PDF line "Average status (N readings): this
reading is high" (grammatically wrong — an average described as a single
reading) was replaced with neutral `trendReportContextLine`: "Context:
review this pattern with your healthcare professional." Pinned in
`test/l10n/medical_safety_wording_test.dart` (§37).

**Deliberately untouched:** the on-screen Trends card and
`trendSummaryLines` still use the old "Average status... this reading is
high" phrasing (`bpCategoryReadingIsHigh` / `trendSummaryAverageStatus`
ARB keys) — the rework was scoped to the PDF only, per explicit
instruction not to redesign app UI.

**Known limitation carried over:** built-in Helvetica (via the `pdf`
package) is Latin-1 only, so non-ASCII characters in user notes / report
titles won't render until a Unicode font is bundled.

**How to apply:** if asked to improve the Trends PDF further, check this
file's current state first — the "make it professional/shareable with a
clinician" ask is already fully implemented; don't rebuild from scratch.
