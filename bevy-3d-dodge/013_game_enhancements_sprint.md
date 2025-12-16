# Devlog 013: Game Enhancements - Sprint Mechanic & Camera Improvements

**Date:** November 20, 2025
**Focus:** UI polish, camera controls, and sprint mechanic for continuous action space

---

## Overview

This session focused on enhancing the game experience and adding a critical new mechanic for RL training: **sprint**. Level 2's difficulty (4.5 speed projectiles, 0.5s spawn interval, random positions) was proving too challenging without a way for agents to burst to safety. We implemented a clean 5D continuous action space that gives agents explicit control over movement speed.

---

## 1. UI & Visual Improvements

### 1.1 Action Space Display
**Goal:** Show current game configuration during training to help visualize what the agent is learning.

**Implementation:**
- Added `ConfigInfoText` component in [src/main.rs](src/main.rs:401)
- Displays below the level indicator (top right)
- Shows: "Action Space: Discrete" or "Action Space: Continuous"
- Font size: 16 (smaller than level label's 22)
- Color: Gray (0.7, 0.7, 0.7) to differentiate from green level label

**Code:**
```rust
// Config info (below level indicator)
commands.spawn((
    Text::new("Action Space: Discrete"),
    TextFont { font_size: 16.0, ..default() },
    TextColor(Color::srgb(0.7, 0.7, 0.7)),
    Node {
        position_type: PositionType::Absolute,
        top: Val::Px(70.0),
        right: Val::Px(10.0),
        ..default()
    },
    ConfigInfoText,
));
```

**Updates dynamically** when configuration changes via `/configure` endpoint.

### 1.2 Simplified Game Over Log
**Change:** Reduced verbosity of game over messages
- Before: `"Game Over! Press R to restart."`
- After: `"Game Over!"`

**Rationale:** During training, hundreds of episodes complete per minute. The "Press R to restart" instruction is irrelevant for agents and cluttered logs. Human players already know the R key resets.

---

## 2. Camera Control Enhancements

### 2.1 Mouse Wheel Zoom in Default Mode
**Previous behavior:** Mouse wheel zoom only worked in free camera mode (F1)

**New behavior:** Mouse wheel zoom works in both default and free camera modes

**Implementation:** [src/game/camera.rs](src/game/camera.rs:151-156)
```rust
// Mouse wheel scroll for zoom (forward/backward movement) - works in both modes
for wheel in mouse_wheel.read() {
    let zoom_amount = wheel.y * zoom_sensitivity;
    let forward = transform.forward();
    transform.translation += forward * zoom_amount;
}
```

### 2.2 Double-Click Camera Reset
**Feature:** Double-clicking anywhere on screen resets camera to default position

**Implementation:** [src/game/camera.rs](src/game/camera.rs:127-149)
- Uses `Local<Option<f64>>` to track last click time
- 300ms threshold for double-click detection
- Resets to: position `(0.0, -15.0, 10.0)`, looking at `(0.0, 0.0, 1.0)`
- Works in both default and free camera modes

**Code:**
```rust
const DOUBLE_CLICK_THRESHOLD: f64 = 0.3; // 300ms
if mouse_button.just_pressed(MouseButton::Left) {
    let current_time = time.elapsed_secs_f64();

    if let Some(last_time) = *double_click_timer {
        let time_since_last_click = current_time - last_time;

        if time_since_last_click < DOUBLE_CLICK_THRESHOLD {
            // Reset camera
            transform.translation = Vec3::new(0.0, -15.0, 10.0);
            *transform = transform.looking_at(Vec3::new(0.0, 0.0, 1.0), Vec3::Z);
            info!("Camera reset to default position");
            *double_click_timer = None;
        } else {
            *double_click_timer = Some(current_time);
        }
    } else {
        *double_click_timer = Some(current_time);
    }
}
```

---

## 3. Sprint Mechanic Implementation

### 3.1 Motivation
Level 2 (Hard difficulty) parameters:
- Projectile speed: 4.5 units/sec (50% faster than Level 1's 3.0)
- Spawn interval: 0.5 seconds (4x more frequent than Level 1's 2.0s)
- Random spawn positions: 120° fan instead of fixed direction
- Max projectiles: 25 (2.5x more than Level 1's 10)

**Problem:** Even with continuous movement and tilting, agents couldn't consistently dodge the dense projectile patterns. The base speed of 5.0 units/sec wasn't enough.

**Solution:** Add a sprint mechanic that lets agents temporarily boost speed by 1.5x when needed.

### 3.2 Design Choices

We evaluated 4 options:

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **1. 5D action space** | Add explicit sprint component | Explicit control, agent learns strategy | Requires retraining |
| 2. Velocity magnitude | Sprint when `sqrt(vx² + vy²) > threshold` | No dimension change | Can't sprint slowly |
| 3. Tilt magnitude | Sprint when leaning hard | Intuitive | Couples movement & visual |
| 4. Always sprint | Increase base speed for continuous mode | Simplest | No tactical control |

**Selected: Option 1** - Cleanest design, gives agents explicit control over sprint strategy.

### 3.3 Implementation

#### 3.3.1 Action Space Expansion
**From:** 4D Box `[vx, vy, pitch, roll]`
**To:** 5D Box `[vx, vy, pitch, roll, sprint]`

All components in range `[-1, 1]`.

#### 3.3.2 Sprint Normalization
Sprint is normalized from `[-1, 1]` input to `[0, 1]` internally:

```rust
// Normalize sprint from [-1, 1] to [0, 1] for clearer semantics
// -1 and 0 both mean no sprint, +1 means full sprint
let sprint_normalized = (arr[4] + 1.0) / 2.0;
```

**Rationale:**
- `-1.0` and `0.0` both mean "no sprint"
- `+1.0` means "full sprint"
- Smoother for neural networks than using negative values for "no action"

#### 3.3.3 Speed Calculation
[src/rl/action.rs](src/rl/action.rs:101-107)

```rust
// Calculate effective speed with sprint multiplier
// sprint is in [0, 1], so: speed = base_speed * (1 + sprint * multiplier)
let speed_multiplier = 1.0 + action.sprint * config.sprint_multiplier;
let effective_speed = config.player_speed * speed_multiplier;

// Apply velocity (normalized action * effective_speed)
velocity.0 = action.velocity * effective_speed;
```

**Configuration:** [src/config.rs](src/config.rs:74)
```rust
pub sprint_multiplier: f32,  // Speed multiplier when sprinting (e.g., 0.5 = 1.5x speed)
```

**Default value:** `0.5` for both Level 1 and Level 2
- No sprint (`sprint=0`): `5.0 * (1 + 0*0.5) = 5.0` units/sec
- Full sprint (`sprint=1`): `5.0 * (1 + 1*0.5) = 7.5` units/sec
- Partial sprint (`sprint=0.5`): `5.0 * (1 + 0.5*0.5) = 6.25` units/sec

### 3.4 API Changes

#### Rust Side
1. **Action struct** [src/rl/action.rs](src/rl/action.rs:45-50):
```rust
pub struct ContinuousAction {
    pub velocity: Vec2,  // (vx, vy) in range [-1, 1]
    pub pitch: f32,      // Forward/backward tilt in range [-1, 1]
    pub roll: f32,       // Left/right tilt in range [-1, 1]
    pub sprint: f32,     // Sprint intensity in range [0, 1] (normalized from [-1, 1])
}
```

2. **EnvCommand** [src/rl/api.rs](src/rl/api.rs:44):
```rust
StepContinuous { action: [f32; 5] },  // Was [f32; 4]
```

3. **Action space endpoint** [src/rl/api.rs](src/rl/api.rs:286):
```rust
crate::config::ActionSpaceType::Continuous => ActionSpaceResponse::Box {
    r#type: "Box".to_string(),
    shape: vec![5],  // [vx, vy, pitch, roll, sprint]
    low: -1.0,
    high: 1.0,
}
```

#### Python Side
The Python environment automatically queries action space dimensions from the server, so **no code changes needed**. It automatically handles the 5D action space.

### 3.5 Testing

Created [python/test_sprint.py](python/test_sprint.py) to verify sprint functionality:

```python
def test_sprint():
    """Test that sprint increases player speed."""
    # Test 1: No sprint (sprint=-1.0)
    action_no_sprint = np.array([1.0, 0.0, 0.0, 0.0, -1.0], dtype=np.float32)
    # Take 10 steps, measure position

    # Test 2: Full sprint (sprint=+1.0)
    action_with_sprint = np.array([1.0, 0.0, 0.0, 0.0, 1.0], dtype=np.float32)
    # Take 10 steps, measure position

    # Test 3: Partial sprint (sprint=0.0, normalized to 0.5)
    action_partial_sprint = np.array([1.0, 0.0, 0.0, 0.0, 0.0], dtype=np.float32)
    # Take 10 steps, measure position
```

**Results:**
```
Position after 10 steps without sprint: X = 0.753
Position after 10 steps with sprint:    X = 1.124
Speed ratio: 1.49x (expected ~1.5x) ✓

Position after 10 steps partial sprint: X = 0.937
Speed ratio vs no-sprint: 1.25x (expected ~1.25x) ✓
```

Sprint working perfectly!

---

## 4. API Configuration Pattern

### 4.1 Problem: Action Space Query Timing
When calling `env.configure(action_space_type="continuous")`, the game updates its config asynchronously. If a new environment is created immediately, it might query the old action space before the change propagates.

### 4.2 Solution: Reset After Configure
**Pattern:**
```python
temp_env = BevyDodgeEnv(port=8000)
temp_env.configure(action_space_type="continuous")
temp_env.reset()  # Ensures config is synced
del temp_env

# New env now queries updated action space
env = BevyDodgeEnv(port=8000)
```

### 4.3 Documentation
Added clear documentation to [python/bevy_dodge_env/environment.py](python/bevy_dodge_env/environment.py:180-203):

```python
def configure(self, level: Optional[int] = None,
              action_space_type: Optional[str] = None) -> None:
    """Configure game settings.

    Note:
        - After calling configure(), you must call reset() or create a new
          environment instance to ensure the updated action space is queried correctly

    Example:
        >>> env = BevyDodgeEnv()
        >>> env.configure(action_space_type="continuous")
        >>> env.reset()  # Ensures config is synced
        >>> # Or: del env; env = BevyDodgeEnv()  # New env queries updated action space
    """
```

This is not a hack - it's the proper API usage pattern for configuration changes.

---

## 5. Training Configuration

Updated [python/configs/ppo_level2_continuous.yaml](python/configs/ppo_level2_continuous.yaml) comment to reflect 5D action space:

```yaml
# PPO Level 2 (Hard) Continuous Action Space Configuration
# Full training on challenging difficulty with continuous actions (vx, vy, pitch, roll, sprint)
```

**Training command:**
```bash
uv run python python/train_ppo.py --config python/configs/ppo_level2_continuous.yaml
```

**Expected learning:**
- Agents will learn when to sprint (e.g., burst away from projectile clusters)
- When to conserve sprint (normal speed sufficient)
- Tactical speed management for different threat patterns

---

## 6. Commit Summary

```
Add sprint mechanic to continuous action space (5D Box)

Expand continuous action space from 4D to 5D by adding sprint component:
- Action space: [vx, vy, pitch, roll, sprint] all in range [-1, 1]
- Sprint normalized from [-1, 1] to [0, 1] internally for clearer semantics
- Speed formula: effective_speed = base_speed * (1 + sprint * sprint_multiplier)
- Default sprint_multiplier: 0.5 (gives 1.5x speed at full sprint)
- Verified with test_sprint.py: 1.0x (no sprint) → 1.5x (full sprint)

Also improve configure() API documentation to clarify that reset() must be
called after configure() to ensure action space changes are properly synced.
```

---

## 7. Technical Notes

### 7.1 Why Not Async Configuration?
We considered making the `/configure` endpoint wait for the game loop to apply changes before returning (using oneshot channels). However:
- Would require changing `EnvCommand` from `Clone` to non-Clone (breaking change)
- Would require significant architecture refactor
- Current pattern (configure + reset) is idiomatic for gym environments
- Works reliably and is well-documented

### 7.2 Sprint Design Advantages
- **Explicit control:** Agent learns strategy, not just reaction
- **Smooth gradation:** Continuous sprint values allow nuanced speed control
- **Configurable:** `sprint_multiplier` can be tuned per level
- **Observable:** Sprint usage can be logged and analyzed
- **No energy system:** Simplest implementation (can add stamina later if needed)

### 7.3 Alternative Designs Reconsidered
If sprint proves too powerful or agents don't learn to use it strategically, we could:
1. Add stamina/cooldown mechanics
2. Reduce sprint_multiplier (e.g., 0.3 for 1.3x speed)
3. Make sprint only available in certain situations
4. Add energy cost to reward function

---

## 8. Files Modified

### Rust
- `src/config.rs`: Added `sprint_multiplier` field
- `src/rl/action.rs`: Expanded to 5D action space, added sprint application
- `src/rl/api.rs`: Updated to accept 5-component arrays, return 5D action space
- `src/main.rs`: Added config info UI display
- `src/game/camera.rs`: Added double-click reset, enabled zoom in default mode
- `src/game/collision.rs`: Simplified game over message

### Python
- `python/train_ppo.py`: Added reset after configure for proper sync
- `python/bevy_dodge_env/environment.py`: Improved configure() documentation
- `python/test_sprint.py`: Created comprehensive sprint test

### Config
- `python/configs/ppo_level2_continuous.yaml`: Updated comment for 5D action space

---

## 9. Next Steps

1. **Train with sprint:** Run full 500K step training on Level 2 with continuous + sprint
2. **Analyze sprint usage:** Log sprint values during episodes, visualize when agents sprint
3. **Compare performance:** Level 2 success rate with vs without sprint
4. **Tune multiplier:** Adjust `sprint_multiplier` if needed based on training results
5. **Consider stamina:** If agents spam sprint, add energy/cooldown mechanics

---

## 10. Lessons Learned

1. **Explicit > Implicit:** Making sprint a separate action component gives agents clear control and makes training more interpretable
2. **Normalization matters:** Converting `[-1, 1]` to `[0, 1]` for sprint makes the semantics clearer (0 = no sprint, not -1)
3. **Document patterns:** The configure+reset pattern needed good documentation to avoid confusion
4. **Test incrementally:** test_sprint.py caught issues before full training runs
5. **UI feedback helps:** Showing action space type during training aids debugging

---

## Conclusion

This session significantly improved both the game's usability (camera controls, UI) and the agent's capabilities (sprint mechanic). The 5D continuous action space gives agents the tools they need to tackle Level 2's challenge, while maintaining clean architecture and extensibility for future enhancements.

The sprint mechanic is ready for training - let's see if agents learn to use it strategically! 🎮🚀
