# Window Title Update Bug Analysis

**Date:** 2025-08-26
**Issue:** Window title stops updating after exiting fullscreen mode
**Platforms Affected:** macOS, Linux (all platforms)

## Problem Description

After exiting fullscreen mode using F11 or Escape, the window title becomes "stuck" and won't update when navigating images using:
- Keyboard navigation (A/D keys) 
- Slider interaction
- Any other navigation method

**Temporary Workaround:** Clicking the window content manually "unfreezes" the title updates.

## Root Cause Analysis

### The `moved` Flag System

The application uses a `moved` boolean flag to control when window title updates are allowed:

```rust
// In main.rs render loop
if !*moved {
    let new_title = state.program().title();
    window.set_title(&new_title);
} else {
    debug!("Title update blocked by moved flag: {} at frame", *moved);
}
```

**Original Intent:** Prevent title updates during user-initiated window moves/resizes to avoid flickering.

### The Problem: Fullscreen Transition Side Effects

**Key Discovery:** Fullscreen transitions automatically trigger `WindowEvent::Moved` events:

```
Timeline of events during F11 fullscreen exit:
1. F11 pressed → fullscreen exit initiated
2. Force title update: *moved = false
3. ~13ms later → WindowEvent::Moved fired by OS
4. WindowEvent::Moved handler: *moved = true  
5. All subsequent title updates blocked indefinitely
```

**Debug Log Evidence:**
```
Setting title after fullscreen transition: 'frame_0001.png'
Window moved event - setting moved = true    ← 13ms later
Title update blocked by moved flag: true at frame
```

### Why WindowEvent::Moved Fires

During fullscreen transitions, the operating system:
1. **Changes window bounds** (fullscreen ↔ windowed)
2. **Repositions the window** to restore previous location
3. **Triggers WindowEvent::Moved** as part of the transition

This is **not user-initiated movement** but **OS-managed window state changes**.

## Investigation History

### Initial Theories (Incorrect)
1. **Window focus issues** → Added `window.focus_window()` calls
2. **Render cycle problems** → Added `window.request_redraw()` calls  
3. **Event loop issues** → Added `ControlFlow::Poll` modifications

### Breakthrough: Message-Based Approach
Added `Message::ResetMovedFlag` to reset the flag after navigation:

```rust
// Reset moved flag after keyboard navigation
Task::done(Message::ResetMovedFlag)

// Handle in main.rs
if let Message::ResetMovedFlag = message {
    *moved = false;
}
```

**Results:**
- ✅ **Keyboard navigation**: Works (A/D keys)
- ✅ **Skate mode**: Works (Shift + A/D) 
- ❌ **Slider navigation**: Still broken

### Slider-Specific Issue
Slider interactions use **Iced's internal widget event system**, not raw winit events. Mouse events are captured by the slider widget and don't reach the `WindowEvent::MouseInput` handler.

**Solution:** Added `ResetMovedFlag` message to `SliderReleased` handler:

```rust
Message::SliderReleased(pane_index, value) => {
    // ... existing slider logic
    let slider_task = // ... 
    // Reset moved flag to allow title updates after slider interaction
    return Task::batch(vec![slider_task, Task::done(Message::ResetMovedFlag)]);
}
```

**Results:**
- ✅ **Keyboard navigation**: Works
- ✅ **Skate mode**: Works  
- ❌ **Slider navigation**: Still broken (same root cause)

### Final Discovery: The Real Culprit

The issue persists because **fullscreen transitions trigger WindowEvent::Moved**, which immediately sets `*moved = true` after we reset it to `false`.

**The Core Problem:** The `moved` flag conflates two different concepts:
1. **User-initiated window movement** (should block title updates)
2. **OS-initiated window state changes** (should NOT block title updates)

## Current Status

### What Works
- ✅ Title updates work normally before entering fullscreen
- ✅ Keyboard navigation resets moved flag (but gets overridden by WindowEvent::Moved)
- ✅ Slider interactions send ResetMovedFlag (but gets overridden by WindowEvent::Moved)

### What's Broken  
- ❌ Any navigation immediately after fullscreen exit
- ❌ `WindowEvent::Moved` from fullscreen transitions blocks all future title updates
- ❌ No clean way to distinguish user vs. OS-initiated window moves

## Potential Solutions

### 1. **Time-Based Ignore Window** (Hacky)
Ignore `WindowEvent::Moved` for a short period after fullscreen transitions.
- ❌ **Cons:** Fragile, timing-dependent, not robust

### 2. **Separate Flag System** (Clean)
Use separate flags for different types of movement:
```rust
*user_moved = false;      // User dragging window
*os_transitioning = false; // OS fullscreen transitions
```

### 3. **Remove moved Flag Entirely** (Radical)
Question whether the `moved` flag is necessary at all:
- What specific problems was it originally solving?
- Are those problems still relevant?
- Can we handle title updates differently?

### 4. **Event Source Detection** (Complex)
Try to detect whether `WindowEvent::Moved` came from user action vs. OS transitions.

## Technical Context

### Platforms
- **macOS**: Uses `set_simple_fullscreen()` for compatibility with RustDesk
- **Linux/Others**: Uses `set_fullscreen()`
- **Both trigger WindowEvent::Moved during transitions**

### Codebase Architecture
- **winit**: Provides raw window events (`WindowEvent::Moved`)
- **Iced**: Higher-level UI framework with internal event handling
- **Mixed event handling**: Some events handled at winit level, others at Iced level

## Recommendations

1. **Investigate the original purpose** of the `moved` flag
2. **Consider removing or redesigning** the flag system
3. **Test whether title updates during window movement actually cause problems**
4. **Implement a cleaner solution** that doesn't conflate user and OS actions

The current message-based approach is a good foundation but doesn't address the fundamental issue of `WindowEvent::Moved` being triggered by fullscreen transitions.
