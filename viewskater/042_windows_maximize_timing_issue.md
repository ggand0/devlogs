# Window State Save & Restore (PR #61)

## Overview

Contributor hml-pip implemented automatic window state persistence (position, size, maximized/fullscreen). Multiple platform-specific timing bugs required workarounds due to winit lacking dedicated maximize/minimize events (winit issue #1578).

## Problems & Fixes

### 1. Windows: Position Loss on Maximize

**Problem**: When maximizing, `PositionChanged(0, 0, is_maximized=false)` fires before `WindowResized(is_maximized=true)`. The (0,0) overwrites the pre-maximize position.

**Fix**: Track `position_before_transition` as a backup. When `WindowResized` detects maximize, restore from backup:

```rust
Message::PositionChanged(position, monitor) => {
    if app.window_state == WindowState::Window {
        app.position_before_transition = app.last_windowed_position;
        app.last_windowed_position = position;
    }
}

// In WindowResized, when maximize detected:
app.last_windowed_position = app.position_before_transition;
```

### 2. X11: Un-maximize Detection Failure

**Problem**: `window.is_maximized()` returns stale `true` during un-maximize transition.

**Fix**: Platform-gated (`#[cfg(target_os = "linux")]`) exact size comparison. On X11, maximized windows cannot be user-resized, so any size change while in Maximized state means un-maximize:

```rust
#[cfg(target_os = "linux")]
{
    let size_changed = app.maximized_size.map_or(false, |max_size| size != max_size);
    if size_changed {
        app.window_state = WindowState::Window;
        app.maximized_size = None;
    }
}
```

Earlier version used a 100px threshold hack that ran on all platforms — replaced with this platform-gated exact comparison.

### 3. X11: Window Attributes Must Be Set at Creation

**Problem**: `set_maximized()` and `set_outer_position()` after window creation are unreliable on X11.

**Fix**: Use `with_maximized()` and `with_position()` on `WindowAttributes` during creation, gated to Linux:

```rust
#[cfg(target_os = "linux")]
{
    window_attrs = window_attrs
        .with_maximized(CONFIG.window_state == WindowState::Maximized)
        .with_position(config_position);
}
```

### 4. macOS: Cmd+Q Bypasses Winit Event Loop

**Problem**: Cmd+Q calls `[NSApp terminate:]` → `exit()` without firing `CloseRequested` or any winit events. Window state never saved.

**Fix**: Register `NSApplicationWillTerminateNotification` observer that reads NSWindow frame directly at termination time:

```rust
// In window_state::setup_macos_window()
let block = RcBlock::new(move |_: NonNull<NSNotification>| {
    // Read isZoomed(), frame position/size directly from NSWindow
    // Save to settings
});
center.addObserverForName_object_queue_usingBlock(
    Some(NSApplicationWillTerminateNotification), None, None, &block,
);
```

### 5. macOS: Double-Click Unzoom Broken

**Problem**: winit's `set_maximized(true)` doesn't establish `_savedFrame` on NSWindow, so double-click title bar to unzoom has no frame to restore to.

**Fix**: Use `NSWindow.zoom(None)` instead, which saves the current frame to `_savedFrame` before zooming:

```rust
#[cfg(target_os = "macos")]
if CONFIG.window_state == WindowState::Maximized {
    ns_window.zoom(None);
}
```

### 6. macOS: isZoomed() Unreliable Mid-Animation

**Problem**: `isZoomed()` flickers during zoom/unzoom animation, causing incorrect state detection in `WindowResized` handler.

**Fix**: Query `isZoomed()` authoritatively at save time (post-animation) in `save_window_state_to_disk()`, overriding whatever the handler tracked:

```rust
#[cfg(target_os = "macos")]
let (window_state, pos_source) = {
    let is_zoomed = query_is_zoomed(window);
    if is_zoomed {
        (WindowState::Maximized, app.position_before_transition)
    } else {
        (WindowState::Window, app.last_windowed_position)
    }
};
```

### 7. wgpu Panic on Multi-Monitor Spanning

**Problem**: Window spanning multiple 4K monitors can exceed wgpu's 8192px max texture size, causing panic on relaunch: `Surface width and height must be within the maximum supported texture size. Requested was (9379, 5694), maximum extent is 8192.`

**Fix**: Cap window dimensions to `MAX_TEXTURE_SIZE` (8192) at window creation, initial surface config, and resize events.

### 8. Window Off-Screen After Monitor Disconnect (Contributor fix)

**Problem**: Window position saved while on secondary monitor. Monitor disconnected. Relaunch places window off-screen.

**Fix**: `get_window_visible()` checks if at least 30px of window is visible on the monitor. If not, resets to monitor origin. Applied at save time and startup (Windows).

### 9. Multi-Monitor Move While Maximized (Contributor fix)

**Problem**: Using Win+Shift+Arrow to move maximized window between monitors doesn't update saved position. Relaunch opens on wrong monitor.

**Fix**: Track `last_monitor` via `MonitorHandle`. In `PositionChanged`, update position when monitor changes even while maximized.

Note: On X11/GNOME, Super+Shift+Arrow doesn't fire `WindowEvent::Moved`, so this fix only works on Windows.

## Removed: size_exceeds_monitor Heuristic

The original `size_exceeds_monitor` check with the -80px taskbar hack was removed. Now maximize on startup is determined solely by `CONFIG.window_state == Maximized`. The window manager handles sizing if saved dimensions don't fit the current monitor.

## Save Points

Window state is saved at:
- `WindowEvent::Focused(false)` — covers alt-tab, crash recovery
- `WindowEvent::CloseRequested` — normal window close
- `Control::Exit` — programmatic exit
- `NSApplicationWillTerminateNotification` — macOS Cmd+Q (macOS only)

## Files Changed

- `src/window_state.rs` (new) — `get_window_visible()`, `save_window_state_to_disk()`, `setup_macos_window()`, `query_is_zoomed()`
- `src/app.rs` — Added `window_state`, `window_size`, `maximized_size`, `window_position`, `last_windowed_position`, `position_before_transition`, `last_monitor`
- `src/app/message.rs` — Updated `WindowResized` to include size and is_maximized, added `PositionChanged` and `SaveWindowState`
- `src/app/message_handlers.rs` — Maximize/un-maximize state tracking, position tracking, window state save handler
- `src/main.rs` — Platform-specific window creation, wgpu texture size cap, save on focus loss/close/exit
- `src/settings.rs` — Added `window_position_x`, `window_position_y`, `window_state` fields
- `src/config.rs` — Added window position and state to CONFIG
- `src/settings_modal.rs` — Removed window dimension UI inputs
- `src/ui.rs` — Use `window_state == FullScreen` instead of `is_fullscreen`
- `src/app/keyboard_handlers.rs` — Save window state before Cmd+Q quit
- `src/pane.rs` — Removed spammy debug log

## Platform Behavior Summary

| Platform | Maximize Restore | Un-maximize Detection | Position Tracking | Save on Quit |
|----------|-----------------|----------------------|-------------------|-------------|
| Windows  | `set_maximized(true)` | `is_maximized()` (reliable) | `position_before_transition` backup | CloseRequested |
| X11/Linux | `.with_maximized()` at creation | Size change detection (stale `is_maximized()` workaround) | `last_windowed_position` | CloseRequested |
| macOS | `NSWindow.zoom(None)` | `isZoomed()` at save time | `position_before_transition` | NSApplicationWillTerminateNotification |

## References

- https://github.com/rust-windowing/winit/issues/1578 (No maximize/minimize events)
- https://github.com/rust-windowing/winit/issues/1765 (Fullscreen doesn't preserve state)
- https://github.com/rust-windowing/winit/issues/2360 (with_maximized issues on X11)
