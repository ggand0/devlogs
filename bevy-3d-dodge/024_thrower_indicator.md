# Devlog 024: Thrower Indicator Feature

## Date: 2025-12-21

## Objective

Add anticipation mechanics to the environment by spawning a visible "thrower" indicator before each projectile. This gives the agent predictive information about incoming throws.

## Implementation

### New Components

**ThrowerIndicator** (`src/game/projectile.rs`):
```rust
pub struct ThrowerIndicator {
    pub spawn_timer: Timer,      // Countdown until throw
    pub spawn_position: Vec3,    // Where projectile will spawn
    pub spawn_velocity: Vec3,    // Pre-computed velocity
}
```

### Observation Modes

**ObservationMode** enum (`src/config.rs`):
- `Standard` - 65-dim (backward compatible with existing models)
- `WithThrowerIndicator` - 69-dim (adds thrower info)

**Extended Observation (69-dim):**
```
[0-4]:   Player position (x,y,z) + velocity (vx,vy)
[5-64]:  10 projectiles × 6 values (pos + vel)
[65-67]: Thrower position (x, y, z)
[68]:    Time until throw (0.0-1.0 normalized)
```

### Configuration

New `GameConfig` fields:
- `observation_mode: ObservationMode` - default: `Standard`
- `thrower_delay_seconds: f32` - default: `1.0`

### API Updates

**Configure endpoint** (`/configure`):
```json
{
  "observation_mode": "with_thrower",
  "thrower_delay_seconds": 1.0
}
```

**Observation space** (`/observation_space`):
- Returns dynamic shape based on `observation_mode`
- Standard: `{"shape": [65], ...}`
- WithThrower: `{"shape": [69], ...}`

## Visual Indicator

- Orange glowing sphere (smaller than projectile)
- Appears at future spawn position
- Visible for `thrower_delay_seconds` before projectile spawns
- Despawns when projectile launches

## Usage

**Default (backward compatible):**
```bash
cargo run --release
# Observation space: 65-dim
```

**With thrower indicator:**
```bash
# Terminal 1: Start game
cargo run --release

# Terminal 2: Configure
curl -X POST http://127.0.0.1:8000/configure \
  -H "Content-Type: application/json" \
  -d '{"observation_mode": "with_thrower", "thrower_delay_seconds": 1.0}'

# Verify
curl http://127.0.0.1:8000/observation_space
# Returns: {"shape":[69],...}
```

## Files Modified

| File | Changes |
|------|---------|
| `src/config.rs` | Added `ObservationMode` enum, new config fields |
| `src/game/projectile.rs` | Added `ThrowerIndicator` component, spawn/process systems |
| `src/rl/observation.rs` | Added `extract_observation_with_mode()`, 69-dim support |
| `src/rl/api.rs` | Updated `/observation_space`, `/configure` endpoints |
| `src/main.rs` | Updated command handlers, observation extraction |

## Backward Compatibility

- Default `observation_mode: Standard` maintains 65-dim observation
- Existing trained models work without changes
- New training can opt-in to 69-dim with `observation_mode: with_thrower`

## Next Steps

1. Train SAC with thrower indicator mode
2. Compare stability vs standard mode
3. Test if agent learns to anticipate throws
4. Consider multi-thrower support for future adversarial training
