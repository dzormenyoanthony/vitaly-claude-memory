---
name: project-data-export-feature
description: "Full BP data export (CSV + scanned reports) as a ZIP, shared via native share sheet — approved scope, implemented and verified on-device 2026-08-28"
metadata: 
  node_type: memory
  type: project
  modified: 2026-09-02T02:15:36.890Z
  originSessionId: 41ef1397-50e4-4ce4-9ff0-2d6578bd9f49
---

Implemented PROJECT_SPEC.md §28's "full data export": Settings > Export
data builds `vitaly_data_export_YYYY-MM-DD.zip` containing
`vitaly_bp_readings_YYYY-MM-DD.csv` (Date, Time, Systolic, Diastolic,
Pulse, Notes, Measurement Context, Reading Source, Related Report ID) plus
`scanned_reports/report_<id>[_page_<n>].<ext>` for every saved report file
belonging to the signed-in user (local storage only, never Firebase
Storage or another user's files). A report file that can't be read is
skipped and listed in a pre-share confirmation dialog rather than failing
silently.

**New feature folder:** `lib/features/data_export/` — `domain/
bp_readings_csv.dart` (pure CSV builder, unit-tested), `domain/
data_export_result.dart`, `data/data_export_service.dart` (builds the ZIP
via the `archive` package, injectable file-reader for testing), `data/
data_export_providers.dart`, `presentation/data_export_share.dart` (writes
to a temp file via `path_provider`, then `SharePlus.instance.share()` —
the modern non-deprecated share_plus 13.x API, not `Share.shareXFiles`).
UI entry point added to `_DataCard` in `settings_screen.dart`.

**New dependencies:** `archive` (promoted from transitive to direct) and
`share_plus: ^13.3.0` (new — needed because the existing `printing`
dependency's `Printing.sharePdf` is PDF-only; share_plus handles generic
files and manages its own Android FileProvider automatically, no manifest
changes needed — it copies the given file into its own
`<cacheDir>/share_plus/` folder before sharing, so the source path/name
just needs to be correct).

`BloodPressureRepository` gained a `getAll()` one-shot method (mirroring
`SavedReportRepository.getAll()`) specifically so this export could read a
snapshot without opening a new `.watch()` stream — see
[[project-flutter-test-drift-stream-pitfalls]].

**Spec updated:** PROJECT_SPEC.md §28 rewritten to document the approved
ZIP export scope (was previously blocking any CSV/export work); §39 and
§40 updated to move this out of "deferred"/"open questions" into
resolved, alongside noting the BP classification work from the prior
session was also never removed from those lists despite being resolved.

**Verified on a real device** (Pixel_8 emulator, signed in with a
temporary test account against the live Firebase project, then deleted
via Settings > Delete account afterward — do not leave test accounts in
production Firebase): tapping Export data produced a real Android share
sheet with a file literally named `vitaly_data_export_2026-08-28.zip`.

**How to apply:** If asked for CSV-only or a different export scope in
the future, this ZIP-based approach and file layout is the currently
approved one — check with the user before changing the structure, column
set, or filename pattern, since these were explicitly specified.

**Update (2026-09-02):** the sibling "Trends PDF summary" export now
embeds colored charts — see [[project-trend-pdf-charts]]. That work was
scoped to the PDF only; the ZIP in this memory was deliberately left
unchanged, so it still contains just the CSV + `scanned_reports/`.
