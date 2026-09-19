# Face-retouch RN / iOS-sim `face_width` ~73px quirk — dig notes

Date: 2026-09-19 (PT)  
Branch / PR: `poc/face-retouch-landmarker-compare` (studio-next #570)  
Scope: read-only code + artifact review (no Slack / Jira / GitHub writes)

## Root-cause verdict (one sentence)

**Vision-on-iOS-simulator (forced `usesCPUOnly`) returns a shrunken landmark mesh in-place — not an RN/Documents scale bug and not a Y-flip / UIKit-vs-Vision math error in our oval code — so sim Vision geometry is unreliable; macOS Vision and device Vision are the fidelity references.**

Hypothesis rank: **(A) primary** ≫ (B) secondary diagnostic ≫ (C) ruled out ≫ (D) none needed.

## How `face_width_px_min` / oval is computed

Shared core (`modules/face-retouch-poc/ios/`):

1. **`VisionLandmarkProvider.buildFace`**
   - `VNDetectFaceLandmarksRequest` → regions via `pointsInImage(imageSize:)` (lower-left) → flip Y to top-left pixels.
   - `#if targetEnvironment(simulator)` sets **`request.usesCPUOnly = true`** (sim cannot create GPU/ANE inference context).
   - Forehead arc synthesized from jaw L/R extremes + `observation.boundingBox` top.
   - **`faceWidth = max(x) − min(x)`** over jaw + eyes + brows + lips + arc (not bbox width).
   - `faceOval = jaw + brows + arc`.

2. **`ClassicalRetouchCore.run`**
   - `face_width_px_min = min(faces.map(\.faceWidth))`
   - `guided_radius ≈ clamp(faceWidth/60, 2…40)` → sim Vision clamps to **2** (73/60), macOS gets **4**.

3. **Same provider** for Expo module (`FaceRetouchPocModule`) and Metro-free smoke (`poc/.../smoke/main.swift` via `face-retouch-sim-smoke.sh` / `face-retouch-smoke.sh`). No separate RN geometry path.

4. **`PocImageIO.loadCGImage`**: `CGImageSourceCreateImageAtIndex` only; no EXIF orientation option. Identical for macOS CLI and sim.

5. **MediaPipe** (sim only): `normalized × CGImage width/height` → top-left pixels; faceWidth = full-mesh X span. Correct ~225 on the **same** sim binary / JPEG.

## Evidence

| Path | obama `face_width` | jemison | notes |
| --- | ---: | ---: | --- |
| Python golden | 225.6 | 163.1 | MediaPipe mesh |
| Vision macOS CLI | **226.1** | **164.9** | same Swift sources, **no** `usesCPUOnly` |
| MediaPipe iOS sim | **225.2** | **163.1** | same process, same `before.jpg` |
| Vision iOS sim (smoke + RN) | **73.2** | **51.9** | `usesCPUOnly`; IoU@0.5 ≈ 0.15 / 0.17 |
| Vision bbox (earlier dig, outside oval path) | ~270–277 px | — | input file / Documents JPEG fine |

- **Shrink ratio not a clean constant**: 226.1/73.2 ≈ **3.09**, 164.9/51.9 ≈ **3.18** → argues against a single global scale (e.g. @3x or fixed analysis buffer) and for **detector quality / shrunken mesh**.
- **Y-flip / UIKit-vs-Vision (B) cannot explain X-only `face_width`**: width is pure X extent after `pointsInImage`; a double Y-flip would not shrink X.
- **Double-applying face-normalized→image via bbox** would predict ~(226/960)×277 ≈ **65**, not 73 → not a clean match for (B).
- **(C) RN / Documents / orientation ruled out**: Metro-free `simctl spawn` smoke on the host golden JPEG reproduces ~73; MediaPipe on that same `CGImage` is correct; public report notes bbox ~277 on Documents JPEG and RN-written `before.png`.
- **Only sim-specific Vision code difference** vs macOS: `usesCPUOnly = true` under `targetEnvironment(simulator)`.
- Repo already documents this: PoC README + `artifacts/README.md` call sim Vision “degraded” and tell readers to use **macOS Vision** as geometry reference.
- Mask collapse matches geometry failure: obama mask_refined 8 440 vs 43 695 macOS; IoU 0.149 vs 0.852 — tiny oval centered on the face (overlap), not a translated wrong-space mask.

## macOS CLI vs iOS sim / Expo

| | macOS smoke | iOS sim smoke / Expo module |
| --- | --- | --- |
| Sources | same `modules/face-retouch-poc/ios/*` | same |
| Target | macOS | `iphonesimulator` |
| Vision | GPU/default path | **`usesCPUOnly = true`** |
| MediaPipe | N/A (UIKit / pods) | optional, correct geometry |
| Image load | same `PocImageIO` | same |
| RN TS | N/A | thin wrapper only |

**RN does not need a geometry fix** — it hosts the same Swift Vision path; the sim Vision stack is the problem.

## Recommended fix plan (by likelihood)

1. **Policy (do now, no code required for product):** Treat Vision-on-simulator as an **execution smoke only**. Geometry / IoU / `face_width` gates use **macOS Vision** (`scripts/face-retouch-smoke.sh`) and/or **MediaPipe-on-sim**. Do **not** use Vision sim numbers to judge RN fidelity. *(Already stated in README — keep that the official stance.)*

2. **Verify on a physical device (highest-value experiment):** Run the Expo harness or a device-targeted smoke **without** `usesCPUOnly` (device builds skip the `#if simulator` branch). Expect `face_width` ≈ **226 / 165**. If device matches macOS, the quirk is confirmed sim-only and **no production code change** is required for shipping Vision.

3. **Optional diagnostic (1–2 hours, if you want to nail A vs residual B):** In `VisionLandmarkProvider` (sim-only log), dump per face:
   - `boundingBox` → pixel width
   - jaw `pointsInImage` X span
   - `normalizedPoints` X span × image width vs × bbox width
   - final `faceWidth`  
   Confirms shrunken mesh in correct location (A) vs mis-mapped norms (B).

4. **Optional harness hardening:** When `targetEnvironment(simulator)` and landmarker == vision, tag `report.json` with `"vision_sim_degraded": true` and/or skip Vision sim in compare tables so RN Debug runs stop looking like “RN is wrong.”

5. **Do not change for this quirk:** RN/TS, `PocImageIO` orientation (not implicated), MediaPipe as a production dependency, or “fixing” the oval math for sim. If device somehow also shrinks (unexpected), only then revisit coordinate math or Apple bug workaround.

## Does MediaPipe need to “fix” RN?

**No.** MediaPipe-on-sim already proves the classical core + image IO + RN/Expo wiring are fine (~225 face_width, IoU ~0.95). The failure mode is **Vision’s simulator CPU landmark path**. Product default remains **Vision on device / macOS reference**; MediaPipe stays an optional legal-gated parity bridge, not an RN fix.
