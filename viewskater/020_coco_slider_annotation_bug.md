# COCO Slider Annotation Bug Debugging

**Date:** 2025-11-01

## Problem Statement

Annotations (bounding boxes) are incorrectly positioned during slider navigation, even though they appear correctly:
- During keyboard navigation (arrow keys)
- After slider is released

The bug was introduced in commit `7a7f86e59359440dbff650aa8b5cb166657aa9e8` which added annotation rendering during slider movement.

## Observations

### What Works
- Keyboard navigation shows annotations correctly
- Annotations appear correctly after slider is released
- Correct annotations are shown for each image (filename lookup works)
- Zoom is maintained during slider navigation

### What's Broken
- During slider movement, annotation positions are wrong
- User reports: "For some images it looks correct and for others it's still wrong"
- Hypothesis: "if the image of slider rendering start is common image resolution like 640x480 it's working for images that have the same res during the slider movement"

## Architecture Overview

### Two Widget Paths

1. **Slider Movement** (fast rendering)
   - Uses `Viewer` widget (simple image widget)
   - Image loaded asynchronously via `SliderImageWidgetLoaded` message
   - Sets `pane.slider_image` handle

2. **Keyboard Navigation / Normal** (feature-rich)
   - Uses `ImageShader` widget (custom WGPU shader with zoom/pan)
   - Uses `pane.scene`

### Annotation Rendering

Both paths overlay COCO annotations using:
- `BBoxShader` - WGPU shader for bounding boxes
- `PolygonShader` - WGPU shader for segmentation masks

Annotations are stacked on top of image widget using Iced's `Stack`.

## Key Code Locations

### Slider Image Loading
- `app.rs:1576` - Sets `pane.slider_image = Some(handle)`
- `app.rs:1579` - Updates `pane.img_cache.current_index = pos`
- `app.rs:1602` - Updates `pane.current_image` with slider data

### UI Rendering
- `ui.rs:285-295` - Viewer widget creation during slider movement
- `ui.rs:296-322` - ImageShader widget creation otherwise
- `ui.rs:325-390` - COCO annotation overlay creation

### Annotation Position Calculation
- `bbox_shader.rs:117-128` - ContentFit::Contain calculation for centering
- `bbox_shader.rs:145-153` - Annotation coordinate transformation
- `viewer.rs:516-535` - Viewer's scaled_image_size calculation

## Debugging Journey

### Attempt 1: Bounds Mismatch
**Hypothesis**: Viewer and BBox overlay had different bounds due to layout differences.

**Debug Output**:
```
Viewer: bounds=(417.9,557.2)
BBox: bounds=(960,557.2)
```

**Fix Attempted**: Removed `center()` wrapper from Viewer, used `container()` with Fill dimensions.

**Result**: Made bounds match but annotations still wrong.

### Attempt 2: Layout Consistency
**Hypothesis**: ImageShader uses `center()` wrapper, Viewer should too for consistency.

**Fix Attempted**: Made both paths use `center()` wrapper.

**Result**: Keyboard navigation worked (already did), slider still broken.

### Attempt 3: Wrong Image Index
**Hypothesis**: Using `current_index` but slider displays `slider_value` position.

**Debug Output**:
```
display_index=0, is_slider_moving=false
```

**Findings**:
- `is_slider_moving` flag not reliable in UI render phase
- `slider_image.is_some()` also showed false

**Fix Attempted**: Use `slider_value` when `is_slider_moving` is true.

**Result**: Annotations stuck on index 0, showing wrong annotations for each image.

### Attempt 4: Current Index Updates
**Discovery**: `SliderImageWidgetLoaded` handler updates `current_index` at line 1579.

**Realization**: Don't need special slider detection - `current_index` automatically updates when slider images load!

**Fix Attempted**: Reverted to simple approach using `current_index` directly.

**Result**: Correct annotations now shown for each image! But positions still wrong.

### Attempt 5: Image Dimension Mismatch
**Critical Debug Output**:
```
Viewer draw: measured_image=(480,640), final_size=(417.9,557.2)
BBox prepare: image=(427,640), base_scale=0.871, center=(294.1,0.0)
```

**Discovery**: Viewer shows 480x640 image, but BBox calculates for 427x640 (OLD image).

**Root Cause**: Race condition - `build_ui()` called before `current_image` updated with new slider data.

**Fix Attempted**: Get dimensions from GPU cache using `current_index`:
```rust
let image_size = app.panes[0].img_cache.cached_image_indices.iter()
    .position(|&idx| idx == current_index as isize)
    .and_then(|cache_idx| app.panes[0].img_cache.cached_data.get(cache_idx))
    .and_then(|opt| opt.as_ref())
    .map(|cached_data| (cached_data.width(), cached_data.height()))
    .unwrap_or_else(|| (
        app.panes[0].current_image.width(),
        app.panes[0].current_image.height()
    ));
```

**Result**: Some images work correctly now, others still wrong. Pattern suggests resolution-dependent bug.

## Current Status

**Partially Working**: Annotations correct for some images during slider movement, wrong for others.

**User Observation**: "if the image of slider rendering start is common image resolution like 640x480 it's working for images that have the same res during the slider movement"

## Hypothesis for Remaining Issue

The BBox shader calculations might be using cached values from the first image resolution:

1. Slider starts on image A (e.g., 640x480)
2. BBox shader initializes with these dimensions
3. User moves slider to image B (e.g., 427x640)
4. BBox shader might be caching or reusing calculations from image A
5. Only images matching image A's resolution render correctly

**Potential Problem Areas**:
- `bounds.x` and `bounds.y` offsets in BBox shader (line 149-150)
- `center_offset` calculation might be cached/stale
- Viewer widget internal state vs BBox shader state sync

## Next Steps

1. Add more detailed logging to compare:
   - Viewer's actual rendering position/translation
   - BBox shader's calculated center offset
   - Both widgets' understanding of image dimensions

2. Check if BBox shader primitive is being reused across different images

3. Investigate if there's widget state caching in Iced that's not being cleared

## Code Changes Made

### src/ui.rs
- Both Viewer and ImageShader now use `center()` wrapper
- Image dimensions looked up from cache using `current_index`
- Removed special slider movement detection logic

### src/widgets/viewer.rs
- Added `with_zoom_state()` method
- Added `diff()` method to sync zoom state
- Added debug logging for draw parameters

### src/coco/overlay/bbox_shader.rs
- Added debug logging for prepare parameters
- No calculation changes yet

## Files Modified
- `src/ui.rs`
- `src/widgets/viewer.rs`
- `src/coco/overlay/bbox_shader.rs`
