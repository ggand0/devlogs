# Spinner Animation Fix

## Problem
The loading spinner showed as a static frame - it only animated when the mouse cursor was moved over the application window.

## Animation Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BEFORE FIX (Canvas with cache)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SpinnerTick fires                                                          │
│       ↓                                                                     │
│  UserEvent handler                                                          │
│       ↓                                                                     │
│  state.queue_message(SpinnerTick)  ←── Message goes to app.update()        │
│       ↓                                                                     │
│  *redraw = true                                                             │
│       ↓                                                                     │
│  renderer.present()  ←── Traverses widget tree, calls draw() on each       │
│       ↓                                                                     │
│  Canvas::draw() called                                                      │
│       ↓                                                                     │
│  cache.draw() returns CACHED geometry  ←── Cache not cleared!              │
│       ↓                                                                     │
│  Same frame rendered  ←── NO ANIMATION                                      │
│                                                                             │
│  Mouse movement triggers Event                                              │
│       ↓                                                                     │
│  Widget receives event in on_event()                                        │
│       ↓                                                                     │
│  cache.clear()  ←── Cache cleared by event                                  │
│       ↓                                                                     │
│  Next draw() generates NEW geometry  ←── ANIMATION WORKS                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         AFTER FIX (Widget without cache)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SpinnerTick fires                                                          │
│       ↓                                                                     │
│  UserEvent handler                                                          │
│       ↓                                                                     │
│  *redraw = true                                                             │
│       ↓                                                                     │
│  renderer.present()                                                         │
│       ↓                                                                     │
│  Circular::draw() called                                                    │
│       ↓                                                                     │
│  Compute animation from Instant::now()  ←── Fresh computation every time   │
│       ↓                                                                     │
│  Create new Frame, draw arc at current angle                                │
│       ↓                                                                     │
│  frame.into_geometry()  ←── No cache, always new geometry                   │
│       ↓                                                                     │
│  Animation updates!                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**The actual fix in circular.rs:**
```rust
// BEFORE: Used cache
let geometry = state.cache.draw(renderer, bounds.size(), |frame| {
    // Only runs when cache is empty
});

// AFTER: No cache, fresh every frame
let mut frame = canvas::Frame::new(renderer, bounds.size());
// Draw directly using Instant::now() for animation
let geometry = frame.into_geometry();
```

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

### Part 2: Add Animation Loop to Original Render Path (main.rs:906)

The original render code (`if *redraw { ... }`) didn't continue the animation loop:

```rust
// After frame.present()
if state.program().is_any_pane_loading() {
    window.request_redraw();
}
```

This ensures any WindowEvent that triggers a render also continues the animation.

### Part 3: Direct Render in UserEvent Handler (main.rs:1043)

Added direct rendering when loading is active to bypass RedrawRequested event delivery issues:

```rust
// In UserEvent handler, after window.request_redraw()
if state.program().is_any_pane_loading() {
    // Render directly
    surface.get_current_texture() -> render -> present
}
```

This ensures animation starts immediately when SpinnerTick fires, without waiting for a WindowEvent.

### Part 4: Fix is_any_pane_loading() Timing (app.rs:537)

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

## Detailed Animation Flow (After Fix)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ 1. SpinnerTick scheduled                                                     │
│    src/app.rs:295-302 (start_neighbor_loading)                               │
│    ┌─────────────────────────────────────────────────────────────────────┐   │
│    │ let spinner_task = Task::perform(                                   │   │
│    │     async { tokio::task::spawn_blocking(|| {                        │   │
│    │         std::thread::sleep(Duration::from_millis(16));              │   │
│    │     }).await.ok(); },                                               │   │
│    │     |_| Message::SpinnerTick                                        │   │
│    │ );                                                                  │   │
│    └─────────────────────────────────────────────────────────────────────┘   │
│         ↓                                                                    │
│ 2. SpinnerTick fires as UserEvent                                            │
│    src/main.rs:974 (Event::UserEvent handler)                                │
│    ┌─────────────────────────────────────────────────────────────────────┐   │
│    │ Event::EventLoopAwakened(winit::event::Event::UserEvent(action))    │   │
│    └─────────────────────────────────────────────────────────────────────┘   │
│         ↓                                                                    │
│ 3. Message queued and processed                                              │
│    src/main.rs:1006-1007                                                     │
│    ┌─────────────────────────────────────────────────────────────────────┐   │
│    │ state.queue_message(message);                                       │   │
│    │ state.queue_event(iced_winit::core::Event::Window(                  │   │
│    │     iced_winit::core::window::Event::RedrawRequested(Instant::now())│   │
│    │ ));                                                                 │   │
│    └─────────────────────────────────────────────────────────────────────┘   │
│         ↓                                                                    │
│ 4. Direct render if loading active                                           │
│    src/main.rs:1043-1067                                                     │
│    ┌─────────────────────────────────────────────────────────────────────┐   │
│    │ if state.program().is_any_pane_loading() {                          │   │
│    │     match surface.get_current_texture() {                           │   │
│    │         Ok(frame) => {                                              │   │
│    │             renderer_guard.present(...);                            │   │
│    │             frame.present();                                        │   │
│    │         }                                                           │   │
│    │     }                                                               │   │
│    │ }                                                                   │   │
│    └─────────────────────────────────────────────────────────────────────┘   │
│         ↓                                                                    │
│ 5. Circular::draw() called during present()                                  │
│    src/widgets/circular.rs:113-195                                           │
│    ┌─────────────────────────────────────────────────────────────────────┐   │
│    │ fn draw(&self, tree: &Tree, renderer: &mut Renderer, ...) {         │   │
│    │     let now = Instant::now();                          // :128      │   │
│    │     let elapsed = now.duration_since(state.start_time);             │   │
│    │     let rotation = (elapsed.as_secs_f32() / rotation_duration)      │   │
│    │         .fract();                                      // :145      │   │
│    │                                                                     │   │
│    │     let mut frame = canvas::Frame::new(...);           // :148      │   │
│    │     // Draw arc at current rotation angle                           │   │
│    │     let geometry = frame.into_geometry();              // :190      │   │
│    │     renderer.draw_geometry(geometry);                  // :193      │   │
│    │ }                                                                   │   │
│    └─────────────────────────────────────────────────────────────────────┘   │
│         ↓                                                                    │
│ 6. SpinnerTick reschedules itself                                            │
│    src/app/message_handlers.rs:46-65                                         │
│    ┌─────────────────────────────────────────────────────────────────────┐   │
│    │ Message::SpinnerTick => {                                           │   │
│    │     let is_loading = app.panes.iter()                               │   │
│    │         .any(|p| p.loading_started_at.is_some());                   │   │
│    │     if is_loading {                                                 │   │
│    │         app.spinner_tick_task = Some(Task::perform(...));           │   │
│    │     }                                                               │   │
│    │ }                                                                   │   │
│    └─────────────────────────────────────────────────────────────────────┘   │
│         ↓                                                                    │
│ 7. Loop continues every 16ms until loading completes                         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Key Code Locations

| Step | File | Line | Purpose |
|------|------|------|---------|
| Schedule SpinnerTick | `src/app.rs` | 295-302 | Start 16ms timer loop |
| UserEvent handler | `src/main.rs` | 974 | Receive SpinnerTick |
| Queue message | `src/main.rs` | 1006 | Send to app.update() |
| Check loading | `src/app.rs` | 537-539 | `is_any_pane_loading()` |
| Direct render | `src/main.rs` | 1043-1067 | Render if loading |
| Animation calc | `src/widgets/circular.rs` | 128-145 | Wall clock rotation |
| Draw geometry | `src/widgets/circular.rs` | 148-193 | Fresh frame each time |
| Reschedule | `src/app/message_handlers.rs` | 57-64 | Next tick in 16ms |

## Files Modified

- **src/widgets/circular.rs**: Complete rewrite as Widget (not canvas::Program)
- **src/widgets/loading_overlay.rs**: Simplified, removed CircularState parameter
- **src/main.rs**: Direct render in UserEvent handler, animation loop continuation in render path
- **src/app.rs**: Added `is_any_pane_loading()` method, fixed timing logic
- **src/app/message_handlers.rs**: Fixed loading_started_at clearing logic
- **src/pane.rs**: Removed spinner_state field

## Key Insight

The iced framework's custom main loop separates Events and Messages:
- **Events** go to widgets via `on_event()`
- **Messages** go to the app via `update()`

Canvas widgets with `canvas::Program` cache geometry and only redraw on events. By rewriting as a direct Widget that computes animation from wall clock time on every `draw()`, the animation now works because each SpinnerTick triggers a direct render that calls `Circular::draw()` with fresh geometry.

## Commits
- `a4ae33e`: Spinner animation works without mouse movement
- `9483156`: Spinner animation starts immediately without cursor movement
- `e368fc3`: Remove redundant WindowEvent::RedrawRequested handler
