---
name: custom-profile
description: Use when authoring or editing a custom profile JSON for Aria Gen 2. A profile fixes which sensors are enabled and at what rate, resolution, and encoding — **the same profile works for both recording and streaming**, despite the historical "recording profile" name. Gives the legal value for every field, the rules that make the device reject a profile or fail to start a session, and the changes the device makes silently. Use whenever the user asks how to write a custom profile, modify a profile, pull an existing profile off the device, define sensor rates, deploy a profile to the device, or asks what a specific profile field means.
---

# Custom Profile Authoring

A **profile** is JSON that fixes which Aria Gen 2 sensors run, at what rate, resolution, and encoding. One profile serves both recording and streaming; the "recording profile" name is historical.

**Default to a pre-defined profile.** Write a custom one only when none fits.

## Don't write a custom profile when

- The user is new to Aria.
- The data goes to MPS — custom profiles risk incompatibility.
- The data must be shareable or comparable across studies.
- A pre-defined profile hasn't been tried yet.

---

## 1. Start from a device profile

Sources on device: **native** (firmware), **cloud** (server-pushed), **user** (yours — the only writable one).

**Pull a base off the device and edit it. Never transcribe one from docs, and never start blank.** A pull returns what the firmware actually runs.

Client SDK does list / pull / add / remove. **Commands are in the `client-sdk` skill — don't guess them.**

`add` is non-destructive, takes one profile, and won't overwrite. To iterate: remove, then add.

| Profile | Use for | Notable |
|---|---|---|
| `profile8` | General-purpose recording | RGB 10 Hz @ 2560×1920, SLAM 30 Hz, all environmental sensors on |
| `profile9` | General-purpose streaming | RGB 5 Hz CBR (8 Mbps + blur filter), no GPS / BLE / WiFi / ET cameras |
| `profile10` | High-frame-rate RGB | RGB 30 Hz @ 2016×1512, otherwise like `profile8` |
| `mp_streaming_demo` | Machine-perception streaming demo | Streaming-only, lighter encoding, no ALS / baro / mag / GPS / BLE / WiFi |

Catalog: https://facebookresearch.github.io/projectaria_tools/gen2/technical-specs/device/profile

---

## 2. Hard rules — breaking these stops the session

| Rule | Failure |
|---|---|
| Every field name spelled exactly; no unknown fields | Upload rejected |
| `name` non-empty and unique across all three sources | Upload rejected |
| `imus` / `magnetometer` / `barometer` / `gps` rate on the allow-list | Rejected |
| Camera `rate_hz` ≤ 90 | Rejected |
| RGB `rate_hz` within the ceiling for its `resolution` (24 at 4032×3024, else 30) | Rejected |
| `cqp.qp` in 0–51 | Stream won't start |
| `fixed_exposure.gain` positive **and** inside the camera's range | Rejected |
| `encoding: YUV` only on `rgb_camera` | Rejected |
| Valid enum values everywhere, including `wifi.scan_mode` | Rejected |
| `rgb_camera.resolution` recognised | **Dataflow service aborts** |
| ALS `exposure_time_us + 6000` ≤ sample period | Rejected |
| ALS `exposure_mode: FIXED` requires `exposure_time_us` | Rejected |
| **Never enable ML ET and `ht` together** | Session won't start |

**Geometric ET: at most one of ET images / gaze results may exceed 5 Hz.** A third-party IP restriction, and a hard requirement — do not rely on the device to catch a violation for you. See the `et.type` section.

---

## 3. Things the device changes without telling you

| You wrote | Device does |
|---|---|
| Camera `rate_hz` not equal to `180/N` | Snaps **up** to the next `180/N` (25 → ≈25.7) |
| `ppg.rate_hz` in range | Snaps to nearest achievable |
| `als.rate_hz` | Snaps to a 20.5 ms grid |
| `als.exposure_time_us` | Rounds to nearest 1600 µs tick |
| `temperature.rate_hz` < 1 | Clamps to 1 |
| `cnr.strength` > 10 | Clamps to 10 |
| `als.gain_index` outside 1–15 | Sensor default |
| `fixed_exposure` value ≤ 0 | Replaced with 1000 µs / gain 1.0 **before** the range check, so it never reaches the driver. Only a positive out-of-range gain is rejected. |
| `iframe_period` ≤ 0 | 1 |
| Blur `threshold` < 0 / `window_ms` ≤ 0 | 60 / 100 |
| `encoding` unset | **H265**, and it beats the RAW8 a CV vertical asks for — see below |
| Any `blur_filter_config` block, even `enabled: 2` | Pulls in `vio_high_frequency_pose` → `vio` → `slam_cameras` + `imus` |
| A whole top-level block omitted | Feature **off**, not defaulted. `"audio": {}` gets you the defaults; no `audio` key gets you no audio. |
| Deprecated top-level `qp` | Migrated to `cqp` |
| `gps` / `ppg` on a board without the hardware | Removed from the profile |
| CQP H265 under thermal/power pressure | Rewritten to CBR with a bitrate cap — recorded quality won't match your QP |

---

## 4. JSON conventions

- Field names are `snake_case`.
- Enums accept the integer or the name: `"encoding": 4` == `"encoding": "H265"`.
- `0` / `UNDEFINED` means unset — it yields to defaults, it does not force anything.
- `OptionalBool` is a 3-state enum, not a JSON bool: `0`=UNDEFINED, `1`=TRUE, `2`=FALSE.
- Presence enables: write empty-message toggles as `"button_state": {}`.
- A file is a JSON array of profiles, or a single profile object for the inline path.
- Units: `rate_hz` Hz (float), `period_ms` ms (int32), `*_us` µs (int32).
- On merge, `rate_hz` takes the **max** and `period_ms` the **min**.

---

## 5. Field reference

### Top-level

| Field | Type | Meaning |
|---|---|---|
| `name` | string | **Required.** Selection key. Unique across native + cloud + user. |
| `description` | string | Free text. |
| `type` | `1`=RECORDING, `2`=STREAMING | Metadata only; does not affect runtime. |
| `recommended` | bool | UI hint. No runtime effect. |
| `public` | bool | `true` = visible in the SDK's profile list. No runtime effect. |

Selection at session time is by `name` — not filename, index, or `type`.

### Inertial & environmental

| Field | Sub-field | Legal values | Standard |
|---|---|---|---|
| `imus` | `rate_hz` | `200, 400, 800, 1600, 3200, 6400` | 800 (VIO forces 800) |
| `magnetometer` | `rate_hz` | `1.562, 3.125, 6.25, 12.5, 25, 50, 100, 200, 400` | 100 |
| `barometer` | `rate_hz` | `0.125, 0.25, 0.5, 1, 2, 3, 4, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 60, 70, 80, 90, 100, 120, 130, 140, 150, 160, 180, 200, 220, 240` | 50 |
| `gps` | `rate_hz` | `1, 2, 4, 5, 8, 10` | 1 |
| `ppg` | `rate_hz` | ≈1 … ≈1927 | 128 |
| `temperature` | `rate_hz` | ≥ 1. Sensor runs even if omitted. | 1 |
| `als` | `rate_hz` | ≈0.19 … ≈131.6 | 10 |
| `als` | `exposure_time_us` | `1600 … 1600000`. In AUTO it only seeds the loop; in FIXED it is held. | 3200 |
| `als` | `exposure_mode` | `0`=UNSPECIFIED(→AUTO), `1`=AUTO, `2`=FIXED | AUTO |
| `als` | `gain_index` | `1 … 15`; `0` = sensor default. No auto-gain. | 0 |
| `utc_time_sync` | `period_ms` | Cadence. **Use `period_ms`** — `rate_hz` parses but never starts the subsystem. | 60000 |
| `device_info` | `period_ms` | Battery + proximity polling. Also enables the proximity sensor. | 5000–30000 |

### Audio

```jsonc
"audio": {
  "sample_rate": 1,           // 1=16kHz, 2=48kHz
  "frame_period": 3,          // 1=5ms, 2=10ms, 3=20ms
  "sample_format": 1,         // 1=S16, 2=S32
  "encoding_format": 2,       // 1=PCM, 2=OPUS
  "mic_selection": { "1": true, "3": true },              // keys 1-8; empty/omitted = all 8
  "opus_config": { "complexity": 10, "bitrate": 256000 }  // OPUS only; complexity 0-10
}
```

Unset sub-fields default to 48 kHz / 20 ms / S32 / PCM.

### Connectivity

```jsonc
"ble":  { "period_ms": 30000, "duration_ms": 2000 }
"wifi": { "period_ms": 30000, "scan_mode": 2 }      // 1=PASSIVE, 2=ACTIVE
```

The numeric values are passthrough — no range check. `scan_mode` is an enum and is validated like every other enum.

### Cameras — shared

All cameras run off one **180 Hz trigger**. Achievable rate = `180/N` for any integer N, capped at 90. Common values: 90, 60, 45, 36, 30, 25.7, 22.5, 20, 18, 15, 12.9, 10, 9, 6, 5. Anything else snaps up to the next `180/N`. `rate_hz: 0` disables the group.

`encoding` (`CameraEncodingFormat`): `1`=RAW8, `2`=RAW10, `3`=YUV (RGB only), `4`=H265 (needs `video_config`).

**`encoding` decides what gets recorded, not what the verticals consume.** VIO / HT / ET always get the RAW8 they need internally; both streams exist. So:

- Want compressed frames on disk: leave `encoding` unset (it defaults to H265) or set `4`. The vertical still works.
- Want raw frames on disk: set `encoding: 1` explicitly. Leaving it unset will **not** give you RAW8 — the H265 default wins over the vertical's RAW8 request.

```jsonc
"video_config": {
  "iframe_period": 30,
  "cqp": { "qp": 24 }                  // or "cbr": { "bitrate": 10000000 }
}
```

`qp` is 0–51 on every camera. `cbr.bitrate` is bps, unvalidated — 900 kbps (SLAM) to 30 Mbps (RGB) are typical. The schema also has a `cvbr` arm: **do not use it**, the firmware has no CVBR support and it silently degrades to QP 0.

Exposure is a `oneof` — `auto_exposure` or `fixed_exposure`:

```jsonc
"auto_exposure": {}                                   // or:
"fixed_exposure": { "exposure_us": 250, "gain": 6 }
```

`auto_exposure`'s range sub-fields are ignored; only `mode` is read (`1`=DEFAULT, `2`=GYRO, the gyro motion-blur-adaptive AEC on RGB).

| Camera | Gain | Exposure (µs) |
|---|---|---|
| SLAM | 0.1–8.0 | 266–16000 |
| ET | 1.0–16.0 | 8–1300 |
| RGB (POV) | 1.0–255.94 | 26–99878 |

RGB auto-exposure works in a narrower window: gain 1.0–128.0, exposure 26–14000 µs.

SLAM auto-exposure is recomputed whenever the SLAM rate changes: the ceiling is `min(frame_period_us, 16000) − 1000`, and the AEC then works within **267–10000 µs**. In practice that means 10 ms at any rate the 90 Hz camera ceiling allows.

### `slam_cameras`

```jsonc
"slam_cameras": {
  "rate_hz": 30,
  "auto_exposure": {},
  "encoding": 4,
  "video_config": { "cqp": { "qp": 24 } }
}
```

512×512 on shipping Aria Gen 2 (a board property, not a profile field). All four cameras always come up together — `camera_selection` exists in the schema but the device ignores it.

### `et_cameras`

```jsonc
"et_cameras": {
  "rate_hz": 5,
  "auto_exposure": {},
  "encoding": 4,
  "video_config": { "cqp": { "qp": 22 } },
  "resolution": 1,                // 1=200×200 (default), 2=400×400
  "ir_led": 1,                    // OptionalBool; UNDEFINED → on
  "trigger_output_enabled": 1     // OptionalBool; pulses the external HW trigger with the ET trigger
}
```

Both ET cameras always come up together; `camera_selection` is ignored here too.

### `rgb_camera`

```jsonc
"rgb_camera": {
  "rate_hz": 10,
  "auto_exposure": {},
  "encoding": 4,
  "video_config": { "cqp": { "qp": 22 } },
  "resolution": 3,                // see the rate ceiling table below
  "width": 2560, "height": 1920,  // downscaled OUTPUT size, applied before encode
  "blur_filter_config": {
    "enabled": 1,                 // OptionalBool; UNDEFINED → false
    "threshold": 60,              // drop frames scoring above this; 0 = drop nothing; 9999 = score only
    "window_ms": 400              // guarantees ≥1 frame per window
  },
  "sharpening_level": 0,          // 0-10, unvalidated
  "denoising_level": 0,           // 0-10, unvalidated
  "awb": { "source": 1 },         // presence enables. 1 = ALS colour temp (needs `als`), 2 = from pixels
  "lsc_enabled": 1,               // needs `awb`
  "cnr": { "strength": 5 },       // presence enables; strength 0-10
  "cac_enabled": 1
}
```

Each `resolution` is a distinct sensor mode with its own rate ceiling — the RGB camera never exceeds 30 fps:

| `resolution` | Capture | Max `rate_hz` |
|---|---|---|
| `1` | 1008×756 | 30 |
| `2` | 2016×1512 (default) | 30 |
| `3` | 4032×3024 | **24** |
| `4` | 3024×3024 | 30 |

`width`/`height` are the downscaled output size, applied before encode, and have no allow-list; 2560×1920 and 320×240 are known-working.

### Machine-perception verticals

Enable the vertical and let the device add its dependencies — don't wire up the raw sensors yourself.

| Field | Form | Auto-adds |
|---|---|---|
| `vio` | `{rate_hz, extract_image_keypoints, output_tracks}` | `slam_cameras`@rate (RAW8) + `imus`@800 |
| `vio_high_frequency_pose` | `{rate_hz}` | `vio` (→ SLAM + IMU); sets `imus` to its rate |
| `et` | `{rate_hz, type}` | `et_cameras`@rate (RAW8) + IR LED on |
| `ht` | `{rate_hz}` | `slam_cameras`@rate (RAW8) |

`vio.extract_image_keypoints` defaults to **off** — set `1` if you want keypoints. `vio.output_tracks` enables image-track output.

**A vertical raises the sensor's physical rate; the `rate_hz` on the raw sensor is what gets delivered.** Rates merge by max, then the device decimates to your requested rate. `et: 30` + `et_cameras: 5` runs the ET cameras at 30 Hz for gaze and records the images at 5 Hz.

### `et.type` — gaze algorithm

`2` / `ET_ML_TURING` selects ML ET. Omitted, `0` / `ET_TYPE_UNDEFINED`, or `3` / `ET_GEOMETRICAL` all mean geometric ET, the default.

- ⛔ **ML ET cannot run with `ht`.** The device errors and the session never starts. Geometric ET has no such conflict — the pre-defined profiles run it alongside `ht`.
- **Geometric ET carries a third-party IP restriction: at most one of ET images / gaze results may exceed 5 Hz.** For gaze above 5 Hz hold `et_cameras.rate_hz` at 5 (this is what `profile8` does: `et` 30 Hz, `et_cameras` 5 Hz); for images above 5 Hz hold `et.rate_hz` at 5. Treat it as a hard requirement. ML ET is exempt.

The gaze stream's flavor differs per algorithm, so readers must dispatch on it.

### Misc

`"button_state": {}` — capture button-state events.

---

## 6. Warn the user about

Thermal shutdown (~44 °C), shorter battery life, missing streams from untested sensor combinations, MPS rejection, PAT load failures. Meta does not guarantee compatibility for custom profiles.

---

## 7. Deploy

**Client SDK:** add it (see the `client-sdk` skill), then select it by `name`.

**Companion App:** Headset Details → **Recording Profiles** → **Manage recording profiles** → pick one → **Make a copy** → edit → **Save**. Or **+** for a blank editor. The saved profile appears under **My profiles** and works for both recording and streaming despite the UI label.
Reference: https://facebookresearch.github.io/projectaria_tools/gen2/ark/companion-app/recording-profiles

---

## 8. Verify

Record something short, then:

- Run `vrs-health-check` (see that skill) — required before MPS.
- Load in PAT (see `projectaria-tools`) and confirm every expected stream is there.
- Compare actual data rates against what you configured; a large gap means thermal throttling or a sensor that didn't start.
- Watch device temperature and battery.

---

## Related plugin skills

- `aria-knowledge` — what a profile is conceptually + the pre-defined list.
- `client-sdk` — pull / add / remove profiles; recording and streaming workflow.
- `vrs-health-check` — validate a recording before downstream use.
- `projectaria-tools` — inspect the resulting VRS file.
