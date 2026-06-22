# Prius 2017 Tuning Log

## v0 — baseline (no changes)
Pinned `9614e5b`. Torque defaults: `LAT_ACCEL_FACTOR=1.60  MAX_LAT_ACCEL_MEASURED=1.502  FRICTION=0.1515`.
Device params (curve/torque) all default/off. Saved on device: `/data/prius-tune-baseline-v0.txt`.

## v1 — enable camera-based curve slowdown  (device params; PENDING test drive)
- `TurnVisionControl = 1`  (slow before curves so it won't bail; targets ~1.9 m/s² lat accel)
- `VisionCurveLaneless = 1`  (follow curve via end-to-end path)
- Left OFF: `TurnSpeedControl` (needs map data), `LiveTorque` (keep tuning deterministic)
- Expected: "curve too tight, take control" stops; car eases speed into curves.
- Result: _pending batched test drive_

## v3 — DM tolerance for sunglasses / setting sun  (code: helpers.py; DRAFT, validate via replay)
`selfdrive/monitoring/helpers.py` — keep the old (non-phone) DM, just relax detection:
- `_SG_THRESHOLD`   0.9 → 0.5   (recognize sunglasses sooner → stop trusting unseeable eyes)
- `_FACE_THRESHOLD` 0.7 → 0.55  (tolerate sun-glare face washout)
- `_POSESTD_THRESHOLD` 0.3 → 0.4 (tolerate noisier head-pose in glare)
- Values are a first pass — tune to the actual sunglassesProb/faceProb/poseStd seen in Ryan's
  replayed sunglasses+sunset footage; confirm it still catches genuine inattention.
- **Replay validation (5 recent routes, 9.6k+ frames):**
  - Sunglasses worn 69-93% of every drive. `_SG_THRESHOLD` 0.9→0.5 eliminates the 234-480 frames/drive
    where old code trusted unseeable eyes → **~0**. Validated, helps every drive. ✅
  - Glare half (`_FACE_THRESHOLD`/`_POSESTD_THRESHOLD`): no sunset/glare drive in recent recordings
    (noFace% ~0 everywhere), so unexercised. Conservative (only act when face detection drops in glare);
    keep, confirm on a real sunset drive. ⚠️ unvalidated.

## v2 — curve target lowered to the Prius's real limit  (code: vision_turn_controller.py)
**Data (replay, 5 routes ~50k lat frames):** when the torque controller SATURATES, desired lateral
accel = **~1.6 m/s² median (1.72 avg)** — i.e. the Prius EPS runs out of steering ~1.6 m/s²
(matches static `MAX_LAT_ACCEL_MEASURED=1.502`). No EPS hardware faults seen → "take control" is
openpilot hitting that ceiling.
**Insight:** Vision Turn Control's `TARGET_LAT_A` was **1.9** > the car's ~1.6 ceiling, so even with
v1 on, it aimed too high and would still bail on tight curves.
**Change:** `TARGET_LAT_A` 1.9 → **1.5** (just under measured ceiling) → slows enough to stay below
saturation. Pairs with v1 (vision turn control enabled).
- Did NOT raise MAX_LAT_ACCEL/torque: saturation reflects the real EPS torque cap; respect it by
  slowing, don't pretend the car can pull more.
- Result: _pending test drive (this changes speed/closed-loop, so replay can't fully simulate it)_

## Planned
- v4: swap in newer driving model (curves + silver cars), verify modeld compatibility
- (maybe) refine TARGET_LAT_A 1.4-1.6 on the drive for feel vs margin
