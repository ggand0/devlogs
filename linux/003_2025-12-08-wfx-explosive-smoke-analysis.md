# WFX_ExplosiveSmoke_Big Effect Analysis

**Date**: 2025-12-08
**Purpose**: Document the detailed analysis of Unity WarFX particle effect for Bevy implementation

## Overview

Analyzed the Unity WarFX `WFX_ExplosiveSmoke_Big2.prefab` explosion effect to extract all parameters needed for recreation in Bevy using the Hanabi particle system.

## Effect Composition

The effect consists of **6 particle emitters** working together:

| Emitter | Role | Key Characteristic |
|---------|------|--------------------|
| **Explosion** | Main fire/smoke cloud | Random start color (orange→brown gradient) |
| **Smoke** | Lingering smoke trail | 0.5s delayed start, constant gray |
| **Embedded Glow** | Central flash/fireball | Grow-then-shrink size curve, only 2 particles |
| **Glow Sparkles** | Wide glowing embers | High speed (24), strong gravity (4) |
| **Dot Sparkles** | Dense spark shower | 75 particles, fastest (12-24 speed) |
| **Dot Sparkles Vertical** | Rising sparks | Mesh shape, zero gravity, floats upward |

## Key Technical Discoveries

### 1. Explosion Emitter - Random Start Color Gradient

The most important discovery was understanding how a single emitter creates both flame AND smoke visuals:

```yaml
startColor:
  minMaxState: 1          # Gradient mode (NOT constant!)
  minColor: {r: 0.349, g: 0.212, b: 0.110, a: 1}  # Dark brown
  maxColor: {r: 1.000, g: 0.816, b: 0.408, a: 1}  # Orange
```

**`minMaxState: 1`** means Unity randomly samples between min and max colors at spawn time. This creates:
- Orange particles → appear as flames
- Brown particles → appear as smoke

Both come from the SAME emitter with identical behavior, just different starting colors.

### 2. UV Scrolling Shader

The Explosion and Smoke emitters use `WFX_M_Explosion Multiply` material with:
- **Texture**: `WFX_T_Explosion_Light.png` (4x4 sprite sheet)
- **UV Animation**: Scrolls through 16 frames for animated smoke

### 3. Blend Modes

Two distinct blend modes used:
- **Multiply** (`Blend DstColor SrcAlpha`): Explosion + Smoke emitters - darkens background
- **Additive**: All sparkle/glow emitters - brightens, creates glow

### 4. Grow-Then-Shrink Size Curve

The Embedded Glow emitter has a unique 4-keyframe size curve:
```
0%   → 0.3  (start small)
20%  → 0.7  (grow)
60%  → 1.0  (peak)
100% → 0.5  (shrink back)
```

This simulates fireball expansion then contraction.

### 5. Vertical Sparkles - Mesh Emission

The "Dot Sparkles Vertical" emitter uses **Mesh** shape type (unique among all emitters) with a very narrow 1.6° arc and zero gravity, creating sparks that float upward rather than arc down.

## Emitter Timing

```
Time:    0.0s          0.5s          1.0s          3.0s          4.0s
         |-------------|-------------|-------------|-------------|
Embedded Glow:  ▓░░                    (0.7s lifetime)
Explosion:      ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░ (3-4s)
Glow Sparkles:  ▓░░                    (0.25-0.35s)
Dot Sparkles:   ▓▓░░░                  (0.2-0.3s)
Dot Vertical:   ▓░░░                   (0.1-0.3s)
Smoke:          ----▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░ (0.5s delay, 3-4s)
```

## Documentation Created

All detailed emitter documentation stored in `data/wfx_explosivesmoke_big/`:

1. `EXPLOSION_EMITTER_DETAILS.md` - Main fire/smoke with random color gradient
2. `SMOKE_EMITTER_DETAILS.md` - Delayed smoke trail emitter
3. `EMBEDDED_GLOW_EMITTER_DETAILS.md` - Central flash with grow-shrink curve
4. `GLOW_SPARKLES_EMITTER_DETAILS.md` - Wide glowing ember particles
5. `DOT_SPARKLES_EMITTER_DETAILS.md` - Dense omnidirectional spark spray
6. `DOT_SPARKLES_VERTICAL_EMITTER_DETAILS.md` - Rising floating sparks

Each document includes:
- Complete parameter tables
- Color/alpha gradient keyframes
- Size over lifetime curves
- Material/texture references
- Bevy Hanabi implementation notes with code snippets
- Visual timelines

## Raw Data Sources

- `data/wfx_explosivesmoke_big/UNITY_PREFAB_COMPLETE.md` - Full extracted prefab data
- `data/TempUnityProject/Assets/JMO Assets/WarFX/_Effects/Explosions/WFX_ExplosiveSmoke_Big2.prefab` - Original Unity YAML

## Implementation Status

- [x] Explosion emitter analysis complete
- [x] Smoke emitter analysis complete
- [x] Embedded Glow analysis complete
- [x] Glow Sparkles analysis complete
- [x] Dot Sparkles analysis complete
- [x] Dot Sparkles Vertical analysis complete
- [ ] Bevy Hanabi implementation (separate project)

## Lessons Learned

1. **Unity's `minMaxState` is critical**:
   - `0` = constant color
   - `1` = random between min/max (gradient)
   - Initial extraction scripts missed this, leading to incorrect color analysis

2. **Shape module can be disabled**: The Embedded Glow emitter has shape disabled, spawning particles at exact origin

3. **Mesh emission for directional control**: Vertical sparkles use mesh shape to bias emission direction upward

4. **Multiply blend for smoke**: Unlike typical additive particles, smoke uses multiply blend to darken the scene

---

## Retrospective

### The Problem We Were Trying to Solve

Reproduce a Unity WarFX explosion effect in Bevy using the Hanabi particle system. The challenge: Unity's particle system has decades of features and tooling, while we needed to reverse-engineer the exact parameters from serialized YAML data and translate them to a completely different engine.

### What We Struggled With

#### 1. The "Missing Flame" Mystery

The biggest struggle was understanding why the Explosion emitter visually produced both **flame** and **smoke** particles. Initial analysis showed:
- Color Over Lifetime gradient: white → gray (no orange!)
- Material: smoke texture

This didn't match what we saw in Unity - there were clearly orange flame particles. We went in circles trying to find a "hidden" second emitter or some other mechanism.

#### 2. Incomplete Initial Extraction

The first extraction scripts focused on obvious fields like `ColorModule` (Color Over Lifetime) but missed the `InitialModule.startColor` field entirely. This was buried deep in the prefab YAML and used a non-obvious `minMaxState` enum.

#### 3. Unity YAML Complexity

The prefab file is ~25,000 lines of deeply nested YAML with:
- Internal file references (`fileID: 19800000`)
- GUID references to external assets
- Implicit defaults (missing fields = default values)
- Multiple serialization formats for the same concept

### Mistakes Made

1. **Assumed Color Over Lifetime was the only color source** - Missed that `startColor` can ALSO be a gradient, not just a constant

2. **Trusted summarized data over raw data** - Initial `UNITY_PREFAB_COMPLETE.md` was a summary that omitted the critical `minMaxState` field. Should have read raw prefab earlier.

3. **Didn't question the obvious** - When user said "I see flame AND smoke from one emitter", I initially pushed back instead of digging deeper

### The Breakthrough

User insisted they saw both flame and smoke when isolating ONLY the Explosion emitter in Unity. This forced a deeper dive into the raw prefab YAML where we found:

```yaml
startColor:
  serializedVersion: 2
  minMaxState: 1    # <-- THE KEY! 1 = Random Between Two Colors
  minColor: {r: 0.349, g: 0.212, b: 0.110, a: 1}  # Brown
  maxColor: {r: 1.000, g: 0.816, b: 0.408, a: 1}  # Orange
```

`minMaxState: 1` means Unity randomly picks a color between min and max for each particle at spawn. Orange particles look like flames, brown particles look like smoke - same emitter, same behavior, different starting colors.

### Problems Solved

| Problem | Solution |
|---------|----------|
| Flame + smoke from one emitter | `startColor` gradient with `minMaxState: 1` |
| Animated smoke texture | UV scrolling through 4x4 sprite sheet |
| Central flash that expands then fades | Grow-then-shrink size curve (4 keyframes) |
| Rising vs falling sparks | Zero gravity + mesh emission for vertical, gravity for others |
| Smoke appearing after explosion | 0.5 second start delay on Smoke emitter |

### What Worked Well

1. **Creating detailed per-emitter documentation** - Breaking down each emitter into its own markdown made the complexity manageable

2. **Including Bevy code snippets** - Made handoff to gamedev AI agent smoother

3. **Visual timelines** - ASCII diagrams showing particle lifetime and timing helped understand the choreography

4. **User testing in Unity** - User isolating individual emitters to verify behavior was crucial for the breakthrough

### If I Did This Again

1. **Read raw prefab YAML first** - Don't trust summaries or extraction scripts until verified
2. **Document ALL `minMaxState` values** - This enum appears in many places (color, size, speed, etc.)
3. **Ask user to test isolated emitters earlier** - Their empirical observation broke the logjam
4. **Create a Unity field glossary** - Quick reference for `minMaxState`, shape types, blend modes, etc.

### Time Spent

- ~2 sessions of analysis and documentation
- Most time lost: chasing the wrong hypothesis about flame colors (Color Over Lifetime instead of Start Color)
- Fastest part: documenting sparkle emitters once the pattern was understood
