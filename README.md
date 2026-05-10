# advanced-playables

R&D for sensor-driven HTML5 playable ads at iion.

This repo holds **diagnostic and prototype creatives** that explore what's possible inside the WebView contexts created by the major mobile ad SDKs (Google Mobile Ads, AppLovin MAX, IronSource LevelPlay) when serving via iion's direct supply path.

## Hosted

GitHub Pages serves the latest `main` at:
**https://creativehubiion.github.io/advanced-playables/**

## Creatives

| Creative | URL | Purpose |
| --- | --- | --- |
| [`sensor-probe`](./sensor-probe/) | [/sensor-probe/](https://creativehubiion.github.io/advanced-playables/sensor-probe/) | Live diagnostics for MRAID / Permissions Policy / DeviceMotion / DeviceOrientation / mic / WebGL / fps inside whatever WebView is rendering it. |

## Why this matters

Sensor APIs (gyroscope, accelerometer, microphone) are technically supported by mobile browsers, but in the **ad-WebView + cross-origin iframe + MRAID/SafeFrame** context they are silently gated by:

1. The host app's WebView configuration
2. The ad SDK's iframe `allow=` attribute (Permissions Policy)
3. iOS 13+ user-gesture permission for `DeviceOrientationEvent.requestPermission()`

Before committing to any sensor-driven game mechanic, it is worth empirically measuring which gates open in which SDK × OS × publisher combinations. That is what `sensor-probe` is for.
