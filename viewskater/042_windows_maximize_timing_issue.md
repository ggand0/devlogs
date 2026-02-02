# Windows Maximize Event Timing Issue

## Problem

On Windows, when user clicks maximize button:
1. `PositionChanged(0, 0, is_maximized=false)` fires first
2. Position (0, 0) gets saved, overwriting the pre-maximize position
3. `WindowResized` fires later with correct `is_maximized=true`

Result: After maximize → quit → relaunch → unmaximize, window goes to (0, 0) instead of original position.

## Root Cause

winit has no dedicated `WindowEvent::Maximized` or `WindowEvent::Minimized` events.

- Issue: https://github.com/rust-windowing/winit/issues/1578
- Status: Open, "needs discussion" label
- Latest winit (0.30.12) still doesn't have these events

Developers must use `Resized` event as workaround, which has timing issues.

## Current Code

```rust
Message::PositionChanged(position, is_maximized) => {
    if !is_maximized && app.window_state == WindowState::Window {
        app.window_position = position;
    }
    Task::none()
}
```

The check `!is_maximized` fails because `is_maximized` is still `false` when the event fires during maximize transition.

## Additional Findings

From winit docs:
- `set_outer_position()` automatically un-maximizes the window if it's maximized
- `request_inner_size()` automatically un-maximizes the window if it's maximized

This means restore logic must apply maximized state AFTER setting position/size.

## Possible Solutions

### 1. Track `last_windowed_position` separately
- Add `last_windowed_position` to app state
- Only update when certain we're in windowed mode (not during transitions)
- Use this for config save instead of `window_position`

### 2. Debounce position saves
- Don't save position immediately on `PositionChanged`
- Wait until state has been stable for N milliseconds
- More complex, adds latency

### 3. Size-based heuristic
- Skip saving position if window size equals monitor size (maximized)
- Fragile on multi-monitor setups

### 4. Accept the limitation
- Keep the TODO comment
- Document as known issue
- Low impact edge case

## Decision

[TBD - pending discussion with contributor]

## References

- https://github.com/rust-windowing/winit/issues/1578 (No maximize/minimize events)
- https://github.com/rust-windowing/winit/issues/1765 (Fullscreen doesn't preserve state)
- https://github.com/rust-windowing/winit/issues/2360 (with_maximized issues on X11)
- https://docs.rs/winit/latest/winit/event/enum.WindowEvent.html (0.30.12 - no Maximized event)
