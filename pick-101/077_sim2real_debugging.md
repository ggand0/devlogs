# 077: Sim2Real Debugging Checklist

## Problem

Policy trained in sim (70% success rate at 1.2M steps) doesn't work on real robot. Arm drifts forward with gripper opening, then moves right while opening/closing. Seg+depth inference visually looks correct.

## Observation Pipeline Checklist

### 1. Normalization
```python
# MUST normalize to [0, 1]
obs = obs.astype(np.float32) / 255.0
```

### 2. Channel Order
| Channel | Content |
|---------|---------|
| 0 | Segmentation mask (0-5) |
| 1 | Disparity (0-255) |

### 3. Frame Stacking
- Stack 3 frames → (6, 84, 84)
- Initialize buffer with 3 copies of first frame
- Concatenate along channel axis

```python
frame_buffer = [first_obs] * 3
stacked = np.concatenate(frame_buffer, axis=0)  # (6, 84, 84)
```

### 4. Disparity Normalization
- Per-frame min-max normalization to 0-255
- Near = 255, Far = 0 (Depth Anything V2 format)

```python
disp = (disp - disp.min()) / (disp.max() - disp.min() + 1e-6)
disp = (disp * 255).astype(np.uint8)
```

### 5. Center Crop
- Real camera: 640x480 → center crop to 480x480
- Then resize to 84x84

```python
crop_x = (640 - 480) // 2  # 80
cropped = frame[:, crop_x:crop_x+480]
resized = cv2.resize(cropped, (84, 84))
```

## Action Space

| Index | Joint | Sim Range |
|-------|-------|-----------|
| 0 | shoulder_pan | [-1, 1] |
| 1 | shoulder_lift | [-1, 1] |
| 2 | elbow | [-1, 1] |
| 3 | wrist_flex | [-1, 1] |
| 4 | wrist_roll | [-1, 1] |
| 5 | gripper | [-1, 1] |

- Actions are **delta positions**
- Training used `action_repeat=2`

## Common Failure Modes

| Symptom | Likely Cause |
|---------|--------------|
| Arm drifts in one direction | Wrong obs normalization or channel order |
| Random jerky movements | Frame stacking not working |
| No gripper response | Action index mismatch |
| Overshooting | Action scaling wrong for real robot |

## Debug Steps

1. Print obs stats before policy inference:
   ```python
   print(f"obs shape: {obs.shape}, min: {obs.min()}, max: {obs.max()}")
   ```

2. Verify channel 0 has discrete values (0-5 after /255 → 0, 0.004, 0.008, ...)

3. Verify channel 1 has continuous disparity values

4. Check action output range matches expected [-1, 1]

## Training Reference

- Checkpoint: `runs/seg_depth_rl/20260119_192807/snapshots/1200000_snapshot.pt`
- Config: `configs/image_based/drqv2_lift_seg_depth_v19.yaml`
- Success rate: 70% (7/10 episodes in sim)
