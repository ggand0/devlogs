# 052: Seg+Depth Real Robot Inference Implementation

## Overview

Implement real robot inference for DrQ-v2 policy trained with segmentation + depth observations instead of RGB images. This enables sim2real transfer using domain-invariant visual features.

## Model Weights

| Model | Checkpoint | Notes |
|-------|------------|-------|
| Segmentation | `pick-101/outputs/efficientvit_seg/best.ckpt` | EfficientViT-B0, ~0.85M params, Val IoU 0.811 |
| Depth | `checkpoints/depth_anything_v2_vits.pth` | Depth Anything V2 Small, ~25M params |
| Policy | `pick-101/runs/seg_depth_rl/20260119_192807/snapshots/1200000_snapshot.pt` | DrQ-v2, 70% sim success |

## Observation Format

**Per-frame**: (2, 84, 84) uint8
- Channel 0: Segmentation class IDs (0-5)
- Channel 1: Disparity (0-255, near=255, far=0)

**Frame stacked**: (6, 84, 84) from frame_stack=3

**Normalized for policy**: float32 [0, 1]

### Segmentation Classes

| ID | Class | Description |
|----|-------|-------------|
| 0 | unlabeled | Pixels with no annotation |
| 1 | background | Arm, sky, everything else |
| 2 | table | Ground/floor surface |
| 3 | cube | Target object |
| 4 | static_finger | Fixed gripper finger |
| 5 | moving_finger | Actuated gripper finger |

## Preprocessing Pipeline

```
Camera (640x480 BGR)
    ↓
┌───────────────────────────────────────┐
│  Segmentation Model (EfficientViT)    │
│  → (480, 640) class IDs 0-5           │
└───────────────────────────────────────┘
    ↓
┌───────────────────────────────────────┐
│  Depth Model (Depth Anything V2)      │
│  → (480, 640) disparity               │
│  → Per-frame normalize to 0-255       │
└───────────────────────────────────────┘
    ↓
Center Crop (480x480)
    ↓
Resize (84x84)
  - Seg: INTER_NEAREST (preserve class IDs)
  - Depth: INTER_LINEAR (smooth gradients)
    ↓
Stack Channels → (2, 84, 84)
    ↓
Frame Stack (3 frames) → (3, 2, 84, 84)
    ↓
Normalize → float32 / 255.0
    ↓
Policy Input
```

## Action Space

Policy outputs 4D Cartesian actions (same as RGB policy):
- delta X, Y, Z for end-effector position
- gripper open/close (-1 to 1)

**IK is required** to convert Cartesian actions to joint commands.

## Files Created

### 1. `src/deploy/perception.py`

New module with three main classes:

**SegmentationModel**
- Wraps `SegmentationInference` from pick-101
- `load()` - loads EfficientViT checkpoint
- `predict(bgr_image)` - returns (H, W) mask with class IDs 0-5

**DepthModel**
- Wraps Depth Anything V2 Small
- `load()` - loads checkpoint from `checkpoints/depth_anything_v2_vits.pth`
- `predict(bgr_image)` - returns (H, W) disparity uint8 (0-255)

**SegDepthPreprocessor**
- Combines camera + segmentation + depth pipeline
- `load_models()` - loads both perception models
- `open_camera()` - opens OpenCV capture
- `fill_buffer()` - initializes frame buffer
- `get_stacked_observation()` - returns (frame_stack, 2, 84, 84)

**MockSegDepthPreprocessor**
- Mock version for dry-run testing

### 2. `src/deploy/policy.py` (modified)

Added `SegDepthPolicyRunner` class:
- Mirrors `PolicyRunner` structure
- Creates agent with 2-channel observation space
- `load()` - auto-detects state_dim from checkpoint weights
- `get_action(seg_depth, low_dim_state)` - normalizes to [0,1] and returns action

Key implementation detail: Uses "rgb" as observation dict key for wrapper compatibility, even though input is 2-channel seg+depth.

### 3. `scripts/rl_inference_seg_depth.py`

Main inference script following `rl_inference.py` structure:

**Arguments**:
```
--checkpoint        Path to seg+depth DrQ-v2 checkpoint
--seg_checkpoint    Path to EfficientViT segmentation checkpoint
--camera_index      Camera device index (default: 0)
--robot_port        Serial port (default: /dev/ttyACM0)
--num_episodes      Number of episodes (default: 5)
--episode_length    Max steps per episode (default: 200)
--dry_run           Mock mode
--device            Torch device (default: cuda)
--action_scale      Meters per unit (default: 0.02)
--control_hz        Control frequency (default: 10.0)
--cube_x/cube_y     Expected cube position
--genesis_mode      Enable Genesis coordinate transform
--record_dir        Recording output directory
--debug_state       Print verbose debug info
--save_obs          Save observation images
```

**Control loop**:
1. Get seg+depth observation from preprocessor
2. Get proprioception state (joint pos/vel, EE pose)
3. Get action from policy
4. Apply Genesis coordinate transform if enabled
5. Convert Cartesian action to joint targets via IK
6. Send to robot

## Usage

```bash
# Dry run test
uv run python scripts/rl_inference_seg_depth.py \
    --checkpoint ~/ggando/ml/pick-101/runs/seg_depth_rl/20260119_192807/snapshots/1200000_snapshot.pt \
    --seg_checkpoint ~/ggando/ml/pick-101/outputs/efficientvit_seg/best.ckpt \
    --dry_run

# Real robot with Genesis mode
uv run python scripts/rl_inference_seg_depth.py \
    --checkpoint ~/ggando/ml/pick-101/runs/seg_depth_rl/20260119_192807/snapshots/1200000_snapshot.pt \
    --seg_checkpoint ~/ggando/ml/pick-101/outputs/efficientvit_seg/best.ckpt \
    --genesis_mode \
    --cube_x 0.25 --cube_y 0.0

# With debug and observation saving
uv run python scripts/rl_inference_seg_depth.py \
    --checkpoint ~/ggando/ml/pick-101/runs/seg_depth_rl/20260119_192807/snapshots/1200000_snapshot.pt \
    --seg_checkpoint ~/ggando/ml/pick-101/outputs/efficientvit_seg/best.ckpt \
    --genesis_mode --debug_state --save_obs
```

## Key Implementation Details

### Disparity Normalization

Training normalizes disparity per-frame:
```python
d_min, d_max = depth.min(), depth.max()
if d_max - d_min > 1e-6:
    disparity_norm = (depth - d_min) / (d_max - d_min)
else:
    disparity_norm = np.ones_like(depth)
disparity_uint8 = (disparity_norm * 255).astype(np.uint8)
```

### Observation Key Convention

DrQ-v2 agent expects observation key "rgb" even for seg+depth:
```python
obs = {
    "rgb": obs_tensor,  # Key is "rgb" to match training wrapper
    "low_dim_state": state_tensor,
}
```

### IK Reset Motion

Same as RGB script - moves to initial pose with wrist locked at π/2:
1. Reset to safe extended position
2. Set wrist joints to π/2 (top-down orientation)
3. Move to safe height above target
4. Lower to grasp height (Genesis mode)

### Genesis Mode Coordinate Transform

Genesis → MuJoCo frame (90° rotation):
```python
delta_xyz = np.array([
    -genesis_y,  # MuJoCo X (forward) = -Genesis Y
    genesis_x,   # MuJoCo Y (sideways) = Genesis X
    genesis_z,   # Z unchanged
])
```

Note: Joint offset correction (elbow_flex -12.5 deg) has been removed since new calibration is in place.

## Dependencies

- pick-101 repo (for SegmentationInference class and robobase)
- Depth Anything V2 (setup with `python scripts/depth_estimation.py --setup`)
- Existing so101-playground deploy modules (robot, IK controller)

## Dry Run Test Output

```
============================================================
SO-101 RL Inference (DrQ-v2 Seg+Depth)
============================================================

[1/4] Loading policy...
  Auto-detected state_dim=18 from checkpoint (total=54, frame_stack=3)
Loaded seg+depth checkpoint: .../1200000_snapshot.pt
  Frame stack: 3
  State dim: 18

[2/4] Initializing perception (segmentation + depth)...
  [DRY RUN] Using mock perception

[3/4] Initializing robot...
[MOCK] Connected to SO-101 at /dev/ttyACM0

[4/4] Initializing IK controller...
  IK controller ready (damping=0.1)

--- Episode 1/1 ---
  Step 1: Reset position (extended forward)...
  Step 2: Setting top-down wrist orientation...
  Step 3a: Moving to safe height above target...
    Safe target: [ 0.25  -0.015  0.08 ]
    Reached: [ 0.24938831 -0.01485335  0.08042311]
  Initial EE position: [ 0.21445947 -0.01241888  0.05282767]
  Step 0: delta=[ 0.99981874  0.7599953  -0.99999976] gripper=-0.93 ee_z=0.053
  Episode 1 complete

Safe return sequence...
Done.
```

## Real Robot Results

### Initial Observations

- Robot successfully executes IK reset motion to initial pose
- Policy outputs reasonable actions (moving toward perceived cube location)
- Gripper closes when near perceived target

### Segmentation Quality Issue

**Problem**: The segmentation model confuses shadows with the cube (class 3). When the robot arm casts a shadow on the table, the shadow region gets classified as "cube", causing the robot to chase the shadow instead of the actual cube.

**Behavior observed**:
- Robot moves toward shadow regions detected as cube
- As robot moves, shadow moves, creating a feedback loop
- Policy appears to work correctly given the (incorrect) segmentation input

**Potential solutions**:
1. **Post-processing**: Add morphological filtering or minimum area threshold
2. **Brightness filtering**: Reject dark "cube" regions (shadows are darker than orange cube)
3. **Retrain segmentation**: Add shadow augmentation to training data
4. **Improve lighting**: Reduce shadows with diffuse lighting

### Recording Options

Added `--save_preview_video` flag to record RGB|Seg|Depth visualization for debugging.

```bash
uv run python scripts/rl_inference_seg_depth.py \
    --checkpoint ... --seg_checkpoint ... \
    --genesis_mode --camera_index 1 \
    --save_preview_video preview.mp4
```

## Next Steps

1. ~~Test with real camera and perception models~~ Done
2. ~~Verify segmentation quality on real images~~ Issue found: shadow confusion
3. Fix segmentation shadow issue (retrain or post-process)
4. Run real robot evaluation and compare to sim success rate (70%)
