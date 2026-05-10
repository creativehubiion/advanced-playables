# advanced-playables

R&D for sensor-driven HTML5 playable ads at iion.

This repo holds **diagnostic and prototype creatives** that explore what's possible inside the WebView contexts created by the major mobile ad SDKs (Google Mobile Ads, AppLovin MAX, IronSource LevelPlay) when serving via iion's direct supply path.

## Hosted

GitHub Pages serves the latest `main` at:
**https://creativehubiion.github.io/advanced-playables/**

## Creatives

| Creative | URL | Purpose |
| --- | --- | --- |
| [`tilt-racer`](./tilt-racer/) | [/tilt-racer/](https://creativehubiion.github.io/advanced-playables/tilt-racer/) | Playable demo — gyro tilt-to-steer endless racer, 25 s loop, with auto-fallback to drag-to-steer when gyro is denied or silently blocked. Validates the full pipeline: iOS permission gate, sensor binding, MRAID `open()` click-through, GAM macros. |
| [`sensor-probe-telemetry`](./sensor-probe-telemetry/) | [GAM](https://creativehubiion.github.io/advanced-playables/sensor-probe-telemetry/index_gam.html) · [PLL](https://creativehubiion.github.io/advanced-playables/sensor-probe-telemetry/index_pll.html) | Real-traffic probe — fires discrete `event_name` events to iion's staging DMP at every sensor lifecycle juncture. Two flavours (GAM macros, PLL/RTB macros). Upload as a creative, run a small test campaign, read per-SDK breakdown of sensor support. |
| [`sensor-probe`](./sensor-probe/) | [/sensor-probe/](https://creativehubiion.github.io/advanced-playables/sensor-probe/) | Live diagnostics for MRAID / Permissions Policy / DeviceMotion / DeviceOrientation / mic / WebGL / fps inside whatever WebView is rendering it. Manual / DevTools-driven sibling of `sensor-probe-telemetry`. |

## Why this matters

Sensor APIs (gyroscope, accelerometer, microphone) are technically supported by mobile browsers, but in the **ad-WebView + cross-origin iframe + MRAID/SafeFrame** context they are silently gated by:

1. The host app's WebView configuration
2. The ad SDK's iframe `allow=` attribute (Permissions Policy)
3. iOS 13+ user-gesture permission for `DeviceOrientationEvent.requestPermission()`

Before committing to any sensor-driven game mechanic, it is worth empirically measuring which gates open in which SDK × OS × publisher combinations. That is what `sensor-probe` is for.
