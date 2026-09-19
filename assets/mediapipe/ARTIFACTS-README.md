# TEMP PoC — landmarker compare results (Vision vs MediaPipe vs Python golden)

**Status: temporary proof-of-concept artifacts. Not production evidence. Do not ship.**

Checked-in results of running the standalone smokes over the two checked-in
golden samples. Layout per sample:

```
artifacts/<sample>/
  sim/     # scripts/face-retouch-sim-smoke.sh — MediaPipe + Vision arms,
           # iOS simulator (see quirk note below on the sim Vision arm)
    strip.png            # before | python | mediapipe | vision
    mask_mediapipe.png   # refined soft mask, MediaPipe arm
    mask_vision.png      # refined soft mask, Vision arm (sim-degraded)
    report.json          # per-arm faces/face_width/timings/diff-vs-golden
  macos/   # scripts/face-retouch-smoke.sh — Vision arm only, macOS host
           # (the geometry reference for the Vision arm)
    strip.png            # before | python | vision
    mask_vision.png
    report.json
```

## Environment / provenance

- Xcode 26.5 (17F42), `swiftc -O`; simulator runs on a throwaway iPhone 16
  device, iOS 27.0 runtime, arm64, via `xcrun simctl spawn` (no app install).
- MediaPipeTasksVision + MediaPipeTasksCommon **0.10.35** (what `pod install`
  resolves for the podspec's `~> 0.10.14`), simulator xcframework slices,
  downloaded at script runtime from the CocoaPods CDN source URLs.
- Model: `face_landmarker.task` (float16, `latest`) downloaded at script
  runtime from `storage.googleapis.com/mediapipe-models`. **Neither the
  library nor the model is committed or bundled — no legal clearance is
  claimed** (see the PoC README licensing section).
- Python golden numbers from `goldens/<sample>/report_python.json`
  (`vsco/ai-lab-eval` classical pipeline, MediaPipe landmarks, full stage
  list — the Swift core intentionally ports only a subset, see PoC README).

## Geometry / mask closeness vs Python golden

Mask IoU computed with `smoke/mask_iou.py` (binary @0.5 vs
`goldens/<sample>/mask_python.png`; soft L1 over [0,1] masks).

| sample | arm (env) | face_width px (python) | mask_refined_px (python) | mask IoU@0.5 | soft L1 | diff vs python after: mean_abs / max_abs (u8) |
|---|---|---|---|---|---|---|
| obama | mediapipe (sim) | 225.2 (225.6) | 45 897 (46 707) | **0.946** | 0.0022 | 0.079 / 54 |
| obama | vision (macOS) | 226.1 (225.6) | 43 695 (46 707) | **0.852** | 0.0058 | 0.088 / 54 |
| obama | vision (sim, degraded) | 73.2 (225.6) | 8 440 (46 707) | 0.149 | 0.0327 | 0.153 / 73 |
| jemison | mediapipe (sim) | 163.1 (163.1) | 21 549 (22 209) | **0.919** | 0.0015 | 0.047 / 33 |
| jemison | vision (macOS) | 164.9 (163.1) | 21 254 (22 209) | **0.878** | 0.0023 | 0.048 / 32 |
| jemison | vision (sim, degraded) | 51.9 (163.1) | 4 507 (22 209) | 0.169 | 0.0148 | 0.058 / 47 |

Reading:

- **MediaPipe arm ≈ Python geometry** (face_width within 0.4 px, IoU
  0.92–0.95). Expected — same landmark source. Residual IoU gap is the
  Swift `refine_mask` deviations (no `skin_probability` term, box-blur
  feather), not landmarks.
- **Vision arm (macOS geometry reference) is close but looser**
  (face_width within ~1.8 px, IoU 0.85–0.88, masks ~4–6 % smaller).
  Gap concentrated at the synthesized forehead arc + missing nostril
  exclusion (documented PoC approximations).
- The `diff_vs_python_golden` after-image stats are similar for both arms
  and dominated by the intentionally unported stages (heal, even-tone,
  dark-circles) — sanity signal only, not a fidelity gate.

### Known quirk, reproduced: Vision-on-simulator landmarks are degraded

On the iOS simulator, Vision only runs with `usesCPUOnly` (the sim cannot
create a GPU/ANE inference context) and returns face_width ≈ 73/52 px vs
≈ 226/165 px from the identical code on macOS (which matches Python). This is
the previously observed RN/sim quirk — **use `macos/` numbers as the Vision
geometry reference**, and the sim Vision run only as proof the arm executes
in an iOS environment.

## Latency

| sample | arm (env) | detect_ms | core total_ms |
|---|---|---|---|
| obama | mediapipe (sim) | 512.6 | 254.5 |
| obama | vision (sim) | 594.4 | 548.7 |
| obama | vision (macOS) | 69.3 | 163.0 |
| jemison | mediapipe (sim) | 547.5 | 629.4 |
| jemison | vision (sim) | 564.9 | 525.5 |
| jemison | vision (macOS) | 63.9 | 172.4 |

Python reference (`report_python.json`, full pipeline incl. unported stages):
obama 1 488.9 ms, jemison 814.8 ms total; detect+landmarks 475.0 / 50.3 ms.

Caveats: simulator timings are CPU-only emulation on an arm64 Mac host
(MediaPipe detect includes first-run model/graph init; Vision sim is the
degraded CPU path) — treat them as existence proof, not device performance.
Real latency comparison needs a device run.

## Repro

```bash
scripts/face-retouch-smoke.sh                                  # macOS Vision arm
FACE_RETOUCH_SMOKE_SAMPLE=jemison scripts/face-retouch-smoke.sh
scripts/face-retouch-sim-smoke.sh                              # sim MediaPipe+Vision arms
FACE_RETOUCH_SMOKE_SAMPLE=jemison scripts/face-retouch-sim-smoke.sh
```
