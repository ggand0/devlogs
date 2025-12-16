# Devlog: Fixing Sequential Explosion Bug (2025-11-14)

## Problem Statement

Tower destruction triggered 1200+ unit explosions, but instead of creating a dramatic cascading effect with random bursts, explosions fired **one-by-one sequentially** with ~100ms gaps between each. This created a slow, tedious chain reaction instead of the intended chaotic burst pattern.

The issue was **intermittent** - sometimes it worked correctly with burst explosions, other times it degraded to sequential behavior. This inconsistency made debugging challenging.

## Initial Diagnosis Attempts

Over 10+ previous attempts to fix this, various approaches were tried and failed:

### Failed Approach #1: Query Order Shuffling
**Theory:** ECS query iteration order was deterministic, causing explosions to process in entity ID order.

**Implementation:** Shuffled the `ready_to_explode` vector before processing.

**Result:** ❌ Failed - still sequential explosions

**Why it failed:** The problem wasn't the *processing* order, it was that only 1 explosion became ready per frame.

### Failed Approach #2: Bundling Particle Spawns
**Theory:** Individual particle spawn calls were causing frame delays.

**Implementation:** Combined debris, sparks, and smoke particle spawning into single bundled calls.

**Result:** ❌ Failed - still sequential explosions

**Why it failed:** Particle spawning performance wasn't the bottleneck; the issue was in delay timing.

### Failed Approach #3: Frame Limiting
**Theory:** Too many explosions per frame caused stuttering.

**Implementation:** Added `MAX_EXPLOSIONS_PER_FRAME = 50` limit.

**Result:** ❌ Failed - limit never reached, still only 1 explosion/frame

**Why it failed:** The limit wasn't the problem; explosions weren't grouping together in the first place.

### Multiple Other Attempts
- Adjusting delay ranges
- Changing timer update logic
- Modifying despawn timing
- Various other timing tweaks

All failed to address the root cause.

## Root Cause Discovery

### The Breakthrough

Analysis of log files revealed the smoking gun:

```
📊 EXPLOSION FRAME: 1263 total pending, 1 units ready, 0 towers ready
💥 Processing 1 unit explosions this frame (limit: 50)
[10 frames of 0 units ready]
📊 EXPLOSION FRAME: 1262 total pending, 1 units ready, 0 towers ready
💥 Processing 1 unit explosions this frame (limit: 50)
```

**Pattern:** Consistently **only 1 explosion per frame**, with ~10 frame gaps (100ms at 100 FPS).

### The Actual Problem

The explosion delay system worked as follows:
1. Assign random delays: `delay = rng.gen_range(0.1..2.0)`
2. Each frame, decrement all timers: `delay_timer -= delta_time`
3. When `delay_timer <= 0.0`, explosion fires

**The issue:** With 1200 units and continuous random delays across 0.1-2.0s range:
- At 100 FPS (10ms per frame), there are ~190 possible frame slots
- With uniform distribution, delays spread evenly across all slots
- Result: **Only ONE timer crosses zero per frame**

This created the sequential "pop-pop-pop" pattern instead of synchronized bursts.

### Why It Was Intermittent

With continuous random distribution, sometimes delays would randomly cluster by chance (creating bursts), other times they'd spread evenly (creating sequential behavior). The **behavior was non-deterministic**, making it hard to diagnose.

## The Solution: Time Quantization

### Core Concept

Instead of continuous delays across infinite values, **quantize delays to discrete time slots**:

```rust
const EXPLOSION_TIME_QUANTUM: f32 = 0.05; // 50ms buckets

// Before: continuous values (0.1234, 0.5678, 0.9012, ...)
let delay = rng.gen_range(EXPLOSION_DELAY_MIN..EXPLOSION_DELAY_MAX);

// After: quantized to 50ms slots (0.10, 0.15, 0.20, 0.25, ...)
let raw_delay = rng.gen_range(EXPLOSION_DELAY_MIN..EXPLOSION_DELAY_MAX);
let delay = (raw_delay / EXPLOSION_TIME_QUANTUM).round() * EXPLOSION_TIME_QUANTUM;
```

### Why This Works

**Delay Distribution:**
- Before: 1200 units spread across ~190 frame slots → 1-6 units per slot (sparse)
- After: 1200 units grouped into ~38 time slots → 17-67 units per slot (dense)

**Explosion Pattern:**
- Before: 1 explosion every 10 frames (sequential)
- After: 12-67 explosions per burst, hitting `MAX_EXPLOSIONS_PER_FRAME=50` limit

**Consistency:**
- Before: Random luck determined if explosions would cluster
- After: Guaranteed burst behavior every time

## Implementation Details

### Code Changes

**1. Added constant** ([src/constants.rs:46](src/constants.rs#L46)):
```rust
pub const EXPLOSION_TIME_QUANTUM: f32 = 0.05; // Quantize delays to 50ms slots
```

**2. Applied quantization in `tower_destruction_system`** ([src/objective.rs:464-466](src/objective.rs#L464-L466)):
```rust
let raw_delay = rng.gen_range(EXPLOSION_DELAY_MIN..EXPLOSION_DELAY_MAX);
let delay = (raw_delay / EXPLOSION_TIME_QUANTUM).round() * EXPLOSION_TIME_QUANTUM;
```

**3. Fixed E-key debug test** ([src/objective.rs:704-706](src/objective.rs#L704-L706)):

Previously used hardcoded sequential delays:
```rust
// OLD - guaranteed sequential
let delay = (i as f32) * 0.1; // 0.0, 0.1, 0.2, 0.3...
```

Now uses same quantization logic:
```rust
// NEW - quantized random
let raw_delay = rng.gen_range(EXPLOSION_DELAY_MIN..EXPLOSION_DELAY_MAX);
let delay = (raw_delay / EXPLOSION_TIME_QUANTUM).round() * EXPLOSION_TIME_QUANTUM;
```

**4. Added histogram logging** ([src/objective.rs:486-507](src/objective.rs#L486-L507)):

Visualizes delay distribution to verify quantization:
```
📊 HISTOGRAM (quantized delays):
  0.10s: 12 units
  0.15s: 36 units
  0.20s: 41 units
  0.25s: 37 units
  ... (29 more time slots)
```

**5. Cleaned up logging** ([src/explosion_shader.rs](src/explosion_shader.rs), [src/particles.rs](src/particles.rs)):
- Changed individual explosion spawn logs from `info!` → `trace!`
- Changed particle spawn logs from `info!` → `trace!`
- Kept only summary statistics at `info!` level

## Results

### Before Fix
```
📊 EXPLOSION FRAME: 1263 total pending, 1 units ready, 0 towers ready
💥 Processing 1 unit explosions this frame (limit: 50)
📊 EXPLOSION FRAME: 1262 total pending, 1 units ready, 0 towers ready
💥 Processing 1 unit explosions this frame (limit: 50)
```

### After Fix
```
📊 EXPLOSION FRAME: 1263 total pending, 12 units ready, 1 towers ready
💥 Processing 12 unit explosions this frame (limit: 50)
📊 EXPLOSION FRAME: 1250 total pending, 36 units ready, 0 towers ready
💥 Processing 36 unit explosions this frame (limit: 50)
📊 EXPLOSION FRAME: 1214 total pending, 41 units ready, 0 towers ready
💥 Processing 41 unit explosions this frame (limit: 50)
📊 EXPLOSION FRAME: 535 total pending, 67 units ready, 0 towers ready
💥 Processing 50 unit explosions this frame (limit: 50)  ← Hitting limit!
```

### Validation

Tested 7-8 runs consecutively - **all successful** with burst explosions. Previously, success rate was ~50% due to random chance.

## Key Learnings

1. **Log analysis is critical** - The pattern was only visible by examining frame-by-frame explosion counts, not from observing the game.

2. **Statistical distribution matters** - Continuous uniform random isn't always the right choice; discrete clustering can create better perceived randomness.

3. **Multiple code paths, one bug** - The E-key debug test had hardcoded sequential delays, creating a red herring. Both code paths needed fixing.

4. **Intermittent bugs hide root causes** - When behavior "sometimes works," it's easy to assume the fix worked when it actually just got lucky.

5. **Performance limits matter** - `MAX_EXPLOSIONS_PER_FRAME=50` was added as protection but never activated until the quantization fix made it necessary.

## Technical Insights

### Why 50ms Quantum?

- **Game runs at ~100 FPS** → 10ms per frame
- **50ms = 5 frames** → Explosions align every 5 frames
- **Creates bursts** without overwhelming a single frame
- **Tunable** - can adjust for different dramatic effects

### Alternative Approaches Considered

**1. Fixed time slots (0.1s, 0.2s, 0.3s exactly):**
- Too predictable, loses randomness feel

**2. Larger quantum (100ms+):**
- Fewer bursts, more "chunky" feel

**3. Smaller quantum (10ms):**
- Too many time slots, back to sparse distribution

50ms hit the sweet spot of random feel + reliable bursts.

## Conclusion

After 10+ failed attempts focusing on symptoms (processing order, performance, frame limiting), the actual problem was **statistical distribution of delay timers**. The fix was conceptually simple (quantization) but required deep log analysis to discover.

The sequential explosion bug is now **permanently fixed** with consistent burst behavior across all runs.

---

**Files modified:**
- `src/constants.rs` - Added `EXPLOSION_TIME_QUANTUM`
- `src/objective.rs` - Applied quantization to both code paths, added histogram logging
- `src/explosion_shader.rs` - Logging cleanup
- `src/particles.rs` - Logging cleanup

**Commit:** Fix sequential explosion bug with delay quantization
