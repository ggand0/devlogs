# Slider Rendering Loop Bug - Analysis and Fix Plan

**Date:** 2025-11-16
**Status:** Under Investigation
**Severity**: Medium (affects UX during rapid slider scrubbing)
**Branch**: TBD (separate from RLE mask support PR)

## Executive Summary

During testing of the RLE mask support feature, a rendering loop bug was discovered that causes the application to become temporarily unresponsive (~2.5 seconds) when rapidly scrubbing the slider on images with many annotations. The issue is a race condition in the slider navigation system that creates redundant async image loading tasks, not a problem with the RLE mask rendering itself.

## Bug Description

### Symptoms
- App becomes stuck in continuous rendering loop when slider is scrubbed very rapidly
- UI unresponsive for ~2.5 seconds
- Continuous log output showing repeated rendering cycles
- More likely to occur with images that have many annotations (195+ in test case)

### When It Occurs
1. User rapidly scrubs the slider back and forth
2. Lands on an image with many annotations (causing initial slow render)
3. UI enters feedback loop creating duplicate async tasks
4. Loop continues until event queue drains

### Impact
- Temporary UI freeze (no permanent damage)
- Poor user experience during rapid navigation
- Wastes CPU/GPU resources on redundant operations
- Can occur with any annotation-heavy dataset (not RLE-specific)

## Root Cause Analysis

### Log Analysis (logs/111625_rendering_loop.log)

**Timeline of Events:**

**Line 1036-1037**: Initial render completes
- 195 mask annotations processed
- Quads created for all masks

**Line 1038**: **Trigger point** - GPU bottleneck detected
```
WARN BOTTLENECK: Renderer present took 122.572888ms
```
- Normal render time: ~14ms (71 FPS)
- This render: 122ms (8.7× slower!)
- Caused by: Initial GPU texture upload for 195 masks

**Line 1040**: First `update_pos` call
```
#####################update_pos - Creating async image loading task for pane 0
```

**Lines 1042-1796**: Rendering loop (2.5 seconds)
- 61 `update_pos` calls creating async tasks
- ~155 "Reusing cached quads" messages (cache working perfectly!)
- Position never changes: `image_top_left=(272.0,1.0)` throughout
- Image dimensions constant: `(480, 640)` throughout
- Rendering at normal speed (~14ms/frame) after initial bottleneck

### Root Cause: Feedback Loop

The 122ms initial render delay triggers a cascade:

```
1. User scrubs slider rapidly
   └→ Multiple slider position updates queued

2. GPU processing heavy initial render (122ms)
   └→ UI event processing delayed

3. GPU finishes → UI update triggered
   └→ Calls update_pos (line 1040)

4. update_pos spawns async image load task
   └→ Task completes → requests UI update

5. UI update → another update_pos call (line 1056, only 16ms later!)
   └→ Even though position hasn't changed!

6. Loop continues...
   └→ Each async task completion triggers more update_pos calls
   └→ 61 redundant tasks created over 2.5 seconds
   └→ Finally drains when event queue empties
```

### Key Evidence

**The bug is NOT in mask_shader.rs:**
- Cache working perfectly: "Reusing cached quads for 195 annotations" × 155 times
- No quad rebuilding after initial creation (state-based caching works!)
- All renders use cached GPU resources efficiently

**The bug IS in navigation_slider.rs:**
- `update_pos` called repeatedly for same position
- No deduplication of async image loading tasks
- No check if position actually changed
- Creates tasks even when image already loaded and displayed

### Why It's Rare But Reproducible

**Requires specific conditions:**
1. Rapid slider scrubbing (creates event queue buildup)
2. Image with many annotations (causes initial slow GPU render)
3. Slow render delays UI responsiveness (triggers cascade)

**Why it affects annotation-heavy images more:**
- More annotations = more GPU textures to upload on first render
- More textures = longer initial render time
- Longer render = more UI events queued
- More queued events = longer feedback loop

## Technical Details

### Location
**File**: [src/navigation_slider.rs:481](../src/navigation_slider.rs#L481)

**Problem Code Pattern:**
```rust
fn update_pos(...) {
    log::debug!("#####################update_pos - Creating async image loading task for pane {}", pane_index);

    // Problem: No check if position changed or task already in flight
    spawn_async_image_load_task(...);
}
```

**What Happens:**
1. UI update event → `update_pos()` called
2. New async task spawned unconditionally
3. Task loads image (even if already loaded)
4. Task completion triggers UI update
5. UI update calls `update_pos()` again → GOTO 2

### Performance Impact

**During Loop (2.5 seconds):**
- 61 redundant async tasks created
- Each task performs file I/O and image decoding (even if cached downstream)
- ~71 FPS rendering maintained (rendering itself is not slow)
- CPU spinning on task scheduling and event processing

**Normal Operation:**
- `update_pos` called only when slider position changes
- Clean one-task-per-position behavior
- No feedback loop

## Proposed Fix

### Strategy: Position Change Detection (Recommended)

**Approach**: Track the last loaded position and only create async tasks when position actually changes.

**Implementation:**

```rust
// In src/navigation_slider.rs

pub struct NavigationSlider {
    // ... existing fields
    last_loaded_position: Option<usize>,  // NEW: Track last position we loaded
}

impl NavigationSlider {
    fn update_pos(&mut self, ...) {
        // Calculate current position from slider state
        let current_position = self.calculate_current_position();

        // Check if position actually changed
        if self.last_loaded_position == Some(current_position) {
            log::debug!("Position unchanged ({}), skipping redundant image load task", current_position);
            return;  // Early exit - no async task created
        }

        // Position changed, create async task
        log::debug!("#####################update_pos - Creating async image loading task for pane {} (position {} → {})",
            pane_index,
            self.last_loaded_position.unwrap_or(0),
            current_position
        );

        // Update tracked position BEFORE spawning task
        self.last_loaded_position = Some(current_position);

        // Spawn async task
        spawn_async_image_load_task(...);
    }
}
```

**Why This Works:**
- Directly addresses root cause: eliminates redundant tasks
- Simple state tracking (single `Option<usize>` field)
- No timing dependencies or race conditions
- Works regardless of render speed
- Minimal performance overhead

**Edge Cases to Handle:**
- First call: `last_loaded_position == None` → allow task
- Position wrapping: handle position overflow gracefully
- Manual navigation vs slider: both use same code path
- Reset on dataset change: clear `last_loaded_position`

### Alternative Approaches Considered

#### Option 2: In-Flight Task Tracking
```rust
struct NavigationSlider {
    pending_loads: HashSet<usize>,  // Track positions being loaded
}
```

**Pros:**
- Prevents multiple tasks for same position
- More granular control

**Cons:**
- More complex state management
- Need to track task completion to remove from set
- Requires message passing from async task back to slider
- Harder to debug if tasks fail silently

**Verdict**: More complex than needed, Option 1 is simpler

#### Option 3: Time-Based Debouncing
```rust
struct NavigationSlider {
    last_update_time: Instant,
}

fn update_pos(...) {
    if now.duration_since(self.last_update_time) < Duration::from_millis(16) {
        return;  // Skip if called within same frame
    }
}
```

**Pros:**
- Very simple to implement
- Reduces event frequency

**Cons:**
- Doesn't address root cause (position not changing)
- Timing-dependent (can still create redundant tasks if >16ms apart)
- Magic number (16ms) may not work well on all systems
- Could skip legitimate position changes if timing unlucky

**Verdict**: Band-aid solution, doesn't fix core issue

### Recommended Solution

**Use Option 1 (Position Change Detection)** because:
1. Directly solves the root cause
2. Simple implementation and maintenance
3. No timing dependencies
4. Works universally across all systems
5. Easy to test and verify

## Implementation Plan

### Phase 1: Create Branch and Setup
```bash
git checkout -b fix/slider-rendering-loop
```

### Phase 2: Implement Fix

**File**: `src/navigation_slider.rs`

1. Add `last_loaded_position` field to `NavigationSlider` struct
2. Initialize to `None` in constructor
3. Add position change check in `update_pos()`
4. Clear `last_loaded_position` when dataset changes
5. Update logging to show old→new position transitions

### Phase 3: Testing

**Manual Testing:**
1. Load dataset with 100+ annotations per image
2. Rapidly scrub slider back and forth
3. Verify: No rendering loop occurs
4. Verify: Logs show "Position unchanged" messages
5. Verify: Normal navigation still works smoothly

**Regression Testing:**
1. Test with small datasets (1-10 images)
2. Test with no annotations
3. Test keyboard navigation (arrow keys)
4. Test mouse wheel navigation
5. Test jump-to-position (clicking slider)

**Performance Testing:**
1. Measure frame times during slider scrubbing
2. Verify no async tasks created for same position
3. Check CPU usage (should be lower without redundant tasks)

### Phase 4: Validation

**Success Criteria:**
- [ ] No rendering loops during rapid slider scrubbing
- [ ] Logs show position change detection working
- [ ] All navigation methods work correctly
- [ ] No performance regression in normal use
- [ ] No new bugs introduced

**Log Output Expected:**
```
Position unchanged (42), skipping redundant image load task
Position unchanged (42), skipping redundant image load task
#####################update_pos - Creating async image loading task for pane 0 (position 42 → 43)
Position unchanged (43), skipping redundant image load task
```

## Additional Considerations

### Why This Wasn't Caught Earlier

1. **Rare occurrence**: Requires specific sequence (rapid scrubbing + heavy annotations)
2. **Appears benign**: App recovers automatically after 2-3 seconds
3. **Annotation-dependent**: Only obvious with annotation-heavy datasets
4. **RLE masks revealed it**: Initial GPU texture upload increased initial render time

### Relationship to RLE Mask PR

**Independent Issues:**
- RLE mask support is working correctly
- State-based caching is working perfectly
- This bug existed before RLE masks (just harder to trigger)

**Why Keep Separate:**
- RLE mask PR is feature-complete and tested
- This bug affects general slider navigation
- Fix applies to all datasets (not COCO-specific)
- Easier to review and merge independently

**Timeline:**
1. Merge RLE mask PR first (proven stable)
2. Address slider loop bug in separate PR
3. Both improvements benefit users independently

### Future Improvements

Beyond this fix, consider:

1. **Async task queue management**
   - Limit concurrent image loading tasks
   - Cancel in-flight tasks when position changes rapidly

2. **Progressive loading**
   - Load lower resolution first for preview
   - Load full resolution in background

3. **Predictive preloading**
   - Preload adjacent images during idle time
   - Reduce latency during normal navigation

4. **GPU resource pooling**
   - Reuse texture objects instead of creating new ones
   - Reduce GPU allocation overhead

## References

- **Log File**: `logs/111625_rendering_loop.log` (1796 lines)
- **Related PR**: RLE mask support (feat/rle-mask-support branch)
- **Related Fix**: Cache collision bug (already fixed in RLE mask PR)
- **Code Location**: [src/navigation_slider.rs:481](../src/navigation_slider.rs#L481)

## Timeline

- **2025-11-15**: Bug discovered during RLE mask testing
- **2025-11-16**: Root cause analysis completed
- **2025-11-16**: Fix plan documented
- **TBD**: Implementation in separate branch
- **TBD**: Testing and validation
- **TBD**: PR creation and review

## Questions for Discussion

1. Should we add telemetry to track how often this pattern occurs in production?
2. Should we add a warning if the same position is requested >5 times in a row?
3. Should we implement task cancellation for in-flight loads when position changes?
4. Should we add a setting for maximum concurrent image loading tasks?

---

**Document Status**: Complete
**Next Steps**: Create fix branch and implement solution
**Owner**: TBD
**Priority**: Medium (UX improvement, not a blocker)
