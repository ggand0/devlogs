# Window Title Update Fix - Complete Analysis & Implementation

**Date:** 2025-08-26
**Issue:** Window title stops updating after exiting fullscreen mode
**Status:** Partially Fixed - New Performance Issues Discovered

## Problem Discovery Timeline

### Original Issue
- **Symptom**: After exiting fullscreen (F11/Escape), window title wouldn't update during navigation
- **Temporary Fix**: Clicking window content would "unfreeze" title updates
- **Platforms**: macOS, Linux (all platforms affected)

### Root Cause Investigation

#### Initial Theories (All Wrong)
1. **Window Focus Issues** → Added `window.focus_window()` - didn't work
2. **Render Cycle Problems** → Added `window.request_redraw()` - didn't work  
3. **Event Loop Issues** → Added `ControlFlow::Poll` - didn't work

#### The Real Culprit: Performance Optimization Gone Wrong

**Key Discovery**: The `moved` flag was originally added as a **performance optimization**:

```rust
// OLD: Called every frame - caused 10fps drop!
let new_title = state.program().title();
window.set_title(&new_title);

// OPTIMIZED: Only update when not "moved"
if !*moved {
    let new_title = state.program().title();
    window.set_title(&new_title);
}
```

**The Problem**: `moved` flag conflated two different concepts:
1. **User dragging window** (should block updates - performance)
2. **OS fullscreen transitions** (should NOT block updates - functionality)

#### Debug Log Evidence

**Timeline of Fullscreen Exit Bug:**
```
Setting title after fullscreen transition: 'frame_0001.png'  // *moved = false
Window moved event - setting moved = true                    // ~13ms later  
Title update blocked by moved flag: true at frame           // Forever stuck
```

**Root Issue**: `WindowEvent::Moved` is triggered by OS during fullscreen transitions, not user actions.

## Solution Attempts

### Attempt 1: Message-Based Reset (Partial Success)
```rust
// Added Message::ResetMovedFlag to reset moved flag after navigation
Message::ResetMovedFlag => {
    *moved = false;
}
```

**Results:**
- ✅ Keyboard navigation: Fixed
- ✅ Skate mode: Fixed  
- ❌ Slider navigation: Still broken (Iced widget events don't reach winit level)
- ❌ Still vulnerable to WindowEvent::Moved overriding the reset

### Attempt 2: Clean Event-Driven Solution (Performance Issues)

**The Concept:**
```rust
// Replace moved flag with title change detection
Message::ImageChanged => {
    let new_title = state.program().title();
    if *current_title != new_title {
        *current_title = new_title.clone();
        window.set_title(&new_title);
    }
}
```

**Implementation:**
- Removed `moved` flag entirely
- Added `ImageChanged` message sent after all navigation
- Only call `window.set_title()` when title actually changes

**Results:**
- ✅ **Functionality**: Title updates work for all navigation types
- ✅ **Reliability**: No more stuck titles after fullscreen
- ❌ **Performance**: Message queue flooding causes rendering to freeze

## Current Status: New Performance Problem

### Message Queue Flooding
```
DEBUG viewskater:190 Message queue size: 26
##########MOVE_RIGHT_ALL()########## (repeated rapidly)
render_happened: false (stuck in loop)
```

**Issue**: `Task::batch(vec![task, Task::done(Message::ImageChanged)])` is being called excessively during skate mode, flooding the message queue and causing:
- Image rendering to freeze
- Navigation to get stuck in loops
- Performance degradation

### Slider Navigation Still Broken
- **Keyboard navigation**: ✅ Working
- **Slider navigation**: ❌ Title doesn't update
- **Reason**: Slider uses Iced's internal widget system, events don't trigger our message flow properly

## Technical Insights

### Performance vs Functionality Trade-off
- **Original `moved` flag**: Good performance, broken functionality after fullscreen
- **Event-driven approach**: Good functionality, performance issues from message flooding
- **Need**: Smart solution that only sends `ImageChanged` when image actually changes, not on every navigation attempt

### Platform-Specific Behavior
- **macOS**: Uses `set_simple_fullscreen()`, triggers `WindowEvent::Moved`
- **Linux**: Uses `set_fullscreen()`, also triggers `WindowEvent::Moved`  
- **Both platforms**: OS-managed fullscreen transitions cause the same stuck title issue

### Message Queue Architecture Issue
The current approach sends `ImageChanged` on every navigation call, but:
- Navigation calls can happen multiple times per actual image change
- Skate mode generates excessive navigation calls
- Need to detect actual image changes, not navigation attempts

## Next Steps Required

### 1. Fix Message Queue Flooding
- Only send `ImageChanged` when an image **actually changes**
- Use the `did_render_happen` boolean from navigation functions
- Avoid sending messages during failed navigation attempts

### 2. Fix Slider Navigation  
- Ensure slider interactions properly trigger `ImageChanged`
- May need different approach since slider events are handled in Iced widget system

### 3. Performance Testing
- Verify no FPS drops after fixes
- Ensure message queue stays manageable during rapid navigation

## Code Changes Made

### Files Modified
- **`src/app.rs`**: Added `ImageChanged` message, integrated with navigation
- **`src/main.rs`**: Replaced `moved` flag with `current_title` tracking, added title change detection
- **Navigation integration**: All keyboard and slider navigation now sends `ImageChanged`

### Key Implementation
```rust
// In main.rs - Only update when title actually changes
if let Message::ImageChanged = message {
    let new_title = state.program().title();
    if *current_title != new_title {
        debug!("Title changed: '{}' -> '{}'", *current_title, new_title);
        *current_title = new_title.clone();
        window.set_title(&new_title);
    }
}
```

## Lessons Learned

1. **Performance optimizations can create functional bugs** when they make incorrect assumptions
2. **OS window events are not always user-initiated** - fullscreen transitions trigger movement events
3. **Event-driven solutions need careful message management** to avoid flooding
4. **Mixed UI frameworks** (winit + Iced) create complex event handling scenarios
5. **Debug logging is essential** for understanding async event timing issues

The core title update mechanism is now correct, but needs refinement to avoid performance issues from excessive message generation.
