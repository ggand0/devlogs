# macOS Fullscreen Toggle Fix

**Date:** 2025-08-26

## Issue Summary

The macOS fullscreen functionality had a critical state detection bug where F11 could enter fullscreen mode but couldn't exit it. The Escape key also didn't work to exit fullscreen mode.

### Symptoms
- F11 first press: Enters fullscreen mode ✅
- F11 second press: Doesn't exit fullscreen mode ❌
- Escape key: Doesn't exit fullscreen mode ❌
- User gets stuck in fullscreen with no way to exit

## Root Cause Analysis

The issue was a **current state vs target state confusion** in the contributor's original implementation, combined with state detection incompatibility between two different fullscreen APIs on macOS.

### The Core Bug: Using Current State Instead of Target State

The contributor's code correctly calculated the target state but then used the current state instead:

```rust
// ❌ CONTRIBUTOR'S BUGGY CODE
let fullscreen = if window.fullscreen().is_some() {
    state.queue_message(Message::ToggleFullScreen(false));
    None  // Target: should EXIT fullscreen
} else {
    state.queue_message(Message::ToggleFullScreen(true));
    Some(winit::window::Fullscreen::Borderless(None))  // Target: should ENTER fullscreen
};

// But then ignore the calculated target and use CURRENT state instead!
window.set_simple_fullscreen(state.program().is_fullscreen);  // ❌ Current state, not target!
```

This created a **one-step lag** where the window API was always one state behind the intended action:
- **First F11 press**: Target = enter fullscreen, but current state = false → Nothing happens
- **Second F11 press**: Target = enter fullscreen, but current state = true (from first message) → Enters fullscreen  
- **Third F11 press**: Target = exit fullscreen, but current state = true → Stays in fullscreen

### The API Incompatibility

Additionally, there was a state detection incompatibility:

**`window.set_simple_fullscreen()`** (used for RustDesk compatibility):
- Creates a borderless window that fills the screen
- Not "real" macOS fullscreen (no Space creation, no animations)
- Required for remote desktop environments like RustDesk

**`window.fullscreen().is_some()`** (used for state detection):
- Only returns `true` for "real" macOS fullscreen mode
- Always returns `false` when using `set_simple_fullscreen()`
- This made the original detection logic unreliable

## The Fix

### 1. Use Target State Instead of Current State

The key fix was to use the calculated target state instead of the current state:

```rust
// ✅ FIXED CODE: Use the calculated target state
#[cfg(target_os = "macos")] {
    let fullscreen = if state.program().is_fullscreen {  // ✅ Correct detection
        state.queue_message(Message::ToggleFullScreen(false));
        None  // Target: should EXIT
    } else {
        state.queue_message(Message::ToggleFullScreen(true));
        Some(winit::window::Fullscreen::Borderless(None))  // Target: should ENTER
    };
    window.set_simple_fullscreen(fullscreen.is_some());  // ✅ Use TARGET, not current!
}
```

### 2. Platform-Specific State Detection

**Other Platforms**: Use window state (works correctly with `set_fullscreen()`)
```rust
#[cfg(not(target_os = "macos"))] {
    let fullscreen = if window.fullscreen().is_some() {  // ✅ Works fine
        state.queue_message(Message::ToggleFullScreen(false));
        None
    } else {
        state.queue_message(Message::ToggleFullScreen(true));
        Some(winit::window::Fullscreen::Borderless(None))
    };
    window.set_fullscreen(fullscreen);
}
```

### 3. Added Escape Key Support

```rust
WindowEvent::KeyboardInput {
    event: winit::event::KeyEvent {
        physical_key: winit::keyboard::PhysicalKey::Code(winit::keyboard::KeyCode::Escape),
        state: ElementState::Pressed,
        ..
    },
    ..
} => {
    // Platform-specific fullscreen exit logic
}
```

## Why This Approach Works

### API Compatibility Preserved
- **macOS**: Still uses `set_simple_fullscreen()` for RustDesk compatibility
- **Other platforms**: Still uses standard `set_fullscreen()` API
- **No breaking changes** to existing functionality

### State Synchronization Fixed
- **macOS**: Uses `state.program().is_fullscreen` which accurately tracks the borderless fullscreen state
- **Other platforms**: Uses `window.fullscreen().is_some()` which works correctly with native fullscreen
- **Consistent behavior** across all platforms

### User Experience Improved
- **F11 toggle**: Now works bidirectionally on all platforms
- **Escape key**: Universal exit from fullscreen mode
- **No more stuck states**: Always possible to exit fullscreen

## Technical Details

### The Two Types of macOS Fullscreen

**Native Fullscreen (`set_fullscreen()`):**
- Creates a new macOS Space (virtual desktop)
- Shows window controls and smooth animations
- Doesn't work in remote desktop environments
- Detectable via `window.fullscreen().is_some()`

**Simple Fullscreen (`set_simple_fullscreen()`):**
- Just a borderless window filling the screen
- No Space creation, no animations, no window controls
- Works in remote desktop environments like RustDesk
- **NOT detectable** via `window.fullscreen().is_some()`

### Why RustDesk Needs `set_simple_fullscreen()`

Remote desktop software like RustDesk can't handle macOS Spaces and animations properly. The `set_simple_fullscreen()` approach creates a fullscreen-like experience that works correctly in remote environments.

## Files Modified

- `src/main.rs` - Fixed F11 toggle logic and added Escape key support

## Testing Results

✅ **F11 first press**: Enters fullscreen mode  
✅ **F11 second press**: Exits fullscreen mode  
✅ **Escape key**: Exits fullscreen mode  
✅ **RustDesk compatibility**: Maintained (still uses `set_simple_fullscreen()`)  
✅ **Other platforms**: Unaffected (still use standard APIs)

## Key Insights

1. **API Incompatibility**: Different fullscreen APIs can have incompatible state detection methods
2. **Platform-Specific Solutions**: Sometimes you need different approaches for different platforms
3. **Remote Desktop Considerations**: Standard APIs don't always work in remote environments
4. **State Tracking**: Internal application state can be more reliable than window API state queries

This fix demonstrates the importance of understanding platform-specific API behaviors and choosing the right state detection method for each implementation.
