---
name: project-design-references
description: design_references/ mockups were fully implemented in the 2026-08-23 UI-polish + feature pass — no longer just a future reference
metadata:
  type: project
  originSessionId: cc450c07-d367-45ae-8c65-710f50a0bf4d
  modified: 2026-08-23T03:08:52.009Z
---

`design_references/` (repo root) holds 13 PNG screen mockups — Splash,
Onboarding (3 steps + name-only variant), Dashboard, History, Reading, Add
reading, Trends, Reminders, Learn, Measure well. These guided a full
visual-design + feature pass completed 2026-08-23, so the app no longer
looks like plain Material defaults.

**What shipped from the mockups:** a real design system (dark-teal hero
fill, 4 decorative accent colors — mint/coral/purple/blue — used
consistently for time-of-day/context badges and stat tiles, uppercase
"eyebrow" section labels, flat large-radius cards, coral FAB) applied
across every screen; a persistent bottom-nav shell (Home/History/
Trends/Learn via `StatefulShellRoute.indexedStack`) — Settings/Reminders
still reached by push, matching the mockups.

Because the user was walked through each non-visual element the mockups
implied and explicitly approved all of them (not just a visual reskin),
these also shipped as real functionality, not placeholders: a computed
logging streak, a rule-based (never AI-generated, never comments on
reading values) logging-pattern insight/nudge card on the dashboard,
History filter chips + swipe-to-edit/delete, richer reading data (body
position, cuff arm, multi-select context tags — replacing the old
single-select), a same-time-of-day comparison mini-chart on Reading
detail, reminder quiet-hours (silent delivery in a time window), a 1-year
Trends period, and PDF export of the Trends summary (via new `pdf` +
`printing` dependencies) — content is exactly the same non-diagnostic
on-screen text, no new stats.

**Why this matters for future work:** don't treat this folder as "not yet
wired into the app" anymore — it's now the *actual* implemented look, so
compare new screens against both the mockups and the already-shipped
screens for consistency (e.g. `lib/core/theme/app_colors.dart`'s
`AppAccentColors`, `lib/core/widgets/tag_chip.dart`) rather than
reinventing a palette.

**How to apply:** For any new screen/feature, reuse the existing accent
system and card/typography theme (`AppTheme`) instead of hardcoding new
colors. If asked for "more UI polish," ask what's actually missing —
the mockup-to-app gap that motivated this memory is now closed.
