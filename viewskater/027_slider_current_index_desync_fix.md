# Fix: Slider current_index Desynchronization Bug

**Date:** 2025-11-16
**Status:** FIXED
**Severity:** CRITICAL - Causes wrong images/annotations to display
**Branch:** feat/rle-mask-support
**Related:** devlogs/plans/111625_wrong_annotations_cache_desync.md

## Executive Summary

Fixed a critical bug where `current_index` was being overwritten by stale async slider image loading tasks after slider release, causing images and annotations to desynchronize during keyboard navigation.

## The Bug

After rapidly scrubbing the slider and releasing it, keyboard navigation would display wrong images and annotations because `current_index` would be reset to a stale value from an earlier slider position.

### Example from Testing

1. User scrubs slider from position 2503 → 2541
2. Releases slider at position 2541
3. Presses 'D' to navigate right
4. **Expected**: Image/annotations for index 2542 (or thereabouts)
5. **Actual**: Image/annotations for index 2504 (from earlier in scrubbing)

## Root Cause

### The Smoking Gun

From `logs/111625_wrong_rendering_after_sliderrelease_3.log`:

```
Line 3: load_full_res_image: Set current_index = 2541 ✅
Line 4: load_remaining_images: After load_full_res_image, pane[0].current_index = 2541 ✅
Line 163: Slider image loaded for pane 0 at position 2503 ❌
Line 5: LoadPos: After processing, pane[0].current_index = 2503 ❌
Line 6: BEGINE RENDERING NEXT: current_index: 2503 ❌
```

### What Was Happening

**The Race Condition Timeline:**

```
T0: User rapidly scrubs slider (2503 → 2504 → ... → 2541)
    - Each slider position triggers async image loading task
    - Tasks queued: SliderImageWidgetLoaded(2503), (2504), ..., (2541)

T1: User releases slider at position 2541
    - SliderReleased event triggers load_full_res_image(2541)
    - current_index correctly set to 2541 ✅

T2: Stale async tasks complete AFTER slider release
    - SliderImageWidgetLoaded(2503) message arrives
    - Handler at message_handlers.rs:327 executes:
      pane.img_cache.current_index = pos;  // ❌ Sets to 2503!
    - Overwrites the correct value (2541) with stale value (2503)

T3: User presses 'D' for keyboard navigation
    - render_next_image() uses current_index = 2503
    - Wrong image/annotations displayed!
```

### Code Location

**File**: `src/app/message_handlers.rs`

**Line 327** (before fix):
```rust
Message::SliderImageWidgetLoaded(result) => {
    match result {
        Ok((pane_idx, pos, handle, dimensions)) => {
            if let Some(pane) = app.panes.get_mut(pane_idx) {
                pane.slider_image = Some(handle);
                pane.slider_image_dimensions = Some(dimensions);
                pane.slider_image_position = Some(pos);
                pane.img_cache.current_index = pos;  // ❌ BUG HERE!
```

### Why This is Wrong

1. **Async tasks don't know about slider release**: The `SliderImageWidgetLoaded` tasks were queued during slider scrubbing, but they complete AFTER the slider has been released and `current_index` has been properly updated.

2. **No ordering guarantee**: Async tasks can complete in any order, so a stale task from position 2503 might complete AFTER the correct `load_full_res_image(2541)` has run.

3. **Overwrites correct value**: Each completing task blindly sets `current_index = pos` without checking if the slider is still active or if this is a stale update.

4. **Redundant anyway**: The slider position is already tracked in `slider_image_position`, so setting `current_index` here was unnecessary duplication.

## The Fix

### Changed Code

**File**: `src/app/message_handlers.rs:327-329`

```rust
// BEFORE:
pane.img_cache.current_index = pos;

// AFTER:
// BUGFIX: Don't update current_index here! It causes desyncs when stale slider images
// load after slider release. The slider position is tracked in slider_image_position instead.
// pane.img_cache.current_index = pos;
```

### Why This Works

1. **Eliminates race condition**: Stale slider image tasks no longer overwrite `current_index`
2. **Preserves correct value**: `current_index` stays at the value set by `load_full_res_image()` after slider release
3. **No information loss**: Slider position is still tracked via `slider_image_position`
4. **Separation of concerns**:
   - During slider: Use `slider_image_position` for rendering
   - After release: Use `current_index` for navigation

## Testing Evidence

### Before Fix

From `logs/111625_wrong_rendering_after_sliderrelease_3.log`:

```
2025-11-16T02:13:53.338421Z DEBUG load_full_res_image: Set current_index = 2541
2025-11-16T02:13:53.338454Z DEBUG load_remaining_images: After load_full_res_image, pane[0].current_index = 2541
2025-11-16T02:13:53.341717Z DEBUG Slider image loaded for pane 0 at position 2503
2025-11-16T02:13:53.382841Z DEBUG LoadPos: After processing, pane[0].current_index = 2503  ❌ WRONG
2025-11-16T02:13:53.679308Z DEBUG BEGINE RENDERING NEXT: current_index: 2503  ❌ WRONG
```

### Expected After Fix

```
DEBUG load_full_res_image: Set current_index = 2541
DEBUG load_remaining_images: After load_full_res_image, pane[0].current_index = 2541
DEBUG Slider image loaded for pane 0 at position 2503  (ignored - doesn't update current_index)
DEBUG LoadPos: After processing, pane[0].current_index = 2541  ✅ STAYS CORRECT
DEBUG BEGINE RENDERING NEXT: current_index: 2541  ✅ CORRECT
```

## Additional Debug Logging Added

To help diagnose this and future issues, added debug logging in:

1. **`src/navigation_slider.rs:134,139,157,162`**: Log when `current_index` is set in `load_full_res_image()`
2. **`src/navigation_slider.rs:296`**: Log `current_index` after `load_full_res_image()` completes
3. **`src/loading_handler.rs:149`**: Prominent separator for LoadPos operations
4. **`src/loading_handler.rs:188,229`**: Log when LoadPos updates images and final `current_index`

## Reduced Noise Logging

Commented out noisy per-frame debug logs that made it hard to see the important events:

1. **`src/ui.rs:391`**: "UI: Using dimensions for annotation_index=..."
2. **`src/coco/overlay/mask_shader.rs:156,160`**: "Reusing/Rebuilding cached quads..."
3. **`src/coco/overlay/bbox_shader.rs:130`**: "BBox prepare: image=..."

## Related Issues

### This Fix Addresses

- ✅ Wrong image displayed after slider release + keyboard nav
- ✅ Wrong annotations displayed after slider release + keyboard nav
- ✅ Image/annotation desynchronization during navigation
- ✅ `current_index` not matching actual displayed image

### Still Related But Separate

- The "images become correct after 4-6 navigations" symptom - this was because navigating 4-6 times would eventually increment `current_index` past the stale values and into the correctly cached range

### Not Related

- RLE mask implementation (working correctly)
- State-based caching (working correctly)
- Annotation ID caching (already fixed)
- Slider rendering loop (separate performance issue)

## Impact

**Before Fix:**
- After rapid slider scrubbing, navigation would show completely wrong images/annotations
- User trust in application correctness undermined
- Data verification workflows broken
- Very confusing user experience

**After Fix:**
- Images and annotations match correctly after slider release
- `current_index` stays synchronized with actual displayed content
- Navigation works correctly immediately after slider release
- Predictable, correct behavior

## Files Modified

1. `src/app/message_handlers.rs` - Commented out line that was overwriting `current_index`
2. `src/navigation_slider.rs` - Added debug logging for `current_index` updates
3. `src/loading_handler.rs` - Added debug logging for LoadPos operations
4. `src/ui.rs` - Commented out noisy per-frame logging
5. `src/coco/overlay/mask_shader.rs` - Commented out noisy per-frame logging
6. `src/coco/overlay/bbox_shader.rs` - Commented out noisy per-frame logging

## Lessons Learned

1. **Async completion timing matters**: Tasks queued during one operation can complete during a completely different operation, causing state corruption
2. **Stale data is dangerous**: Always check if data is still relevant before applying updates
3. **Single source of truth**: Having multiple places update the same state variable leads to race conditions
4. **Debug logging is essential**: Without the detailed logging, this bug would have been nearly impossible to track down
5. **Test async boundaries**: Bugs at async operation boundaries are subtle and require careful testing

## Timeline

- **2025-11-16 01:36**: Bug reported by user after slider scrubbing
- **2025-11-16 02:10**: Added debug logging to investigate
- **2025-11-16 02:13**: Reproduced bug, captured detailed logs
- **2025-11-16 02:13**: Analyzed logs, found root cause
- **2025-11-16 02:13**: Implemented fix
- **2025-11-16 02:13**: Build successful, ready for testing

## Additional Defensive Fix: current_image_index Tracking

### Overview

In addition to the root cause fix above, we also added `current_image_index` tracking to provide an extra layer of safety. This change was implemented BEFORE discovering the root cause, and may not be strictly necessary with the `SliderImageWidgetLoaded` fix in place.

### Changes Made

**File**: `src/pane.rs`

Added a new field to track which index is actually loaded in `current_image`:

```rust
pub struct Pane {
    // ... existing fields
    pub current_image_index: Option<usize>,  // Track which index current_image contains
}
```

Updated in:
- `Default` impl (line 89): `current_image_index: None`
- Constructor (line 141): `current_image_index: None`
- `reset()` (line 191): `current_image_index = None`

**File**: `src/navigation_slider.rs`

Set `current_image_index` when loading images:
- Line 138: After setting `current_index` in `load_full_res_image()`
- Line 161: After loading image in `load_full_res_image()`
- Line 627: In `load_initial_image()`

```rust
pane.current_image = loaded_image;
pane.current_image_index = Some(pos);
debug!("load_full_res_image: Set current_image_index = {} for pane {}", pos, idx);
```

**File**: `src/pane.rs` (navigation functions)

Update `current_image_index` after navigation:
- Line 329: In `render_next_image()` after incrementing `current_index`
- Line 402: In `render_prev_image()` after decrementing `current_index`

```rust
self.img_cache.current_index += 1;
self.current_image_index = Some(self.img_cache.current_index);
```

**File**: `src/ui.rs`

Use `current_image_index` for annotation lookup:

```rust
let annotation_index = if app.use_slider_image_for_render && app.panes[0].slider_image.is_some() {
    // Slider mode: use slider_image_position
    app.panes[0].slider_image_position
        .or(app.panes[0].current_image_index)
        .unwrap_or(app.panes[0].img_cache.current_index)
} else {
    // Normal mode: use current_image_index
    app.panes[0].current_image_index
        .unwrap_or(app.panes[0].img_cache.current_index)
};
```

### Rationale for This Defensive Fix

**Pros:**
- Provides an independent source of truth for which image is actually loaded
- Protects against OTHER potential bugs where `current_index` and `current_image` might desync
- Makes the code more explicit about the relationship between index and loaded image
- Useful for future debugging if similar issues occur

**Cons:**
- Adds extra state that needs to be maintained
- Requires updates in multiple locations
- May be unnecessary if the root cause fix is sufficient

### Is This Fix Necessary?

**With just the `SliderImageWidgetLoaded` fix:**
- `current_index` should always be correct after `load_full_res_image()` completes
- `current_image` should match `current_index` after loading completes
- The root cause (stale slider tasks overwriting `current_index`) is eliminated

**However, keeping `current_image_index` provides:**
- Defense in depth against other potential desyncs
- Explicit tracking of what's actually loaded vs. what we're navigating to
- Better debugging information if issues occur

### Recommendation

**For now**: Keep both fixes to ensure maximum correctness and provide debugging capability.

**Future**: After the RLE PR is merged and the fix has been validated in production use, we could consider:
1. Removing `current_image_index` if no other desyncs are observed
2. OR keeping it as defensive programming if it proves useful for debugging

**To revert this defensive fix**: See git diff and revert changes to:
- `src/pane.rs` (struct field, Default, constructor, reset, render_next/prev)
- `src/navigation_slider.rs` (current_image_index assignments)
- `src/ui.rs` (annotation_index lookup logic)

## Next Steps

1. ✅ Test the fix with rapid slider scrubbing + keyboard navigation
2. ✅ Verify annotations and images match correctly
3. ✅ Check that `current_index` stays at correct value
4. ✅ User tested 10 slider scrubbing attempts - no issues observed
5. ⏳ Test with just the root cause fix (revert `current_image_index` changes)
6. ⏳ Commit the fix
7. ⏳ Merge into feat/rle-mask-support branch

---

**Status**: Fix implemented and tested successfully (10 attempts, no issues)
**Priority**: CRITICAL (blocker for RLE mask PR)
**Confidence**: HIGH - Root cause clearly identified and fix is surgical
**Testing Status**: Awaiting revert test to confirm root cause fix alone is sufficient
