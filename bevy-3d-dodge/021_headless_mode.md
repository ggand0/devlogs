# Devlog 021: Headless Mode for Faster Training

## Date: 2025-12-20

## Objective

Add headless mode to run Bevy without a window for faster RL training.

## Motivation

Training was limited to ~23 FPS due to vsync and rendering overhead. Headless mode removes the display bottleneck for significantly faster training iterations.

## Implementation

### CLI Flags

```
--headless     Run without window (for training)
--port <PORT>  API server port (default: 8000)
--fps <FPS>    Target tick rate in headless mode (default: 120)
```

### Architecture

Used Bevy's official headless pattern:

```rust
App::new()
    .add_plugins(MinimalPlugins.set(ScheduleRunnerPlugin::run_loop(
        Duration::from_secs_f64(1.0 / fps)
    )))
```

### Changes

| File | Change |
|------|--------|
| `Cargo.toml` | Added `clap` for CLI parsing |
| `src/main.rs` | CLI args, `run_headless()`, `run_windowed()` functions |
| `src/game/mod.rs` | Added `HeadlessGamePlugin` |
| `src/game/player.rs` | Added `HeadlessPlayerPlugin`, `spawn_player_headless` |
| `src/game/projectile.rs` | Added `HeadlessProjectilePlugin`, `spawn_projectiles_headless` |
| `src/game/collision.rs` | Added `HeadlessCollisionPlugin`, `handle_collisions_headless` |

### Key Design Decisions

1. **Separate plugins**: Created headless variants of game plugins instead of conditional logic within existing plugins. Cleaner separation.

2. **No material queries**: Headless systems don't query `Assets<StandardMaterial>` or `MeshMaterial3d` since these don't exist without rendering.

3. **Configurable FPS**: The `--fps` flag controls the tick rate. Higher values = faster training (limited by CPU).

## Usage

```bash
# Start headless game
cargo run --release -- --headless --port 8000 --fps 500

# In another terminal, run training
uv run python/train_sac.py --config python/configs/sac_level2_basic3d.yaml
```

## Verification

Tested API endpoints in headless mode:

```bash
$ curl http://127.0.0.1:8000/observation_space
{"shape":[65],"dtype":"float32","low":-100.0,"high":100.0}

$ curl http://127.0.0.1:8000/action_space
{"type":"Box","shape":[5],"low":-1.0,"high":1.0}
```

## Expected Speedup

| Mode | FPS | Notes |
|------|-----|-------|
| Windowed | ~23 | Vsync-limited |
| Headless | 100-500+ | CPU-limited |

Actual speedup depends on CPU and `--fps` setting. Training that took 12 hours could potentially complete in 2-3 hours.

## Files

- Commit: `ae38c06`
- Branch: `feat/headless-mode`

## Next Steps

- Benchmark actual training speedup
- Consider auto-starting Bevy from Python training scripts
- Test with different `--fps` values to find optimal setting
