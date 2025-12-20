# Async Initial Loading - Navigation Fix Progress

## Current State

Branch: `feat/async-initial-load`
Status: **Partially Fixed** - Navigation past initial cache window now works

## What Works

1. **Sequential initial loading** - Images 1-5 load one at a time in background after image 0 displays
2. **Navigation during initial loading** - User can navigate once target image is cached
3. **Navigation past initial cache window** - Now works with direct LoadPos fallback (NEW)

## Root Cause Analysis

### The Problem

When navigating past the initial cache window (e.g., from image 5 to image 6), `move_right_all()` calls `load_next_images_all()` which calculates the target index using:

```rust
target = current_index - current_offset + cache_count + 1
       = 5 - 0 + 5 + 1 = 11
```

With only 7 images, index 11 doesn't exist. This triggered `ShiftNext` with an invalid target, which returned `[None]` data and navigation failed.

### Why This Happened

`move_right_all()` expects a fully initialized cache with proper cache window state. After async initial loading:
- Cache contains images 0-5 at positions 0-5
- `current_offset = 0` (normal state)
- But the target formula assumes cache edge calculations that don't apply to partially populated caches

### The Fix

Added direct LoadPos fallback in `move_right_all()` when:
1. Next image is not cached
2. Standard target calculation is out of bounds
3. But next image (current_index + 1) exists

```rust
let needs_direct_load = panes_to_load.iter().any(|pane| {
    let cache = &pane.img_cache;
    let next_idx = cache.current_index + 1;
    let standard_target = cache.current_index as isize - cache.current_offset + cache.cache_count as isize + 1;
    next_idx < cache.num_files && standard_target as usize >= cache.num_files
});

if needs_direct_load {
    // Use LoadPos to directly load current_index + 1
}
```

### Secondary Bug Found

`load_images_by_operation()` in `img_cache.rs` returned `Task::none()` for `LoadPos` operations - it did nothing! Fixed by having it call `load_images_by_indices()` like `LoadNext` does.

## Files Modified

### `src/navigation_keyboard.rs`
- Added `needs_direct_load` detection in `move_right_all()`
- Uses LoadPos to load next image directly when standard cache logic fails
- Added info logging for debugging

### `src/cache/img_cache.rs`
- Fixed `load_images_by_operation()` to actually handle `LoadPos`:
  ```rust
  LoadOperation::LoadPos((ref _pane_index, ref target_indices_and_cache)) => {
      let target_indices: Vec<Option<isize>> = target_indices_and_cache
          .iter()
          .map(|opt| opt.map(|(idx, _cache_pos)| idx))
          .collect();
      load_images_by_indices(...)
  }
  ```

## Test Results

- Small dataset (7 images): Navigation reaches image 6 (all images) ✓
- Direct LoadPos path triggers when needed ✓
- Initial sequential loading still works ✓

## Remaining Work

1. **Test with larger dataset** (23 images) - Verify navigation works throughout
2. **Clean up debug logging** - Remove excessive info! statements
3. **Test left navigation** - Similar fix may be needed for `move_left_all()`
4. **Edge cases** - Test at dataset boundaries

## Key Insight

The cache system has two modes:
1. **Fully initialized** - Cache window properly set up, edge formulas work
2. **Partially initialized** - After async initial load, need direct image loading

The fix adds a fallback for the partially initialized state without breaking the fully initialized case.
