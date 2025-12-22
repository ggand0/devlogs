# GPU Fireball Velocity Direction Bug - Handoff

**Date**: December 22, 2025
**Status**: BLOCKED - particles go INWARD instead of OUTWARD
**Branch**: feat/gpu-particle-perf

## Problem

GPU fireball particles go **INWARD** toward the spawn center instead of **OUTWARD**. This is a velocity direction issue, NOT an orientation/mirroring issue.

## Root Cause

bevy_hanabi's `ExprWriter` re-evaluates `rand()` calls on every expression use. Cloning an expression does NOT preserve its computed value - each use generates a separate `frand()` call in the generated WGSL shader.

Example of the problem:
```rust
let dir = rx.vec3(ry, rz).normalized();  // Uses rand() internally
let pos = dir.clone() * radius;          // dir.clone() generates NEW random values
let vel = dir.clone() * speed;           // dir.clone() generates DIFFERENT random values
// Result: pos and vel point in completely different directions!
```

## What Was Tried (All Failed)

1. **Spherical coordinates with clone()** - Failed because clone re-evaluates random values
2. **SetPositionSphereModifier + SetVelocitySphereModifier** - Failed, still inward
3. **Reading POSITION attribute after setting it** - Failed, position not set when velocity expression builds
4. **Computing velocity = position * scalar** - Failed, different random values
5. **Negating velocity** - Failed, position and velocity already have different random directions
6. **Storing direction in F32_0/1/2 attributes then reading back** - Current state, STILL FAILS

## Current Code (particles.rs lines 996-1075)

```rust
let writer_fireball = ExprWriter::new();

// Store direction components in F32_0/1/2, then read back for velocity
let rx = writer_fireball.rand(ScalarType::Float) * writer_fireball.lit(2.0) - writer_fireball.lit(1.0);
let ry = writer_fireball.rand(ScalarType::Float); // [0,1] for Y >= 0
let rz = writer_fireball.rand(ScalarType::Float) * writer_fireball.lit(2.0) - writer_fireball.lit(1.0);
let dir = rx.vec3(ry, rz).normalized();
let dir_x = dir.clone().x();
let dir_y = dir.clone().y();
let dir_z = dir.z();

// Store direction in attributes
let store_dir_x = SetAttributeModifier::new(Attribute::F32_0, dir_x.expr());
let store_dir_y = SetAttributeModifier::new(Attribute::F32_1, dir_y.expr());
let store_dir_z = SetAttributeModifier::new(Attribute::F32_2, dir_z.expr());

// Read direction back and compute position/velocity
let dx = writer_fireball.attr(Attribute::F32_0);
let dy = writer_fireball.attr(Attribute::F32_1);
let dz = writer_fireball.attr(Attribute::F32_2);
let stored_dir = dx.vec3(dy, dz);

let fb_pos = stored_dir.clone() * writer_fireball.lit(0.5);
let fb_speed = writer_fireball.lit(3.0) + writer_fireball.rand(ScalarType::Float) * writer_fireball.lit(2.0);
let fb_vel = stored_dir * fb_speed;

let fireball_init_pos = SetAttributeModifier::new(Attribute::POSITION, fb_pos.expr());
let fireball_init_vel = SetAttributeModifier::new(Attribute::VELOCITY, fb_vel.expr());
```

Init chain order:
```rust
.init(store_dir_x)
.init(store_dir_y)
.init(store_dir_z)
.init(fireball_init_pos)
.init(fireball_init_vel)
```

**Why this SHOULD work but DOESN'T**:
- We store dir.x/y/z in F32_0/1/2 attributes FIRST
- Then we read those attributes back to compute position and velocity
- Both position and velocity should use the SAME stored direction
- But particles still go inward

## Possible Issues to Investigate

1. **The clone() on dir before .x()/.y()/.z() might re-evaluate**
   - `dir.clone().x()` might generate new random values
   - Try storing the scalar components directly without intermediate vec3

2. **Init modifier execution order**
   - Are F32_0/1/2 actually set before POSITION/VELOCITY are computed?
   - Check bevy_hanabi init modifier execution order

3. **Generated WGSL inspection**
   - Look at actual shader code being generated
   - Verify frand() calls aren't duplicated

4. **bevy_hanabi examples**
   - Check `firework.rs`, `force_field.rs` for outward velocity patterns
   - How do official examples handle position-dependent velocity?

5. **SimulationSpace::Local**
   - Could be inverting coordinates somehow
   - Try SimulationSpace::Global to test

6. **Custom modifier approach**
   - Write a single modifier that sets both POSITION and VELOCITY atomically
   - Bypass the expression system entirely for this

## Key Files

- `/home/gota/ggando/gamedev/bevy-mass-render/src/particles.rs` - GPU fireball effect (lines 996-1092)
- `/home/gota/ggando/gamedev/bevy_hanabi/` - Local fork of bevy_hanabi
- `/home/gota/ggando/gamedev/bevy_hanabi/examples/firework.rs` - May have outward pattern
- `/home/gota/ggando/gamedev/bevy_hanabi/src/modifier/` - Modifier implementations

## Goal

Particles spawn on hemisphere surface (radius 0.5, Y >= 0) and move **OUTWARD** from center:
- velocity direction = position direction (normalized)
- velocity magnitude = 3.0 to 5.0 (random)

## NOT the Problem

- Orientation/mirroring (that's a separate resolved issue - use X=velocity + rotate texture)
- Texture direction (solved by rotating texture asset 90°)
- The math concept (it's simple: vel = normalize(pos) * speed)

## The Problem IS

Getting bevy_hanabi's ExprWriter to use the SAME random direction for both position and velocity computation.
