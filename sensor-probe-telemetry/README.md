# sensor-probe-telemetry

A real **playable creative** (tilt-to-steer endless racer) that doubles as a sensor-API probe — fires discrete events to **iion's staging DMP** at every lifecycle juncture so a small test campaign tells you, per real-publisher SDK, whether sensor APIs actually work end-to-end.

**Live URLs:**
- GAM: https://creativehubiion.github.io/advanced-playables/sensor-probe-telemetry/index_gam.html
- PLL: https://creativehubiion.github.io/advanced-playables/sensor-probe-telemetry/index_pll.html

## What it is

The creative is the same `tilt-racer` mini-game shipped at `/tilt-racer/`, instrumented with telemetry. User taps **Tap to start**, the iOS permission flow runs (where applicable), then a 25-second tilt-to-steer racer begins. Throughout the lifecycle the creative fires `event_name` events to iion's DMP. Same WebView the publisher's SDK uses — same answer about whether sensors work as in production. No bypass.

## The headline question — and the events that *unambiguously* answer it

**Will gyro/accelerometer work natively when iion demand serves to iion supply?**

There are two classes of events: weak signals (the API exists, events arrive) and **foolproof signals** (real human engagement *via tilt* drove a measurable game outcome). Trust the foolproof ones — they only fire when the entire pipeline survived contact with reality.

### Foolproof success signals

These cannot fire unless gyro actually works, the user actually moved the device, and the game actually responded. **One impression with any of these is hard proof that tilt mechanics are viable in that SDK / publisher / device combo.**

| Event | Fires only if |
| --- | --- |
| `SensorTiltLeftDetected` | gyro events arrived AND user tilted past **−15°** off baseline |
| `SensorTiltRightDetected` | gyro events arrived AND user tilted past **+15°** off baseline |
| `SensorTiltBothDirections` | both above fired — proves real two-way human steering, not a stationary phone |
| `SensorRealAcceleration` | accel magnitude > **2.0 m/s²** for **3 consecutive samples** — sustained motion, rules out one-off noise spikes |
| `SensorTiltSteeredCar` | gyro was the active input mode AT the moment the player car X position deviated > 30 px from centre — proves API → velocity → game-state pipeline works |
| `SensorTiltCoinCollected` | a coin was collected while gyro was the active input mode — full pipeline including collision detection |
| `SensorTiltScoreOver500` | score crossed 500 in gyro mode — proves sustained engagement, not a single accidental movement |
| `SensorTiltScoreOver1000` | score crossed 1000 in gyro mode — the strongest score-based engagement proof |
| `SensorTiltOnlyMode` | game ended with input mode still gyro AND was never bumped to touch fallback — proves gyro held up for the whole run |
| `SensorTiltGameCompleted` | survived the full 25 s in gyro-only mode AND tilted both directions — **the strongest single end-of-impression "everything worked" signal** |

### Foolproof failure signals

These only fire when the creative was alive long enough to know it was a real failure (not just an early close):

| Event | Fires only if |
| --- | --- |
| `SensorOrientSilentBlock` | DeviceOrientation API exists, listener attached, **0 events in 2000 ms** — almost certainly Permissions Policy blocking the iframe `allow=` |
| `SensorMotionSilentBlock` | same for DeviceMotion |
| `SensorOrientIOSDenied` | iOS user explicitly tapped Don't Allow in the permission dialog |

## Disambiguating from inconclusive impressions

Some impressions tell you nothing — discard them, don't count as failure:

| Closing condition | Bucket |
| --- | --- |
| `SensorProbeClosed_lt2s` AND no first-event AND no silent-block | **inconclusive** — closed before the API had time to fire (or fail) |
| `SensorClosedBeforeStart` | **inconclusive** — user closed before tapping Tap to Start, no chance to enter game |
| iOS UA AND `SensorOrientIOSPrompted` did NOT fire | **inconclusive** — passive impression, no permission dialog ever shown |

Every closed impression fires exactly one of `SensorProbeClosed_lt2s | _2to5s | _5to10s | _gt10s`, so the duration distribution per SDK is always available.

## Full event taxonomy (all events the creative can fire)

### Lifecycle (always)
- `SensorProbeLoaded` — creative loaded
- `SensorProbeIframed` / `SensorProbeTopLevel`
- `SensorProbeClosed_lt2s` / `_2to5s` / `_5to10s` / `_gt10s`
- `SensorClosedBeforeStart` / `SensorClosedDuringPlay`

### Permissions Policy (one per sensor, always)
For `gyroscope`, `accelerometer`, `magnetometer`, `microphone`, `camera`, `geolocation`, `autoplay`, `fullscreen`:
- `SensorPolicy_<feature>_allowed` / `_blocked` / `_unknown`

### MRAID
- `SensorMraidPresent` / `SensorMraidAbsent`
- `SensorMraidVersion_<v>`

### DeviceOrientation API
- `SensorOrientApiPresent` / `SensorOrientApiAbsent`
- `SensorOrientGestureRequired` / `SensorOrientAutoBind`
- `SensorOrientFirstEvent_lt100ms` / `_100to500ms` / `_500to2000ms` / `_gt2000ms`
- `SensorOrientConfirmed10` / `SensorOrientConfirmed50`
- `SensorOrientSilentBlock`
- `SensorOrientIOSPrompted` / `SensorOrientIOSGranted` / `SensorOrientIOSDenied` / `SensorOrientIOSError`
- `SensorOrientNoEventsAtClose`

### DeviceMotion API
- `SensorMotionApiPresent` / `SensorMotionApiAbsent`
- `SensorMotionGestureRequired` / `SensorMotionAutoBind`
- `SensorMotionFirstEvent_<bucket>` / `SensorMotionConfirmed10` / `_50` / `SensorMotionSilentBlock`
- `SensorMotionNoEventsAtClose`

### User interaction
- `SensorUserTapped` / `SensorUserTilted`
- `SensorTiltLeftDetected` / `SensorTiltRightDetected` / `SensorTiltBothDirections`
- `SensorRealAcceleration`
- `SensorCtaClicked`

### Game lifecycle
- `SensorGameStarted`
- `SensorGameInputMode_gyro` / `SensorGameInputMode_touch_<reason>`
- `SensorGameCarMoved` (any mode) / `SensorTiltSteeredCar` (gyro only)
- `SensorGameCoinCollected` (any mode) / `SensorTiltCoinCollected` (gyro only)
- `SensorGameCrashed` / `SensorGameCrashedEnd`
- `SensorGameSurvived` (full 25 s, any mode)
- `SensorGameScore_lt100` / `_100to500` / `_500to1000` / `_gt1000`
- `SensorTiltScoreOver500` / `SensorTiltScoreOver1000`
- `SensorTiltOnlyMode` (no fallback ever happened)
- `SensorTiltGameCompleted` (the headline success event)

## Per-SDK analysis

Group DMP events by SDK signature (from the existing macros — `user_agent`, `app_bundle`, `domain`, `page_url` for PLL; the same plus `publisher_id` for GAM). Then compute per SDK:

- **Gyro support rate** = `count(SensorTiltSteeredCar) / count(SensorGameStarted)` — the share of started games where gyro actually drove gameplay.
- **Permissions Policy block rate** = `count(SensorOrientSilentBlock) / count(SensorGameStarted)` — share where gyro was silently blocked at the iframe level (file an SDK ticket for these).
- **Full-pipeline success rate** = `count(SensorTiltGameCompleted) / count(SensorTiltBothDirections)` — of users who tried tilting both directions, how many made it all the way through.
- **iOS permission denial rate** = `count(SensorOrientIOSDenied) / count(SensorOrientIOSPrompted)`.

Exclude impressions that fired neither a foolproof success NOR a foolproof failure event (the inconclusive bucket) from these rates.

## Why these specific thresholds

- **15° gamma** for tilt detection: a deliberate intentional movement; ambient hand tremor is typically <5°.
- **2.0 m/s² for 3 consecutive samples** for "real acceleration": rules out one-off bumps and noise spikes; sustained motion peaks reach 3–5 m/s² easily.
- **30 px car displacement** for SteeredCar: half a lane width — meaningful steering, not jitter.
- **2000 ms timeout** for SilentBlock: long enough to rule out browser scheduling delays; short enough to fire before most ad close events.
- **500 / 1000 score** for sustained engagement: 500 ≈ 10s of survival in gyro mode, 1000 ≈ 18-20s.

## Files

| File | Use |
| --- | --- |
| [`index_gam.html`](./index_gam.html) | Upload as the HTML5 creative for **GAM** line items. |
| [`index_pll.html`](./index_pll.html) | Upload as the HTML5 creative for **PLL / RTB** programmatic deals. |
| `README.md` | This document. |

The two HTML files differ only in the trailing `<script>` macro block at the end of `<body>` (GAM `%ebuy!` / `%%CLICK_URL_UNESC%%` style vs PLL `%%campaignId%%` / `%%clickUrl%%` style). Game logic and telemetry are identical.

Both files are **fully self-contained** — single HTML, no external CSS / JS / images / audio / fonts. All graphics are vector (Canvas2D), all sounds are synthesised live (Web Audio oscillators). Drop the file into any GAM/PLL HTML5 creative slot. The only external script reference is `<script src="mraid.js"></script>` — the SDK injects the stub at runtime; if absent the code defensively checks `window.mraid`.

## Caveats

- **Ad-quality review** — the creative looks like a normal "Tilt Racer" mini-game ad with brand placeholder. Should pass automated and human review on AdMob, MAX, LevelPlay. If a network rejects, soften the visual cosmetics; the telemetry layer is invisible to reviewers.
- **Production DMP** — change `staging-dmp-producer.iion.io` to the production endpoint before any non-R&D campaign.
- **Click-through** — `landingPageUrl` defaults to `https://www.iion.io/`. Change per campaign before traffic.
- **Test sample size** — for SDK-level conclusions, aim for ≥100 impressions per SDK per OS. At ~3000-impression total budget you typically cover MAX, LevelPlay, AdMob × Android + iOS with reasonable confidence.
