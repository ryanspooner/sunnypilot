# sunnypilot — Toyota Prius 2017 (TSS-P) Tune

A focused tune of sunnypilot for the 2017 Toyota Prius Eco, built by pinning a known-good
release and **cherry-picking** improvements while **excluding** known regressions.

## Setup
- **Car:** TOYOTA_PRIUS (TSS-P / TSS1) — fingerprint confirmed on device
- **Device:** comma 3-class (aarch64), sunnypilot
- **Pinned base:** `release-c3` @ `9614e5b` (2025-03-30) — known-good; `release-c3` was retired upstream Oct 2025
- **Upstream now:** `v2026.001.x` line (latest seen: v2026.001.007, 2026-05-27)

## Strategy
Stay pinned to the known-good base. Take the gems, skip the landmines:
- 🟢 **Grab:** newer driving model (1.5M-min retrain → better curves + low-contrast/silver lead detection)
- 🔴 **Avoid:** newer driver-monitoring (adds `phoneProbValidCount` phone-detection → false "pay attention" alerts)
- 🟡 **Intel:** upstream reverted lateral to the older "v0 torque tune" — start Prius lateral tuning from that

Key enabler: sunnypilot supports a `CUSTOM_MODEL_PATH` (`supercombo-{name}.thneed`) — driving-model swap
is a supported feature. Driving model (`modeld`) and driver-monitoring model (`dmonitoringmodeld`) are
**separate**, so we can update the driver-facing model without touching driver monitoring.

## Issue tracks
1. **Curves bail ("curve too tight, take control")** — root cause: curve-slowdown was disabled.
   Fix: enable Vision Turn Control + tune `TARGET_LAT_A` and Prius torque limits.
2. **Driver-monitoring false alerts (sunglasses / setting sun)** — relax `_SG_THRESHOLD`,
   `_FACE_THRESHOLD`, `_POSESTD_THRESHOLD` in `selfdrive/monitoring/helpers.py`. Keep old (non-phone) DM.
3. **Silver / low-contrast lead cars detected late** — driving-model limitation; addressed by newer model
   + healthy radar fusion.
4. **Selective upgrade** — pull newer driving model, exclude newer DM.

## Workflow (low-driving)
- **Mac = workshop:** analyze history, prepare/commit/version changes here.
- **comma = test bench:** deploy, then **replay recorded drives** to validate offline (DM false-positives,
  lead detection, curve slowdown) — only steering *feel* needs a real drive.
- **Batch** changes; one test drive per batch; each change is its own versioned commit/tag.
