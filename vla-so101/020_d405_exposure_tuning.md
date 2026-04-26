# 020 — D405 RGB Exposure Tuning

**Date**: 2026-04-26

## Problem

The D405 RGB feed looks slightly darker and cooler than a typical USB webcam (e.g. Logitech C920). This is because the D405 has no dedicated RGB sensor — RGB is derived from the depth sensor's global shutter, which has lower quantum efficiency and an ISP tuned for depth, not color.

## Root Cause

- D405 auto-exposure is tied to the depth sensor, not a separate RGB pipeline
- Default exposure: 33000µs, white balance: 4600 (cool)
- Global shutter captures less light than rolling shutter per exposure
- Residual near-IR from the stereo projector shifts color balance cooler

## Key Finding: D405 Settings Persist Until Power Cycle

Settings written via `rs.option.*` persist on the device even after the pipeline stops and the script exits. Running the preview with manual exposure/gain and then running it again without overrides does NOT reset — the device keeps the old values. A USB unplug/replug (power cycle) is required to return to factory defaults.

This is critical for data collection: if a preview/test script sets exposure values, they will silently carry over into subsequent recording sessions unless explicitly reset.

## What Was Done

Updated `scripts/camera_preview.py` with RealSense exposure controls:

- **Default mode** (`--realsense`): no sensor overrides, runs stock D405 auto-exposure
- **Tuned mode** (`--realsense --tuned`): applies manual exposure, gain, and white balance via CLI flags:
  - `--exposure` (default 8000µs, stock is 33000)
  - `--gain` (default 16/stock, range 16-248)
  - `--white-balance` (default 4600/stock)
- **Auto-restore on exit**: when `--tuned` is used, the script resets auto-exposure, gain, and white balance back to factory defaults before stopping the pipeline, so no power cycle is needed afterward

### Tested Values

| Exposure (µs) | Gain | Result |
|----------------|------|--------|
| auto (stock) | 16 | Overexposes in indoor lighting — "on the sun" |
| 33000 (manual) | 32 | Way too bright |
| 8000 | 16 | Not yet tested (default for --tuned) |
| 5000 | 16 | Looks normal brightness, warm color temperature |

## TODO

- Find the right exposure/gain/wb combo for the recording environment
- Consider whether recording should use tuned settings or stock auto-exposure
- If tuned settings are used for recording, the `record.py` script (via LeRobot's RealSenseCamera) would need patching to apply them after pipeline start — LeRobot's config doesn't expose exposure/gain/wb

## References

- [Intel Tuning Guide](https://dev.intelrealsense.com/docs/tuning-depth-cameras-for-best-performance)
- D405 depth sensor is at index 0 (`query_sensors()[0]`), provides both depth and RGB
- Gain range: 16–248 (Intel recommends keeping at 16, increase exposure first)
- White balance above 4600 = warmer, below = cooler/blue
