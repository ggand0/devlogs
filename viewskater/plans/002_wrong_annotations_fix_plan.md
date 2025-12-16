# Fix Plan: Image/Annotation Cache Desynchronization

**Date:** 2025-11-16
**Related Bug:** 001_wrong_annotations_cache_desync.md
**Severity**: CRITICAL - Affects data correctness
**Files to Modify**: src/ui.rs

## Root Cause (Code Level)

### Location: src/ui.rs lines 365-406

**The Bug**:
```rust
// Line 365-369: Annotation lookup using current_index
let current_index = app.panes[0].img_cache.current_index;
if let Some(path_source) = app.panes[0].img_cache.image_paths.get(current_index) {
    let filename = path_source.file_name();
    if let Some(annotations) = app.annotation_manager.get_annotations(&filename) {

        // Line 393-406: Dimension lookup with WRONG fallback
        app.panes[0].img_cache.cached_image_indices.iter()
            .position(|&idx| idx == current_index as isize)
            .and_then(|cache_idx| app.panes[0].img_cache.cached_data.get(cache_idx))
            .and_then(|opt| opt.as_ref())
            .map(|cached_data| (cached_data.width(), cached_data.height()))
            .unwrap_or_else(|| {
                // BUG: This fallback uses current_image which may be from a different index!
                let fallback = (
                    app.panes[0].current_image.width(),
                    app.panes[0].current_image.height()
                );
                log::debug!("UI: Using fallback dimensions from current_image: {:?}", fallback);
                fallback  // ← WRONG! current_image is from index 3203, but annotations are for index 3226!
            })
```

**What happens**:
1. `current_index` = 3226 (000000372307.jpg - horse)
2. Annotations fetched for index 3226 ✅
3. Cache lookup for index 3226 dimensions → MISS ❌
4. Falls back to `current_image.width()` and `current_image.height()` ❌
5. But `current_image` still contains texture from index 3203 (000000369757.jpg - pizza) ❌
6. Result: Horse annotations rendered with pizza image dimensions on top of pizza image

### Visual Image Display

**Line 287-289** (slider mode):
```rust
if app.use_slider_image_for_render && app.panes[0].slider_image.is_some() {
    let image_handle = app.panes[0].slider_image.clone().unwrap();
    // Uses slider_image for display
```

**Line 318-321** (normal mode):
```rust
} else if let Some(scene) = app.panes[0].scene.as_ref() {
    let mut shader = ImageShader::new(Some(scene))
    // Scene contains current_image texture
```

**The Mismatch**:
- Visual display uses: `current_image` (or `slider_image`)
- Annotations fetched for: filename at `current_index`
- Dimensions used: fallback to `current_image` dimensions
- **But `current_index` != index of image in `current_image`!**

## Proposed Fix

### Option 1: Skip Annotations When Cache Miss (SAFEST)

**Strategy**: Don't render annotations if the image at `current_index` isn't fully loaded in cache.

```rust
// Line 393-406: Modified dimension lookup
let image_size = app.panes[0].img_cache.cached_image_indices.iter()
    .position(|&idx| idx == current_index as isize)
    .and_then(|cache_idx| app.panes[0].img_cache.cached_data.get(cache_idx))
    .and_then(|opt| opt.as_ref())
    .map(|cached_data| (cached_data.width(), cached_data.height()));

// NEW: Only render annotations if we have the correct image dimensions
if let Some(dims) = image_size {
    log::debug!("UI: Using cached dimensions for current_index={}: {:?}", current_index, dims);

    // Check if this image has invalid annotations
    let has_invalid = app.annotation_manager.has_invalid_annotations(&filename);

    // Create bbox/mask overlay
    let bbox_overlay = crate::coco::overlay::render_bbox_overlay(
        annotations,
        dims,  // Use confirmed correct dimensions
        app.panes[0].zoom_scale,
        app.panes[0].zoom_offset,
        app.panes[0].show_bboxes,
        app.panes[0].show_masks,
        has_invalid,
        app.coco_mask_render_mode,
        app.coco_disable_simplification,
    );

    // Stack image and annotations
    container(
        Stack::new()
            .push(base_image_widget)
            .push(bbox_overlay)
    )
    .width(Length::Fill)
    .height(Length::Fill)
    .padding(0)
} else {
    // Image not in cache yet - skip annotations for this frame
    log::debug!("UI: Image at current_index={} not in cache, skipping annotations", current_index);
    container(base_image_widget)
        .width(Length::Fill)
        .height(Length::Fill)
        .padding(0)
}
```

**Pros**:
- Guarantees annotations always match the displayed image
- Simple and safe
- No wrong data ever shown to user

**Cons**:
- Annotations may briefly disappear during navigation
- User sees flickering (image without annotations → image with annotations)

### Option 2: Use Scene/Current_Image Index (BETTER)

**Strategy**: Track which index the current_image/scene contains, and fetch annotations for that index instead of current_index.

**Required Changes**:

1. Add index tracking to Pane struct:
```rust
// In src/pane.rs
pub struct Pane {
    // ... existing fields
    pub current_image_index: Option<usize>,  // NEW: Track which index current_image contains
    // ...
}
```

2. Update index when image is loaded:
```rust
// Wherever current_image is updated (e.g., in loading_handler.rs)
app.panes[0].current_image = new_texture;
app.panes[0].current_image_index = Some(loaded_index);  // NEW: Store the index
```

3. Use current_image_index for annotations:
```rust
// In src/ui.rs line 365
// OLD:
let current_index = app.panes[0].img_cache.current_index;

// NEW:
let annotation_index = app.panes[0].current_image_index
    .unwrap_or(app.panes[0].img_cache.current_index);

if let Some(path_source) = app.panes[0].img_cache.image_paths.get(annotation_index) {
    let filename = path_source.file_name();

    if let Some(annotations) = app.annotation_manager.get_annotations(&filename) {
        // Use current_image dimensions (which now match the annotation_index)
        let image_size = (
            app.panes[0].current_image.width(),
            app.panes[0].current_image.height()
        );
        log::debug!("UI: Using current_image dimensions for index {}: {:?}", annotation_index, image_size);

        // ... render annotations
    }
}
```

**Pros**:
- Annotations always match displayed image
- No flickering - annotations always rendered
- More responsive UX

**Cons**:
- Requires tracking extra state
- Need to update index in multiple places
- More code changes

### Option 3: Hybrid Approach (RECOMMENDED)

**Strategy**: Combine both approaches - prefer Option 2 when available, fall back to Option 1 when not.

```rust
// Try to get the actual displayed image's index
let displayed_image_index = if app.use_slider_image_for_render {
    // During slider: slider_image might not have index tracking yet
    // Use current_index but verify cache
    Some(app.panes[0].img_cache.current_index)
} else {
    // Normal mode: use tracked index from current_image
    app.panes[0].current_image_index
        .or(Some(app.panes[0].img_cache.current_index))
};

if let Some(ann_index) = displayed_image_index {
    if let Some(path_source) = app.panes[0].img_cache.image_paths.get(ann_index) {
        let filename = path_source.file_name();

        if let Some(annotations) = app.annotation_manager.get_annotations(&filename) {
            // Verify we have correct dimensions
            let image_size = if app.use_slider_image_for_render {
                // Slider mode: use slider_image_dimensions if available
                app.panes[0].slider_image_dimensions
            } else {
                // Normal mode: look up from cache
                app.panes[0].img_cache.cached_image_indices.iter()
                    .position(|&idx| idx == ann_index as isize)
                    .and_then(|cache_idx| app.panes[0].img_cache.cached_data.get(cache_idx))
                    .and_then(|opt| opt.as_ref())
                    .map(|cached_data| (cached_data.width(), cached_data.height()))
            };

            if let Some(dims) = image_size {
                // We have correct dimensions - render annotations
                log::debug!("UI: Rendering annotations for index {} with dims {:?}", ann_index, dims);
                // ... render
            } else {
                // No dimensions - skip annotations
                log::debug!("UI: No dimensions available for index {}, skipping annotations", ann_index);
                // ... skip
            }
        }
    }
}
```

## Recommended Implementation: Option 1 (Quick Fix)

For the RLE mask PR, implement **Option 1** as a quick, safe fix:

**Changes to src/ui.rs**:

```rust
// Replace lines 393-423 with:

let image_size_opt = app.panes[0].img_cache.cached_image_indices.iter()
    .position(|&idx| idx == current_index as isize)
    .and_then(|cache_idx| app.panes[0].img_cache.cached_data.get(cache_idx))
    .and_then(|opt| opt.as_ref())
    .map(|cached_data| (cached_data.width(), cached_data.height()));

if let Some(image_size) = image_size_opt {
    log::debug!("UI: Using cached dimensions for current_index={}: {:?}", current_index, image_size);

    // Check if this image has invalid annotations
    let has_invalid = app.annotation_manager.has_invalid_annotations(&filename);

    // Create bbox/mask overlay
    let bbox_overlay = crate::coco::overlay::render_bbox_overlay(
        annotations,
        image_size,
        app.panes[0].zoom_scale,
        app.panes[0].zoom_offset,
        app.panes[0].show_bboxes,
        app.panes[0].show_masks,
        has_invalid,
        app.coco_mask_render_mode,
        app.coco_disable_simplification,
    );

    // Stack image and annotations
    container(
        Stack::new()
            .push(base_image_widget)
            .push(bbox_overlay)
    )
    .width(Length::Fill)
    .height(Length::Fill)
    .padding(0)
} else {
    // Image dimensions not available in cache - skip annotations
    log::warn!("UI: Dimensions for current_index={} not in cache, skipping COCO annotations to avoid mismatch", current_index);
    container(base_image_widget)
        .width(Length::Fill)
        .height(Length::Fill)
        .padding(0)
}
```

**Testing**:
1. Rapidly scrub slider
2. Release slider
3. Navigate with keyboard
4. **Expected**: Annotations may briefly disappear, then reappear when image loads
5. **Verify**: Annotations ALWAYS match the visual image when displayed
6. **Check logs**: Should see "skipping COCO annotations to avoid mismatch" warnings

## Future Improvement: Option 2

After the RLE PR is merged, implement **Option 2** as a follow-up to eliminate flickering:

1. Add `current_image_index: Option<usize>` to Pane struct
2. Update index wherever current_image is set
3. Use current_image_index for annotation lookup
4. Remove the cache lookup fallback

## Timeline

- **Immediate**: Implement Option 1 for RLE mask PR
- **Post-merge**: Consider Option 2 for better UX
- **Related**: This fix will also help with the slider rendering loop issue

---

**Status**: Ready for implementation
**Priority**: CRITICAL - Must fix before RLE mask PR merge
**Estimated effort**: 30 minutes (Option 1), 2 hours (Option 2)
