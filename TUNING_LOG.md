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

## Planned
- v2: tune `TARGET_LAT_A` (vision_turn_controller.py) + Prius `MAX_LAT_ACCEL`/friction (override.toml)
- v3: relax DM thresholds for sunglasses/sun (helpers.py) — keep old DM
- v4: swap in newer driving model (curves + silver cars), verify modeld compatibility
