# Slider Atlas Performance Investigation

**Date:** 2025-11-03

## Summary
Investigation into performance bottlenecks of the new slider atlas rendering system. The slider is significantly slower than the original iced `Viewer` widget-based implementation, experiencing lag and inability to keep up with slider UI movement.

## Background
After implementing the slider atlas integration (Phase 1-5), the slider rendering works but is unacceptably slow for 1080p images. The system sometimes cannot render fast enough to keep up with slider movement, causing visual lag and in extreme cases overwhelming the message queue.

## Performance Measurements

### FPS Tracking Implementation
Added performance tracking similar to `iced_wgpu` fork:
- `SliderPerfTracker` struct with FPS calculation
- Timing for `prepare()` and `render()` phases
- Logs every 50 frames

### Key Findings from Logs

**Performance Metrics (Frame 1650):**
```
SLIDER PERF: Frames: 1650, FPS: 54.6, Prepare: 30.58ms, Render: 0.06ms
```

**Critical Discovery:**
- **FPS: 54.6** - Below target 60 FPS
- **Prepare: 30.58ms** - **BOTTLENECK IDENTIFIED** ⚠️
- **Render: 0.06ms** - Fast (not the bottleneck)

**Bottleneck Analysis:**
The initial hypothesis that render pass creation was the bottleneck was **INCORRECT**. The actual bottleneck is in the `prepare()` phase, which takes 30.58ms on average - over **500x slower** than render (0.06ms).

### Render Pass Creation Investigation

Traced through iced's rendering architecture to understand the performance difference:

**iced's Built-in Image Pipeline (Fast):**
1. Uses shared render pass across all images in a layer
2. Instanced rendering - draws 0..6 vertices × N instances in ONE draw call
3. Integrated with `image_pipeline.render()` which receives pre-created render pass

**Our Custom Slider Shader (Slow):**
1. Each custom primitive creates its own render pass (by design in iced)
2. Separate draw call per image
3. `encoder.begin_render_pass()` called every frame at `pipeline.rs:234`

**Finding:** While render pass creation adds overhead (~0.06ms), it's not the primary bottleneck. The 30ms prepare time is the real issue.

## Observed Behavior

### Caching is Working
- LRU cache (50 images) successfully caching GPU resources
- Logs show "Reusing cached GPU resources" repeatedly
- Cache eviction working: `Evicted slider resources for pane=0, image=93`

### Atlas System is Working
- Images uploading to atlas: 59 layers allocated
- Uncompressed (RGBA8) uploads working
- Atlas entries being reused correctly

### Performance Degrades Over Time
- Initial slider movement: Acceptable
- After sustained use: Message queue overwhelms system
- Extreme case: Visual artifacts on OS level, requiring machine restart

## Potential Causes of 30ms Prepare Time

1. **Atlas Upload Overhead** (Most Likely)
   - Even with caching, new images take 30-50ms to upload
   - Log shows: `Uploading image to atlas` followed by delays
   - May be synchronous texture uploads blocking prepare()

2. **Vertex Buffer Creation**
   - Creating new vertex buffers each frame for new images
   - NDC coordinate calculation on CPU

3. **Storage Lookups**
   - Multiple `storage.get()` calls per prepare
   - HashMap lookups for registry, atlas state, pipeline

4. **Mutex Contention**
   - `SLIDER_PERF_TRACKER.lock()` at start/end of prepare/render
   - Could be slowing down with many frames

## Architecture Comparison

### Original iced Viewer Widget
- ✅ Shared render pass
- ✅ Instanced rendering
- ✅ Optimized atlas pipeline
- ✅ Fast prepare phase
- ❌ Slight "jump" when switching modes
- ❌ Requires custom iced fork for BC1

### Current Custom Slider Shader
- ✅ No "jump" between modes
- ✅ LRU cache working
- ✅ Atlas integration working
- ❌ 30ms prepare bottleneck
- ❌ New render pass per frame
- ❌ Cannot batch with other renders
- ❌ Overwhelms system under sustained use

## Code Locations

### Performance Tracking
- `/src/widgets/slider_image_shader.rs:480-580` - `SliderPerfTracker`
- Instrumented in `prepare()` at lines 151-153, 276-278
- Instrumented in `render()` at lines 289-291, 320-322

### Bottleneck Locations
- `/src/widgets/slider_image_shader.rs:141-279` - `prepare()` method (30ms)
- `/src/slider_atlas/atlas.rs:113-213` - Atlas upload logic
- `/src/slider_atlas/pipeline.rs:234` - Render pass creation (0.06ms)

## Next Steps

### Immediate Actions (Post-Restart)
1. **Fix FPS Display** - Make metrics visible in F3 debug menu (not just logs)
2. **Profile prepare()** - Add timing for individual operations:
   - Atlas state checks
   - Registry lookups  
   - Vertex buffer creation
   - Resource creation
   - Atlas uploads

### Investigation Priorities
1. **Atlas Upload Timing**
   - Measure time spent in `state.atlas.upload()`
   - Check if uploads are synchronous vs async
   - Compare with iced's upload strategy

2. **Prepare Phase Optimization**
   - Identify which operation(s) take 30ms
   - Consider moving work to async tasks
   - Evaluate if atlas uploads can be deferred

3. **Architecture Decision**
   - If bottleneck is fundamental to custom primitive approach
   - May need to reconsider using iced's built-in image pipeline
   - Or implement custom render pass management at higher level

## Questions to Answer

1. Why does prepare() take 30ms when resources are cached?
2. Is atlas upload happening every prepare even with cache hits?
3. Can we move atlas uploads to a background thread?
4. Should we abandon custom primitive approach and use iced's image pipeline?

## Status

**Current State:** Slider rendering functional but unacceptably slow. Performance tracking implemented. Bottleneck identified in prepare() phase (30ms), not render (0.06ms) as initially suspected.

**Blocker:** Need to profile individual operations within prepare() to identify specific 30ms bottleneck.

**Risk:** System instability under sustained slider use - message queue overflow causing OS-level issues.


