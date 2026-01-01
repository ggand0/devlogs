# Devlog 041: v13 Reward Breakthrough & Sim-to-Real Transfer Plan

## Breakthrough Results

**First successful image-based grasping at 800k steps!**

| Metric | Before (v11) | After (v13) |
|--------|--------------|-------------|
| Reward | ~320 (plateau) | 650+ |
| Success Rate | 0% (2M steps) | **10%** (800k steps) |
| Behavior | Single-finger tilt exploit | Proper two-finger grasp |

### Training Progress (v13)
```
800k steps | 3:08:40 elapsed
ep_rew_mean: 650.17
success_rate: 0.10 (10%)
New best model saved!
```

### What Fixed It
v13 gates the binary lift bonus on `is_grasping`:
```python
# v11 (exploitable):
if cube_z > 0.02:
    reward += 1.0  # Tilt exploit gets this

# v13 (fixed):
if is_grasping:
    if cube_z > 0.02:
        reward += 1.0  # Only with proper grasp
```

Reward gap: Exploit ~0.9/step vs Grasp ~2.33/step (was 1.9 vs 2.33 in v11)

## Current Training Status

- **Run dir**: `runs/image_rl/` (latest with v13)
- **Config**: `configs/drqv2_lift_s3.yaml`
- **Target**: 2M steps
- **Current**: 800k steps (40%)
- **ETA**: ~4.5 more hours

## Sim-to-Real Transfer Plan

### Phase 1: Hardware Setup

#### Camera Calibration
| Parameter | Sim | Real (innoMaker) | Action |
|-----------|-----|------------------|--------|
| FOV | 75° | 130° diagonal | Crop real to match sim |
| Resolution | 84×84 | 1080p | Resize + center crop |
| Mount | Front of gripper | Same position | Verify alignment |
| Frame rate | 50 Hz | 30 fps | Account for latency |

#### Robot Calibration
- [ ] Joint encoder zero positions
- [ ] Gripper servo range mapping (sim [-1,1] → real PWM)
- [ ] IK solver validation on real hardware
- [ ] Control frequency test (target: 10-20 Hz)

### Phase 2: Software Integration

#### Inference Script Requirements
```python
# Key components needed:
1. Camera capture + preprocessing (84×84, CHW, uint8)
2. Frame stacking buffer (3 frames)
3. Low-dim state extraction (joint pos/vel, gripper state)
4. Policy inference (load checkpoint, eval mode)
5. Action scaling + robot command interface
6. Safety limits (workspace bounds, velocity caps)
```

#### Observation Pipeline
```
Real camera (1080p, 130° FOV)
    ↓
Center crop to match 75° FOV
    ↓
Resize to 84×84
    ↓
RGB → CHW format
    ↓
Frame stack (3 frames)
    ↓
Combine with low_dim_state
    ↓
Policy inference
```

### Phase 3: Zero-Shot Transfer Test

#### Test Protocol
1. **Reaching only** (no grasp): Verify camera-to-action mapping
2. **Slow motion** (0.25× speed): Safe initial test
3. **Single grasp attempt**: Full pipeline test
4. **10 episode evaluation**: Measure transfer success rate

#### Expected Outcomes
| Scenario | Likelihood | Action |
|----------|------------|--------|
| Works zero-shot | 20% | Celebrate, fine-tune for robustness |
| Reaches but misses | 50% | Camera calibration issue |
| Random behavior | 30% | Visual domain gap, need DR or fine-tuning |

### Phase 4: Fine-tuning (if needed)

#### Option A: Domain Randomization (retrain in sim)
- Randomize: lighting, textures, cube color, camera noise
- Retrain with augmented sim for better transfer

#### Option B: Real-world Fine-tuning
- Collect 10-50 real demonstrations
- Behavior cloning warm-start
- Few-shot RL fine-tuning (TRANSIC approach)

#### Option C: Residual Policy
- Keep sim policy frozen
- Train small residual network on real data
- Output: sim_action + residual_correction

## Hardware Checklist

### Required
- [x] SO-101 robot assembled
- [x] innoMaker USB camera (130° FOV)
- [ ] Camera mount at gripper (match sim position)
- [ ] USB connection to control PC
- [ ] Workspace with cube and table

### Safety
- [ ] Emergency stop accessible
- [ ] Soft/foam cube for initial tests
- [ ] Workspace boundaries defined
- [ ] Joint limit software checks

### Software
- [ ] Camera driver (UVC standard, should work)
- [ ] Robot control interface (LeRobot / custom)
- [ ] Real-time inference script
- [ ] Logging/recording for debugging

## Files

- Training run: `runs/image_rl/` (v13, ongoing)
- Best checkpoint: `snapshots/best_snapshot.pt` (650.17 reward)
- Config: `configs/drqv2_lift_s3.yaml`

## Next Steps

1. **Continue training to 2M** - monitor for higher success rate
2. **Evaluate at 1M, 1.5M, 2M** - track improvement curve
3. **If >30% success**: Start hardware integration
4. **Create inference script**: `scripts/real_robot_inference.py`
5. **Camera calibration**: Match sim/real FOV

## Success Criteria

| Stage | Metric | Target |
|-------|--------|--------|
| Sim training | Success rate | >50% |
| Zero-shot transfer | Any successful grasp | 1/10 |
| After fine-tuning | Success rate | >30% |
| Production ready | Success rate | >70% |
