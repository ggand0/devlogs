# Devlog 030: State Normalization for Sim-to-Real Transfer

**Date:** 2026-01-19

## Context

Investigating how RoboBase/Genesis normalizes `low_dim_state` observations for sim-to-real transfer of the pretrained DrQ-v2 checkpoint.

**Checkpoint**: `runs/drqv2/20260116_002440/snapshots/best_snapshot.pt`

---

## Summary

**RoboBase DrQ-v2 uses IDENTITY normalization (no normalization).**

The `low_dim_state` observations are passed directly to the actor network without any scaling, centering, or normalization. The real robot must provide states in the **exact same units** as Genesis.

---

## 1. Normalization Scheme

| Question | Answer |
|----------|--------|
| **Normalization type** | **IDENTITY (none)** |
| **MIN_MAX normalization?** | No |
| **MEAN_STD normalization?** | No |
| **Running statistics?** | No |
| **Saved normalization params in checkpoint?** | No |

### Code Evidence

From `external/robobase/robobase/method/actor_critic.py:376-380`:

```python
def _act_extract_low_dim_state(self, observations: dict[str, torch.Tensor]):
    low_dim_obs = extract_from_spec(observations, "low_dim_state")
    if self.frame_stack_on_channel:
        low_dim_obs = flatten_time_dim_into_channel_dim(low_dim_obs)
    return low_dim_obs  # <-- No normalization, raw values passed through
```

The actor receives raw `low_dim_state` values directly.

---

## 2. Genesis State Units

**CRITICAL: Genesis outputs states in RADIANS and METERS, not degrees!**

### Measured from live env (curriculum_stage=3, no DR):

| Component | Index | Example Value | Unit |
|-----------|-------|---------------|------|
| Joint positions | 0:6 | `[0.15, 1.12, -1.37, 1.57, 1.57, 1.00]` | **RADIANS** |
| Joint velocities | 6:12 | `[0.002, 0.29, -0.42, -0.001, 0.0001, 0.00004]` | **RAD/S** |
| Gripper XYZ | 12:15 | `[0.206, -0.010, 0.121]` | **METERS** |
| Gripper euler | 15:18 | `[-0.258, -0.012, -0.199]` | **RADIANS** |

### Actual value ranges observed:

| State Component | Typical Range | Unit |
|-----------------|---------------|------|
| Joint positions | -pi to +pi (~-3.14 to +3.14) | radians |
| Joint velocities | -1 to +1 (can spike higher) | rad/s |
| EE position xyz | 0.0 to 0.5 | meters |
| EE euler angles | -pi to +pi | radians |

---

## 3. low_dim_state Structure

**Total dimensions: 18**

```
low_dim_state = [
    joint_pos[0],    # 0:  shoulder (rad)
    joint_pos[1],    # 1:  upper_arm (rad)
    joint_pos[2],    # 2:  lower_arm (rad)
    joint_pos[3],    # 3:  wrist (rad)
    joint_pos[4],    # 4:  gripper rotation (rad)
    joint_pos[5],    # 5:  gripper open/close (rad, 0=closed, 1=open)
    joint_vel[0],    # 6:  shoulder velocity (rad/s)
    joint_vel[1],    # 7:  upper_arm velocity (rad/s)
    joint_vel[2],    # 8:  lower_arm velocity (rad/s)
    joint_vel[3],    # 9:  wrist velocity (rad/s)
    joint_vel[4],    # 10: gripper rot velocity (rad/s)
    joint_vel[5],    # 11: gripper velocity (rad/s)
    gripper_x,       # 12: EE x position (m)
    gripper_y,       # 13: EE y position (m)
    gripper_z,       # 14: EE z position (m)
    gripper_roll,    # 15: EE roll (rad)
    gripper_pitch,   # 16: EE pitch (rad)
    gripper_yaw,     # 17: EE yaw (rad)
]
```

**Note: Cube position is NOT included** - it's privileged information only available in simulation.

---

## 4. Action for Real Robot

Since there's no normalization, the real robot MUST:

1. **Convert joint positions from degrees to RADIANS**:
   ```python
   joint_pos_rad = np.radians(joint_pos_deg)
   ```

2. **Convert joint velocities from deg/s to RAD/S**:
   ```python
   joint_vel_rads = np.radians(joint_vel_degs)
   ```

3. **Keep EE position in METERS** (already correct if using meters)

4. **Keep EE euler angles in RADIANS** (convert if using degrees)

---

## 5. Recommended dataset_stats for HIL-SERL

Since RoboBase uses no normalization, your HIL-SERL `dataset_stats` should reflect the actual Genesis ranges in radians/meters:

```json
{
  "observation.state": {
    "min": [
      -3.14, -3.14, -3.14, -3.14, -3.14, 0.0,
      -5.0, -5.0, -5.0, -5.0, -5.0, -5.0,
      0.0, -0.3, 0.0,
      -3.14, -3.14, -3.14
    ],
    "max": [
      3.14, 3.14, 3.14, 3.14, 3.14, 1.0,
      5.0, 5.0, 5.0, 5.0, 5.0, 5.0,
      0.5, 0.3, 0.4,
      3.14, 3.14, 3.14
    ]
  }
}
```

**WARNING: Using degree-based stats `[-180, 180]` is WRONG** - the actor was trained on radian values.

---

## 6. Critical Conversion for Sim-to-Real

If your real robot outputs degrees, convert BEFORE feeding to the actor:

```python
def prepare_state_for_actor(real_state):
    """Convert real robot state to Genesis/RoboBase format."""
    return np.array([
        # Joint positions: degrees -> radians
        np.radians(real_state['joint_pos'][0]),
        np.radians(real_state['joint_pos'][1]),
        np.radians(real_state['joint_pos'][2]),
        np.radians(real_state['joint_pos'][3]),
        np.radians(real_state['joint_pos'][4]),
        np.radians(real_state['joint_pos'][5]),
        # Joint velocities: deg/s -> rad/s
        np.radians(real_state['joint_vel'][0]),
        np.radians(real_state['joint_vel'][1]),
        np.radians(real_state['joint_vel'][2]),
        np.radians(real_state['joint_vel'][3]),
        np.radians(real_state['joint_vel'][4]),
        np.radians(real_state['joint_vel'][5]),
        # EE position: already in meters
        real_state['ee_pos'][0],
        real_state['ee_pos'][1],
        real_state['ee_pos'][2],
        # EE euler: radians
        real_state['ee_euler'][0],
        real_state['ee_euler'][1],
        real_state['ee_euler'][2],
    ], dtype=np.float32)
```

---

## 7. Summary Table

| What | Genesis Training | Real Robot (Required) |
|------|------------------|----------------------|
| Joint positions | RADIANS | Must convert to RADIANS |
| Joint velocities | RAD/S | Must convert to RAD/S |
| EE position | METERS | METERS |
| EE euler angles | RADIANS | RADIANS |
| Normalization | None (identity) | None (identity) |

---

## Files Checked

- `external/robobase/robobase/method/drqv2.py` - DrQ-v2 actor implementation
- `external/robobase/robobase/method/actor_critic.py` - Base class, low_dim extraction
- `external/robobase/robobase/method/core.py` - Method base class
- `src/envs/lift_cube_env.py` - Genesis env, `_get_obs()`, `get_image_obs()`

---

## TLDR

The pretrained actor expects raw values in **RADIANS** and **METERS**. No normalization is applied. If your real robot outputs degrees, you **MUST** convert to radians before inference.
