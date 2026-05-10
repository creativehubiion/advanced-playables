# tilt-racer

A self-contained HTML5 playable demo that uses the phone's gyroscope to steer a car, with a touch-drag fallback when the gyro is denied or silently blocked.

**Live:** https://creativehubiion.github.io/advanced-playables/tilt-racer/

## Mechanic

- 25-second endless racer, top-down view, road scrolls vertically
- Tilt phone left / right → steer the player car
- Dodge oncoming cars (collision = crash, run ends)
- Collect coins (+50 score each)
- End screen with score + **Install Now** button (calls `mraid.open(landingPageUrl)` if MRAID present, otherwise `window.open`)

## Input handling

The probe-style permission flow is wired in:

1. **iOS 13+** — taps **Tap to start** → calls `DeviceOrientationEvent.requestPermission()`. On grant, gyro mode engages. On deny, falls back to touch.
2. **Android / older iOS** — binds `deviceorientation` directly. If no events arrive within **1500 ms**, the creative assumes Permissions Policy / iframe `allow=` is silently blocking the API and switches to touch mode automatically.
3. **Touch fallback** — drag finger anywhere on the canvas; the car eases toward your finger's x-position.

A small pill at the top of the screen shows which input mode is active (`Tilt to steer` vs `Drag to steer`) so QA always knows which path the WebView ended up on.

Calibration: the first `deviceorientation.gamma` value becomes the neutral baseline, so the user holding the phone at any natural angle is treated as "centered." The car velocity is low-pass filtered (78/22) to prevent jitter.

## Self-contained

- Single `index.html` — no external CSS, JS, fonts, images, or audio files
- All graphics are vector (Canvas2D)
- All sounds are synthesised live (Web Audio oscillators)
- Includes the standard `<script src="mraid.js"></script>` MRAID hook
- Includes the iion GAM tracking macro block at end of body — same shape as production creatives

## Running it

### A. Standalone (control sample)
Open https://creativehubiion.github.io/advanced-playables/tilt-racer/ in mobile Chrome on the test phone. This is your "no SDK in the way" baseline.

### B. As a GAM HTML5 creative
1. ZIP `index.html`.
2. Upload as **HTML5** creative in GAM.
3. Traffic to your test ad unit; serve into the Android test app.

### C. As a GAM third-party hosted URL
Point the third-party tag at the live URL above; GAM substitutes the macros at serve time.

## What this validates

- That gyroscope events fire inside the rendering SDK's WebView (or are silently blocked → fallback engages cleanly)
- That the iOS user-gesture permission flow can be triggered from inside an MRAID interstitial
- That `mraid.open()` opens the landing page correctly on click-through
- That the iion GAM tracking macros resolve at serve time (cross-check with the sensor-probe's GAM card)

## What's intentionally simple (room for v2)

- No images / no licensed audio — the demo runs at any size with no asset-pipeline cost
- Single difficulty curve, single car colour for the player
- No close button — relies on the SDK's interstitial chrome
- 25 s loop is short enough to play within a typical interstitial dwell time
