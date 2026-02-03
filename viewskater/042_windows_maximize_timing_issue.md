# Window Maximize/Unmaximize Timing Issues

## Problem 1: Windows Position Loss on Maximize

On Windows, when user clicks maximize button:
1. `PositionChanged(0, 0, is_maximized=false)` fires first
2. Position (0, 0) gets saved, overwriting the pre-maximize position
3. `WindowResized` fires later with correct `is_maximized=true`

Result: After maximize → quit → relaunch → unmaximize, window goes to (0, 0) instead of original position.

## Problem 2: X11 Un-maximize Detection Failure

On X11/Linux, when user un-maximizes:
1. `WindowEvent::Resized` fires with smaller size
2. But `window.is_maximized()` still returns `true`
3. `window_state` stays `Maximized` even though window is now windowed

Result: Window state saved incorrectly, app relaunches maximized when it shouldn't.

## Root Cause

winit has no dedicated `WindowEvent::Maximized` or `WindowEvent::Minimized` events.

- Issue: https://github.com/rust-windowing/winit/issues/1578
- Status: Open, "needs discussion" label
- Latest winit (0.30.12) still doesn't have these events

Developers must use `Resized` event as workaround, but `is_maximized()` has timing issues on both Windows and X11:
- **Windows**: Returns `false` during maximize transition (before state updates)
- **X11**: Returns `true` during un-maximize transition (before state updates)

## Solution Implemented

### 1. Track `last_windowed_position` separately (Windows fix)

Added `last_windowed_position` field to app state:
- Only updated when `app.window_state == WindowState::Window`
- Used for config save instead of `window_position`
- Avoids saving the (0, 0) position during maximize transition

```rust
Message::PositionChanged(position, _is_maximized) => {
    app.window_position = position;
    // Only track when certain we're in windowed state
    if app.window_state == WindowState::Window {
        app.last_windowed_position = position;
    }
    Task::none()
}
```

### 2. Size-based un-maximize detection (X11 fix)

Track the largest window size seen while maximized, detect un-maximize when size drops:

```rust
// Track the largest size seen while maximized
if is_maximized {
    let should_update = app.maximized_size.map_or(true, |max_size| {
        size.width > max_size.width || size.height > max_size.height
    });
    if should_update {
        app.maximized_size = Some(size);
    }
}

// X11 workaround: is_maximized() may still return true when un-maximizing
// Detect un-maximize by checking if size dropped significantly
let size_dropped = app.maximized_size.map_or(false, |max_size| {
    size.width < max_size.width.saturating_sub(100) ||
    size.height < max_size.height.saturating_sub(100)
});
if !is_maximized || size_dropped {
    app.window_state = WindowState::Window;
}
```

### 3. Restore maximized state on startup

Use saved `window_state` to determine if window should start maximized:

```rust
let should_maximize = CONFIG.window_state == WindowState::Maximized || size_exceeds_monitor;
```

## Files Changed

- `src/app.rs`: Added `last_windowed_position` and `maximized_size` fields
- `src/app/message_handlers.rs`: Updated WindowResized and PositionChanged handlers
- `src/main.rs`: Use saved window_state for maximize on startup

## Platform Behavior Summary

| Platform | Position Tracking | Un-maximize Detection |
|----------|------------------|----------------------|
| Windows  | `last_windowed_position` (avoids 0,0) | `is_maximized()` works |
| X11      | `last_windowed_position` | Size-based heuristic |
| macOS    | Native `setFrameAutosaveName` | Native API handles it |

## References

- https://github.com/rust-windowing/winit/issues/1578 (No maximize/minimize events)
- https://github.com/rust-windowing/winit/issues/1765 (Fullscreen doesn't preserve state)
- https://github.com/rust-windowing/winit/issues/2360 (with_maximized issues on X11)
- https://docs.rs/winit/latest/winit/event/enum.WindowEvent.html (0.30.12 - no Maximized event)
