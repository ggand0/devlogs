# Devlog 032: URDF & Sim2Real Research

## Summary

Researched SO-100/SO-101 URDF issues and sim2real gaps after discovering the lerobot-sim2real repo claiming zero-shot sim2real with ManiSkill. Key finding: the sim2real gap is primarily visual/dynamics, not kinematic.

## Resources Investigated

- https://github.com/StoneT2000/lerobot-sim2real - Zero-shot RGB sim2real with SO-100
- https://github.com/StoneT2000/lerobot-sim2real/issues/9 - SO-101 sim2real failures
- https://github.com/TheRobotStudio/SO-ARM100/pull/87 - URDF fixes
- https://github.com/TheRobotStudio/SO-ARM100/issues/54 - URDF issues
- https://github.com/haosulab/ManiSkill/pull/1171 - SO-101 ManiSkill fixes

## URDF Analysis

### Our URDF vs ManiSkill's SO-100

| Joint | Our/Official Axis | ManiSkill Axis |
|-------|-------------------|----------------|
| shoulder_pan | `0 0 1` | `0 -1 0` |
| shoulder_lift | `0 0 1` | `1 0 0` |
| elbow_flex | `0 0 1` | `1 0 0` |
| wrist_flex | `0 0 1` | `1 0 0` |
| wrist_roll | `0 0 1` | `0 1 0` |
| gripper | `0 0 1` | `0 0 1` |

**Key insight**: Our URDF uses local Z-axis rotation for all joints (axis `0 0 1`) with origin RPY transforms to achieve correct orientation. ManiSkill uses world-frame aligned axes. Both approaches are mathematically equivalent if transforms are correct.

### Known URDF Issues (PR #87)

1. **Joint types**: Should be `revolute` not `continuous` - **OK in ours**
2. **Joint limits**: Must be defined - **OK in ours**
3. **Elbow 5° offset**: Motor zero is 180° from middle position - **Noted in our URDF comment**
4. **Tiny floating-point values**: e.g., `1.73763e-16` should be `0` - Present in ours

### SO-100 vs SO-101 Differences (from lerobot fork)

| Aspect | SO-100 | SO-101 |
|--------|--------|--------|
| wrist_roll calibration | Full turn (0-4095) | Standard range |
| Everything else | Same | Same |

## lerobot-sim2real System Identification

The `system_id_so100.npy` contains 142 samples of:
- `qpos`: Actual robot joint positions
- `target_qpos`: Commanded joint positions

This captures servo tracking error (lag, overshoot) for sim2real compensation. Example from their data:
- Commands ramp from 0 to ~2.7 rad on shoulder_lift
- Actual positions lag behind and show overshoot at direction changes

## Root Cause Analysis

From devlog 014, sim2real failed due to:

1. **Visual domain gap** (primary) - MuJoCo renders vs real USB camera
2. **Servo dynamics** - Response lag, overshoot (captured in system_id)
3. **NOT the URDF** - Kinematics should be correct if calibration is good

## Verification Plan

### 1. Kinematic Verification
Command same joint angles in MuJoCo and real robot, compare EE positions:
- Send joints to known configuration
- Read MuJoCo EE position
- Measure real robot EE position (manually or with marker)
- Compare

### 2. System ID Collection
Record (target_qpos, actual_qpos) pairs like lerobot-sim2real:
- Send sinusoidal or ramp commands to each joint
- Record commanded vs achieved positions
- Analyze tracking error characteristics

### 3. Visual Alignment
Overlay sim render on real camera:
- Match camera intrinsics (FOV, resolution)
- Match camera extrinsics (position, orientation)
- Verify robot pose alignment

## Domain Randomization Priority

After verification, prioritize visual randomization since that's the confirmed gap:

1. **Lighting**: Intensity, direction, color temperature
2. **Textures**: Table, background, robot surface
3. **Camera**: Small FOV variation, noise
4. **Already have**: Random shift augmentation in DrQ-v2

## Files Referenced

- `models/so101_new_calib.urdf` - Our URDF (matches official SO-ARM100 repo)
- `~/.cache/huggingface/lerobot/calibration/robots/so101_follower/ggando_so101_follower.json` - Calibration
- `/home/gota/ggando/ml/lerobot-sim2real/system_id_so100.npy` - Their system ID data

## Verification Scripts Created

### 1. Kinematic Verification (`scripts/verify_kinematics.py`)

Moves robot to several test configurations and compares MuJoCo FK vs actual positions.

```bash
cd /home/gota/ggando/ml/lerobot
uv run python /home/gota/ggando/ml/so101-playground/scripts/verify_kinematics.py
```

Test configurations:
- Home (all zeros)
- Top-down ready (wrist at 90°, -90°)
- Bent elbow
- Side reach
- Low position

Outputs: `kinematic_verification_results.json`

### 2. System ID Collection (`scripts/collect_system_id.py`)

Records target_qpos vs actual_qpos pairs using sinusoidal and ramp trajectories.

```bash
cd /home/gota/ggando/ml/lerobot
uv run python /home/gota/ggando/ml/so101-playground/scripts/collect_system_id.py
```

Outputs:
- `system_id_so101.npy` - NumPy dict with `target_qpos` and `qpos` arrays
- `system_id_so101.json` - Human-readable error statistics

### 3. Visual Alignment Check (`scripts/verify_visual_alignment.py`)

Real-time overlay of MuJoCo render on real camera feed.

```bash
cd /home/gota/ggando/ml/lerobot
uv run python /home/gota/ggando/ml/so101-playground/scripts/verify_visual_alignment.py
```

Controls:
- `q` - Quit
- `s` - Save frame
- `r` - Reset to home
- `+/-` - Adjust overlay opacity

## Kinematic Verification Results

### Calibration File

Best calibration (after elbow_flex recalibration):
```
/home/gota/.cache/huggingface/lerobot/calibration/robots/so101_follower/ggando_so101_follower_bak_110926_3.json
```

### Error Measurements

Test pose: gripper on table, arm bent to side.
Reference point: `gripperframe` site, measured from front base marker (8cm forward from origin).

| Axis | Real | MuJoCo | Error |
|------|------|--------|-------|
| X (forward) | 15cm | 10.2cm | 4.8cm |
| Y (left) | 0cm | 0.2cm | 0.2cm |
| Z (up) | 0cm | -1.3cm | 1.3cm |

### Reference Points

- **Origin**: Center of shoulder_pan axis (inside robot base)
- **Front base marker**: Red sphere at X=8cm, Y=0, Z=1.5cm (on table, front of base)
- **gripperframe**: MuJoCo site on gripper body

### Elbow Flex Offset Correction

The raw calibration showed significant X and Z errors due to elbow_flex mismatch. Testing offsets:

| Offset | X (real=15cm) | Z (real=0cm) |
|--------|---------------|--------------|
| 0° | 10.2cm | -1.3cm |
| -10° | 14.1cm | 0.0cm |
| -12.5° | 15.1cm | 0.5cm |
| -15° | 16.0cm | 0.9cm |

**Best offset: -12.5° on elbow_flex (joint index 2)**

With -12.5° offset applied in simulation:

| Axis | Real | MuJoCo | Error |
|------|------|--------|-------|
| X | 15cm | 15.1cm | 0.1cm |
| Y | 0cm | 0.2cm | 0.2cm |
| Z | 0cm | 0.5cm | 0.5cm |

![Corrected pose](032_corrected_pose_mujoco.png)

### Conclusion

- With -12.5° elbow_flex offset, all errors under 0.5cm
- Y axis excellent (~0.2cm error)
- X and Z now accurate with offset correction
- Apply this offset in simulation code when reading joint positions

## Real Robot Inference Test

### Setup

Ran `rl_inference.py` with 800k checkpoint:
```bash
cd /home/gota/ggando/ml/lerobot && uv run python /home/gota/ggando/ml/so101-playground/scripts/rl_inference.py \
    --checkpoint /home/gota/ggando/ml/pick-101/runs/image_rl/20260106_221053/snapshots/800000_snapshot.pt
```

### Result

**Failed** - Robot did not grab the cube. Behavior observed:
- Reaching forward continuously
- Flapping gripper open/closed
- Did not converge on cube position

### Analysis

Despite kinematic corrections (elbow_flex -12.5° offset), significant sim2real gap remains. Likely causes:

1. **Visual domain gap** - MuJoCo renders vs real USB camera (lighting, textures, colors)
2. **Camera calibration** - Wrist camera FOV/position may differ from simulation
3. **Policy overfitting** - Trained on sim visuals, not transferring to real
4. **Observation mismatch** - low_dim_state may have different distributions

### Next Steps

1. Implement visual domain randomization in training
2. Fine-tune policy with real robot data (HIL-SERL)
3. Check camera intrinsics match between sim and real
4. Consider image augmentation during inference

## Next Steps

1. ~~Run kinematic verification to check URDF correctness~~ Done
2. Collect system ID data to characterize servo dynamics
3. Use visual alignment to tune camera parameters
4. After verification, implement domain randomization in MuJoCo
