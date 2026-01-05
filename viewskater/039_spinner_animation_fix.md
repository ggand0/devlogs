# Spinner Animation Fix

## Problem
The loading spinner showed as a static frame - it only animated when the mouse cursor was moved over the application window.

## Root Cause Analysis

### Why Mouse Movement Made It Work
When the mouse moves, winit sends `WindowEvent::CursorMoved`, which triggers:
1. `state.queue_event(event)` - queues an **Event** for widgets
2. Widgets receive the event in `on_event()`
3. Render code runs with `*redraw = true`

### Why SpinnerTick Didn't Work
When SpinnerTick fires as a UserEvent:
1. `state.queue_message(message)` - only queues a **Message** for the app
2. Widgets' `on_event()` is **never called** - they don't see messages
3. The original Canvas widget cached its geometry and never invalidated

The fundamental issue: **widgets only see Events, not Messages**.

## Solution: Multi-Part Fix

### Part 1: Rewrite Circular Widget (circular.rs)

Replaced `canvas::Program` implementation with direct `Widget` trait implementation:

```rust
impl<'a, Message> Widget<Message, WinitTheme, Renderer> for Circular<'a> {
    fn draw(&self, ...) {
        // Compute animation from wall clock time on EVERY draw
        let now = Instant::now();
        let elapsed = now.duration_since(state.start_time);
        let rotation = (elapsed.as_secs_f32() / rotation_duration).fract();
        // Draw fresh geometry each frame - no caching
    }
}
```

Key changes:
- Animation computed from `Instant::now()` on every draw
- No geometry caching - fresh frame each render
- Simplified state to just `start_time: Instant`

### Part 2: Add WindowEvent::RedrawRequested Handler (main.rs:698)

Added explicit handler for RedrawRequested events inside the WindowEvent match:

```rust
WindowEvent::RedrawRequested => {
    // Queue iced's RedrawRequested event for widgets
    state.queue_event(iced_winit::core::Event::Window(
        iced_winit::core::window::Event::RedrawRequested(Instant::now())
    ));

    // Process events, render frame
    state.update(...);
    renderer.present(...);

    // Continue animation loop if spinner is active
    if state.program().is_any_pane_loading() {
        window.request_redraw();
    }
}
```

This was missing entirely - `window.request_redraw()` was being called but the event was never handled.

### Part 3: Add Animation Loop to Original Render Path (main.rs:1001)

The original render code (`if *redraw { ... }`) didn't continue the animation loop:

```rust
// After frame.present()
if state.program().is_any_pane_loading() {
    window.request_redraw();
}
```

This ensures any WindowEvent that triggers a render also continues the animation.

### Part 4: Direct Render in UserEvent Handler (main.rs:1138)

Added direct rendering when loading is active to bypass RedrawRequested event delivery issues:

```rust
// In UserEvent handler, after window.request_redraw()
if state.program().is_any_pane_loading() {
    // Render directly
    surface.get_current_texture() -> render -> present
}
```

This was needed because the initial RedrawRequested events weren't being delivered before any WindowEvent occurred.

### Part 5: Fix is_any_pane_loading() Timing (app.rs:537)

Original code only returned true after 1 second:
```rust
// WRONG - animation loop stopped during first second
pane.loading_started_at.map_or(false, |start| start.elapsed() > Duration::from_secs(1))
```

Fixed to return true as soon as loading starts:
```rust
// CORRECT - animation loop runs immediately
pane.loading_started_at.is_some()
```

The 1-second delay for *showing* the spinner is handled separately in the view code.

## Architecture Summary

```
SpinnerTick fires (every 16ms)
    ↓
UserEvent handler
    ↓
queue_message() + queue_event(RedrawRequested) + window.request_redraw()
    ↓
Direct render if is_any_pane_loading() == true
    ↓
WindowEvent::RedrawRequested received
    ↓
RedrawRequested handler: update() + render + request_redraw() if loading
    ↓
Loop continues until loading completes
```

## Files Modified

- **src/widgets/circular.rs**: Complete rewrite as Widget (not canvas::Program)
- **src/widgets/loading_overlay.rs**: Simplified, removed CircularState parameter
- **src/main.rs**: Added RedrawRequested handler, animation loop continuations, direct render
- **src/app.rs**: Added `is_any_pane_loading()` method, fixed timing logic
- **src/app/message_handlers.rs**: Fixed loading_started_at clearing logic
- **src/pane.rs**: Removed spinner_state field

## Key Insight

The iced framework's custom main loop separates Events and Messages:
- **Events** go to widgets via `on_event()`
- **Messages** go to the app via `update()`

Canvas widgets with `canvas::Program` cache geometry and only redraw on events. By rewriting as a direct Widget that computes animation from wall clock time on every `draw()`, and ensuring the main loop keeps calling `window.request_redraw()`, the animation now works.

## Commits
- `a4ae33e`: fix: spinner animation works without mouse movement
- `9483156`: fix: spinner animation starts immediately without cursor movement
