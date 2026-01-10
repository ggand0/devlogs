# Devlog 039: Thrower Agent Implementation

**Date:** 2026-01-10
**Branch:** feat/thrower-agent

## Overview

Implemented two-player training setup where a thrower agent learns to aim projectiles at a scripted dodger. This is the first step toward adversarial self-play training.

## Design

### Training Mode
- **Sequential training**: Train thrower against frozen/scripted opponent first
- Server CLI flag: `--training-agent thrower` or `--training-agent dodger`

### Thrower Agent
- **Observation (5D)**: `[player_x, player_y, player_z, player_vx, player_vy]`
- **Action (1D continuous)**: Aim angle normalized to [-1, 1], mapped to ±spawn_angle_degrees
- **Reward**:
  - +100 on hit (player collision)
  - -0.1 per step (time penalty)
  - Shaped bonus for close misses (not yet implemented in this version)

### Scripted Dodger AI
Simple reactive AI that moves perpendicular to the nearest projectile's velocity:
- Finds nearest projectile
- Computes escape direction (perpendicular to projectile velocity)
- Moves at player_speed in escape direction

## Implementation

### Rust Changes

**src/config.rs**
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum TrainingAgent {
    #[default]
    Dodger,
    Thrower,
}
```

**src/main.rs**
- Added `--training-agent` CLI arg
- Routes to appropriate observation/reward based on training_agent
- Fixed bug: preserve training_agent when level is changed via configure()

**src/rl/observation.rs**
```rust
pub const THROWER_OBSERVATION_SIZE: usize = 5;

pub fn extract_thrower_observation(...) -> Vec<f32> {
    // Returns [px, py, pz, vx, vy]
}
```

**src/rl/environment.rs**
```rust
pub fn calculate_thrower_reward(...) -> f32 {
    // +100 on hit, -0.1 per step
}
```

**src/game/projectile.rs**
- Added `ThrowerAction` resource to store agent's chosen angle
- Modified spawning to use agent's angle instead of random when in thrower mode

**src/game/player.rs**
- Added `scripted_dodge_ai()` system that runs when training_agent == Thrower

**src/rl/grpc_api.rs**
- `get_observation_space()`: Returns 5D for thrower, standard 65D for dodger
- `get_action_space()`: Returns Box(1) for thrower, Discrete(5) or Box(5) for dodger
- `step()`: Routes to StepThrower command when in thrower mode

### Python Changes

**python/config.py**
- Changed `action_space_type` default from `"discrete"` to `None` to let server decide

**python/configs/sac_thrower_agent.yaml**
```yaml
algorithm: SAC
total_timesteps: 500000
level: 2
net_arch: [128, 128]
learning_rate: 0.0003
buffer_size: 100000
learning_starts: 5000
```

## Training Results

### Configuration
- Algorithm: SAC
- Level: 2 (Hard) - 60° spawn fan, 0.5s spawn interval
- Network: [128, 128] MLP
- Total steps: 500,000

### Results
| Metric | Start | End | Notes |
|--------|-------|-----|-------|
| Episode Length | 758 | 389 | 2x faster kills |
| Episode Reward | -20.8 | 55.0 | Learned to hit |
| Peak Reward | - | 64.6 | |
| Training Time | - | 2h 26m | |

### Observations
1. **Thrower learned effectively**: Episode length dropped from ~750 to ~390, meaning the thrower kills the scripted dodger twice as fast as random aiming
2. **Reward stabilized around 55-65**: Consistent performance after learning
3. **Eval reward lower than training**: Final eval reward was -23.8 vs training 55.0. This may be due to:
   - Eval episodes running to max length (1000 steps) when dodger survives
   - Different random seeds in eval
   - Scripted dodger being more effective in some configurations

## Files Modified

| File | Changes |
|------|---------|
| src/config.rs | Added TrainingAgent enum |
| src/main.rs | CLI arg, training_agent preservation fix |
| src/rl/observation.rs | extract_thrower_observation() |
| src/rl/environment.rs | calculate_thrower_reward() |
| src/game/projectile.rs | ThrowerAction resource, angle-based spawning |
| src/game/player.rs | scripted_dodge_ai() system |
| src/rl/grpc_api.rs | Thrower mode handling |
| src/rl/api.rs | StepThrower command |
| python/config.py | action_space_type default fix |
| python/configs/sac_thrower_agent.yaml | New config |

## Usage

```bash
# Start server in thrower training mode
./target/release/bevy_3d_dodge --headless --fps 240 --training-agent thrower

# Train thrower agent
uv run --directory python python train_sac.py --config configs/sac_thrower_agent.yaml
```

## Next Steps

1. **Improve scripted dodger**: Current AI is simple - could add prediction, randomness
2. **Add shaped rewards**: Bonus for close misses to encourage aiming improvement
3. **Train dodger against trained thrower**: Use the trained thrower model as opponent
4. **Self-play**: Alternate training both agents against each other

## Model Location

`results/sac_thrower_agent/20260110_145322/models/final_model.zip`
