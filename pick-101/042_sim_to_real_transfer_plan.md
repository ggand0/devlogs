# Devlog 042: Sim-to-Real Transfer Plan

## Phase 1: Hardware Setup

### Camera Calibration
| Parameter | Sim | Real (innoMaker) | Action |
|-----------|-----|------------------|--------|
| FOV | 75° | 130° diagonal | Crop real to match sim |
| Resolution | 84×84 | 1080p | Resize + center crop |
| Mount | Front of gripper | Same position | Verify alignment |
| Frame rate | 50 Hz | 30 fps | Account for latency |

### Robot Calibration
- [ ] Joint encoder zero positions
- [ ] Gripper servo range mapping (sim [-1,1] → real PWM)
- [ ] IK solver validation on real hardware
- [ ] Control frequency test (target: 10-20 Hz)

## Phase 2: Software Integration

### Inference Script Requirements
```python
# Key components needed:
1. Camera capture + preprocessing (84×84, CHW, uint8)
2. Frame stacking buffer (3 frames)
3. Low-dim state extraction (joint pos/vel, gripper state)
4. Policy inference (load checkpoint, eval mode)
5. Action scaling + robot command interface
6. Safety limits (workspace bounds, velocity caps)
```

### Observation Pipeline
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

## Phase 3: Zero-Shot Transfer Test

### Test Protocol
1. **Reaching only** (no grasp): Verify camera-to-action mapping
2. **Slow motion** (0.25× speed): Safe initial test
3. **Single grasp attempt**: Full pipeline test
4. **10 episode evaluation**: Measure transfer success rate

### Expected Outcomes
| Scenario | Likelihood | Action |
|----------|------------|--------|
| Works zero-shot | 20% | Celebrate, fine-tune for robustness |
| Reaches but misses | 50% | Camera calibration issue |
| Random behavior | 30% | Visual domain gap, need DR or fine-tuning |

## Phase 4: Fine-tuning (if needed)

### Option A: Domain Randomization (retrain in sim)
- Randomize: lighting, textures, cube color, camera noise
- Retrain with augmented sim for better transfer

### Option B: Real-world Fine-tuning
- Collect 10-50 real demonstrations
- Behavior cloning warm-start
- Few-shot RL fine-tuning (TRANSIC approach)

### Option C: Residual Policy
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

## Success Criteria

| Stage | Metric | Target |
|-------|--------|--------|
| Sim training | Success rate | >50% |
| Zero-shot transfer | Any successful grasp | 1/10 |
| After fine-tuning | Success rate | >30% |
| Production ready | Success rate | >70% |

## Next Steps

1. **Complete sim training** - achieve >30% success rate
2. **Create inference script**: `scripts/real_robot_inference.py`
3. **Camera calibration**: Match sim/real FOV
4. **Hardware integration test**: Verify joint control
