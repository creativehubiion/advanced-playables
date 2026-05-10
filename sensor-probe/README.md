# sensor-probe

A single-file HTML5 creative that detects, in real time, what the surrounding ad WebView allows.

**Live:** https://creativehubiion.github.io/advanced-playables/sensor-probe/

## What it reports

- **Environment** — user agent, platform, viewport, DPR, touch points, online state, connection type, hardware
- **Iframe context** — top-level vs iframed, origin, referrer
- **MRAID** — presence, version, state, viewability, placement, max size, supports flags
- **Permissions Policy** — for each of: `accelerometer`, `gyroscope`, `magnetometer`, `microphone`, `camera`, `geolocation`, `autoplay`, `fullscreen`, `web-share`, `screen-wake-lock`, `ambient-light-sensor`
- **DeviceOrientation** — API present, `requestPermission` required, permission state, live `alpha` / `beta` / `gamma` / `webkitCompassHeading`, event count
- **DeviceMotion** — API present, `requestPermission` required, permission state, live acceleration (with and without gravity), rotation rate, sample interval, sample rate (Hz), peak |a|
- **Microphone** — `getUserMedia` available, `AudioContext` available, permission state, live RMS volume + meter, peak RMS
- **Other capabilities** — camera/geolocation permission states, `navigator.vibrate`, WebGL (with unmasked renderer), Web Audio, Battery API, Wake Lock, Fullscreen
- **Frame rate** — rolling 60-frame fps badge in the header
- **Event log** — last ~40 events (MRAID lifecycle, permission outcomes, errors)

A **Copy** button in the header serializes everything (including the event log) to JSON and copies to clipboard, or pastes into the event-log box if clipboard write is denied.

## How to use it

### A. As a third-party hosted GAM creative
1. In Google Ad Manager → New creative → **Third party** (or "Custom").
2. Set the snippet to load `https://creativehubiion.github.io/advanced-playables/sensor-probe/` in an iframe (or use GAM's URL-based creative type).
3. Traffic to your test ad unit and serve into the Android test app.
4. When the interstitial loads, tap **Enable tilt** (and/or **Enable motion**, **Enable mic**) to start each sensor. Tap **Copy** when you've gathered enough data and paste the JSON dump somewhere you can analyse it.

### B. As a packaged HTML5 creative
1. ZIP the contents of this directory (`index.html` at the root of the ZIP).
2. In GAM, upload as an **HTML5** creative.
3. The `<script src="mraid.js"></script>` tag is intentional — the SDK injects an `mraid.js` stub when serving as MRAID. When loaded standalone (e.g., in mobile Chrome) it 404s harmlessly; the probe defensively checks `window.mraid` before using it.

### C. Standalone in mobile Chrome (control sample)
Open `https://creativehubiion.github.io/advanced-playables/sensor-probe/` directly in Chrome on the test phone and run the same checks. This gives you a "no SDK in the way" baseline to compare with the in-app reading.

## Reading the results

| Badge | Meaning |
| --- | --- |
| `firing` (green) | Sensor events are arriving. Mechanic feasible on this surface. |
| `silent` (red) | API exists, but no events arrived in 1500 ms. **Almost always Permissions Policy on the iframe `allow=` attribute** — talk to the SDK / wrapper. |
| `denied` (red) | iOS user denied the permission prompt, or `getUserMedia` rejected. |
| `absent` (grey) | `window.mraid` not present — you're either running outside an MRAID host (e.g., browser) or the SDK didn't inject it. |

## Caveats

- **Single sample, single placement.** The probe only tells you about the exact WebView it is currently running in. To generalise to a publisher / SDK / OS combination you need multiple readings.
- **iOS gesture chain.** iOS will only show the `DeviceOrientationEvent.requestPermission()` dialog if the call originates from a real user touch. Some ad SDKs intercept gestures with their own overlays — if the prompt never appears even though you tapped, that is the bug.
- **GitHub Pages serves over HTTPS** (required by `getUserMedia`, `DeviceOrientation`, etc.). Do not test from a plain-HTTP mirror.
