# Devlog 016: Spawn Angle Experiments and Configurable Parameters

**Date:** December 8, 2025
**Focus:** Testing narrower spawn angles and making environment parameters configurable from Python

---

## Overview

Following the 3x sprint experiments (devlog 015), this session focused on:
1. Making `spawn_angle_degrees` and `sprint_multiplier` configurable from Python training scripts
2. Testing a narrower spawn angle (±30° vs ±60°) with 2x sprint
3. Fixing eval script to use correct training configuration

---

## Key Implementation: Configurable Parameters

### Problem
Previously, sprint multiplier and spawn angle were hardcoded in Rust. To run parameter sweep experiments, we needed to recompile for each variation.

### Solution
Extended the `/configure` API endpoint to accept optional parameters:

**Rust Side:**
```rust
// src/rl/api.rs
struct ConfigureRequest {
    level: Option<u8>,
    action_space_type: Option<String>,
    sprint_multiplier: Option<f32>,      // NEW
    spawn_angle_degrees: Option<f32>,    // NEW
}
```

**Python Side:**
```python
# environment.py
def configure(
    self,
    level: Optional[int] = None,
    action_space_type: Optional[str] = None,
    sprint_multiplier: Optional[float] = None,
    spawn_angle_degrees: Optional[float] = None,
) -> None:
```

Parameters can now be set in YAML configs:
```yaml
# ppo_level2_basic3d_narrow.yaml
sprint_multiplier: 1.0       # 2x speed
spawn_angle_degrees: 30      # ±30° = 60° total fan
```

---

## Experiment: 2x Sprint + ±30° Spawn Angle

### Hypothesis
The core difficulty in Level 2 isn't just projectile speed—it's spawn clustering. When consecutive projectiles spawn from similar angles, they create unavoidable "walls". A narrower spawn fan (±30° vs ±60°) should reduce these unfair RNG deaths.

### Configuration
| Parameter | Default (015) | Narrow (016) |
|-----------|---------------|--------------|
| Sprint multiplier | 2.0 (3x) | 1.0 (2x) |
| Spawn angle | ±60° (120° total) | ±30° (60° total) |
| Speed advantage | 3.3x | 2.2x |

### Training Results (500K steps)

**Best Checkpoint (Step 220K):**
| Metric | Value |
|--------|-------|
| Eval mean reward | 130.59 ± 132.16 |
| Eval mean ep length | 227.3 ± 131.91 |

**Final Model (Step 500K):**
| Metric | Value |
|--------|-------|
| Eval mean reward | 72.30 ± 81.22 |
| Eval mean ep length | 170.5 ± 80.74 |

---

## Critical Bug Fix: Eval Script Configuration

### Issue Discovered
Initial evaluation showed poor results (95.41 mean reward). The agent's movement looked off—it wasn't reacting to projectiles properly.

**Root Cause:** The eval script was using default Level 2 parameters (±60° angle) instead of the training configuration (±30° angle).

### Fix Applied
Added CLI arguments to `eval_ppo.py`:
```bash
python eval_ppo.py model.zip --sprint-multiplier 1.0 --spawn-angle 30
```

### Before/After Fix
| Eval Config | Mean Reward | Mean Steps |
|-------------|-------------|------------|
| Wrong (±60°) | 95.41 ± 148 | 192.8 |
| Correct (±30°) | **130.59 ± 132** | **227.3** |

**37% improvement** when using correct configuration!

---

## Comparison: All Experiments

| Experiment | Sprint | Angle | Best Eval Reward | Best Steps |
|------------|--------|-------|------------------|------------|
| Exp 1 (015) | 2x | ±60° | 38.10 | 137 |
| Exp 2 (015) | 3x | ±60° | 106.24 | 205 |
| **Exp 3 (016)** | **2x** | **±30°** | **130.59** | **227** |

### Key Finding
**2x sprint + ±30° angle outperforms 3x sprint + ±60° angle** (130.59 vs 106.24 reward).

This confirms that spawn clustering from wide angles is a bigger obstacle than raw speed. A narrower fan creates more "fair" projectile patterns that the agent can learn to dodge.

---

## Agent Behavior Observations

### 3x Sprint Agent (Exp 2)
- Relies heavily on speed to escape
- "Sliding fast" behavior—less refined movement
- Brute-forces through clusters with raw velocity

### 2x Sprint Agent (Exp 3)
- More deliberate dodging patterns
- Cool "zig-zag" movements between projectiles
- Better demonstrates learned evasion strategy
- Closer to desired behavior for the game

The 2x sprint agent's behavior is preferable—it shows actual skill rather than just speed exploitation.

---

## Human Sprint Control

Added Shift key sprint for human playtesting:
```rust
// src/game/player.rs
let sprint = if keyboard_input.pressed(KeyCode::ShiftLeft)
           || keyboard_input.pressed(KeyCode::ShiftRight) {
    1.0
} else {
    0.0
};
```

This allows humans to experience the same speed capabilities as the RL agent.

---

## Files Modified

**Rust:**
- `src/config.rs`: Added `spawn_angle_degrees` field
- `src/rl/api.rs`: Extended ConfigureRequest with new parameters
- `src/main.rs`: Handle new config parameters in Configure command
- `src/game/projectile.rs`: Use configurable spawn angle
- `src/game/player.rs`: Added Shift key sprint

**Python:**
- `python/config.py`: Added `sprint_multiplier`, `spawn_angle_degrees` fields
- `python/bevy_dodge_env/environment.py`: Updated `configure()` method
- `python/train_ppo.py`: Pass new params to configure
- `python/eval_ppo.py`: Added `--sprint-multiplier` and `--spawn-angle` CLI args
- `python/configs/ppo_level2_basic3d_narrow.yaml`: New config for narrow angle experiment

---

## Next Steps

1. **Further narrow angle tests:** Try ±20° or ±25° to find optimal balance
2. **Combined tuning:** Test ±30° with slightly higher sprint (2.5x)
3. **Longer training:** Run 1M steps to see if narrow angle achieves higher success rate
4. **Spawn separation:** Consider adding minimum angle delta between consecutive spawns

---

## Conclusion

Making environment parameters configurable from Python enables rapid experimentation. The key insight from this session is that **spawn pattern fairness matters more than raw escape speed**. The 2x sprint + ±30° angle configuration:

- Achieves best reward (130.59 vs 106.24)
- Produces more skilled-looking agent behavior
- Confirms spawn clustering as the primary challenge

The configurable parameters infrastructure now makes it easy to run systematic parameter sweeps.
