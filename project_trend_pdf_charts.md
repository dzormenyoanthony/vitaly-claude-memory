---
name: project-trend-pdf-charts
description: "Trends PDF export now embeds colored vector line charts (systolic/diastolic + pulse) via the pdf package's native pw.Chart — shipped 2026-09-02, commits 3002591 + a84fa36"
metadata: 
  node_type: memory
  type: project
  originSessionId: 247eb89e-4bcc-4af7-b068-92b57033f42c
  modified: 2026-09-02T02:15:20.336Z
---

`buildTrendSummaryPdf` (`lib/features/blood_pressure/presentation/
trend_pdf_export.dart`) now draws colored line charts between the summary
sentences and the readings table: a systolic/diastolic chart (teal solid
+ solid coral, `_systolicColor`/`_diastolicColor` mirrored from
`AppColors`) with a swatch legend, plus a pulse chart (purple) when
`includePulse` is on and ≥2 readings in the period carry a pulse.

**Approach — native PDF vector, not raster.** Uses the `pdf` package's
own `pw.Chart` + `pw.CartesianGrid` + `pw.FixedAxis<double>` +
`pw.LineDataSet` (`pw.PointChartValue` points). Deliberately NOT a
RepaintBoundary/`toImage` widget capture of the on-screen `fl_chart`
`_BpChart` — keeping it a pure, synchronous-to-build function with no
Flutter rendering engine means it unit-tests like the rest of the file.
Y-axis fitted with the same padded/10-rounded logic as the on-screen
chart (`_AxisRange.fit`); x-axis labels only first/middle/last reading
dates (`_xTicks` / `_xLabel`, `d/M` format). `marginStart: 22` on the
x-axis insets the plot so the first date label clears the y-axis number
column (was overprinting it — fixed in a84fa36).

**Scope: Trends PDF only.** When asked (AskUserQuestion) the user chose
"PDF summary only" — the §28 full-archive ZIP ([[project-data-export-feature]])
was deliberately left untouched. Dropping the chart-bearing PDF into the
ZIP was explicitly deferred, not forgotten.

**§14 constraint:** data lines only — no reference bands, thresholds, or
color-coded zones, exactly like the on-screen Trends chart. Diastolic is
a *solid* coral line here (on-screen it's dashed); `pw.LineDataSet` has
no dash option, so it's distinguished by color + legend only.

Also closed a copy-vs-reality gap: `exportFormatPdfSubtitle` already
advertised "averages, chart, notes" but the PDF had had no chart. New
ARB keys `trendPdfChartHeader`, `trendPdfPulseChartHeader`,
`trendPdfChartOmittedSingleReading` (none medical-safety-pinned). A
1-reading period shows the "needs at least two readings" note instead of
a chart; an empty period is unchanged.

**Verified:** `flutter analyze` clean, full suite 311 passing (3 new
`trend_pdf_export_test.dart` cases). Also visually confirmed by rendering
the generated PDF bytes to PNG — a throwaway `test/` file writes the PDF,
then `pip install pypdfium2` + `PdfDocument(path)[0].render(scale=3).to_pil()`.

**How to apply:** For any future "see what a generated PDF looks like" on
this repo, use the pypdfium2 route above — `adb screencap` returns an
all-black frame for this Flutter app on the emulator (Skia surface not
captured), so on-device screenshots of PDF/app content don't work here.
