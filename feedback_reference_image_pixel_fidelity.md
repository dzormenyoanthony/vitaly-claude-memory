---
name: feedback-reference-image-pixel-fidelity
description: Technique for matching a screen to a design mockup screenshot precisely, and why live emulator capture failed for this
metadata:
  type: feedback
  originSessionId: 8a750d3c-465d-4f1c-8e61-e19f29cd24ef
  modified: 2026-08-23T04:54:14.938Z
---

For the 2026-08-23 Splash screen task, the user gave an explicit,
narrow-scope instruction: match `design_references/Splash.png` closely
("the reference screenshot is the source of truth... do not redesign or
approximate it"), touching only that one screen, then stop. Eyeballing
the reference wasn't precise enough — my first implementation was
visually "in the spirit of" the mockup but had circle sizes/positions off
by 2x in places and reused one muted color for two blobs that are
actually different shades in the source.

**What worked — measure the reference numerically instead of eyeballing:**
Installed Pillow/numpy in the project's Python (`/c/Python314/python -m
pip install pillow numpy`) and wrote short throwaway scripts to: (1) find
a shape's exact fill color by sampling/histogramming pixels, (2) find
unclipped boundary points of a circle (leftmost/rightmost/topmost/
bottommost, or edge-trace + least-squares Kasa circle fit) to get precise
center/radius, (3) convert those to *fractions* of the screen's content
area (not the raw mockup canvas — mockups often have design-tool chrome:
browser-tab labels, device bezel padding, a fake status bar — so first
find the real screen bounds by scanning for the frame-bezel color and
excluding it). Fractions transfer directly into Flutter's
`Positioned(top: -height*x, left: -width*y, ...)` pattern already used
for this app's decorative blobs.

**Why live on-device capture didn't work here:** the auth gate
(`authGateProvider`) resolves synchronously off a cached session in well
under a frame, so `android screen capture` right after a cold launch
only ever caught either the native pre-Flutter splash or the
already-redirected Dashboard — never the Dart `SplashScreen` widget
itself, even with force-stop + relaunch + immediate capture attempts.

**The fallback that worked — render the widget directly:** a throwaway
`flutter test` file that pumps the target screen inside `ProviderScope`
with the blocking provider overridden to a permanently-loading state
(e.g. `authGateProvider.overrideWithValue(const AuthGateLoading())`),
sets `tester.view.physicalSize`/`devicePixelRatio` to the **real target
device's** values (get these via `adb shell wm size` / `wm density` —
density/160 = devicePixelRatio; using the wrong ratio here initially made
fixed-dp elements look wrong in the render even though they were fine),
then captures via `matchesGoldenFile(...)` with `--update-goldens`. This
sidesteps any timing race entirely and gives a pixel-accurate layout
render (though text renders as placeholder boxes, not real glyphs, since
`flutter test` doesn't load real fonts by default — fine for verifying
layout/spacing/color but not exact font metrics). Delete the temp test
and golden PNG afterward; don't commit them.

**How to apply:** For any future "match this screen to this reference
image exactly" request on this project (other mockups in
`design_references/` haven't had this treatment yet — see
[[project-design-references]]), reach for this measure-then-render
workflow rather than eyeballing + live-capture-and-iterate, especially on
screens that redirect quickly (splash/loading/gate screens) where live
capture may not be feasible at all.
