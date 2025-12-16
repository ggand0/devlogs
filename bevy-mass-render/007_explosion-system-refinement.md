# Explosion System Refinement - Sprite Sheet Implementation

**Date:** June 15, 2025  
**Files Modified:** `src/explosion_shader.rs`, `src/objective.rs`, `assets/shaders/explosion.wgsl`, `Cargo.toml`  
**Session Focus:** Fixing shadow artifacts and overlapping explosion issues in sprite sheet-based explosion system

---

## Context

Working on a 3D RTS game inspired by Star Wars: Empire at War, featuring 10,000 unit battle simulations with modular architecture. The game includes formation, movement, combat, and objective systems with teams of 5,000 battle droids competing to destroy opponent's command uplink towers.

### Previous Development History

The explosion system went through several iterations:

1. **Initial Attempt:** Procedural explosion generation - looked "super ugly"
2. **First Sprite Attempt:** Smoke-only sprite sheet with normal maps - "didn't work well"
3. **Current Approach:** Self-contained 5x5 flipbook sprite sheet with complete explosion lifecycle (bright core → expanding fireball → trailing smoke) baked into frames

---

## Problems Identified

At the start of this session, two main issues were reported:

### Issue 1: Persistent Shadows
Despite previous attempts to disable shadows, explosions were still casting/receiving shadows, creating unwanted artifacts on the sprite sheet quads.

### Issue 2: Overlapping Explosion Effects
Multiple explosion instances were rendering at the same time with decreasing transparency, making it difficult to observe a single complete explosion animation. This was caused by:
- Automatic smoke spawning for large explosions
- Phase transition system creating "three explosions" effect
- Debug keys spawning 4 explosions at once

---

## Solutions Implemented

### 1. Shadow Removal

**Root Cause:** Even though materials had `unlit: true`, the `DirectionalLight` in the scene had `shadows_enabled: true`, causing explosions to participate in shadow casting/receiving.

**Fix Applied:**
- Added imports: `NotShadowCaster` and `NotShadowReceiver` from `bevy::pbr`
- Applied these components to all explosion entity spawning:
  - `spawn_animated_sprite_explosion()` (StandardMaterial-based)
  - `spawn_custom_shader_explosion()` (Custom shader-based)
  - `spawn_debug_explosion_effect()` (Debug colored quads)
  - `spawn_explosion_effect()` (Backward compatibility)
  - Simple test explosions (I key)

**Code Example:**
```rust
commands.spawn((
    PbrBundle { /* ... */ },
    ExplosionTimer { /* ... */ },
    SpriteExplosion { /* ... */ },
    NotShadowCaster,    // Prevents casting shadows
    NotShadowReceiver,  // Prevents receiving shadows
    Name::new("AnimatedSpriteExplosion"),
));
```

### 2. Overlapping Explosions Fix

**Phase Transition System Removal:**

The system was transitioning through three phases (Initial → Secondary → Smoke) with different alpha and emissive values, causing the appearance of "three explosions with decreasing transparency."

**Before:**
```rust
let (alpha_fade, emissive_strength) = match (&sprite_explosion.current_phase, progress) {
    (ExplosionPhase::Initial, p) => {
        (sprite_explosion.fade_alpha, 3.0 * (1.0 - p * 0.3))
    },
    (ExplosionPhase::Secondary, p) => {
        (sprite_explosion.fade_alpha * 0.9, 2.0 * (1.0 - p * 0.5))
    },
    (ExplosionPhase::Smoke, p) => {
        (sprite_explosion.fade_alpha * (1.0 - p * 0.7), 0.5 * (1.0 - p))
    },
};
```

**After:**
```rust
// Maintain full intensity and only fade near the end
let alpha_fade = if progress > 0.9 {
    sprite_explosion.fade_alpha * (1.0 - (progress - 0.9) * 10.0) // Fade only in last 10%
} else {
    sprite_explosion.fade_alpha // Full alpha for 90% of animation
};

let emissive_strength = if progress > 0.9 {
    2.0 * (1.0 - (progress - 0.9) * 5.0) // Reduce emissive only near end
} else {
    2.0 // Full emissive strength for most of animation
};
```

**Rationale:** The self-contained sprite sheet already has all explosion phases baked into the frames, so artificial phase transitions were redundant and created visual artifacts.

**Automatic Smoke Spawning Removal:**

Previously, large explosions (radius > 5.0) would automatically spawn a secondary smoke explosion entity, doubling the visual complexity.

**Removed:**
```rust
// Add smoke for large explosions
if radius > 5.0 {
    // ... spawned additional smoke sprite entity
}
```

**Replaced with:**
```rust
// Removed automatic smoke spawning to prevent overlapping explosions
```

### 3. Debug Control Simplification

**Previous Behavior:**
- T key: Spawned 4 debug colored explosions at different positions
- Y key: Spawned 4 animated sprite explosions
- U key: Spawned 4 custom shader explosions
- I key: Spawned 4 simple test explosions

**Updated Behavior:**
- **T key:** Completely disabled (was spawning 4 pink debug cubes + explosions in `objective.rs`)
- **Y key:** Single animated sprite explosion at center (StandardMaterial approach)
- **U key:** Single custom shader explosion at center (Primary method) ✨
- **I key:** Single simple solid color explosion at center (positioning tests)

**Additional Cleanup:**

Found and removed T key handler in `src/objective.rs` that was spawning 4 bright magenta debug cubes:

```rust
// T key functionality removed - debug cubes no longer needed
```

This was creating `PbrBundle` entities with 10x10x10 magenta cubes that were no longer useful after the sprite sheet system was working.

---

## Technical Architecture

### Custom Shader Material System

**ExplosionMaterial Definition:**
```rust
#[derive(Asset, TypePath, AsBindGroup, Debug, Clone)]
pub struct ExplosionMaterial {
    #[uniform(0)]
    pub frame_data: Vec4, // x: frame_x, y: frame_y, z: grid_size, w: alpha
    #[uniform(1)]
    pub color_data: Vec4, // RGB: base color, A: emissive strength
    #[texture(2, dimension = "2d")]
    #[sampler(3)]
    pub sprite_texture: Handle<Image>,
}
```

**WGSL Shader (explosion.wgsl):**

Simplified from 332 lines of complex procedural noise functions to 41 lines of straightforward frame-based UV sampling:

```wgsl
// Calculate UV coordinates for the specific frame in 5x5 grid
let frame_size = 1.0 / grid_size;
let frame_offset = vec2<f32>(frame_x * frame_size, frame_y * frame_size);

// Scale UV coordinates to fit within the frame
let frame_uv = in.uv * frame_size + frame_offset;

// Sample the flipbook texture directly
let sprite_sample = textureSample(sprite_texture, sprite_sampler, frame_uv);
```

**Key Benefits:**
- Direct texture sampling (no complex noise calculations)
- Frame-by-frame animation control from Rust side
- Simple emissive enhancement for bloom compatibility
- Efficient GPU processing

### Asset Configuration

**Texture Format:**
- Added TGA support to `Cargo.toml`: `bevy = { version = "0.14", features = ["dynamic_linking", "wav", "tga"] }`
- Using `Explosion02HD_5x5.tga` sprite sheet

**Sprite Grid:**
- Grid Size: 5x5 (25 frames total)
- Frame Duration: `(duration * 0.8) / 25.0` - faster animation
- Animation Duration: 80% of specified duration (faster playback)

### Component Architecture

**Two Component Types:**

1. **SpriteExplosion** (StandardMaterial-based):
   - Used for simple animated sprites with StandardMaterial
   - Billboard rotation for camera-facing quads
   - Frame-based animation
   
2. **CustomShaderExplosion** (Custom material-based):
   - Uses ExplosionMaterial with custom shader
   - Frame data passed via uniforms
   - More control over rendering

Both share similar structure:
```rust
pub struct CustomShaderExplosion {
    pub explosion_type: ExplosionType,
    pub current_phase: ExplosionPhase,  // Now unused, kept for compatibility
    pub frame_count: usize,             // 25 frames
    pub current_frame: usize,
    pub frame_duration: f32,
    pub frame_timer: f32,
    pub scale: f32,
    pub fade_alpha: f32,
}
```

### Animation System

**Frame Update Logic:**
```rust
sprite_explosion.frame_timer += time.delta_seconds();
if sprite_explosion.frame_timer >= sprite_explosion.frame_duration {
    sprite_explosion.frame_timer = 0.0;
    sprite_explosion.current_frame += 1;
    
    if sprite_explosion.current_frame >= sprite_explosion.frame_count {
        sprite_explosion.current_frame = sprite_explosion.frame_count - 1; // Hold on last frame
    }
}
```

**Billboard Effect:**
Explosions always face the camera using quaternion-based rotation:

```rust
let to_camera = (camera_position - explosion_position).normalize();
let forward = to_camera;
let right = Vec3::Y.cross(forward).normalize();
let up = forward.cross(right).normalize();
transform.rotation = Quat::from_mat3(&Mat3::from_cols(right, up, forward));
```

---

## Results

### Before
- ❌ Explosions had visible shadows on the ground
- ❌ Multiple overlapping explosion effects made animation hard to see
- ❌ "Three explosions" effect with decreasing transparency
- ❌ Debug keys spawned 4 explosions simultaneously
- ❌ Pink debug cubes appearing when pressing T

### After
- ✅ Shadow-free explosions (no cast, no receive)
- ✅ Single, clean explosion per trigger
- ✅ Full intensity maintained for 90% of animation
- ✅ Simple debug controls (one explosion per key)
- ✅ Clean sprite sheet playback from frame 0 to 24

### Performance Characteristics
- **Shader Complexity:** Reduced from 332 lines (procedural noise) to 41 lines (texture sampling)
- **GPU Load:** Minimal - simple texture lookups vs complex noise functions
- **Entity Count:** Reduced from 2+ entities per explosion (main + smoke) to 1 entity
- **Animation Quality:** Consistent frame timing, smooth playback

---

## Debug Controls Summary

| Key | Function | Description |
|-----|----------|-------------|
| **Y** | StandardMaterial Explosion | Single animated sprite using StandardMaterial |
| **U** | Custom Shader Explosion ⭐ | Primary method - uses custom shader material |
| **I** | Simple Test Explosion | Solid color quad for positioning tests |
| **E** | Tower Destruction | Triggers tower explosion for gameplay testing |
| **R** | Bright Test Explosions | Creates maximum brightness explosions |

**Note:** T key completely removed (previously spawned debug cubes)

---

## Code Quality Improvements

### Removed Dead Code
- Disabled but kept phase transition logic (commented out)
- Removed automatic smoke spawning branches
- Cleaned up debug test functions

### Maintained Backward Compatibility
- `spawn_explosion_effect()` function still exists for old code
- `ExplosionPhase` enum kept but no longer actively used
- Component structure unchanged (easier migration)

### Added Documentation
- Clear comments explaining why systems were disabled
- Inline explanations for shader uniforms
- Updated function documentation

---

## Commit Summary

```
feat: implement sprite sheet explosion system with custom shader

- Replace complex procedural explosion shader with simple 5x5 flipbook animation
- Add custom ExplosionMaterial with frame-based UV sampling in explosion.wgsl
- Switch from 8x8 to 5x5 sprite grid (25 frames) with faster animation timing
- Remove shadow casting/receiving from explosions (NotShadowCaster/NotShadowReceiver)
- Disable phase transitions to prevent "three explosions" effect - use self-contained sprite sheet
- Remove automatic smoke spawning to eliminate overlapping explosions
- Simplify debug controls: single explosion per key (Y/U/I), remove T key debug cubes
- Add TGA texture support in Cargo.toml for sprite sheet assets
- Keep full intensity for 90% of animation, only fade in final 10%

Results in clean, shadow-free, single-shot sprite sheet explosions with proper frame animation.
```

---

## Future Considerations

From the draft message found in chat UI:
> "Thanks, this works as a baseline of explosion effects. I'd like to explore more on how to improve this in the following order:
> 1. particle"

### Potential Next Steps

**1. Particle System Integration:**
- Add GPU-based particle systems for debris/sparks
- Integrate with existing sprite sheet for combined effects
- Use Bevy's particle system or custom compute shaders

**2. Audio Integration:**
- Explosion sound effects synchronized with visual
- Randomized variations for different explosion types
- 3D spatial audio with distance falloff

**3. Screen-Space Effects:**
- Camera shake on explosions
- Screen-space distortion (heat waves)
- Post-processing bloom enhancement

**4. Gameplay Integration:**
- Damage radius visualization
- Unit ragdoll/death animations
- Debris physics with Bevy's physics engine

**5. Optimization:**
- Explosion pooling system for 10,000 unit battles
- LOD system (distant explosions use simpler sprites)
- Culling for off-screen explosions

---

## Technical Decisions & Rationale

### Why Sprite Sheets over Procedural?
1. **Artistic Control:** Easier to achieve specific visual style
2. **Performance:** Texture sampling is faster than complex noise functions
3. **Consistency:** Every explosion looks the same (important for RTS readability)
4. **Iteration Speed:** Artists can update assets without shader changes

### Why Custom Shader over StandardMaterial?
1. **Frame Control:** Direct control over which frame displays via uniforms
2. **Performance:** Avoid material cloning per frame update
3. **Extensibility:** Can add effects (distortion, color grading) later
4. **Emissive Control:** Better bloom compatibility with custom emissive math

### Why 5x5 Grid over 8x8?
1. **Quality:** Fewer frames = higher resolution per frame for same texture size
2. **Duration:** 25 frames @ 60fps = ~0.4s explosion (perfect for RTS)
3. **Performance:** Less frame updates needed
4. **Artist Feedback:** Original asset was provided as 5x5

---

## Known Limitations

1. **Static Sprite:** Billboarded quads don't rotate with viewer movement mid-animation
2. **No Particle Effects:** Pure sprite sheet approach lacks fine detail particles
3. **No Physics:** Explosions don't interact with environment or cast light
4. **Uniform Scaling:** All explosions scale uniformly (no stretching/deformation)
5. **No Audio:** Visual only - audio system not integrated yet

---

## References

- **Bevy Version:** 0.14
- **Asset Path:** `assets/textures/Explosion02HD_5x5.tga`
- **Shader Path:** `assets/shaders/explosion.wgsl`
- **Main Components:** `SpriteExplosion`, `CustomShaderExplosion`, `ExplosionMaterial`
- **Main Systems:** `animate_sprite_explosions`, `animate_custom_shader_explosions`

---

## Testing

To test the explosion system:

1. **Run the game:** `cargo run`
2. **Press U key:** Spawns a single custom shader explosion at battlefield center
3. **Press Y key:** Spawns StandardMaterial version for comparison
4. **Press I key:** Spawns simple solid color for positioning verification

**Expected Results:**
- Single explosion appears at center
- Animation plays through all 25 frames smoothly
- Full intensity maintained for ~90% of duration
- Fades out in final ~10%
- No shadows visible on ground
- Billboard faces camera at all angles

---

**End of Log**

