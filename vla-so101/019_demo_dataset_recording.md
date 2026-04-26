# 019 — Demo Dataset Recording

**Date**: 2026-04-26

Recorded a demo dataset for showcasing the lerobot dataset viewer app.

## Dataset

| Field | Value |
|-------|-------|
| HF repo | `gtgando/so101_pick_place_demo` |
| Local path | `~/.cache/huggingface/lerobot/gtgando/so101_pick_place_demo` |
| Config | `configs/record_demo.yaml` |
| Episodes | 37 |
| Frames | 11,100 |
| FPS | 30 |
| Episode length | 10s (~300 frames) |
| Task | "Pick up the cube and place it in the bowl" |
| Format | lerobot v2.1, parquet + AV1 video |

### Cameras

| Camera | Hardware | Identifier | Resolution |
|--------|----------|------------|------------|
| `observation.images.wrist` | Intel RealSense D405 | serial `335122272499` (pyrealsense2) | 640×480 @ 30fps |
| `observation.images.overhead` | Logitech C920 | `/dev/v4l/by-id/usb-046d_HD_Pro_Webcam_C920-video-index0` | 640×480 @ 30fps |

### Recording sessions

Originally targeted 20 episodes but ended up with 37 across multiple sessions due to resume behavior:

- **Session 1**: Recorded ~16 episodes before `ConnectionError` crash (servo bus drop).
- **Session 2**: Resumed with `--resume`, recorded another 20 episodes (the script re-reads `num_episodes` from config and records that many more, rather than recording up to the total). Crashed at episode 36.
- Total: 37 episodes saved.

### Resume bug

When using `--resume`, the script records `num_episodes` *additional* episodes on top of whatever already exists, rather than recording *up to* `num_episodes` total. This is because `recorded_episodes` starts at 0 on resume instead of accounting for already-recorded episodes. Mildly annoying but not blocking — just means you get more data than expected.

## Servo Bus Communication Errors

Recurring `ConnectionError` crashes during recording, happening every 4–7 episodes initially, then stable for a longer stretch (36 episodes) in the final session.

### Error

```
ConnectionError: Failed to sync read 'Present_Position' on ids=[1, 2, 3, 4, 5, 6] after 1 tries.
[TxRxResult] There is no status packet!
```

All 6 servo IDs fail simultaneously — the entire bus drops, not a single servo.

### Likely hardware causes (in order of probability)

1. **USB cable to servo controller board** — arm movement during teleop and reset wiggles the connection. Most likely cause since the failure is intermittent and correlated with physical movement.
2. **First JST connector in daisy chain** — cable between controller board and servo ID 1. If this disconnects, all downstream servos go silent.
3. **Power sag** — during high-torque moves (especially the snap-to-safe-joints reset), PSU may not deliver enough current, causing servos to brown out.
4. **Controller board solder joints** — cold joint on serial or power pins.

### Diagnostic

After a crash, run `uv run lerobot-find-port`:
- If `/dev/ttyACM0` disappeared → USB connection issue
- If port exists but servos don't respond → daisy chain or power issue

### Software mitigation

The `sync_read` call uses `num_retry=1` by default. Increasing retry count in the record loop could help survive transient dropouts. The error triggers `terminate called without an active exception` which kills the process — a retry or graceful recovery would prevent data loss.
