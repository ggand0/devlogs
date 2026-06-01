# Fix: macOS Continuous Rendering Performance Issue

**Date**: November 2, 2025  
**AI Assistant**: Claude Sonnet 4.0  
**Branch**: `exp/custom-0.30.11`  
**Winit Version**: 0.30.11

## Problem Summary

After updating from a partially merged winit 0.30.0/0.30.1 to winit 0.30.11, the image viewer app experienced severe performance degradation during continuous rendering scenarios:

### Symptoms
- **Keyboard-based rendering**: Dropped from 60fps to ~30fps with stuttering during continuous key presses (arrow keys for image navigation)
- **Slider-based rendering**: Dropped from 40fps to 10-12fps; fast slider movement would render nothing or only 1-2 images
- **Overall**: Continuous user input (holding keys, dragging sliders) caused rendering to become choppy and unresponsive

### Root Cause

The issue was introduced by **winit PR #3708** ("Use AppKit's internal queuing mechanisms"), which fundamentally changed how events are processed on macOS:

#### Before PR #3708
- All events were queued in a `VecDeque<QueuedEvent>`
- Events were processed in **batches** in the `cleared()` method
- `AboutToWait` was called **once** after processing the entire batch
- **Predictable timing** for apps to call `window.request_redraw()`

#### After PR #3708
- Events are handled **immediately** when possible (when not already in an event handler)
- Only queued via `CFRunLoopPerformBlock` if handling would cause re-entrancy
- Events processed **individually** instead of in batches
- **Unpredictable `AboutToWait` timing** breaks continuous rendering

### Why This Broke Continuous Rendering

Apps using async rendering (like `iced::Command::perform` or `Task::perform`) rely on:
1. **Event batching**: Multiple input events processed together
2. **Predictable `AboutToWait`**: Consistent timing to trigger redraws
3. **Uninterrupted async operations**: Time between batches for async tasks to complete

PR #3708's immediate event handling fragmented this flow, causing async rendering tasks to be constantly interrupted.

## Solution: Hybrid Event Handling

We implemented a **hybrid approach** that restores event batching for input events while preserving PR #3708's benefits for system events.

### Implementation

**File Modified**: `src/platform_impl/macos/app_state.rs`

#### Changes Made

1. **Added Event Queue**
   ```rust
   pending_events: RefCell<VecDeque<Event<HandlePendingUserEvents>>>
   ```

2. **Modified `maybe_queue_event()` for Selective Batching**
   - **Always queued** (for batching):
     - `WindowEvent::KeyboardInput`
     - `WindowEvent::CursorMoved`
     - `WindowEvent::MouseInput`
     - `WindowEvent::MouseWheel`
     - `WindowEvent::DragOver`
     - `WindowEvent::DragEnter`
     - All `DeviceEvent`s
   
   - **Conditionally handled** (original PR #3708 behavior):
     - Window management events (resize, focus, etc.)
     - System events

3. **Modified `cleared()` to Process Batched Events**
   ```rust
   // Process queued input events in batch for continuous rendering compatibility
   let events = mem::take(&mut *self.ivars().pending_events.borrow_mut());
   for event in events {
       self.handle_event(event);
   }
   ```

### Why This Works

- **Predictable timing**: Input events are batched and processed together before `AboutToWait`
- **Smooth async rendering**: Async operations get uninterrupted time to execute between event batches
- **Maintains PR #3708 benefits**: Non-input events still get improved immediate handling
- **Best of both worlds**: Combines event batching for continuous rendering with immediate handling for responsiveness

## Results

✅ **Keyboard-based rendering**: Restored to ~50-60fps with no stuttering  
✅ **Slider-based rendering**: Restored to ~30-40fps with smooth, fluid interaction  
✅ **Fast slider movement**: Now renders properly instead of showing nothing  
✅ **Continuous input**: Smooth async rendering during all user interactions

## Commit Message

```
Fix macOS continuous rendering by restoring event batching for input events

Restore VecDeque-based event batching for input events (keyboard, mouse, drag)
while keeping PR #3708's immediate handling for system events. This fixes
performance issues in apps that rely on continuous rendering during user
interaction, such as image viewers with sliders or rapid keyboard navigation.

- Add pending_events VecDeque to AppState
- Queue input events for batch processing in cleared()
- Maintain immediate handling for window management events
- Restores smooth async rendering during continuous user input
```

## Technical Analysis

### Comparison: Hybrid vs. Upstream

#### Our Hybrid Approach Wins For:
- Image/video viewers
- Games with continuous input
- Drawing/design applications
- Slider-heavy UIs
- Real-time data visualization
- Any app doing continuous rendering with async operations

#### Upstream Approach Wins For:
- Traditional desktop applications
- Apps without continuous rendering
- Simpler event handling requirements
- Better AppKit integration
- Cleaner call stacks for debugging

### Ideal Future Solution

The best approach would be to make this **configurable** in upstream winit:

```rust
EventLoop::with_options(EventLoopOptions {
    input_event_batching: true, // For continuous rendering apps
})
```

This would let developers choose based on their app's needs, similar to game engines that offer immediate vs. batched input modes.

## Related Work

### Previous Custom Modifications
- **Drag-and-drop cursor position tracking**: Custom `DragEnter`, `DragOver`, `DragDrop`, `DragLeave` events with position data (replaced original `HoveredFile`, `DroppedFile`, `HoveredFileCancelled`)
- Implemented in `window_delegate.rs` with proper coordinate conversion from macOS to Winit coordinate system

### Dependencies
- App uses **Iced framework v0.13.1** with `Task::perform` for async image loading
- Relies on `wgpu`-accelerated rendering
- Implements `winit::application::ApplicationHandler` pattern

## Notes for Future Maintenance

1. **Upstream compatibility**: This patch may need adjustment when merging future winit versions
2. **PR #3708 context**: Understanding this PR is critical for maintaining the fix
3. **Event tracking mode**: The fix ensures events are processed even during `NSEventTrackingRunLoopMode` (active during slider dragging)
4. **Testing scenarios**: Always test with:
   - Rapid keyboard input (holding arrow keys)
   - Fast slider movement
   - Slow slider movement
   - Continuous rendering with async image loading

## References

- **Problematic PR**: [winit #3708](https://github.com/rust-windowing/winit/pull/3708) - "Use AppKit's internal queuing mechanisms"
- **Original branch**: `exp/upstream-crashfix-0.30.1-partial` (pre-0.30.11, working state)
- **Current branch**: `exp/custom-0.30.11` (with this fix applied)

