---
name: project_launcher_and_native_splash
description: App launcher icon made adaptive + Android system splash restyled to match the branded Flutter splash (2026-09-01)
metadata: 
  node_type: memory
  type: project
  originSessionId: a9a2c75f-ce97-4534-bdef-6188a567dce6
  modified: 2026-09-01T03:22:51.266Z
---

Work on 2026-09-01, verified on the Pixel 8 / API 36 emulator and pushed
to master: commit a93700b (launcher icon), commit 33665b0 (splash).

**Launcher icon** — was a legacy square PNG, so Android 8+ wrapped it in a
white circle with a wide margin (visible on the home screen and inside the
Android 12 system splash). Fixed by adding an adaptive icon:
`tool/generate_adaptive_icon.py` derives `assets/icon/icon_adaptive_background.png`
(brand green gradient, waveform removed by fitting a quadratic surface) and
`assets/icon/icon_adaptive_foreground.png` (the pulse mark) from
`assets/icon/icon.png`; `flutter_launcher_icons` config in `pubspec.yaml` gained
`adaptive_icon_background` / `adaptive_icon_foreground`. Regenerate the two
layers then rerun `dart run flutter_launcher_icons` if `icon.png` changes.

**Android system splash** — the OS cold-start splash showed the launcher icon
on white; user wanted it to *be* `design_references/Splash.png`. That is
impossible: the Android 12+ SplashScreen API only accepts one background
**colour** + one centred **icon** (+ an optional bottom branding image). It
cannot draw the reference's gradient, blobs, rings, VITALY wordmark, tagline,
floating dots, heartbeat trace, loading bar or "NOT A MEDICAL DEVICE" caption.
Best achievable, and what was done: the real ask was "one splash, not
native + branded". So the system splash and the branded `SplashScreen`'s
first frame are made identical (teal + badge, dead centre), and everything
else fades in around the steady badge.
- `android/.../res/drawable-nodpi/splash_badge.png` — the mint pulse badge
  lifted pixel-for-pixel from `Splash.png` (badge = 35% of the icon canvas so
  it renders ~24% of screen width ≈ the reference's ~23%).
- `res/values-v31/styles.xml` + `res/values-night-v31/styles.xml` — `LaunchTheme`
  sets `windowSplashScreenBackground` = `@color/launch_background` and
  `windowSplashScreenAnimatedIcon` = `@drawable/splash_badge` (no icon
  background colour, so no circle mask).
- `res/drawable/launch_background.xml` + `res/drawable-v21/…` — pre-12 path:
  teal fill + `splash_badge` centred at 272dp.
- `res/values/styles.xml` + `res/values-night/styles.xml` — `NormalTheme`
  windowBackground → `@drawable/launch_background` (was near-white
  `?android:colorBackground`), kills the white flash before Flutter's first
  frame.
- `lib/features/splash/presentation/splash_screen.dart` — now
  `ConsumerStatefulWidget`. Badge was in a Column that pushed it ~7% of
  screen height above the OS splash badge (visible jump). Now it's a
  `Positioned` at its resting centre `height * 0.474` (`_badgeCenterY`,
  measured from `Splash.png`) but starts translated down by
  `0.5 - 0.474` of screen height — i.e. exactly at the OS splash's centred
  position — and glides up via `_badgeSettle` (`easeOutCubic`) as the scene
  assembles. So: no jump at hand-off AND the composition lands on the
  mockup. `iconCenter` (rings/coral dot) + wordmark/tagline anchor to
  0.474; heartbeat trace 0.70, mint dot 0.695. Decorations + text
  `FadeTransition` in over 550ms (`_backdrop` Interval 0–0.75, `_foreground`
  0.3–1.0). Commit e1a2c12.

Note: the Flutter splash's `_PulsePainter`
(`lib/features/splash/presentation/splash_screen.dart`) still has a slightly
different wave (an extra small second peak) than `Splash.png`'s glyph; the
native `splash_badge.png` uses the reference glyph. See
[[project_design_references]], [[feedback_reference_image_pixel_fidelity]].
