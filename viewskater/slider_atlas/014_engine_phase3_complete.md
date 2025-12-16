# Phase 3 Complete: Custom Primitive Widget

**Date:** 2025-11-04
**Status:** ✅ Compiles successfully, ready for testing

---

## What Was Implemented

### 1. SliderEngineWidget (`src/widgets/slider_engine_widget.rs`)
A custom shader widget that uses SliderEngine for rendering:

```rust
pub struct SliderEngineWidget<Message> {
    pane_idx: usize,
    image_idx: usize,
    width: Length,
    height: Length,
    content_fit: ContentFit,
    engine: Arc<Mutex<SliderEngine>>,  // Direct reference to engine
    _phantom: PhantomData<Message>,
}
```

**Key Design:** Instead of using iced's shader storage, the widget directly holds a reference to SliderEngine via `Arc<Mutex<>>`. This simplifies access and avoids storage lifetime issues.

### 2. SliderEnginePrimitive
Custom primitive that implements `shader::Primitive` trait:

```rust
impl shader::Primitive for SliderEnginePrimitive {
    fn prepare(&self, ...) {
        // Nothing to do - SliderEngine.prepare() already called in main loop
    }

    fn render(&self, encoder, ...) {
        // Lock engine and render directly
        if let Ok(engine_guard) = self.engine.lock() {
            engine_guard.render(key, encoder, target, clip_bounds);
        }
    }
}
```

**Architecture:**
- `prepare()`: Does nothing (upload already handled in main loop)
- `render()`: Locks SliderEngine and calls its `render()` method
- No separate render pass creation - uses encoder from iced

### 3. UI Integration (`src/ui.rs`)
Replaced `SliderImageShader` with `SliderEngineWidget`:

```rust
if app.is_slider_moving {
    center(
        crate::widgets::slider_engine_widget::slider_engine_image::<_, WinitTheme>(
            app.panes[0].pane_id,
            pos,
            app.slider_engine.clone(),  // Pass engine reference
        )
        .width(Length::Fill)
        .height(Length::Fill)
        .content_fit(iced_winit::core::ContentFit::Contain)
    )
} else {
    // ... normal viewer widget
}
```

---

## Rendering Flow

### Complete Frame Pipeline
```
1. User moves slider
   ↓
2. Async image load
   ↓
3. Message handler queues upload (instant)
   ↓
4. NEXT FRAME:
   ├─ SliderEngine.prepare() in main loop (max 1 upload)
   ├─ iced builds UI tree
   ├─ SliderEngineWidget::draw() called
   ├─  ├─ Draws primitive
   ├─  └─ iced's renderer calls prepare() (no-op)
   ├─ iced's renderer calls render() on all primitives
   ├─  └─ SliderEnginePrimitive.render() → SliderEngine.render()
   ├─ ONE encoder with all commands
   └─ ONE queue.submit() for everything
```

###  Key Benefits
- **Non-blocking uploads**: Queued instantly, processed incrementally
- **Batched GPU work**: All commands in one submission
- **No redundant render passes**: SliderEngine renders into iced's encoder
- **Clean separation**: Upload logic in engine, rendering in widget

---

## Files Changed in Phase 3

| File | Changes | LOC |
|------|---------|-----|
| `src/widgets/slider_engine_widget.rs` | Created custom widget + primitive | +180 |
| `src/widgets/mod.rs` | Exported new widget | +1 |
| `src/ui.rs` | Replaced SliderImageShader with SliderEngineWidget | ~15 |
| `src/slider_engine/engine.rs` | Fixed `render()` signature (immutable) | ~5 |
| `src/slider_engine/cache.rs` | Made `get()` immutable | ~5 |
| `src/slider_atlas/pipeline.rs` | Added `indices()` accessor | +3 |

**Total:** ~210 new/changed lines

---

## Technical Details

### Why Not Use Shader Storage?
Initial approach tried storing SliderEngine in iced's `shader::Storage`:
```rust
// ❌ Doesn't work - Renderer doesn't have store() method
renderer.store(engine);
```

**Solution:** Pass engine directly through widget/primitive via `Arc<Mutex<>>`. This:
- Avoids lifetime issues
- Simplifies access
- Matches Rust's ownership model

### Immutable Render
SliderEngine's `render()` is immutable (`&self`) because:
- Called from widget's `render()` which receives `&self`
- Only reads from cache (no mutations)
- Cache updates happen in `prepare()`, not `render()`

### Type Annotations
Helper function requires explicit theme annotation:
```rust
slider_engine_image::<_, WinitTheme>(pane_idx, pos, engine)
```

This is necessary because Rust can't infer the theme type from context.

---

## Testing Checklist

### Basic Functionality
- [ ] Load directory
- [ ] Start moving slider
- [ ] Verify `is_slider_moving` triggers widget
- [ ] Check images appear during movement
- [ ] Stop slider - verify switch back to normal viewer

### Rendering Quality
- [ ] Check atlas upload logs
- [ ] Verify images render correctly
- [ ] Test with various image sizes
- [ ] Check 4k images (should show first fragment only)

### Performance
- [ ] Measure FPS during slider movement
- [ ] Check GPU memory usage
- [ ] Verify no blocking (fast slider movement)
- [ ] Monitor upload queue length

### Edge Cases
- [ ] Empty directory
- [ ] Single image
- [ ] Corrupted images
- [ ] Very fast slider movement
- [ ] Directory change during movement

---

## Known Limitations

1. **Fragmented images show only first fragment**
   - **Impact:** 4k images show partial view
   - **Status:** Expected behavior for Phase 3
   - **Fix:** Will be addressed in future phases

2. **No FPS tracking for SliderEngine**
   - **Impact:** Can't measure slider-specific performance
   - **Status:** Old `SliderImageShader` FPS code removed
   - **Fix:** Could add similar tracking if needed

3. **Single pane only**
   - **Impact:** Dual pane mode might not work
   - **Status:** Hardcoded to pane 0
   - **Fix:** Will be addressed in multi-pane support phase

---

## Success Criteria ✅

- [x] Compiles without errors
- [x] Widget properly implements iced traits
- [x] Primitive correctly implements shader::Primitive
- [x] UI integration complete
- [x] Rendering pipeline in place
- [ ] Runtime testing pending

---

## Next Steps

After testing Phase 3:
1. **Test basic rendering**: Does it show images?
2. **Measure performance**: Is it faster than old approach?
3. **Check stability**: Any crashes or visual glitches?

If successful, consider:
- Adding slider-specific FPS tracking
- Implementing full fragmented rendering
- Supporting dual pane mode
- Optimizing upload strategy

---

## Commit Message

```
feat(slider): implement custom widget using SliderEngine

Create SliderEngineWidget that renders via SliderEngine:
- Custom primitive directly holds engine reference via Arc<Mutex<>>
- prepare() is no-op (uploads handled in main loop)
- render() locks engine and calls its render() method
- Replace SliderImageShader with SliderEngineWidget in UI

Widget integrates seamlessly with iced's rendering:
- Uses shared encoder (no redundant render passes)
- Batched with UI rendering in single queue.submit()
- Non-blocking uploads with incremental processing

Ready for testing with real images.
```

