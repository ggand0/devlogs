# 003 — Datasets

**Date**: 2026-02-15

Two datasets recorded for the "pick up cube, place in bowl" task using teleoperation with joint-space reset between episodes.

---

## Dataset 1: `so101_pick_place_smolvla_v3` (30cm workspace)

| Field | Value |
|-------|-------|
| HF repo | `gtgando/so101_pick_place_smolvla_v3` |
| Local path | `~/.cache/huggingface/lerobot/gtgando/so101_pick_place_smolvla_v3` |
| Config | `configs/record_30cm.yaml` |
| Episodes | 50 |
| Frames | 14,839 |
| FPS | 30 |
| Episode length | 10s (~300 frames) |
| Task | "Pick up the cube and place it in the bowl" |

### Cameras

| Camera | Hardware | Device | Resolution |
|--------|----------|--------|------------|
| `observation.images.wrist` | InnoMaker USB RGB | `/dev/video0` | 640x480 @ 30fps |
| `observation.images.overhead` | Generic USB webcam | `/dev/video2` | 640x480 @ 30fps |

### Recording conditions

- Cube placed randomly within ~30x30cm area in front of the arm
- Bowl positioned to the right
- Video codec: AV1 (yuv420p)

### Training result

- Trained SmolVLA for 20,000 steps (bs=64, ~4.75h on RTX 3090)
- Loss: 0.162 -> 0.005
- Checkpoint: `outputs/train/smolvla_so101_pick_place/checkpoints/last/pretrained_model`
- Inference: robot moves toward cube but fails to grasp — likely too sparse (50 demos across 30cm workspace)

---

## Dataset 2: `so101_pick_place_10cm_v1` (10cm workspace)

| Field | Value |
|-------|-------|
| HF repo | `gtgando/so101_pick_place_10cm_v1` |
| Local path | `~/.cache/huggingface/lerobot/gtgando/so101_pick_place_10cm_v1` |
| Config | `configs/record_10cm.yaml` |
| Episodes | 75 (recorded 80, removed 5 failed grasps) |
| Frames | 22,458 |
| FPS | 30 |
| Episode length | 10s (~299 frames) |
| Task | "Pick up the cube and place it in the bowl" |

### Cameras

| Camera | Hardware | Device / Serial | Resolution |
|--------|----------|-----------------|------------|
| `observation.images.wrist` | Intel RealSense D405 | serial `335122272499` (pyrealsense2 SDK) | 640x480 @ 30fps |
| `observation.images.overhead` | Generic USB webcam | `/dev/video2` | 640x480 @ 30fps |

### Recording conditions

- Cube placed within tighter ~10x10cm area for better grasp density
- Bowl positioned to the right (same as v3)
- Wrist cam swapped from InnoMaker (was dying / disconnecting) to D405 RealSense
- D405 connected via pyrealsense2 SDK (V4L2 limited to 424x240)
- Video codec: AV1 (yuv420p)
- Two recording sessions:
  - Session 1: 50 episodes (`configs/record_10cm.yaml`)
  - Session 2: 30 episodes with varied cube rotation (`configs/record_10cm_extra.yaml`, `--resume`)

### Removed episodes

- Session 1: episodes 11, 26 (failed grasps)
- Session 2: episodes 49, 74, 76 (failed grasps)
- All removed using `scripts/remove_episodes.py`, remaining renumbered 0-74
- Backup of session 1 at `~/.cache/huggingface/lerobot/gtgando/so101_pick_place_10cm_v1.bak`

### Training result

Not yet trained.

---

## Common settings

Both datasets share:

- **Robot**: SO-101 follower (Feetech STS3215 servos), `use_degrees=True`
- **Teleop**: SO-101 leader arm
- **Action space**: 6-DOF joint positions in degrees (shoulder_pan, shoulder_lift, elbow_flex, wrist_flex, wrist_roll, gripper)
- **Observation**: joint state (6-DOF) + 2 camera views
- **Home position**: `[0.0, -104.66, 96.09, 48.92, 90.0]` degrees, gripper=5.0
- **Reset**: 2-step interpolated — snap to safe joints (all zeros) then interpolate to home
- **Format**: lerobot v2.1, parquet + AV1 video
