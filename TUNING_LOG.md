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

## v4 — NNLC (neural-net steering) ENABLED  (device param)
`NNFF = 1`. Replaces the generic linear torque controller with a **per-car neural-net feedforward**
trained on Prius EPS data. Verified: matched `lat_models/TOYOTA_PRIUS.json`, **similarity 1.0** (exact),
FluxModel loads OK. Engages at next ignition (reads `NNFF` at car init).
- Why: biggest steering-quality upgrade for Toyotas; smoother/more precise, and **holds curves with less
  torque → less saturation → less bailing** (stacks with v1/v2's slow-for-curves).
- Closed-loop change → only a real drive confirms feel. Reversible: `NNFF=0`.
- Result: _pending test drive_

## Deployed build = "full" + NNLC  (v1 params + v2 + v3 code + v4 NNLC param)
All active on next drive. Reverts: see README profile-switching.

## Planned / future
- v5 (deferred, risky): swap in newer driving model (silver cars), needs road-side modeld port
- (maybe) refine TARGET_LAT_A 1.4-1.6 on the drive for feel vs margin
- after a drive: pull logs, check NN engaged + curve/DM behavior, retune
