# Loading Spinner Implementation

## Overview
Material Design-style circular loading spinner that appears when image loading takes >1 second.

## Design Decisions
- **Style**: Circular spinner with "Emphasized" easing, 2.0s cycle duration
- **Placement**: Centered overlay on image area
- **Backdrop**: Semi-transparent black (0.5 alpha)
- **Timing**: Only appears after 1 second delay to prevent flicker on fast loads
- **Animation**: Driven by SpinnerTick message at ~60fps using smol timer

## Files Created

### `src/widgets/easing.rs`
- Ports the `EMPHASIZED` easing constant from iced loading_spinners example
- Uses `lyon_algorithms` for cubic Bezier curve interpolation
- Two-part curve: `[0.05, 0.0] -> [0.133333, 0.06] -> [0.166666, 0.4]` then `[0.208333, 0.82] -> [0.25, 1.0] -> [1.0, 1.0]`

### `src/widgets/circular.rs`
- `CircularState`: Stores animation state (Expanding/Contracting phases)
- `Circular`: Canvas Program that draws the spinner
- Uses canvas `Path::circle` for track, `arc` for animated bar
- Hardcoded: size=48px, bar_height=4px, white bar on semi-transparent track

### `src/widgets/loading_overlay.rs`
- `loading_overlay(base, show_spinner, spinner_state)` function
- Uses `stack!` macro (same pattern as `modal.rs`)
- Semi-transparent black backdrop with centered spinner
- Only renders overlay when `show_spinner=true`

## Files Modified

### `Cargo.toml`
- Added `lyon_algorithms = "1.0"` dependency
- Added `smol = "2.0"` dependency (for async timers on separate thread pool)

### `src/widgets/mod.rs`
- Exported `easing`, `circular`, `loading_overlay` modules

### `src/pane.rs`
- Added `loading_started_at: Option<Instant>` field to `Pane` struct
- Added `spinner_state: CircularState` field to `Pane` struct
- Modified `build_ui_container()` to return `Element` (was `Container`)
- Wrapped base container with `loading_overlay()` based on timer check (>1 second)

### `src/app.rs`
- `start_neighbor_loading()`: Sets `loading_started_at` timer and starts SpinnerTick loop
- Uses `smol::Timer` for SpinnerTick which runs on separate thread pool (avoids blocking main tokio runtime)

### `src/app/message.rs`
- Added `SpinnerTick` message variant for driving spinner animation

### `src/app/message_handlers.rs`
- Added `SpinnerTick` handler that:
  - Updates `CircularState` animation for each pane
  - Schedules next tick at 16ms interval (~60fps) while loading
  - Stops when no panes have `loading_started_at` set
- In `ImagesLoaded` handler: Clear `loading_started_at` for affected panes

### `src/navigation_keyboard.rs`
- In `move_right_all()`: Set `loading_started_at` when images not fully loaded
- Clear timer immediately if render from cache succeeds
- In `move_left_all()`: Same pattern for backward navigation

### `src/navigation_slider.rs`
- In `get_loading_tasks_slider()`: Set `loading_started_at` when LoadPos enqueued

## Trigger Conditions

The spinner shows when:
```rust
let show_spinner = self.loading_started_at
    .map_or(false, |start| start.elapsed() > Duration::from_secs(1));
```

Timer is set in:
1. **Neighbor loading** (`app.rs:start_neighbor_loading`): After first image loads, timer starts for background neighbor loading
2. **Keyboard navigation** (`navigation_keyboard.rs`): Only when `render_next_image_all()` returns false (image NOT in cache)
3. **Slider navigation** (`navigation_slider.rs`): When LoadPos operation is enqueued

Timer is cleared in:
1. **ImagesLoaded handler**: When async image loading completes

## Animation Mechanism

The spinner animation is driven by `SpinnerTick` messages:

1. When `start_neighbor_loading()` is called:
   - Sets `loading_started_at` timer
   - Dispatches initial `SpinnerTick` via `smol::Timer` (50ms delay)

2. `SpinnerTick` handler:
   - Checks if any pane has `loading_started_at` set
   - If loading: updates `CircularState` and schedules next tick at 16ms (~60fps)
   - If not loading: stops the tick loop

3. `CircularState.update()` advances the animation:
   - Cycles between Expanding and Contracting phases
   - Uses wall-clock time for smooth animation regardless of frame rate

## Debug Logging

Run with `RUST_LOG=viewskater=debug` to see spinner-related logs:
- `SPINNER: Set loading_started_at for neighbor loading (pane X)`
- `SPINNER: SpinnerTick stopped - loading complete`
- `SPINNER: build_ui_container called, loading_started_at=X, show_spinner=Y`

## Reference
- iced loading_spinners example: https://github.com/iced-rs/iced/tree/0.13/examples/loading_spinners
- Existing modal pattern: `src/widgets/modal.rs`
