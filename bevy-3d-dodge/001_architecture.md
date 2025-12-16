# Architecture Document: Bevy 3D Dodge Game with RL Training

**Date:** 2025-11-16
**Status:** Initial Design
**Author:** System Architecture

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Component Specifications](#component-specifications)
4. [Communication Protocol](#communication-protocol)
5. [Observation & Action Spaces](#observation--action-spaces)
6. [Reward Design](#reward-design)
7. [Project Structure](#project-structure)
8. [Technology Stack](#technology-stack)
9. [Implementation Phases](#implementation-phases)
10. [Future Enhancements](#future-enhancements)

---

## Overview

This project implements a 3D projectile dodging game using the Bevy game engine (Rust) with reinforcement learning capabilities. The goal is to train RL agents in Python that learn to dodge incoming projectiles through a clean HTTP/REST API interface.

### Goals

- **Game Development**: Create an engaging 3D dodge game with physics and collision detection
- **RL Training**: Enable training of RL agents using state-of-the-art algorithms (starting with DQN)
- **Performance**: Support vectorized training with multiple parallel game instances
- **Extensibility**: Design for easy migration to gRPC and future feature additions
- **Content Creation**: Visual rendering suitable for YouTube videos and blog posts

### Key Features

- State-based observations (positions, velocities)
- XY plane movement for player (initial version)
- Projectiles spawning from one direction
- HTTP/REST API for Python communication
- Stable-Baselines3 integration
- AMD ROCm 6.4.3 + PyTorch 2.4.0 optimized

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Training Pipeline                        │
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                    │
│  │ Python       │      │ Python       │                     │
│  │ Training     │◄────►│ Gym Env      │                     │
│  │ Script       │      │ Wrapper      │                     │
│  │ (SB3/PyTorch)│      │              │                     │
│  └──────────────┘      └──────┬───────┘                     │
│                               │                              │
│                               │ HTTP/REST                    │
│                               │                              │
└───────────────────────────────┼──────────────────────────────┘
                                │
                                │
┌───────────────────────────────┼──────────────────────────────┐
│                     Bevy Game Engine (Rust)                  │
│                               │                              │
│  ┌────────────────────────────▼─────────────────────┐       │
│  │          HTTP API Server (Axum)                  │       │
│  │  /reset  /step  /observation_space  /action_space│       │
│  └────────────────────┬─────────────────────────────┘       │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────┐       │
│  │         RL Environment Manager                   │       │
│  │  - Observation extraction                        │       │
│  │  - Action application                            │       │
│  │  - Reward calculation                            │       │
│  │  - State management                              │       │
│  └────────────────────┬─────────────────────────────┘       │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────┐       │
│  │            Game Core Systems                     │       │
│  │  - Player movement                               │       │
│  │  - Projectile spawning/physics                   │       │
│  │  - Collision detection                           │       │
│  │  - 3D rendering                                  │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Vectorized Training Architecture

For parallel environment training:

```
┌──────────────────────────────────────────────────────┐
│         Python Training Process                      │
│                                                      │
│  ┌────────────────────────────────────────────┐     │
│  │  SubprocVecEnv (Stable-Baselines3)        │     │
│  │                                            │     │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐   │     │
│  │  │ Env 1   │  │ Env 2   │  │ Env N   │   │     │
│  │  │Wrapper  │  │Wrapper  │  │Wrapper  │   │     │
│  │  └────┬────┘  └────┬────┘  └────┬────┘   │     │
│  └───────┼────────────┼────────────┼─────────┘     │
│          │            │            │               │
└──────────┼────────────┼────────────┼───────────────┘
           │ HTTP       │ HTTP       │ HTTP
           │ :8000      │ :8001      │ :800N
           │            │            │
    ┌──────▼──────┐ ┌──▼──────┐ ┌──▼──────┐
    │   Bevy      │ │  Bevy   │ │  Bevy   │
    │ Instance 1  │ │Instance2│ │InstanceN│
    └─────────────┘ └─────────┘ └─────────┘
```

---

## Component Specifications

### 1. Bevy Game Core (Rust)

#### Player System
- **Entity**: 3D capsule or sphere mesh
- **Components**:
  - `Transform` (position, rotation)
  - `Player` marker component
  - `Velocity` (vx, vy)
  - `Collider` for collision detection
- **Movement**: Keyboard controls (WASD) for manual play, API-driven for RL
- **Constraints**: Movement limited to XY plane (constant Z height)

#### Projectile System
- **Spawning**: Timer-based spawning from fixed direction (e.g., from +X axis)
- **Components**:
  - `Transform`
  - `Projectile` marker with speed/direction
  - `Velocity`
  - `Collider`
- **Physics**: Simple linear motion toward -X direction
- **Cleanup**: Despawn when out of bounds

#### Collision System
- **Detection**: Bevy's collision detection or simple distance-based
- **Events**: Emit `CollisionEvent` when player hit
- **Response**: Trigger game over state

#### Camera System
- **Type**: Perspective 3D camera
- **Position**: Angled view showing player and incoming projectiles
- **Control**: Fixed position initially, potential for dynamic camera later

#### Rendering
- **Mode**: Visual by default (for content creation)
- **Future**: Headless mode flag (`--headless`) for faster training
- **Assets**: Simple colored materials (player: blue, projectiles: red, ground: gray)

### 2. RL Environment Manager (Rust)

#### State Management
```rust
pub enum GameState {
    Running,
    GameOver,
    Paused,
}

pub struct RLEnvironment {
    state: GameState,
    episode_steps: u32,
    total_reward: f32,
    last_action: Option<Action>,
}
```

#### Observation Extraction
```rust
pub struct Observation {
    // Player state (5 floats)
    player_position: Vec3,      // x, y, z
    player_velocity: Vec2,      // vx, vy

    // Projectile states (6 * MAX_PROJECTILES floats)
    projectiles: Vec<ProjectileObs>,  // Each: position (3) + velocity (3)
}

pub struct ProjectileObs {
    position: Vec3,
    velocity: Vec3,
}

const MAX_PROJECTILES: usize = 10;  // Pad/mask for fixed-size observations
```

#### Action Application
```rust
pub enum Action {
    Noop,
    Up,
    Down,
    Left,
    Right,
}

// Alternative: Continuous action space
pub struct ContinuousAction {
    delta_x: f32,  // [-1, 1]
    delta_y: f32,  // [-1, 1]
}
```

#### Reward Calculator
```rust
pub struct RewardCalculator;

impl RewardCalculator {
    pub fn calculate(
        &self,
        player_pos: Vec3,
        projectiles: &[Projectile],
        collision: bool,
    ) -> f32 {
        if collision {
            return -100.0;  // Large penalty for getting hit
        }

        let mut reward = 1.0;  // +1 for surviving each timestep

        // Bonus for close dodges (optional)
        let min_distance = projectiles
            .iter()
            .map(|p| player_pos.distance(p.position))
            .min_by(|a, b| a.partial_cmp(b).unwrap())
            .unwrap_or(f32::MAX);

        if min_distance < 2.0 {
            reward += (2.0 - min_distance) * 0.5;  // Bonus for risky dodges
        }

        reward
    }
}
```

### 3. HTTP API Server (Rust - Axum)

#### Endpoints

**POST /reset**
- **Purpose**: Reset the game to initial state
- **Request**: `{}`
- **Response**:
```json
{
  "observation": [0.0, 0.0, 1.0, 0.0, 0.0, ...],
  "info": {}
}
```

**POST /step**
- **Purpose**: Execute one step with given action
- **Request**:
```json
{
  "action": 2  // Discrete action index
}
```
- **Response**:
```json
{
  "observation": [0.1, 0.2, 1.0, 0.5, 0.0, ...],
  "reward": 1.5,
  "done": false,
  "truncated": false,
  "info": {
    "episode_steps": 42,
    "projectile_count": 3
  }
}
```

**GET /observation_space**
- **Purpose**: Query observation space specification
- **Response**:
```json
{
  "shape": [65],  // 5 + 6*10
  "dtype": "float32",
  "low": -100.0,
  "high": 100.0
}
```

**GET /action_space**
- **Purpose**: Query action space specification
- **Response**:
```json
{
  "type": "Discrete",
  "n": 5
}
```

#### Server Configuration
```rust
pub struct ApiConfig {
    pub host: String,      // "127.0.0.1"
    pub port: u16,         // 8000 (configurable for vectorized envs)
    pub timeout_ms: u64,   // Request timeout
}
```

### 4. Python Gym Environment Wrapper

#### Base Environment Class
```python
import gymnasium as gym
import numpy as np
import requests
from typing import Tuple, Dict, Any

class BevyDodgeEnv(gym.Env):
    """Gymnasium environment wrapper for Bevy 3D Dodge game."""

    metadata = {"render_modes": ["human"], "render_fps": 60}

    def __init__(self, host: str = "127.0.0.1", port: int = 8000):
        super().__init__()
        self.base_url = f"http://{host}:{port}"

        # Fetch space specifications from Bevy
        obs_space_info = self._get(f"{self.base_url}/observation_space")
        action_space_info = self._get(f"{self.base_url}/action_space")

        # Define spaces
        self.observation_space = gym.spaces.Box(
            low=obs_space_info["low"],
            high=obs_space_info["high"],
            shape=tuple(obs_space_info["shape"]),
            dtype=np.float32
        )

        self.action_space = gym.spaces.Discrete(action_space_info["n"])

    def reset(self, seed=None, options=None) -> Tuple[np.ndarray, Dict]:
        super().reset(seed=seed)
        response = self._post(f"{self.base_url}/reset", {})
        return np.array(response["observation"], dtype=np.float32), response["info"]

    def step(self, action: int) -> Tuple[np.ndarray, float, bool, bool, Dict]:
        response = self._post(f"{self.base_url}/step", {"action": int(action)})
        return (
            np.array(response["observation"], dtype=np.float32),
            response["reward"],
            response["done"],
            response.get("truncated", False),
            response["info"]
        )

    def _post(self, url: str, data: Dict) -> Dict:
        response = requests.post(url, json=data, timeout=5.0)
        response.raise_for_status()
        return response.json()

    def _get(self, url: str) -> Dict:
        response = requests.get(url, timeout=5.0)
        response.raise_for_status()
        return response.json()
```

#### Vectorized Environment Wrapper
```python
from stable_baselines3.common.vec_env import SubprocVecEnv

def make_env(port: int):
    """Create a single environment instance."""
    def _init():
        return BevyDodgeEnv(port=port)
    return _init

def make_vec_env(n_envs: int, start_port: int = 8000):
    """Create vectorized environment with n_envs parallel instances."""
    return SubprocVecEnv([make_env(start_port + i) for i in range(n_envs)])
```

---

## Communication Protocol

### Request/Response Format

All API communication uses JSON over HTTP.

#### Reset Sequence
```
Python → POST /reset
       ← 200 OK {"observation": [...], "info": {}}
```

#### Step Sequence
```
Python → POST /step {"action": 2}
       ← 200 OK {"observation": [...], "reward": 1.0, "done": false, "truncated": false, "info": {}}
```

#### Error Handling
```
Python → POST /step {"action": 999}
       ← 400 Bad Request {"error": "Invalid action: 999 not in range [0, 5)"}
```

### Thread Safety

The Bevy game runs on the main thread, while the Axum HTTP server runs on separate async runtime threads. Communication handled via:
- **Channels**: `tokio::sync::mpsc` for sending commands from API to game
- **Shared State**: `Arc<Mutex<EnvState>>` for reading observations

```rust
pub struct SharedEnvState {
    pub observation: Vec<f32>,
    pub reward: f32,
    pub done: bool,
    pub info: HashMap<String, serde_json::Value>,
}

// In API handler
let state = env_state.lock().unwrap();
let obs = state.observation.clone();
```

---

## Observation & Action Spaces

### Observation Space

**Type**: `Box(low=-100, high=100, shape=(65,), dtype=float32)`

**Layout** (65-dimensional vector):
```
Indices 0-2:   Player position (x, y, z)
Indices 3-4:   Player velocity (vx, vy)
Indices 5-64:  Projectile data (10 projectiles × 6 values)
  - For each projectile:
    - Position (x, y, z)
    - Velocity (vx, vy, vz)
  - If fewer than 10 projectiles exist, remaining slots are zero-padded
```

**Normalization**: Positions and velocities are in world coordinates. Consider normalizing to [-1, 1] for better neural network training.

### Action Space

**Type**: `Discrete(5)`

**Actions**:
```
0: NOOP  (no movement)
1: UP    (move in +Y direction)
2: DOWN  (move in -Y direction)
3: LEFT  (move in -X direction)
4: RIGHT (move in +X direction)
```

**Alternative Continuous Action Space** (future):
```
Box(low=-1, high=1, shape=(2,), dtype=float32)
[delta_x, delta_y]  # Applied as velocity or acceleration
```

---

## Reward Design

### Reward Components

1. **Survival Reward**: +1.0 per timestep
2. **Dodge Bonus** (optional): +0.5 * (2.0 - min_distance) for close dodges (distance < 2.0)
3. **Collision Penalty**: -100.0 (terminal)

### Reward Shaping Considerations

**Current Design** (simple, sparse):
```
R(t) = 1.0 (if alive) + dodge_bonus - 100.0 (if collision)
```

**Potential Improvements**:
- Distance-based shaping: Small negative reward for being close to projectiles
- Velocity penalty: Discourage excessive movement
- Efficiency bonus: Reward staying near center
- Curriculum learning: Start with slow projectiles, increase difficulty

### Episode Termination

**Terminal Conditions**:
- Player collides with projectile (`done=True`)
- Maximum episode length reached (e.g., 1000 steps) (`truncated=True`)

**Reset Behavior**:
- Player returns to origin (0, 0, 1)
- All projectiles despawned
- Timers reset
- Episode counters reset

---

## Project Structure

```
bevy_3d_dodge/
├── Cargo.toml                          # Rust project manifest
├── Cargo.lock                          # Dependency lock file
├── .gitignore                          # Git ignore rules
├── README.md                           # Project documentation
│
├── src/                                # Rust source code
│   ├── main.rs                         # Entry point, Bevy app setup
│   ├── config.rs                       # Configuration structs
│   │
│   ├── game/                           # Game logic modules
│   │   ├── mod.rs                      # Module exports
│   │   ├── player.rs                   # Player entity, components, movement
│   │   ├── projectile.rs               # Projectile spawning, physics
│   │   ├── camera.rs                   # Camera setup and control
│   │   └── collision.rs                # Collision detection system
│   │
│   └── rl/                             # RL integration modules
│       ├── mod.rs                      # Module exports
│       ├── environment.rs              # RL environment state management
│       ├── observation.rs              # Observation extraction logic
│       ├── action.rs                   # Action parsing and application
│       └── api.rs                      # HTTP API server (Axum)
│
├── pyproject.toml                      # Python project configuration (uv)
├── uv.lock                             # Python dependency lock (generated)
│
├── python/                             # Python RL training code
│   ├── bevy_dodge_env/                 # Gym environment package
│   │   ├── __init__.py                 # Package exports
│   │   ├── environment.py              # BevyDodgeEnv class
│   │   └── vec_env.py                  # Vectorized environment utilities
│   │
│   ├── train.py                        # Training script (DQN with SB3)
│   ├── eval.py                         # Evaluation and visualization
│   ├── config.py                       # Training hyperparameters
│   └── utils.py                        # Helper functions
│
├── devlogs/                            # Development logs and documentation
│   └── 001_architecture.md             # This document
│
├── models/                             # Saved RL models (gitignored)
│   └── .gitkeep
│
├── logs/                               # Training logs, TensorBoard (gitignored)
│   └── .gitkeep
│
└── assets/                             # Game assets (future)
    └── .gitkeep
```

---

## Technology Stack

### Rust (Bevy Game)

| Library | Version | Purpose |
|---------|---------|---------|
| `bevy` | 0.14 | Game engine (ECS, rendering, physics) |
| `tokio` | 1.40 | Async runtime for HTTP server |
| `axum` | 0.7 | HTTP web framework |
| `serde` | 1.0 | Serialization/deserialization |
| `serde_json` | 1.0 | JSON support |

### Python (RL Training)

| Library | Version | Purpose |
|---------|---------|---------|
| `torch` | 2.4.0 | Deep learning framework (ROCm 6.4.3) |
| `stable-baselines3` | 2.3.0+ | RL algorithms (DQN, PPO, etc.) |
| `gymnasium` | 0.29.0+ | RL environment interface |
| `numpy` | 1.24.0+ | Numerical operations |
| `tensorboard` | 2.20.0+ | Training visualization |
| `requests` | 2.31.0+ | HTTP client for API calls |
| `tqdm` | 4.65.0+ | Progress bars |

### Development Tools

- **Rust**: `cargo`, `rustfmt`, `clippy`
- **Python**: `uv` (package management), `ruff` (linting), `mypy` (type checking)
- **Version Control**: Git

---

## Implementation Phases

### Phase 1: Bevy Game Core (Days 1-3)
**Goal**: Playable 3D game with keyboard controls

- [ ] Initialize Bevy project with basic 3D scene
  - Ground plane, directional lighting, camera
  - Sky/background color
- [ ] Create player entity
  - 3D capsule mesh with material
  - WASD movement controls (XY plane)
  - Velocity-based physics
- [ ] Implement projectile system
  - Spawn projectiles at fixed intervals (every 2 seconds)
  - Linear motion from +X toward -X
  - Despawn when out of bounds
- [ ] Add collision detection
  - Distance-based or Bevy collision detection
  - Game over on player-projectile collision
  - Visual feedback (color change, game over text)
- [ ] Game state management
  - Running, GameOver, Paused states
  - Reset functionality (R key)
  - UI text displaying state

**Validation**: Play the game manually, verify collision detection works

### Phase 2: RL Environment Integration (Days 4-5)
**Goal**: Expose game as RL environment via HTTP API

- [ ] Define observation extraction
  - Implement `extract_observation()` function
  - Return 65-dimensional float vector
  - Handle variable projectile count (padding)
- [ ] Define action application
  - Parse action enum/index
  - Apply to player velocity
  - Clamp/limit movement
- [ ] Implement reward calculation
  - Survival reward: +1 per step
  - Collision penalty: -100
  - Optional dodge bonus
- [ ] Create HTTP API with Axum
  - `/reset` endpoint
  - `/step` endpoint
  - `/observation_space` and `/action_space` info endpoints
  - Error handling and validation
- [ ] Thread-safe state sharing
  - Use `Arc<Mutex<>>` for shared state
  - Channels for game loop communication

**Validation**: Test API with `curl` or Postman

### Phase 3: Python Gym Wrapper (Days 6-7)
**Goal**: Python environment that talks to Bevy via HTTP

- [ ] Set up Python project with uv
  - Create `pyproject.toml`
  - Install gymnasium, requests, numpy
- [ ] Implement `BevyDodgeEnv` class
  - Inherit from `gym.Env`
  - Implement `reset()` and `step()`
  - HTTP client for API communication
  - Space definitions
- [ ] Create vectorized environment wrapper
  - Use `SubprocVecEnv` from SB3
  - Helper function to spawn multiple Bevy instances
  - Port management for parallel envs
- [ ] Write tests
  - Test single environment
  - Test vectorized environments
  - Verify observation/action space consistency

**Validation**: Run random agent in environment, check for errors

### Phase 4: RL Training Setup (Days 8-9)
**Goal**: Train DQN agent successfully

- [ ] Set up PyTorch with ROCm
  - Install torch for ROCm 6.4.3
  - Verify GPU detection (`torch.cuda.is_available()`)
- [ ] Install Stable-Baselines3
  - Install sb3 and sb3-contrib
  - Test DQN import
- [ ] Write training script
  - Create DQN agent with hyperparameters
  - TensorBoard logging
  - Model checkpointing
  - Training loop
- [ ] Write evaluation script
  - Load trained model
  - Run episodes with visualization
  - Record metrics (avg reward, success rate)
- [ ] Run baseline training
  - Train for 100k steps
  - Monitor TensorBoard
  - Verify learning (reward increases)

**Validation**: Agent performance improves over time, reward curve trends upward

### Phase 5: Documentation & Refinement (Days 10-11)
**Goal**: Clean, documented, reproducible project

- [ ] Write comprehensive README
  - Project description
  - Installation instructions (Rust + Python)
  - Usage examples (manual play, training, evaluation)
  - Architecture overview
  - Troubleshooting
- [ ] Add configuration files
  - `config.toml` for game settings (projectile speed, spawn rate, etc.)
  - `train_config.yaml` for hyperparameters
- [ ] Code cleanup
  - Add comments and documentation
  - Run `rustfmt` and `clippy`
  - Run `ruff` and `mypy` on Python code
- [ ] Create `.gitignore`
  - Rust: `/target`, `Cargo.lock` (for binaries)
  - Python: `__pycache__`, `.venv`, `*.pyc`
  - Models/logs: `/models`, `/logs`
- [ ] Test end-to-end pipeline
  - Fresh clone of repo
  - Follow README to set up
  - Train agent from scratch
  - Verify reproducibility

**Validation**: Another person can set up and train successfully

---

## Future Enhancements

### Short-Term (1-2 weeks)

1. **Headless Mode**
   - Add `--headless` CLI flag
   - Disable rendering for faster training
   - Bevy headless example integration

2. **Pixel-Based Observations**
   - Capture camera render to image
   - Send RGB pixels as observation
   - Integrate with CNN-based policies

3. **Curriculum Learning**
   - Start with slow projectiles
   - Gradually increase speed/frequency
   - Automatic difficulty adjustment

4. **Advanced Algorithms**
   - PPO (better for continuous control)
   - SAC (if switching to continuous actions)
   - Rainbow DQN improvements

### Medium-Term (1-2 months)

5. **gRPC Migration**
   - Implement `GrpcCommunicator` trait
   - Protocol buffer definitions
   - Performance benchmarking vs HTTP
   - Unix domain socket optimization

6. **3D Movement (Yaw/Pitch/Roll)**
   - Extend action space to 6DOF
   - Projectiles from multiple directions
   - More complex dodging strategies

7. **Multi-Agent Support**
   - Multiple players in same environment
   - Cooperative or competitive scenarios
   - MARL algorithms (QMIX, MAPPO)

8. **Procedural Projectile Patterns**
   - Varied projectile types (fast, slow, homing)
   - Wave-based difficulty progression
   - Boss patterns (bullet hell style)

### Long-Term (3+ months)

9. **Advanced Graphics**
   - Particle effects for projectiles
   - Trail effects for player movement
   - Better materials and lighting

10. **Imitation Learning**
    - Record human gameplay
    - Behavioral cloning baseline
    - GAIL or AIRL for reward learning

11. **Transfer Learning**
    - Pre-train in simple environment
    - Fine-tune in complex scenarios
    - Domain randomization

12. **Web Deployment**
    - WASM build of Bevy game
    - Run trained model in browser
    - WebGPU for inference

---

## Performance Considerations

### Training Performance

**Expected Throughput** (single environment):
- HTTP API: ~50-100 steps/sec
- Bottleneck: JSON serialization + network overhead

**Optimization Strategies**:
1. **Vectorized Environments**: 4-8 parallel instances on different ports
2. **Observation Compression**: Send only changed projectiles (delta encoding)
3. **Batch API**: Send multiple actions, receive multiple transitions (future)
4. **gRPC Migration**: 5-10x speedup with binary protocol

### GPU Utilization (AMD 7900XTX)

**Current Setup**:
- Bevy rendering: Minimal GPU usage (simple 3D scene)
- PyTorch training: High GPU usage (NN forward/backward passes)

**Optimizations**:
- `torch.compile`: 2-3x speedup on AMD GPUs (PyTorch 2.0+)
- Mixed precision training: FP16 for faster training
- Larger batch sizes: Maximize GPU memory usage
- Asynchronous data collection: Separate processes for env rollout and training

**ROCm-Specific**:
- Ensure `HSA_OVERRIDE_GFX_VERSION=11.0.0` for 7900XTX (if needed)
- Use PyTorch ROCm wheels from official repo
- Profile with `rocprof` if performance issues arise

### Memory Usage

**Bevy Game** (per instance):
- Estimated: 100-200 MB RAM
- Scales linearly with projectile count

**Python Training**:
- Model parameters: ~1-10 MB (DQN with small network)
- Replay buffer: ~100-500 MB (capacity 100k transitions)
- GPU memory: 1-2 GB (batch size 32-64)

**Vectorized Setup** (8 environments):
- Total RAM: ~2-3 GB
- GPU RAM: ~2-3 GB
- Well within 24 GB VRAM of 7900XTX

---

## Testing Strategy

### Unit Tests (Rust)

- `rl::observation`: Test observation extraction with various game states
- `rl::action`: Test action parsing and application
- `rl::reward`: Test reward calculation edge cases
- `game::collision`: Test collision detection accuracy

### Integration Tests (Rust)

- `api::reset`: Test full reset flow via HTTP
- `api::step`: Test step execution via HTTP
- `api::error_handling`: Test invalid inputs

### Environment Tests (Python)

- `test_env_creation`: Ensure environment initializes correctly
- `test_reset`: Verify reset returns valid observation
- `test_step`: Verify step returns valid transition
- `test_spaces`: Check observation/action spaces match
- `test_vec_env`: Test vectorized environment creation

### End-to-End Tests

- Manual gameplay test (human can play and enjoy)
- Random agent test (no crashes over 10k steps)
- Training test (reward increases over 50k steps)
- Evaluation test (load model and run episode)

---

## Risk Mitigation

### Potential Issues

1. **API Latency**: HTTP too slow for training
   - **Mitigation**: Start simple, profile, migrate to gRPC if needed
   - **Fallback**: Shared memory or PyO3 integration

2. **ROCm Compatibility**: PyTorch issues with ROCm 6.4.3
   - **Mitigation**: Use tested PyTorch version (2.4.0 as reference)
   - **Fallback**: Docker with official ROCm PyTorch image

3. **Thread Safety**: Race conditions in shared state
   - **Mitigation**: Careful use of `Arc<Mutex<>>` and channels
   - **Testing**: Stress test with multiple concurrent requests

4. **Game Balance**: Too hard/easy for RL to learn
   - **Mitigation**: Configurable difficulty (projectile speed, spawn rate)
   - **Testing**: Baseline with random agent, adjust until learnable

5. **Observation Space**: Insufficient information for policy
   - **Mitigation**: Start with rich state, ablation study later
   - **Extension**: Add pixel observations if state insufficient

---

## Success Metrics

### Minimum Viable Product (MVP)

- [ ] Bevy game is playable manually
- [ ] Python can control game via API
- [ ] DQN agent trains without errors
- [ ] Reward increases over time (learning signal)
- [ ] Trained agent performs better than random

### Stretch Goals

- [ ] Agent achieves >90% survival rate in 1000-step episodes
- [ ] Training completes in <2 hours on 7900XTX
- [ ] Vectorized training with 8 environments works
- [ ] Visual rendering suitable for YouTube (60 fps, smooth)
- [ ] Comprehensive documentation and tutorials

---

## Conclusion

This architecture provides a solid foundation for building an RL-trainable 3D dodge game. The modular design allows for:

- **Rapid Prototyping**: HTTP API for quick iteration
- **Performance Scaling**: Easy migration to gRPC or faster methods
- **Extensibility**: Clean separation of game logic and RL interface
- **Content Creation**: Visual rendering for educational content

The choice of Bevy (Rust) + Stable-Baselines3 (Python) leverages:
- Bevy's excellent performance and ECS architecture
- SB3's mature RL implementations and documentation
- ROCm's AMD GPU optimization for PyTorch

By following the phased implementation plan, we can build and validate each component incrementally, reducing risk and enabling early testing.

---

**Next Steps**: Begin Phase 1 implementation with Bevy game core setup.
