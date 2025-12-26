# Project Overview - Bevy 3D Dodge with RL

**Quick Summary:** A 3D dodgeball game built with Bevy (Rust) that trains AI agents via reinforcement learning. PPO achieves 100% success rate in 10k steps.

## Core Architecture

### Technology Stack
- **Game Engine:** Bevy 0.15 (Rust ECS)
- **RL Framework:** Stable-Baselines3 (Python)
- **Deep Learning:** PyTorch 2.9.1 + ROCm 6.4 (AMD GPU)
- **Environment Interface:** Gymnasium (OpenAI Gym standard)
- **API Layer:** Axum HTTP server (Rust async)
- **Visualization:** TensorBoard

### System Design

```
Python (RL Training)          Rust (Game Engine)
┌─────────────────┐           ┌──────────────────┐
│ DQN/PPO Agent   │◄────HTTP──┤ Axum Server      │
│ (SB3)           │  REST API │ (port 8000)      │
└─────────────────┘           └──────────────────┘
                                      │
                              ┌───────▼──────────┐
                              │ RL Environment   │
                              │ - Observations   │
                              │ - Actions        │
                              │ - Rewards        │
                              └───────┬──────────┘
                                      │
                              ┌───────▼──────────┐
                              │ Game Core (ECS)  │
                              │ - Player physics │
                              │ - Projectiles    │
                              │ - Collisions     │
                              │ - 3D rendering   │
                              └──────────────────┘
```

**Communication Flow:**
1. Python agent requests observation via HTTP GET
2. Rust extracts game state → 65D vector
3. Python computes action using neural network
4. Rust applies action to player velocity
5. Rust simulates physics, checks collisions
6. Rust calculates reward, returns next observation

## Game Mechanics

### Player
- **Movement:** 5 discrete actions (Noop, Up, Down, Left, Right)
- **Speed:** 3.0 units/second
- **Bounds:** Confined to 8×8 play zone
- **Starting position:** Center (0, 0, 0)

### Projectiles
- **Spawn rate:** 1 every 2 seconds
- **Trajectory:** Arc with gravity (spawn at height 5, target ground)
- **Speed:** Randomized initial velocity
- **Max simultaneous:** Up to 3 active projectiles
- **Collision:** Sphere collider (radius 0.5)

### Rendering
- **Camera:** Fixed orbital view (0, -15, 10) looking at origin
- **Lighting:** HDR skybox + image-based lighting
- **Materials:** PBR (physically-based rendering)
- **Performance:** 47-50 FPS during training

## RL Environment

### Observation Space
**Type:** `Box(65,)` float32, range [-100, 100]

**Structure:**
```
[0-2]:   Player position (x, y, z)
[3-4]:   Player velocity (vx, vy)
[5-64]:  Up to 10 projectiles × 6 values:
         - Position (x, y, z)
         - Velocity (vx, vy, vz)
         Zero-padded if < 10 projectiles
```

### Action Space
**Type:** `Discrete(5)`

```
0: NOOP   - No movement
1: UP     - Move +Y direction
2: DOWN   - Move -Y direction
3: LEFT   - Move -X direction
4: RIGHT  - Move +X direction
```

### Reward Function
```python
+1.0   per timestep (survival)
-100.0 on collision (terminal)
+0.5   close dodge bonus (distance < 2.0, scaled)
```

### Episode Termination
- **Done:** Collision with projectile
- **Truncated:** Max 1000 steps reached
- **Success:** Reaching 1000 steps = perfect dodge

## Training Results

### DQN (Deep Q-Network)

**Initial 100k training (Nov 16):**
- Network: [64, 64] MLP
- Learning rate: 1e-4
- Buffer: 50k experiences
- Result: Mean reward 158, 0% success rate
- Issue: Movement bug (race condition)

**Improved 300k training (Nov 17):**
- Network: [256, 256] MLP
- Learning rate: 5e-5 (lower for stability)
- Buffer: 100k experiences
- **Bug fix:** Player movement system race condition
- Result: Mean reward 641, 30% success rate (6/20 episodes)
- High variance: σ=325.30 (inconsistent performance)
- Training time: 2h 49min

**DQN Challenges:**
- High variance in episode performance (2-751 reward range)
- Sample inefficiency (300k steps for 30% success)
- Replay buffer can hold stale experiences
- ε-greedy exploration less effective for this task

### PPO (Proximal Policy Optimization) ⭐

**Baseline training (Nov 17):**
- Network: [256, 256] MLP (same as DQN for fair comparison)
- Learning rate: 3e-4
- Rollout: 2048 steps, 10 epochs
- Result: **100% success rate (20/20 episodes)**
- Training time: **~10 minutes (10k steps)**
- Perfect consistency: σ=1.31

**PPO Breakthrough:**
- 30x more sample efficient (10k vs 300k steps)
- 17x faster training (10 min vs 2h 49min)
- Zero failures across all evaluation episodes
- All episodes reached max 1000 steps

**Why PPO Succeeded:**
1. **On-policy learning** - No stale data from replay buffer
2. **Entropy regularization** - Better exploration (ent_coef=0.01)
3. **Clipped updates** - Stable policy changes (clip_range=0.2)
4. **GAE** - Superior credit assignment (gae_lambda=0.95)
5. **Sample reuse** - 10 epochs per rollout (efficient)

### Performance Comparison

| Metric | DQN (300k) | PPO (10k) | Winner |
|--------|------------|-----------|---------|
| Success rate | 30% | **100%** | PPO |
| Mean reward | 641±325 | **1002±1** | PPO |
| Episode length | 703±290 | **1000±0** | PPO |
| Training steps | 300,000 | **10,000** | PPO |
| Training time | 2h 49min | **10 min** | PPO |
| Variance | High (σ=325) | **Minimal (σ=1)** | PPO |

**Verdict:** PPO comprehensively solved the task.

## Critical Bug Discovery

### The Race Condition (Nov 17)

**Symptom:** Agent didn't move during training despite code appearing correct.

**Root cause:** `player_movement` system unconditionally overwrote velocity every frame:
```rust
// BUGGY CODE
fn player_movement(...) {
    let mut direction = Vec2::ZERO;
    // Check keyboard...
    velocity.0 = direction * config.player_speed;  // Always executes!
    //           ^^^^^^^^^ Zero when no keys pressed
}
```

**Impact:**
- RL actions applied by `handle_rl_commands` were immediately overwritten
- Non-deterministic: depended on Bevy system execution order
- Sometimes worked (lucky ordering), sometimes didn't

**Fix:**
```rust
// FIXED CODE
if keyboard_input.get_pressed().next().is_some() {
    // Only update velocity when keys actually pressed
    velocity.0 = direction * config.player_speed;
}
```

**Lesson:** Bevy systems without explicit `.before()`/`.after()` ordering are non-deterministic. This caused training to work initially (100k) but fail later (300k attempt).

## Project Structure

```
bevy_3d_dodge/
├── src/
│   ├── main.rs              # Bevy app + HTTP server
│   ├── game/
│   │   ├── player.rs        # Player movement (5 actions)
│   │   ├── projectile.rs    # Spawn & physics
│   │   ├── collision.rs     # Detection & game state
│   │   └── camera.rs        # Orbit camera
│   └── rl/
│       ├── api.rs           # Axum HTTP endpoints
│       ├── observation.rs   # 65D state extraction
│       ├── action.rs        # Action → velocity mapping
│       └── environment.rs   # Reward calculation
│
├── python/
│   ├── bevy_dodge_env/      # Gymnasium wrapper
│   ├── train.py             # DQN training
│   ├── train_ppo.py         # PPO training
│   ├── eval.py              # DQN evaluation
│   ├── eval_ppo.py          # PPO evaluation
│   ├── plot_training.py     # TensorBoard plotting
│   ├── config.py            # Unified DQN/PPO config
│   └── configs/
│       ├── dqn_baseline.yaml
│       ├── dqn_improved_baseline.yaml
│       ├── dqn_quick_test.yaml
│       └── ppo_baseline.yaml
│
├── devlogs/                 # Development documentation
├── results/                 # Training artifacts (gitignored)
│   ├── dqn_improved_baseline/
│   │   └── 20251117_020433/
│   │       ├── models/
│   │       ├── logs/
│   │       └── plots/
│   └── ppo_baseline/
│       └── 20251117_213229/
│           ├── models/
│           ├── logs/
│           └── plots/
└── assets/                  # HDR skybox, textures
```

## Key Implementation Details

### HTTP API Endpoints
```
GET  /observation_space  → {shape: [65], dtype: "float32"}
GET  /action_space       → {type: "Discrete", n: 5}
POST /reset              → {observation: float[65], info: {}}
POST /step {action: int} → {observation, reward, done, truncated, info}
```

### Bevy Systems Architecture
```rust
Update schedule:
  - player_movement      // Apply keyboard/RL actions
  - apply_velocity       // Move entities
  - projectile_spawner   // Spawn new projectiles
  - apply_gravity        // Physics
  - collision_detection  // Check collisions
  - handle_rl_commands   // Process RL API requests
  - update_rl_state      // Update shared observation
```

**Critical:** `handle_rl_commands` must not be overridden by `player_movement`.

### Training Pipeline
```
1. Start Bevy game (cargo run)
2. Start Python training (uv run python train_ppo.py --config ...)
3. Agent requests reset
4. Loop:
   - Get observation (65D vector)
   - Compute action (neural network forward pass)
   - Send action to game
   - Receive reward & next observation
   - Store experience
   - Update policy (PPO: every 2048 steps)
5. Evaluate periodically (every 5k steps)
6. Save best model based on eval reward
```

## Configurations

### DQN Best Config
```yaml
algorithm: DQN
total_timesteps: 300000
learning_rate: 0.00005
buffer_size: 100000
batch_size: 32
gamma: 0.99
exploration_fraction: 0.3
net_arch: [256, 256]
```

### PPO Best Config (Winning Setup)
```yaml
algorithm: PPO
total_timesteps: 300000  # Only needed 10k
learning_rate: 0.0003
batch_size: 64
n_steps: 2048
n_epochs: 10
gamma: 0.99
gae_lambda: 0.95
clip_range: 0.2
ent_coef: 0.01
net_arch: [256, 256]
```

## Usage Quick Reference

### Train PPO (Recommended)
```bash
# Terminal 1: Start game
VK_LOADER_DEBUG=error VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json cargo run

# Terminal 2: Train PPO
uv run python python/train_ppo.py --config python/configs/ppo_baseline.yaml
```

### Evaluate Model
```bash
uv run python python/eval_ppo.py \
  results/ppo_baseline/<timestamp>/models/best/best_model.zip \
  --episodes 20
```

### Plot Training Curves
```bash
uv run python python/plot_training.py \
  --logdir results/ppo_baseline/<timestamp>/logs \
  --output results/ppo_baseline/<timestamp>/plots
```

### TensorBoard Monitoring
```bash
tensorboard --logdir results/ppo_baseline/<timestamp>/logs
```

## Development Timeline

1. **Phase 1:** Bevy game foundation (3D rendering, physics)
2. **Phase 2:** RL environment design (observations, actions, rewards)
3. **Phase 3:** HTTP API + Gymnasium wrapper
4. **Phase 4:** DQN training (initial 100k)
5. **Phase 5:** Bug discovery & fix (race condition)
6. **Phase 6:** DQN improved training (300k, 30% success)
7. **Phase 7:** PPO implementation (10k, 100% success) ⭐

## Lessons Learned

### Algorithm Selection
- **PPO vastly superior** for this high-variance task
- On-policy learning avoided replay buffer issues
- Entropy regularization critical for exploration

### System Design
- **Bevy system ordering matters** - use `.before()`/`.after()`
- Race conditions can be non-deterministic and hard to debug
- HTTP API adds ~5-10ms latency (acceptable for 50 FPS training)

### Training Efficiency
- **Sample efficiency >> wall-clock time** for RL
- PPO's 10k steps (10 min) beat DQN's 300k steps (3 hours)
- GPU utilization poor for MLPs but still faster than CPU

### Debugging
- **Visual observation crucial** - watching agent movement revealed bug
- TensorBoard plots can be misleading (variance looks like plateau)
- Devlogs essential for maintaining context across sessions

## Current Status

**Task:** SOLVED ✅

PPO agent achieves 100% success rate with perfect consistency. The dodgeball environment is comprehensively mastered.

**Best Model:** `results/ppo_baseline/20251117_213229/models/best/best_model.zip`

**Performance:**
- Mean reward: 1002 ± 1
- Success rate: 100% (20/20 episodes)
- All episodes reach max 1000 steps
- Training time: 10 minutes

## Future Directions

Potential next steps (not critical, task is solved):
1. Increase difficulty (more/faster projectiles)
2. Test generalization (different spawn patterns)
3. Model compression (distillation, quantization)
4. Recurrent policies (LSTM for memory)
5. Multi-agent scenarios

## References

- **Devlogs:** See `devlogs/` directory for detailed progress
- **Key devlogs:**
  - `010_initial_training_results.md` - DQN 100k baseline
  - `011_bugfix_and_300k_training.md` - Race condition fix
  - `012_ppo_breakthrough.md` - PPO perfect performance

- **Frameworks:**
  - Bevy: https://bevyengine.org/
  - Stable-Baselines3: https://stable-baselines3.readthedocs.io/
  - Gymnasium: https://gymnasium.farama.org/


## From me (user)
Don't commit devlog markdowns unless I tell you so.
github repo name is "bevy-3d-dodge".
Don't commit yourself, I'll commit myself. When I say "make a commit", give me a commit msg draft not directly committing on your side unless I tell you so.