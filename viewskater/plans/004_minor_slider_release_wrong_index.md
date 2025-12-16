# Minor Glitch: Wrong Image After Slider Release (Non-COCO Mode)

**Date:** 2025-11-16
**Status:** NOT FIXED - Deferred to post-RLE PR
**Severity:** MINOR - Self-correcting after first navigation
**Branch:** feat/rle-mask-support
**Related:** ../027_slider_current_index_desync_fix.md

## Symptom

After rapid slider scrubbing and release, the first navigation shows the wrong image, but subsequent navigation self-corrects.

### Example

1. Rapidly scrub slider back and forth
2. Release slider at position/index 36
3. Press 'D' to navigate right
4. **Expected**: Image at index 37
5. **Actual**: Image at index 31 (wrong, off by ~5)
6. Press 'D' again → Image at index 32 (following from 31)
7. Press 'A' to go back → Image at index 31, then 30, 29...
8. Index 36 is "forgotten" - navigation continues from the wrong starting point

### Key Characteristics

- **Self-correcting**: After the first wrong image, navigation works correctly from that wrong starting point
- **Consistent offset**: Wrong index is typically ~5 positions off (possibly related to cache size)
- **Persistent shift**: The original slider position (36) is lost; system treats wrong position (31) as the new base
- **No COCO required**: Happens even without COCO annotations enabled

## Difference from the Fixed Bug

The bug we just fixed ([devlogs/111625_slider_current_index_desync_fix.md](devlogs/111625_slider_current_index_desync_fix.md)) was:
- **That bug**: Stale `SliderImageWidgetLoaded` tasks overwriting `current_index` AFTER slider release
- **This bug**: `current_index` is set to wrong value during/immediately after slider release

The key difference: That bug caused ongoing corruption (every stale task would overwrite). This bug is a one-time incorrect initialization.

## Hypothesis

Based on the recently fixed slider desync bug, likely causes:

### 1. Cache Offset Calculation Error

The ring buffer cache uses `current_offset` to track position. After slider release:
- `current_index` might be set correctly to 36
- But `current_offset` might point to cache slot containing index 31
- When navigation happens, it reads from wrong cache slot

**Location to investigate**: `src/navigation_slider.rs` - `load_full_res_image()` and cache offset calculations

### 2. Stale LoadPos Operation

Similar to the `SliderImageWidgetLoaded` bug we just fixed:
- A `LoadPos` async task from earlier slider position completes after slider release
- Sets `current_index` to stale value (31)
- But this happens BEFORE the user navigates, not after

**Location to investigate**: `src/loading_handler.rs` - `handle_load_pos_operation()`

### 3. Wrong Target Index Calculation

During slider release, the code calculates which index to load:
- Might be using wrong slider position value
- Off-by-5 suggests possible rounding or cache size arithmetic error
- Could be reading from `slider_image_position` vs `slider_value` inconsistency

**Location to investigate**: Slider release event handler, position-to-index conversion

## Why Off By ~5?

The consistent ~5 offset is suspicious. Possibilities:
- **Cache size**: If cache size is 5-6, might be reading wrong cache slot
- **Prefetch offset**: Code might be loading 5 positions ahead/behind
- **Stale cache entry**: Cache at offset 1 might contain index 31, while slider is at 36

## Impact

**Severity: MINOR** because:
- Self-corrects after first navigation
- User can immediately navigate to correct position
- No data corruption or persistent desync
- Relatively rare (requires rapid slider scrubbing)

**User Experience Impact**:
- Confusing: User releases slider at one position, navigates to another
- Loss of position: Original slider position (36) is lost
- Requires re-navigation to get back to intended position

## Reproduction

1. Load any image dataset (no COCO required)
2. Rapidly scrub slider back and forth
3. Release slider at a noted position (e.g., image filename visible)
4. Immediately press 'D' or 'A' to navigate
5. **Observe**: Wrong image displayed, but subsequent navigation works

**Reliability**: Intermittent - doesn't happen every time

## Investigation Plan

1. **Add debug logging** similar to the slider desync fix:
   - Log `current_index` and `current_offset` at slider release
   - Log cache slot contents at slider release
   - Log which image is loaded in `load_full_res_image()`
   - Log any `LoadPos` operations completing around slider release

2. **Check for race conditions**:
   - Are there async tasks completing after slider release?
   - Is `current_index` being overwritten by stale operations?

3. **Verify cache offset calculations**:
   - Is `current_offset` pointing to correct cache slot?
   - Does cache slot actually contain the expected index?

4. **Timeline analysis** (similar to previous fix):
   - Capture detailed logs during slider scrub → release → navigation
   - Identify exact moment when `current_index` becomes wrong
   - Trace backwards to find what set it

## Deferred Fix

**Decision**: Fix this AFTER merging the RLE mask PR

**Rationale**:
- Minor severity (self-correcting)
- RLE PR already has critical fix (slider desync with COCO)
- This issue exists independently of RLE feature
- Can be investigated and fixed in separate PR
- Debug logging infrastructure from slider desync fix will help

## Related Code Locations

Based on the slider desync fix, likely locations:

1. **`src/app/message_handlers.rs`** - Slider release handler
2. **`src/navigation_slider.rs`** - `load_full_res_image()`, cache calculations
3. **`src/loading_handler.rs`** - `handle_load_pos_operation()`, async task completion
4. **`src/pane.rs`** - `render_next_image()`, `render_prev_image()`

## Testing After Fix

When this is fixed, verify:
1. Slider release at position X → first navigation goes to X+1 or X-1 ✅
2. No "lost" positions - can navigate back to X ✅
3. Multiple rapid slider scrubbing attempts don't cause wrong positions ✅
4. Works with both COCO and non-COCO modes ✅

---

**Status**: Documented, deferred to post-RLE PR
**Priority**: LOW-MEDIUM (minor UX issue, self-correcting)
**Next Steps**:
1. Merge RLE PR with critical slider desync fix
2. Create separate issue/PR for this minor glitch
3. Use existing debug logging infrastructure to investigate
