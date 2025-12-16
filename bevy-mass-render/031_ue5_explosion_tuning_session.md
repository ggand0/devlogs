# Devlog 031: UE5 Ground Explosion Port - Retrospective

A meta-devlog collecting retrospective notes from different work sessions on the UE5 explosion port.

---

## Session: 2024-12-10 (Smoke Billboard Rotation & Local-Space Velocity)

### Context

This was a context-continued session. The previous thread had run out of context while iterating on the F3 Smoke Cloud emitter. At the start, smoke was described as looking like "a bunch of ghosts playing around" - the basic components existed but the visual result was wrong.

### What Was Already Done (from previous session)

- `SmokeScaleOverLife` component existed (grows 1x → 3x with ease-out)
- `SmokeColorOverLife` component existed (color curve from UE5)
- `SmokePhysics` with velocity, drag, acceleration was implemented
- `loop_animation` flag had been added to `FlipbookSprite`
- Basic smoke was spawning but looked bad

### What We Fixed

#### 1. Billboard Rotation Being Overwritten

**Problem**: Each smoke particle had a random rotation (0-360°) set at spawn via `Transform::with_rotation(Quat::from_rotation_z(rotation_angle))`. But the `update_camera_facing_billboards` system was completely overwriting this every frame:

```rust
transform.rotation = Quat::from_mat3(&Mat3::from_cols(right, up, forward));
```

All smoke particles ended up with identical orientation - no variety.

**Solution**: Created `SpriteRotation { angle: f32 }` component to store the rotation separately, then apply it AFTER the billboard calculation:

```rust
let billboard_rotation = Quat::from_mat3(&Mat3::from_cols(right, up, forward));
if let Some(sprite_rot) = sprite_rotation {
    let local_rotation = Quat::from_rotation_z(sprite_rot.angle);
    transform.rotation = billboard_rotation * local_rotation;
}
```

This was a "duh" moment in hindsight - the billboarding system needs to preserve sprite-local rotation while still facing the camera.

#### 2. Camera-Local Velocity (The Big Insight)

**Problem**: Smoke was spreading on the world XZ plane (horizontal ground spread). It looked unnatural because particles moved "into" and "out of" the screen depth in ways that felt disconnected from the view.

**User provided the key insight**: UE5 uses `bLocalSpace=True` for smoke velocity. The XY velocity spread is relative to the *emitter's orientation*, which typically faces the camera. This means:
- Local X = camera's right direction (spread left/right ON SCREEN)
- Local Y = camera's up direction (spread up/down ON SCREEN)
- Local Z = camera's forward (toward/away - minimal)

**Solution**: Transform velocity from camera-local space to world space at spawn time:

```rust
let (cam_right, cam_up, cam_forward) = if let Some(cam_tf) = camera_transform {
    (cam_tf.right().as_vec3(), cam_tf.up().as_vec3(), cam_tf.forward().as_vec3())
} else {
    (Vec3::X, Vec3::Y, Vec3::NEG_Z)
};

let velocity = cam_right * local_x + cam_up * local_y + cam_forward * local_z;
```

This required threading `camera_transform: Option<&GlobalTransform>` through `spawn_ground_explosion`, `spawn_single_emitter`, and updating the debug menu system to query the camera.

### Hardest Parts

#### Understanding "Local Space" in VFX Context

The term `bLocalSpace=True` sounds like it means "relative to the parent transform" - standard game engine stuff. But in VFX context, it often means "relative to the VIEW". UE5's Niagara emitters are frequently camera-facing, so "local XY" actually means "screen XY".

This is a paradigm shift if you're thinking in world coordinates. The smoke doesn't spread "outward from explosion center" - it spreads "across your field of view". Subtle but completely changes how it reads on screen.

#### The Billboarding vs. Sprite Rotation Conflict

This felt obvious after solving it, but the initial confusion was: "why does my rotation not work?" The answer required understanding the *order of operations*:
1. Sprite has local rotation (random angle for visual variety)
2. Billboard system orients sprite to face camera
3. These must be COMPOSED, not one overwriting the other

The fix was architectural - store rotation separately, apply in correct order.

### Files Changed

- `src/ground_explosion.rs` - Added `SpriteRotation` component, updated billboard system, changed velocity calculation
- `src/objective.rs` - Updated `debug_ground_explosion_system` to pass camera transform
- `src/main.rs` - Registered new systems

### Commit

`9b813f2` - "Improve smoke effect with camera-local velocity and sprite rotation"

---

## Session: Earlier (Initial Implementation)

*[To be filled in by earlier session agents]*

---

## Session: 2024-12-09 (Tuning & Polish)

### Context

This documents a tuning session for the UE5 ground explosion port. The basic implementation was already done - all 11 emitters were spawning. This session was about making them *look right*.

### What We Worked On

#### 1. Dirt Size Scaling

The dirt emitters were too small to see. We went through multiple iterations:
- First tried 3× UE5 spec
- Then bumped to 4× (8-14m final size)

The core problem: UE5 uses centimeter units internally but displays in a way that looks much larger on screen. Direct 1:1 porting of sizes doesn't work - you need artistic judgment.

#### 2. Fireball Vertical Stretch

The main/main001 fireballs looked stretched vertically - too tall and thin. Two factors:
- **Scale curve** was 0.5→2.0 (too much growth)
- **Velocity** was 4.5-6.5 m/s (too fast upward)

Fixed by reducing scale curve to 0.5→1.3 and velocity to 3-5 m/s.

#### 3. Spark Spawn Position

After adjusting fireball size, sparks looked disconnected - spawning too high above the explosion core. They had a Y offset (`position + Vec3::Y * 3.0 * scale`) that was added earlier to match a larger fireball.

Reset to spawn at explosion core (`position`) with gravity back to 9.8 m/s².

#### 4. Wisp Emitter (Major Rework)

This was the biggest change. The wisp looked like **lingering smoke** instead of a **punchy burst**.

Problems identified:
- Double-spawning (2 sets of 3 = 6 particles)
- 2× scale modifier making them huge
- Long lifetime (2-9 seconds)
- Two-phase velocity system (complex up-then-down)
- Alpha curve starting at 3.0, dropping to 1.0 at t=0.1

The user wanted it to behave more like dirt001 - launched upward, then gravity pulls it down in an arc.

Changes made:
- Single set of 3 particles (disabled double-spawn)
- Scale reduced to 1.5×
- Added gravity (9.8 m/s²) - same as dirt
- Simple upward velocity (3-6 m/s) instead of two-phase
- Lifetime shortened to 1-2 seconds
- Alpha curve adjusted: 4.0→1.0 at t=0.2, then linear fade

#### 5. Debug Hotkey Remapping

The F1-F4 keys were being used for both map switching AND explosion emitters. User wanted:
- F1-F4 for map switching (Flat, RollingHills, FirebaseDelta, Debug)
- Number keys 1-9 for emitters while in P menu
- J for "group 1-6" (main, main001, dirt, dirt001, dust, wisp)
- K for full explosion

Also added on-screen debug menu UI (bottom-left, positioned above WFX debug menu at 40px from bottom).

### Hardest Parts / Lessons Learned

#### Unit Translation is Not Linear

UE5 Niagara specs give you numbers like "80-180 units" but these don't translate directly to meters. The relationship between UE5's internal units and visual appearance depends on:
- Camera FOV
- Typical viewing distance
- Surrounding particle sizes
- The specific effect's role in the composition

You can't just convert units - you have to **eyeball it**.

#### Emitter Interdependence

Changing one emitter affects how others look:
- Making fireball smaller → sparks look disconnected
- Changing spark spawn height → changes relationship to dirt
- Scaling dirt up → fireball looks relatively smaller

It's a balancing act. You tune one thing, then need to revisit others.

#### "Lingering" vs "Punchy" is About Physics + Lifetime

The wisp's problem wasn't the texture or animation - it was the **motion profile**:
- Long lifetime = lingers
- No gravity = floats unnaturally
- Two-phase velocity = confusing motion

Adding gravity and shortening lifetime transformed it from smoke to debris-like burst.

#### Alpha > 1.0 as Brightness Multiplier

UE5's trick: alpha values above 1.0 act as brightness multipliers. This is how dust/wisp get their initial "punch" - starting at 3.0 or 4.0 alpha, then dropping to normal opacity.

Had to implement this in the flipbook shader:
```wgsl
let alpha_multiplier = color_data.a;
let tinted_color = sprite_sample.rgb * color_data.rgb * alpha_multiplier;
let final_alpha = sprite_sample.a * min(alpha_multiplier, 1.0);
```

#### Debug UI Positioning Matters

When you have multiple debug menus, they'll overlap. Had to calculate exact pixel positions:
- WFX debug menu: bottom 10px, font 16px (~20px line height)
- Ground explosion menu: bottom 40px to sit just above it

### Files Changed

- `src/ground_explosion.rs` - All particle spawning and physics
- `src/terrain.rs` - Map switching hotkeys (Digit1-4 → F1-F4)
- `src/main.rs` - Register new debug UI systems

### What Was Already Working

When this session started, all emitters were implemented and spawning. The spawn functions existed, physics systems were running, flipbook animation worked. This was purely a **tuning and polish** session.
