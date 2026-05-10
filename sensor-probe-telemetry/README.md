# sensor-probe-telemetry

Real-traffic sensor probe — a self-contained HTML5 creative that exercises sensor APIs (gyro, accel, mic-policy, etc.) and reports the outcomes back to **iion's staging DMP** via discrete `event_name` events. Upload as a GAM creative or PLL/RTB creative, run a small test campaign, and read the per-SDK breakdown of which sensors actually work in real publisher inventory.

**Live URLs:**
- GAM: https://creativehubiion.github.io/advanced-playables/sensor-probe-telemetry/index_gam.html
- PLL: https://creativehubiion.github.io/advanced-playables/sensor-probe-telemetry/index_pll.html

## Why this exists

In-app testing tells you about *one* device + *one* SDK config. Real-traffic testing tells you about the actual distribution of sensor-API support across hundreds of SDK × OS × publisher-app combinations the iion supply pipeline reaches every day. A small test budget (~3000 impressions) returns a real per-SDK answer in hours.

Two known traps when designing this kind of test:

1. **Telemetry from the wrong place.** Firing a tracker pixel from `addEventListener('deviceorientation', handler)` always succeeds — that call returns nothing about whether events actually arrive. **All "API works" events in this probe fire from inside the actual handler**, never from the attach.
2. **Ambiguity between "blocked" and "not enough time."** If the probe closes 500ms after load and we conclude "blocked," that's a false negative. The probe waits **2000ms** before firing `SensorOrientSilentBlock`, and emits separate `SensorProbeClosed_<bucket>` events so the analyst can filter out impressions that didn't live long enough.

## Files

| File | Use |
| --- | --- |
| [`index_gam.html`](./index_gam.html) | Upload as the HTML5 creative for **GAM** (Google Ad Manager) line items. |
| [`index_pll.html`](./index_pll.html) | Upload as the HTML5 creative for **PLL / RTB** programmatic deals. |
| `README.md` | This document. |

The two files differ ONLY in the trailing `<script>` macro block at the end of `<body>` (different macro syntax for GAM vs PLL). The probe logic is identical.

## DMP endpoint

```
https://staging-dmp-producer.iion.io/tracker/impressions
```

The probe builds the full URL by concatenating `window.trackingType` (which ends with `event_name=`) and the event name string. GET request, fire-and-forget. Falls back to `Image()` requests if `fetch` is blocked or CSP-restricted in the SDK's WebView.

## Event taxonomy

All events are **discrete and countable** — designed to be aggregated on the DMP side with `GROUP BY event_name`. No payload data is encoded anywhere except the event name itself, because the DMP only captures the named query params from `trackingType` (extra `&foo=bar` would be ignored).

### Lifecycle (always fires)

| Event | Meaning |
| --- | --- |
| `SensorProbeLoaded` | Creative loaded successfully |
| `SensorProbeIframed` | Running inside a cross-origin iframe (almost always true in ad context) |
| `SensorProbeTopLevel` | Running as the top window (rare — only standalone Chrome) |
| `SensorProbeClosed_lt2s` | Impression ended in under 2s (too short to draw conclusions) |
| `SensorProbeClosed_2to5s` | 2–5s impression |
| `SensorProbeClosed_5to10s` | 5–10s impression |
| `SensorProbeClosed_gt10s` | 10s+ impression |

### Permissions Policy snapshot (one per sensor)

For each of `gyroscope`, `accelerometer`, `magnetometer`, `microphone`, `camera`, `geolocation`, `autoplay`, `fullscreen`:

| Event | Meaning |
| --- | --- |
| `SensorPolicy_<feature>_allowed` | iframe `allow=` includes this feature |
| `SensorPolicy_<feature>_blocked` | iframe explicitly blocks it |
| `SensorPolicy_<feature>_unknown` | Permissions Policy API not available (very old WebView) |

### MRAID

| Event | Meaning |
| --- | --- |
| `SensorMraidPresent` | `window.mraid` exists — running in an MRAID-compliant SDK host |
| `SensorMraidAbsent` | No MRAID — likely web browser or non-MRAID host |
| `SensorMraidVersion_<v>` | MRAID version, normalised (e.g., `SensorMraidVersion_30` for 3.0) |

### DeviceOrientation API (the headline question)

| Event | Meaning |
| --- | --- |
| `SensorOrientApiPresent` | `DeviceOrientationEvent` class exists |
| `SensorOrientApiAbsent` | Class doesn't exist (extremely rare) |
| `SensorOrientGestureRequired` | iOS 13+ — `requestPermission()` exists, needs user tap |
| `SensorOrientAutoBind` | Android — events fire automatically, no permission needed |
| `SensorOrientFirstEvent_lt100ms` | Time-to-first-event bucket: <100ms after attach |
| `SensorOrientFirstEvent_100to500ms` | 100–500ms |
| `SensorOrientFirstEvent_500to2000ms` | 500–2000ms |
| `SensorOrientFirstEvent_gt2000ms` | >2s — late but firing |
| `SensorOrientConfirmed10` | 10+ events received (high confidence the API works) |
| `SensorOrientConfirmed50` | 50+ events received (very high confidence) |
| `SensorOrientSilentBlock` | API present, listener attached, **zero events in 2000ms** — almost certainly Permissions Policy blocking the iframe |
| `SensorOrientIOSPrompted` | iOS permission dialog was triggered |
| `SensorOrientIOSGranted` | User granted on iOS |
| `SensorOrientIOSDenied` | User denied on iOS |
| `SensorOrientNoEventsAtClose` | Probe closed with 0 orientation events ever received |

### DeviceMotion API (acceleration / rotation rate)

Same shape as DeviceOrientation:
- `SensorMotionApiPresent` / `SensorMotionApiAbsent`
- `SensorMotionGestureRequired` / `SensorMotionAutoBind`
- `SensorMotionFirstEvent_<bucket>`
- `SensorMotionConfirmed10` / `SensorMotionConfirmed50`
- `SensorMotionSilentBlock`
- `SensorMotionNoEventsAtClose`

### User interaction signals

| Event | Meaning |
| --- | --- |
| `SensorUserTapped` | User tapped the creative at least once |
| `SensorUserTilted` | Real motion detected — peak acceleration > 1.5 m/s² (proves the user actively engaged with the device) |
| `SensorNoUserTapAtClose` | User closed without tapping — useful to filter "passive" impressions |
| `SensorCtaClicked` | User clicked the "Learn more" CTA |

## Decision matrix — how to interpret the events per impression bucket

For each rendering SDK (identifiable from the DMP's `user_agent`, `app_bundle`, `domain`, `page_url` macros), bucket impressions like this:

| Bucket | Signal in DMP | Conclusion |
| --- | --- | --- |
| ✅ **Allowed (high conf)** | `SensorOrientConfirmed10` or `SensorOrientConfirmed50` fired | DeviceOrientation works in this SDK's WebView |
| ✅ **Allowed (low sample)** | `SensorOrientFirstEvent_*` fired but not Confirmed10, AND `SensorProbeClosed_lt2s` (early close) | API works (insufficient time) — count as ✓ |
| ❌ **Silent block** | `SensorOrientSilentBlock` fired AND impression lived ≥2s (`SensorProbeClosed_2to5s` or higher) | iframe Permissions Policy is missing `gyroscope` — cannot do tilt mechanics in this SDK without infra change |
| ❌ **iOS denied** | iOS UA AND `SensorOrientIOSDenied` fired | User refused permission |
| ⚠️ **iOS no gesture** | iOS UA AND `SensorOrientIOSPrompted` did NOT fire AND `SensorNoUserTapAtClose` fired | Inconclusive — passive impression on iOS, no chance to ask permission |
| ⚠️ **Inconclusive (early close)** | `SensorProbeClosed_lt2s` AND no first-event AND no silent-block (timeout never fired) | Inconclusive — probe didn't run long enough |

The unambiguous buckets (Allowed / Silent block / iOS denied) define the success / failure rates per SDK. **Inconclusive buckets must be excluded from rate calculations**, not counted as failures — that's the rule that prevents false-negative bias.

## Example DMP queries

Assuming the DMP team can `GROUP BY event_name, user_agent, app_bundle`:

```sql
-- Per-SDK confirmation rate for DeviceOrientation
SELECT
  app_bundle,
  COUNT(*) FILTER (WHERE event_name = 'SensorProbeLoaded')         AS impressions,
  COUNT(*) FILTER (WHERE event_name = 'SensorOrientConfirmed10')   AS confirmed,
  COUNT(*) FILTER (WHERE event_name = 'SensorOrientSilentBlock')   AS silent_blocks
FROM dmp_events
WHERE creative_id = '<sensor-probe-creative-id>'
GROUP BY app_bundle
ORDER BY impressions DESC;
```

```sql
-- Permissions Policy state across the supply
SELECT
  app_bundle,
  COUNT(*) FILTER (WHERE event_name = 'SensorPolicy_gyroscope_allowed') AS gyro_allowed,
  COUNT(*) FILTER (WHERE event_name = 'SensorPolicy_gyroscope_blocked') AS gyro_blocked,
  COUNT(*) FILTER (WHERE event_name = 'SensorPolicy_microphone_allowed') AS mic_allowed,
  COUNT(*) FILTER (WHERE event_name = 'SensorPolicy_microphone_blocked') AS mic_blocked
FROM dmp_events
WHERE creative_id = '<sensor-probe-creative-id>'
GROUP BY app_bundle;
```

## Caveats

- **Ad-quality review**. Some networks flag creatives that read sensors without an obvious "tilt to play" UX. The probe presents a "Tilt to play / Learn more" splash, which gives reviewers a plausible product justification. If a network rejects, soften the visual further (no animated arrows, no explicit mention of tilt).
- **MAID / IDFA** is captured by the existing macro block. No additional PII is collected by the probe.
- **Production DMP URL** is different from staging. Update both files before any non-R&D campaign.
- **Click-through** lands on `https://www.iion.io/` by default. Change `window.landingPageUrl` per campaign.
- **iOS Low Power Mode** throttles DeviceMotion to ~1 Hz. `SensorMotionConfirmed50` may be skipped on devices in LPM even when the API works — use `SensorMotionConfirmed10` as the primary "works" signal for motion.
- **Test devices**. For initial validation before broad targeting, run the campaign with a tight test-device segment in iion's GAM/SSP to avoid noise.

## Iteration

Re-uploading a new creative version requires re-passing ad-quality review on most networks (24–72h). To minimise iteration cost, **add new event names liberally up-front** — the DMP doesn't care about extra events it doesn't recognise, but adding events later means another review cycle.
