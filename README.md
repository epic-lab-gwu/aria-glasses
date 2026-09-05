# Meta Project Aria Gen 2

## 1. What this project is

Meta Project Aria Gen 2 research glasses — egocentric capture: 7 cameras, 2 IMUs, and on-device machine perception (VIO, eye tracking, hand tracking).

**Downstream consumer:**

| Destination | What it wants | Status |
|---|---|---|
| GI Labs (gilabs.xyz) | egocentric video + hand tracking for robot learning | repo access pending |

## 2. Status as of 2026-09-05

Bring-up complete. Every sensor confirmed live in rerun on 2026-09-03: 4 CV cameras (independent viewpoints), RGB, both ET cameras, both IMUs, 7 mics + contact mic, barometer, magnetometer. Hand skeletons rendering in 2D and articulated in 3D. VIO trajectory accumulating.


## 3. Hardware

- 7 cameras: 4 CV/SLAM + 1 RGB + 2 eye-tracking
- CV cameras: global shutter, 120 dB dynamic range, 80° stereo overlap
- 2× IMU, barometer, magnetometer, 7 spatial mics, contact mic, GNSS (L1), PPG (heart rate), ambient light sensor with UV
- On-device machine perception on a dedicated coprocessor, streamable live:
  - **VIO** — 6DoF pose
  - **Eye tracking** — per-eye gaze, vergence, blink, pupil center + diameter
  - **Hand tracking** — 21 landmarks, articulated 3D joint poses

> Gen 2 runs eye and hand tracking on-device. Gen 1 required MPS post-processing for these. Do not follow Gen 1 guides here.

## 4. Environment — the setup that works

**Platform constraint:** the Client SDK has no Windows build. Linux x64 (Ubuntu 22.04/24.04 LTS, Fedora 40/41) or macOS ARM64 only.

```bash
conda create -y -n aria python=3.12
conda activate aria
pip install projectaria-client-sdk      # SDK + CLI
pip install projectaria-tools           # VRS reading
pip install projectaria-mps             # MPS requests
aria_doctor                             # configures PC for device discovery
python -m aria.extract_sdk_samples --out ~/Downloads
```

Python must be 3.10–3.12. Not 3.13.

Every terminal needs `conda activate aria`. `auto_activate` is set to false, and bare `python` does not exist on Ubuntu outside the env. This is the cause of `"python: command not found"` in a fresh tab.

**The detour — do not repeat it**

- `python3 --version` said 3.13.13. That was miniforge base auto-activating, shadowing the system Python — not a distro Python. Diagnose with `readlink -f $(which python3)` before believing any version.
- Built CPython 3.12.14 from source. Unnecessary.
- `python3.12 -m venv` then failed — apt's `python3.12` ships without ensurepip (needs `apt install python3.12-venv`).
- `conda create -n aria python=3.12` worked first try. Miniforge was already installed the whole time.

## 5. Quick start — zero to validated

```bash
# 1. Phone: Companion App — pair glasses, run EYE GAZE CALIBRATION
# 2. Connect glasses by USB (data cable, direct port, no hub)

conda activate aria
aria_gen2 device list        # PC sees the glasses?
aria_gen2 auth pair          # APPROVE THE PROMPT ON THE PHONE
aria_gen2 auth check

# 3. Live validation — two terminals
aria_gen2 streaming start                          # tab 1, defaults are correct
aria_streaming_viewer --real-time --interpolate    # tab 2

# 4. Numbers, not impressions
python stream_audit.py --list-api                  # fix callback names first
python stream_audit.py --seconds 30 --expect slam=30,rgb=30,imu=800

aria_gen2 streaming stop     # BEFORE unplugging

# 5. Record (untethered from the phone is fine and higher fidelity)
aria_gen2 recording start --profile profile10 --recording-name test01
aria_gen2 recording stop
aria_gen2 recording list                           # note the UUID
aria_gen2 recording info -u <uuid>
aria_gen2 recording download -u <uuid> -o ~/aria_data/

# 6. Verify the file
aria_rerun_viewer --vrs ~/aria_data/<file>.vrs
python vrs_inspect.py ~/aria_data/<file>.vrs
```

`auth pair` is one-time per device+PC. It survives unplugging. It looks frozen — it is waiting on the phone.

## 6. The four data paths

| Path | Use for | Notes |
|---|---|---|
| Companion App (phone) | pairing, eye gaze calibration, profile select, start/stop | Cannot show live camera or tracking output |
| Client SDK + CLI | scripted recording, bulk download, live streaming | Linux/macOS only |
| projectaria_tools | reading/decoding VRS offline | separate pip package |
| MPS (cloud) | closed-loop trajectory, semi-dense points, hand tracking | uploads to Meta |

**Rule of thumb:** calibrate on the phone, validate tethered on Linux, record untethered from the phone, download to Linux over USB.

## 7. Recording profiles

Use **profile10**. Same CV cameras and IMU as profile8 (so VIO work is indifferent), but 30 Hz RGB instead of 10 Hz — and 10 fps is not video.

| | profile8 | profile10 |
|---|---|---|
| CV cameras | 30 Hz, 512×512 | 30 Hz, 512×512 |
| RGB | 10 Hz, 2560×1920 | 30 Hz, 2016×1512 |
| ET cameras (raw) | 5 Hz, 200×200 | 5 Hz, 200×200 |
| IMUs | 800 Hz | 800 Hz |
| ET · hands · VIO | 30 · 30 · 10 Hz | 30 · 30 · 10 Hz |

Cost: roughly 2× the RGB data rate. Matters only for long sessions.

`profile9` and `mp_streaming_demo` are streaming profiles and are not valid for `recording start`. `mp_streaming_demo` is the streaming default and is the right one for live eye/hand validation.

Full tables in [[recording-profiles]].

## 8. Confirmed stream labels

From rerun on this device, 2026-09-03. Not the Gen 1 names.

```
camera-rgb
slam-front-left    slam-front-right
slam-side-left     slam-side-right
camera-et-left     camera-et-right
imu-left           imu-right
mic0 … mic6        contact_mic
baro_P   baro_T    mag0
```

## 9. Files in this folder

| File | Purpose |
|---|---|
| README.md | this file |
| CLAUDE.md | scope + working rules |
| MEMORY.md | running state: versions, corrections, verified checklist |
| setup-runbook.md | step-by-step from unboxing to VRS |
| sensor-validation-checklist.md | what to do, in order |
| acceptance-tests.md | how to judge whether it's really working |
| recording-profiles.md | full profile sensor tables |
| vrs_inspect.py | enumerate streams, counts, rates, calibration |
| vrs_extract.py | one camera + one IMU → undistorted PNGs + CSV + calib |
| stream_audit.py | live stream correctness with hard numbers |
| live_sensor_check.py | superseded by stream_audit.py |

All scripts are marked UNTESTED. The projectaria_tools and SDK APIs have moved between releases. Correct them in place and record the fix in MEMORY.md.

## 10. Validation methodology

Three failure modes, none of which throw an error, all of which produce a healthy-looking sample count:

- **FROZEN** — value stops updating but keeps being reported
- **ZEROED** — output pinned at 0.0 regardless of input
- **DECAYING** — rate degrades over seconds (bandwidth or thermal)

Every test is built so you already know the correct answer before you look at the output. No published accuracy specs needed.

| Channel | Test | Pass |
|---|---|---|
| Cameras | cover one lens | exactly one feed goes dark |
| Eyes | cup hands over eyes 10s, remove | pupil dilates then constricts |
| Hands | hands into pockets | goes None within a beat |
| VIO | walk a loop, return to start | translation returns near origin |

The pupil test is strongest — pupillary light response is involuntary. The hand dropout test matters most — a frozen last-known pose looks perfect in a viewer.

`stream_audit.py` checks these numerically, plus gap percentiles, timestamp monotonicity, and arrival-vs-capture jitter (which separates transport batching from sensor faults).

## 11. MPS (Machine Perception Services)

```bash
pip install projectaria-mps
aria_mps single -i ~/aria_data/<recording>.vrs
```

Outputs:

```
slam/closed_loop_trajectory.csv      ← loop-closed 6DoF trajectory
slam/open_loop_trajectory.csv        ← odometry-only trajectory
slam/semidense_points.csv.gz         ← 3D structure
slam/online_calibration.jsonl        ← time-varying calibration, beats factory
hand_tracking/hand_tracking_results.csv
```

MPS is how you get an offline closed-loop trajectory, semi-dense structure, and hand tracking from a recording.

Two caveats: MPS is a cloud service (recordings upload to Meta — decide deliberately which ones, given GI Labs involvement). And the Gen 2 MPS docs list no eye gaze service; Gen 1 had one.

## 12. Documentation drift — corrections found

The published docs have been wrong or stale more than once. The installed binary and the hardware are authoritative.

| Documented | Reality |
|---|---|
| `aria_gen2 device profile list` | `device` subcommand not found on this build |
| `extract_sdk_samples --output` | one page says `--out`; check `--help` |
| Gen 1 labels `camera-slam-left/right` | Gen 2 uses `slam-front-*` / `slam-side-*` |
| Gen 1 MPS includes eye gaze | Gen 2 MPS page does not list it |

Always start from `aria_gen2 --help`.

## 13. Open questions

1. Are gaze / hands / VIO written into the VRS, or stream-only? The docs list them in the recording profile table with rates but never say where they land. If stream-only: hands recoverable via MPS, but eye gaze has no offline path on Gen 2. Settle with `vrs_inspect.py` on a real recording.
2. Eye gaze output (yaw/pitch/vergence) not yet observed — raw ET camera images confirmed, computed estimate not.
3. Ubuntu version on the laptop.
4. Whether `aria_streaming_viewer` and `stream_audit.py` can bind port 6768 at the same time.
5. Real profile-listing command on this SDK build.

## 14. Gotchas

- No Windows build for the Client SDK.
- Conda base auto-activation masquerades as the system Python.
- apt's `python3.12` has no ensurepip.
- `auth pair` waits silently on the phone.
- Use a data USB cable, direct port, no hub.
- Run `streaming stop` before unplugging.
- Eye gaze calibration is per-user and Companion-App-only. Uncalibrated gaze fails acceptance tests on perfectly good hardware.
- Raw ET cameras record at 5 Hz even though ET processing runs at 30 Hz.
- Mixed time domains in one file — baro/mag showed epoch-1970 stamps while cameras used `device_time`. Harmless in a viewer, silently fatal in a converter. Read every stream in the same TimeDomain.
- Wi-Fi streaming batches and drops frames under load. USB for anything full-rate. Untethered recording has no such problem — it writes to internal storage at full profile rates.
- Verify a downloaded VRS opens before any recording delete. `delete-all` has no undo.

## 15. References

- [Gen 2 docs](https://facebookresearch.github.io/projectaria_tools/gen2/)
- [CLI reference](https://facebookresearch.github.io/projectaria_tools/gen2/ark/client-sdk/cli-reference)
- [Client SDK install](https://facebookresearch.github.io/projectaria_tools/gen2/ark/client-sdk/start)
- [Recording profiles](https://facebookresearch.github.io/projectaria_tools/gen2/technical-specs/device/profile)
- [ROS 2 example](https://facebookresearch.github.io/projectaria_tools/gen2/ark/client-sdk/python-sdk/ros2-example)
- [MPS](https://facebookresearch.github.io/projectaria_tools/gen2/ark/mps/start)
- [Camera intrinsic models](https://facebookresearch.github.io/projectaria_tools/docs/tech_insights/camera_intrinsic_models)
- [projectaria_tools](https://github.com/facebookresearch/projectaria_tools)
- [Depth from stereo (Gen 2)](https://github.com/facebookresearch/projectaria_gen2_depth_from_stereo)
- [GI Labs](https://www.gilabs.xyz/) · [EGO1 writeup](https://www.gilabs.xyz/blog/ego1)
- Support — AriaOps@meta.com
