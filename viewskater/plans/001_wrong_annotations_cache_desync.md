# Wrong Annotations Bug - Image/Annotation Cache Desynchronization

**Date:** 2025-11-16
**Status:** Critical Bug - Affects Correctness
**Severity**: High (wrong data displayed to user)
**Branch**: feat/rle-mask-support

## Executive Summary

After slider release and keyboard navigation, the app sometimes renders annotations from the correct image index but overlays them on a different image. This is caused by a desynchronization between the image cache and COCO annotation lookup during navigation transitions.

## Bug Description

### Symptoms
- User scrubs slider rapidly, then releases
- Navigates a few images to the right using keyboard
- App displays wrong annotations:
  - **Example**: Window title shows `000000372307.jpg` (horse image, index 3226)
  - **Visual**: Displays pizza image from `000000369757.jpg` (index 3203)
  - **Annotations**: Shows horse annotation (correct for index 3226)
  - **Result**: Horse annotation rendered on top of pizza image!
- Annotations are correctly sized for their actual image
- But rendered on top of a different image from an earlier index

### Reproduction Steps
1. Load COCO dataset (e.g., coco-rem at /home/gota/ggando/rust_gui/data/ml/val2017/)
2. Rapidly scrub slider back and forth
3. Release slider
4. Press 'D' key to navigate right 1-2 images
5. **Observe**: Annotations from one image rendered on a different image

## Root Cause Analysis

### Log Evidence (logs/111625_wrong_anno_after_sliderreleased.log)

**The smoking gun:**

```
Line 535: END RENDERING NEXT: current_index: 3226, current_offset: 1
Line 549: UI: Looking up image size from cache for current_index=3226
Line 550: UI: Using fallback dimensions from current_image: (640, 480)  ← STALE!
Line 553: MaskShader: Rebuilding quads for 1 annotations (state changed)
Line 555: MaskShader: Found RLE mask, size: [428, 640]  ← CORRECT annotation for 3226
Line 559: BBox prepare: image=(640,480)  ← WRONG image dimensions!
```

**What should have happened:**
- Index 3226 = 000000372307.jpg = 640×428 image with 1 horse annotation
- Annotation size [428, 640] is correct for this image
- But image dimensions used are (640, 480) from a DIFFERENT image!

**What actually happened (verified):**
- Window title: `000000372307.jpg` (index 3226) ✅ CORRECT
- Annotations: Horse (correct for index 3226) ✅ CORRECT
- Visual image displayed: Pizza image from `000000369757.jpg` (index 3203) ❌ WRONG
- Image dimensions used: (640, 480) from index 3203's cached data ❌ WRONG

**Verification from COCO JSON:**
```bash
Image 000000372307.jpg:
  - Image ID: 372307
  - Actual size: 640×428
  - Annotations: 1 horse (cat_id=19), bbox=[222, 346, 47, 43], RLE size=[428, 640]
```

### The Desynchronization

**Three separate data sources with different states:**

1. **Window Title** (CORRECT):
   - Uses `current_index` (3226)
   - Shows: `000000372307.jpg` ✅

2. **Annotation Lookup** (CORRECT):
   - Uses `current_index` (3226)
   - Fetches annotations for 000000372307.jpg
   - Returns horse annotation with size [428, 640] ✅

3. **Visual Image Display** (WRONG):
   - Uses `current_image` from GPU texture cache
   - Still showing: 000000369757.jpg (index 3203, pizza image) ❌
   - Dimensions: (640×480) from the pizza image ❌

**The critical issue**: `current_index` has been updated to 3226, but `current_image` texture hasn't been updated yet and still contains index 3203's data.

**Timeline of the bug:**

```
T0: User rapidly scrubbing slider
    - Passes through many images including index 3203 (000000369757.jpg - pizzas)
    - current_image GPU texture = 000000369757.jpg (640×480)
    - Cache has index 3203 data

T1: User releases slider at position ~3200
    - Slider stops around index 3200
    - Full resolution image loaded for that position

T2: User presses 'D' to move right
    - current_index updated to 3226
    - Window title updates to 000000372307.jpg ✅
    - render_next_image_all() called

T3: Image cache lookup
    - Tries to render image at index 3226
    - Index 3226 NOT in cache yet (async loading hasn't completed)
    - current_image GPU texture STILL contains index 3203 (pizza image) ❌

T4: UI render triggered
    - Window title shows: 000000372307.jpg (from current_index=3226) ✅
    - Looks up image size for index 3226 from cache → MISS
    - Falls back to current_image dimensions: (640, 480) from index 3203 ❌

T5: COCO annotation lookup
    - Correctly fetches annotations for current_index=3226
    - Gets horse annotation with RLE size [428, 640] ✅

T6: Rendering
    - Visual: Displays current_image GPU texture = 000000369757.jpg (pizzas) ❌
    - Overlays: Renders horse annotation (correct for 000000372307.jpg) ✅
    - Scaling: Uses dimensions (640×480) from pizza image ❌
    - Result: Horse annotation on pizza image!
```

### Why This Happens

**The race condition:**

1. Navigation updates `current_index` immediately
2. Image cache loading is async
3. UI renders before image cache is updated
4. Annotation lookup uses new `current_index`
5. Image dimensions use old `current_image`
6. **Mismatch!**

## Code Analysis

### Where Annotations Are Looked Up

Need to examine where COCO annotations are fetched during rendering. The annotation lookup likely happens in the UI rendering code where it queries based on `current_index`, but image dimensions come from a different source.

**Likely location**: Somewhere in the rendering pipeline where:
```rust
// Annotation lookup (uses current_index)
let annotations = get_annotations_for_image(current_index);

// Image dimensions (uses current_image)
let image_dims = current_image.dimensions(); // ← STALE during transition!

// Render (mismatch!)
render_annotations(annotations, image_dims);
```

### The Fallback Mechanism

From line 550: "Using fallback dimensions from current_image"

This fallback is intended to handle cases where image size isn't in cache yet, but it creates a desynchronization bug when:
- Annotations are fetched for NEW index
- Image dimensions are from OLD index

## Impact Analysis

### User Impact
- **Data Integrity**: Users see incorrect annotations on images
- **Trust**: Undermines confidence in the application
- **Workflows**: Annotation verification workflows are broken
- **Severity**: HIGH - This is a correctness bug, not just performance

### When It Occurs
- Most likely during rapid navigation after slider scrubbing
- Happens when image cache hasn't caught up with navigation
- More common with:
  - Large datasets
  - Rapid slider movements
  - Keyboard navigation immediately after slider release

### Scope
- Affects ALL annotation types (bboxes, masks, polygons)
- Not specific to RLE masks
- Pre-existing bug made more visible by RLE implementation

## Proposed Fix

### Strategy: Synchronized Lookup

**Ensure annotations and image dimensions always come from the same source:**

```rust
// Option 1: Use the ACTUAL loaded image for both
let current_image = get_current_image(); // The actual displayed image
let image_index = current_image.index;
let annotations = get_annotations_for_index(image_index);
let dimensions = current_image.dimensions();

// Option 2: Wait for cache to be consistent
if !is_image_loaded(current_index) {
    // Don't render annotations until image is loaded
    return;
}
let annotations = get_annotations_for_index(current_index);
let dimensions = get_image_dimensions_from_cache(current_index);
```

### Recommendation

**Option 2 is safer**: Don't render COCO annotations until the corresponding image is fully loaded and dimensions are available.

**Implementation:**
1. Check if image at `current_index` is loaded before fetching annotations
2. If not loaded, skip annotation rendering for this frame
3. Once image loads, both dimensions and annotations will be correct

### Alternative: Store Index with Image

```rust
struct CurrentImage {
    texture: GpuTexture,
    dimensions: (u32, u32),
    index: usize,  // NEW: Track which index this image is from
}

// During annotation rendering:
let annotations = get_annotations_for_index(current_image.index); // Always synchronized!
let dimensions = current_image.dimensions;
```

This ensures annotations always match the displayed image, even during transitions.

## Testing Plan

### Reproduction Test
1. Load COCO dataset
2. Rapidly scrub slider (trigger cache churn)
3. Release slider
4. Immediately navigate with keyboard
5. **Verify**: Annotations match the displayed image

### Validation
- Check that image filename matches annotation content
- Verify annotation count matches expected for that image
- Compare bbox positions with visual image content
- Test with different navigation patterns:
  - Slider → keyboard
  - Rapid keyboard navigation
  - Jump to random positions

### Regression Testing
- Ensure normal navigation still works
- Verify no performance degradation
- Test with various dataset sizes

## Implementation Priority

**Priority**: **CRITICAL** - Must fix before merging RLE mask PR

**Rationale**:
- This is a correctness bug (wrong data shown)
- Affects user trust and data integrity
- Not a performance issue - this is functional incorrectness
- Makes the RLE feature unreliable if annotations don't match images

## Related Issues

### Not Related To
- RLE mask implementation (this is a pre-existing bug)
- Cache collision fix (that was about wrong annotations for same image)
- Rendering loop bug (that's about performance, not correctness)

### Similar To
- Cache collision bug: Both involve wrong annotations, but different root causes
  - Cache collision: Wrong annotation ID cached for same image
  - This bug: Right annotation ID but wrong image being displayed

## Timeline

- **2025-11-15 23:17**: Bug discovered during RLE testing
- **2025-11-16**: Root cause identified through log analysis
- **2025-11-16**: Fix plan documented
- **TBD**: Implementation
- **TBD**: Testing and validation

## Questions for Discussion

1. Should we add an assertion that annotation index matches current_image index?
2. Should we display a warning when annotations are skipped due to cache miss?
3. Should we add telemetry to track how often this desync occurs?
4. Should we preload annotations along with images to avoid this race?

---

**Document Status**: Complete - Ready for implementation
**Next Steps**: Investigate code to find exact location of annotation/dimension lookup
**Owner**: TBD
**Blocker For**: RLE mask PR merge
