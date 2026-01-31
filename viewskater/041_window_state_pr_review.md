# PR #61 Review: Window State Save/Restore

## PR Overview
- **PR:** https://github.com/ggand0/viewskater/pull/61
- **Contributor:** hml-pip
- **Feature:** Automatic window position, size, and fullscreen state persistence

## What the PR Does
1. Saves window position, size, and fullscreen state when app exits
2. Restores these values on next startup
3. Handles edge case where saved window size exceeds current monitor (maximizes instead)

## Files Changed
- `src/app.rs` - Added `window_size` and `window_position` fields to DataViewer
- `src/app/keyboard_handlers.rs` - Saves state before Cmd+Q quit
- `src/app/message.rs` - Added `SaveWindowState`, `PositionChanged` messages
- `src/app/message_handlers.rs` - Implements save/restore handlers
- `src/config.rs` - Added position/fullscreen to static CONFIG
- `src/main.rs` - Restores window state on startup
- `src/settings.rs` - Added fields to UserSettings, fixed comment typo (MB not bytes)

## Testing Results

### macOS
- Original PR code: Window position worked, but window size/zoom wasn't retained
- `.with_position()` alone doesn't work on macOS
- Final solution: Use native `setFrameAutosaveName` API
- Works: position, size, double-click zoom, green button maximize

### Linux X11
Found multiple issues:

1. **First launch worked, second launch failed**
   - Window opened on 2nd monitor first time
   - Second launch opened at (0, 0) on main monitor

2. **Debug investigation revealed:**
   - `PositionChanged` events DO fire when window is moved
   - `SaveWindowState` receives correct position
   - Settings file gets correct values
   - But window still opens at wrong position on relaunch

## Bugs Found

### Bug 1: `window_position` initialized to (0,0)

**Location:** `src/app.rs:231`

**Problem:**
```rust
window_position: PhysicalPosition { x: 0, y: 0 },  // BUG
```
Should be:
```rust
window_position: PhysicalPosition {
    x: crate::config::CONFIG.window_position_x,
    y: crate::config::CONFIG.window_position_y
},
```

**Why it caused issues:**
- App state starts at (0,0)
- `WindowEvent::Moved` doesn't fire immediately on X11
- If user closes before Moved event fires, (0,0) gets saved
- Next launch opens at (0,0)

**Why it worked on Windows/macOS:**
- `WindowEvent::Moved` fires immediately and reliably on these platforms
- `app.window_position` gets updated before user can close

**Status:** Committed as `2dbf65a`

### Bug 2: `set_outer_position()` doesn't work on X11

**Location:** `src/main.rs:1130`

**Original code:**
```rust
window.set_outer_position(PhysicalPosition::new(CONFIG.window_position_x, CONFIG.window_position_y));
```

**Problem:** On X11, calling `set_outer_position()` after window creation is unreliable. The window manager may ignore requests made before the window is mapped.

**Evidence:**
- https://github.com/rust-windowing/winit/issues/978
- User John-Nagle: "When a window is created, its initial position is (0, 0) until the window manager updates this with the X server."

**Fix:** Use `.with_position()` in WindowAttributes during window creation:
```rust
.with_position(PhysicalPosition::new(CONFIG.window_position_x, CONFIG.window_position_y))
```

### Bug 3: `set_maximized()` doesn't work on X11

**Location:** `src/main.rs:1137`

**Original code:**
```rust
if CONFIG.window_width >= size.width && CONFIG.window_height > (size.height - 80) {
    window.set_maximized(true);
}
```

**Problem:** Same as above - `set_maximized()` after window creation is ignored on X11.

**Debug logs showed:**
```
Monitor size: PhysicalSize { width: 2560, height: 1440 }, Window config: 2873x1527
Window too large, maximizing
```
The condition triggered, but window wasn't maximized.

**Evidence:** https://github.com/rust-windowing/winit/issues/2360
- User John-Nagle: "Tried, after the window was open, calling window.set_maximized(true) No effect on Linux 20.04 LTS with GNOME/X11."

**Fix:** Use `.with_maximized()` in WindowAttributes. Requires computing the maximize condition before window creation:
```rust
let monitor_size = event_loop.available_monitors().next().unwrap().size();
let maximized = CONFIG.window_width >= monitor_size.width && CONFIG.window_height > (monitor_size.height - 80);
// Then in WindowAttributes:
.with_maximized(maximized)
```

## Why X11 Behaves Differently

Windows processes window API calls synchronously - when you call SetWindowPos() or similar Win32 APIs, they execute immediately.

X11 uses client-server architecture - the app sends requests to the X server/window manager (a separate process). Requests made before the window is mapped may be queued, ignored, or overridden by the WM's placement policy.

Setting attributes during window creation (`.with_position()`, `.with_maximized()`) works because they become part of the window's initial state that the WM sees when mapping.

## The `-80` Magic Number

```rust
CONFIG.window_height > (size.height - 80)
```

**Purpose:** Buffer for taskbar/panel height. Detects "is this window basically fullscreen-sized?" and maximizes properly instead of having a slightly-too-large window.

**Cross-platform behavior:**
- Tested with value `0` on Linux - still works
- The contributor added it as a Windows taskbar workaround
- Works as a heuristic threshold on all platforms

**Contributor's explanation:** "It was a dirty hack to prevent the window from overlapping the Windows taskbar."

**Status:** Asked contributor what value works on Windows. May need documentation or constant.

## Final Changes

### Committed
1. `src/app.rs`: Initialize `window_position` from CONFIG instead of (0,0)
   - Commit: `2dbf65a`

2. `src/main.rs`: Add `.with_position()` and `.with_maximized()` for X11 compatibility
   - Commit: `9fdcf72`
   - +4 lines, -1 line

3. `src/main.rs`: Fix window position restore on macOS with platform-specific handling
   - Commit: `7fbcd5f`
   - Added back `set_outer_position()` for macOS/Windows
   - X11 uses `.with_position()`, macOS/Windows use `set_outer_position()`

4. `src/main.rs`: Use macOS native `setFrameAutosaveName` for window state persistence
   - Commit: `d2b7f4e`
   - Uses `NSWindow.setFrameAutosaveName("ViewSkaterMainWindow")`
   - macOS automatically saves/restores window frame including zoom state
   - More reliable than manual state tracking for macOS

### Platform Differences Summary
| Platform | Position | Size | Maximize/Zoom |
|----------|----------|------|---------------|
| X11 | `.with_position()` | `.with_inner_size()` | `.with_maximized()` |
| macOS | `setFrameAutosaveName` (native) | `setFrameAutosaveName` (native) | `setFrameAutosaveName` (native) |
| Windows | `set_outer_position()` | `.with_inner_size()` | `set_maximized()` |

## Open Questions for Contributor
1. What does `-80` represent? Should it be a named constant?
2. Does `0` work on Windows, or is the buffer needed there?
