# Slider Release Interaction Fixes

**Date:** 2025-11-11

## Overview
Fixed visual jumps and interaction issues after slider release by implementing zoom/pan support directly in the Viewer widget, eliminating the need to switch between Viewer and ImageShader widgets. Added proper COCO zoom state synchronization to ensure double-click reset works correctly.

## Problem Statement

### Initial Issues
1. **Visual Jumps**: After releasing the slider, any mouse click or cursor movement caused visual jumps when switching from Viewer widget (showing slider_image) to ImageShader widget
2. **COCO Zoom State Persistence**: When zoomed in with COCO feature enabled, moving the slider and then attempting to double-click reset would fail because the COCO zoom state (`pane.zoom_scale`, `pane.zoom_offset`) persisted while the Viewer widget had no way to update it
3. **Missing Interactions**: After slider release, zoom and pan interactions didn't work because the Viewer widget was being displayed but lacked these capabilities

### Root Cause
The application used two different widgets for rendering:
- **Viewer widget**: Fast rendering for slider_image during slider movement (uses `iced::widget::image::Handle`)
- **ImageShader widget**: Full-resolution rendering with GPU shader, supports zoom/pan

Previous attempts to fix this by clearing `use_slider_image_for_render` on `ButtonPressed` or `CursorMoved` events caused visual jumps because switching widgets mid-interaction disrupted the rendering pipeline.

## Solution

### 1. Make Viewer Widget Message-Aware
Made the Viewer widget generic over `Message` type to support event callbacks:

**src/widgets/viewer.rs**:
```rust
pub struct Viewer<Handle, Message = ()> {
    // ... existing fields
    #[cfg(feature = "coco")]
    pane_index: usize,
    #[cfg(feature = "coco")]
    on_zoom_change: Option<Box<dyn Fn(usize, f32, Vector) -> Message>>,
    _phantom: std::marker::PhantomData<Message>,
}
```

### 2. Add Zoom Change Callbacks
Added methods to configure zoom change callbacks (mirroring ImageShader's approach):

```rust
#[cfg(feature = "coco")]
pub fn pane_index(mut self, pane_index: usize) -> Self {
    self.pane_index = pane_index;
    self
}

#[cfg(feature = "coco")]
pub fn on_zoom_change<F>(mut self, f: F) -> Self
where
    F: 'static + Fn(usize, f32, Vector) -> Message,
{
    self.on_zoom_change = Some(Box::new(f));
    self
}
```

### 3. Emit Zoom Change Events
Updated Viewer's event handlers to emit zoom change messages:

#### Double-Click Reset
```rust
if now.duration_since(last_click).as_millis() < threshold_ms {
    // Reset zoom and pan
    state.scale = 1.0;
    state.current_offset = Vector::default();
    state.starting_offset = Vector::default();
    state.last_click_time = None;

    // Emit zoom change event for COCO feature
    #[cfg(feature = "coco")]
    if let Some(ref on_zoom_change) = self.on_zoom_change {
        shell.publish(on_zoom_change(self.pane_index, 1.0, Vector::default()));
    }

    return event::Status::Captured;
}
```

#### Mouse Wheel Zoom
```rust
state.current_offset = Vector::new(/* calculated values */);

// Emit zoom change event for COCO feature
#[cfg(feature = "coco")]
if let Some(ref on_zoom_change) = self.on_zoom_change {
    shell.publish(on_zoom_change(self.pane_index, state.scale, state.current_offset));
}
```

#### Panning
```rust
state.current_offset = Vector::new(x, y);

// Emit zoom change event for COCO feature when panning
#[cfg(feature = "coco")]
if let Some(ref on_zoom_change) = self.on_zoom_change {
    shell.publish(on_zoom_change(self.pane_index, state.scale, state.current_offset));
}
```

### 4. Wire Up Callbacks in UI
**src/ui.rs**:
```rust
#[cfg(feature = "coco")]
{
    viewer = viewer
        .with_zoom_state(app.panes[0].zoom_scale, app.panes[0].zoom_offset)
        .pane_index(0)
        .on_zoom_change(|pane_idx, scale, offset| {
            Message::CocoAction(crate::coco::widget::CocoMessage::ZoomChanged(
                pane_idx, scale, offset
            ))
        });
}
```

### 5. Remove Visual Jump Handlers
Removed the problematic event handlers from `src/app/message_handlers.rs`:
- Removed `Event::Mouse(ButtonPressed)` handler that was clearing slider state
- Removed `Event::Mouse(CursorMoved)` handler that was clearing slider state

These handlers were causing visual jumps by switching widgets on any mouse interaction.

### 6. Update Widget Implementation
Updated the `Widget` trait implementation and `From` trait to use the new generic parameters:

```rust
impl<Msg, Theme, Renderer, Handle> Widget<Msg, Theme, Renderer>
    for Viewer<Handle, Msg>
where
    Renderer: image::Renderer<Handle = Handle>,
    Handle: Clone,
{
    // ...
}

impl<'a, Message, Theme, Renderer, Handle> From<Viewer<Handle, Message>>
    for Element<'a, Message, Theme, Renderer>
where
    Renderer: 'a + image::Renderer<Handle = Handle>,
    Message: 'a,
    Handle: Clone + 'a,
{
    // ...
}
```

## Technical Details

### State Tracking
Added `last_click_time` field to Viewer's `State` struct for double-click detection:

```rust
pub struct State {
    scale: f32,
    starting_offset: Vector,
    current_offset: Vector,
    cursor_grabbed_at: Option<Point>,
    last_click_time: Option<std::time::Instant>,
}
```

### Double-Click Detection
Uses `CONFIG.double_click_threshold_ms` (consistent with ImageShader) to detect double-clicks:
- Records timestamp on each button press
- Compares with previous timestamp
- If within threshold, treats as double-click and resets zoom/pan

### Message Flow
1. User performs zoom/pan interaction in Viewer widget
2. Viewer updates its internal state (scale, offset)
3. Viewer calls `on_zoom_change` callback with new values
4. Callback publishes `Message::CocoAction(CocoMessage::ZoomChanged(...))`
5. Message handler updates `pane.zoom_scale` and `pane.zoom_offset`
6. Next render uses updated zoom state from pane

## Results

### Fixed Issues
✅ **No Visual Jumps**: Viewer widget handles all interactions internally without switching widgets
✅ **Double-Click Reset Works**: Properly resets both Viewer's internal state and COCO's pane zoom state
✅ **Panning Works**: Click-and-drag panning works after slider release
✅ **Wheel Zoom Works**: Mouse wheel zoom works after slider release and syncs with COCO state
✅ **State Consistency**: COCO zoom state stays synchronized with Viewer's internal state

### Interaction Flow After Slider Release
1. **Slider moving**: Shows `slider_image` via Viewer widget (fast rendering)
2. **Slider released**: Continues showing `slider_image` via Viewer widget
3. **User interacts**:
   - Double-click → Resets zoom to 1.0, pan to (0,0), updates COCO state
   - Wheel scroll → Zooms in/out, updates COCO state
   - Click-drag → Pans image, updates COCO state
4. **Keyboard navigation**: Clears slider state, switches to ImageShader for full-resolution rendering

### Keyboard Navigation Remains Unchanged
The following interactions still clear slider state and switch to ImageShader:
- Arrow keys / A/D keys
- Shift+arrow (skate mode)
- Cmd/Ctrl+arrow (jump to first/last)
- Mouse wheel navigation (without Ctrl)
- Ctrl+wheel zoom (when enabled)

## Files Modified

### Core Changes
- **src/widgets/viewer.rs** (459 lines → 600 lines)
  - Made generic over Message type
  - Added COCO zoom state callback mechanism
  - Added zoom change event emissions
  - Added double-click detection

- **src/ui.rs**
  - Wired up zoom change callbacks for Viewer widget
  - Added pane_index configuration

- **src/app/message_handlers.rs**
  - Removed ButtonPressed handler
  - Removed CursorMoved handler

### Supporting Files
- **commit_msg.txt**: Updated commit message
- **devlogs/111125_slider_release_interaction_fixes.md**: This document

## Benefits

1. **Architectural Consistency**: Viewer widget now has feature parity with ImageShader for event handling
2. **No Widget Switching**: Eliminates visual artifacts from mid-interaction widget changes
3. **State Synchronization**: Proper bidirectional sync between widget state and app state
4. **Code Reusability**: Zoom change callback pattern can be extended to other features
5. **User Experience**: Smooth, responsive interactions with no visual glitches

## Future Considerations

- The Viewer widget's zoom/pan functionality could be extracted into a shared trait
- Consider unifying the zoom state management between Viewer and ImageShader
- Could add additional interaction modes (e.g., click-to-zoom-point)
