# Phase 1 Completion: 3D Dodge Game MVP

**Date:** 2025-11-16
**Status:** Complete

## Overview

Successfully implemented a fully functional 3D projectile dodging game using Bevy 0.14 as the foundation for RL agent training. The game provides a clean, visually pleasant environment with all core mechanics needed for reinforcement learning experimentation.

## Implemented Features

### Core Game Mechanics
- **Player Movement:** WASD/Arrow key controls with normalized diagonal movement
  - Movement constrained to XY plane at fixed Z height
  - Position clamped to playable bounds (-15 to +15 in X and Y)
  - Blue capsule entity standing upright with proper rotation

- **Projectile System:** Automated spawning and movement
  - Red spheres spawning from +Y direction at configurable intervals
  - Linear movement toward -Y at configurable speed
  - Random X-axis spawn positions for variety
  - Automatic cleanup of off-screen projectiles

- **Collision Detection:** AABB-based collision system
  - Player-projectile collision triggers game over state
  - Visual feedback: player turns red on collision
  - Game freezes on collision (no further updates)

- **Game State Management:**
  - Reset functionality (R key) restores initial state
  - Despawns all active projectiles on reset
  - Resets player position, velocity, and color

### Visual Environment

- **Isaac Sim-Style Ground Plane:**
  - Light blue color (RGB: 0.45, 0.65, 0.85)
  - 50x50 unit rectangle in XY plane
  - White semi-transparent grid overlay (1-unit spacing)
  - Unlit material for consistent bright appearance

- **Coordinate System Visualization:**
  - RGB color-coded axes (Red=X, Green=Y, Blue=Z)
  - 5-unit length with arrow heads for direction indication
  - Hidden by default, shown only in debug mode (F1)

- **Lighting Setup:**
  - Directional light from elevated position
  - High ambient brightness (500.0) for clear visibility
  - Shadows disabled for cleaner aesthetic

### Camera System

- **Fixed Perspective Camera:**
  - Position: (-15, 0, 10)
  - Looking at: (0, 0, 1)
  - Up vector: Z-axis

- **Debug Mode (F1 Toggle):**
  - Middle mouse + drag: Pan camera
  - Right mouse + drag: Rotate view
  - Scroll wheel: Zoom in/out
  - U/O keys: Vertical movement
  - Arrow keys: Camera rotation
  - Shows coordinate axes when enabled

### UI Elements

- **Main Control Text:** Always visible instruction banner
- **Camera Debug Help:** Yellow text shown only when F1 is active
- **Game Over Message:** Large red text on collision

## Technical Architecture

### Project Structure
```
bevy_3d_dodge/
├── src/
│   ├── main.rs              # Entry point, scene setup, UI management
│   ├── config.rs            # Game configuration parameters
│   └── game/
│       ├── mod.rs           # Game plugin aggregator
│       ├── player.rs        # Player entity and movement logic
│       ├── projectile.rs    # Projectile spawning and movement
│       ├── camera.rs        # Camera setup and debug controls
│       └── collision.rs     # Collision detection and game state
├── run.sh                   # AMD GPU helper script
└── Cargo.toml              # Dependencies and build configuration
```

### Key Dependencies
- **bevy:** 0.14 (Game engine with ECS architecture)
- **rand:** 0.8 (Random projectile spawn positions)
- **axum, tokio:** 0.7, 1.40 (Prepared for Phase 2 HTTP API)
- **serde:** 1.0 (Serialization for future API)

### Build Optimization
- Dev builds: opt-level 1 for faster iteration
- Dependencies: opt-level 3 even in dev for Bevy performance
- Release: LTO thin, single codegen unit

## Coordinate System

**Z-up orientation:**
- +X: Forward (away from camera)
- +Y: Left (from camera view)
- +Z: Up (vertical)

**Player Movement Mapping:**
- W/Up Arrow: +X (forward)
- S/Down Arrow: -X (backward)
- A/Left Arrow: +Y (left)
- D/Right Arrow: -Y (right)

## Configuration Parameters

Default values in `GameConfig`:
```rust
player_speed: 5.0               // Units per second
player_start_height: 1.0        // Z position
projectile_speed: 3.0           // Units per second
projectile_spawn_interval: 2.0  // Seconds between spawns
projectile_spawn_distance: 20.0 // Y position for spawning
max_projectiles: 10             // Maximum concurrent count
```

## Platform Support

- **Primary:** AMD GPU with Vulkan (tested on 7900XTX with ROCm 6.4.3)
- **Compatible:** NVIDIA GPU, CPU-only systems
- **Helper Script:** `run.sh` sets Vulkan ICD for AMD GPUs

## Issues Resolved

1. **Ground Plane Orientation:** Changed from `Plane3d` to `Rectangle` mesh to ensure XY plane alignment
2. **Player Orientation:** Added 90° rotation around X-axis to make capsule stand upright
3. **Control Mapping:** Iteratively adjusted WASD mapping based on camera viewpoint testing
4. **Ground Color:** Adjusted to Isaac Sim-style blue with unlit + emissive material
5. **Coordinate Axes:** Initially visible, moved to debug-mode-only for cleaner default view

## Git History

- Initial commit: Added README.md
- Second commit: Implemented complete Phase 1 game with visual environment
- Third commit: Hidden coordinate axes by default, show only in debug mode

## Testing

Manual testing confirmed:
- ✅ Player movement responsive and intuitive
- ✅ Projectiles spawn and move correctly
- ✅ Collision detection accurate
- ✅ Game reset fully restores state
- ✅ Camera debug mode functional
- ✅ Visual appearance clean and pleasant
- ✅ Performance smooth on AMD 7900XTX

## Next Steps (Phase 2)

1. Define observation space (player position, velocities, projectile states)
2. Define action space (discrete 9-action or continuous 2D)
3. Implement reward function (survival time, collision penalty)
4. Create HTTP REST API with axum:
   - POST /reset - Reset environment
   - POST /step - Take action, return observation/reward/done
   - GET /info - Environment metadata
5. Test API endpoints with basic Python client

## Lessons Learned

- Bevy's coordinate system (Z-up) requires careful orientation planning
- Visual debugging tools (coordinate axes, debug camera) are essential
- Unlit materials with emissive colors provide consistent appearance
- ECS plugin architecture scales well for modular game systems
- Camera viewpoint significantly affects control mapping intuitiveness

## Resources

- [Bevy 0.14 Documentation](https://docs.rs/bevy/0.14)
- [Isaac Sim Visual Reference](https://docs.omniverse.nvidia.com/isaacsim/)
- Architecture document: `001_architecture.md`
