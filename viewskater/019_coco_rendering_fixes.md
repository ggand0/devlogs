# COCO Annotation Rendering Fixes

**Date:** 2025-10-31

## Issues Fixed

1. **Slider Navigation Rendering**
   - Problem: Annotations and labels wouldn't render when navigating with the slider
   - Root cause: UI used `slider_image` widget instead of `ImageShader` during slider movement
   - Solution: Refactored to create unified `base_image_widget` that works for both slider and shader paths, then stack annotations on top regardless
   - File: [src/ui.rs](../src/ui.rs)

2. **Real-time Pan Updates**
   - Problem: Annotations didn't update during pan, only on mouse release
   - Root cause: `ZoomChanged` callback only emitted on `ButtonReleased`, not during `CursorMoved`
   - Solution: Added callback emission during panning in ImageShader
   - File: [src/widgets/shader/image_shader.rs:727-732](../src/widgets/shader/image_shader.rs#L727-L732)

3. **UI Space Invasion**
   - Problem: Annotation masks and bboxes stretching into menu bar and footer
   - Root cause: Container `.clip(true)` only works for CPU widgets, not GPU shaders
   - Solution: Used WGPU scissor rectangles in both BBoxShader and PolygonShader render passes
   - Files:
     - [src/widgets/shader/bbox_shader.rs:227-232](../src/widgets/shader/bbox_shader.rs#L227-L232)
     - [src/widgets/shader/polygon_shader.rs:233-238](../src/widgets/shader/polygon_shader.rs#L233-L238)

4. **Zoom State Sync on Navigation**
   - Problem: Mask scale incorrect when zoomed in on one image, then navigating to another
   - Root cause: Zoom state in Pane wasn't syncing with ImageShader's internal state on image changes
   - Solution: Track `image_index` in ImageShader, detect changes, sync current zoom state to Pane (don't reset)
   - Files:
     - [src/widgets/shader/image_shader.rs:564-578](../src/widgets/shader/image_shader.rs#L564-L578)
     - [src/ui.rs](../src/ui.rs)

## Invalid Annotation Handling

### Implementation

Added graceful handling for invalid COCO annotations instead of failing the entire dataset load:

1. **Validation Changes**
   - Changed `validate()` to `validate_and_clean()` - filters instead of failing
   - Returns `(skipped_count, warnings, images_with_invalid_annos: HashSet<u64>)`
   - Uses `retain()` to filter out invalid annotations
   - Tracks which images had invalid data by image_id
   - File: [src/coco_parser.rs:76-129](../src/coco_parser.rs#L76-L129)

2. **Invalid Annotation Types Caught**
   - Empty bbox arrays (e.g., `bbox: []`)
   - Incorrect bbox length (must be exactly 4 values: `[x, y, width, height]`)
   - References to non-existent image_ids
   - References to non-existent category_ids

3. **Tracking Invalid Annotations**
   - Added `images_with_invalid_annos: HashSet<u64>` to `LoadedDataset` struct
   - Added `has_invalid_annotations(filename)` method to AnnotationManager
   - Propagated invalid set through entire call chain from validation to UI
   - Files:
     - [src/annotation_manager.rs:227-236](../src/annotation_manager.rs#L227-L236)
     - [src/coco_widget.rs](../src/coco_widget.rs)
     - [src/ui.rs:335-346](../src/ui.rs#L335-L346)

4. **Visual Indicator**
   - Display "WARNING: Invalid annotations skipped" in orange text
   - Shows at top of yellow category summary overlay
   - Only appears for images that had annotations filtered out
   - File: [src/bbox_overlay.rs:111-120](../src/bbox_overlay.rs#L111-L120)

### Example Invalid Annotation Found

In `/home/gota/ggando/rust_gui/data/ml/c/lowerleft/lane3_lowerleft.json`:
- Image: `lane3_lowerleft_image_0022.png` (image_id: 21)
- Annotation 2250 has empty bbox: `bbox: []`
- This is correctly caught and skipped, warning displayed

## Commits Made

1. `Fix annotation rendering during slider navigation`
2. `Emit zoom changes during pan for real-time annotation updates`
3. `Clip annotation rendering to image bounds using scissor rects`
4. `Sync zoom state on image navigation without resetting`
5. `Skip invalid COCO annotations instead of failing dataset load`
6. `Display warning for images with invalid annotations`

## Technical Notes

- WGPU scissor rectangles are necessary for GPU shader clipping (CPU `.clip(true)` doesn't work)
- Image index tracking is more reliable than scene pointer tracking for detecting navigation
- Zoom state sync on navigation preserves user's zoom level between images
- Filter pattern with `retain()` is cleaner than error propagation for data cleaning
