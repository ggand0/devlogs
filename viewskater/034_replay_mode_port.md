# Devlog 034: Replay Mode Port from Legacy Branch

**Date:** 2025-12-26
**Branch:** `feat/replay-mode` (new, ported from `feat/replay-mode-legacy`)

## Context

The original `feat/replay-mode` branch had diverged significantly from `main` over ~3 months, with 30+ commits in main that the replay branch didn't have. Merging would have required resolving 11 conflict markers across `app.rs` and `main.rs` in core initialization paths.

**Decision:** Port the replay code fresh to current main instead of merging.

## What is Replay Mode?

Replay mode is an automated performance benchmarking system that:
1. Loads specified test directories
2. Automatically navigates through images (right then left)
3. Collects FPS and memory metrics during navigation
4. Outputs benchmark results for performance comparison

This is useful for:
- Validating no performance regression after code changes
- Testing macOS keyboard navigation performance (target: 50-60 FPS)
- Comparing performance across different platforms

## Implementation Details

### Files Added/Modified

| File | Change |
|------|--------|
| `Cargo.toml` | Added `clap = { version = "4.0", features = ["derive"] }` |
| `src/replay.rs` | New file (483 lines) - ReplayController, ReplayState, ReplayMetrics, ReplayAction |
| `src/main.rs` | Added `mod replay`, clap `Args` struct, replay config creation |
| `src/app.rs` | Added `replay_controller` + `replay_keep_alive_task` fields, helper methods |
| `src/app/message.rs` | Added `ReplayKeepAlive` variant |
| `src/app/message_handlers.rs` | Added handler for `ReplayKeepAlive` |

### Key Components

#### 1. ReplayController (`src/replay.rs`)
- Manages replay state machine: `Inactive` → `LoadingDirectory` → `WaitingForReady` → `NavigatingRight` → `NavigatingLeft` → `Finished`
- Tracks metrics: UI FPS, Image FPS, memory usage
- Handles multiple iterations and multiple test directories

#### 2. CLI Arguments (`src/main.rs`)
```rust
#[derive(Parser)]
struct Args {
    path: Option<PathBuf>,           // Normal mode: path to open
    #[arg(long)] replay: bool,       // Enable replay mode
    #[arg(long = "test-dir")] test_directories: Vec<PathBuf>,
    #[arg(long, default_value = "10")] duration: u64,      // seconds per dir
    #[arg(long, default_value = "50")] nav_interval: u64,  // ms between navs
    #[arg(long, default_value = "both")] directions: String,
    #[arg(long)] output: Option<PathBuf>,
    #[arg(long, default_value = "1")] iterations: u32,
    #[arg(long)] verbose: bool,
}
```

#### 3. Replay Update Logic (`src/app.rs`)
- `update_replay_mode()`: Called each frame to update replay state and metrics
- `process_replay_action()`: Handles actions like LoadDirectory, NavigateRight, NavigateLeft
- Keep-alive task ensures continuous update loop during timing phases

### Usage

```bash
# Basic replay mode
cargo run --profile opt-dev -- --replay /path/to/images

# With options
cargo run --profile opt-dev -- --replay \
    --test-dir /path/to/images \
    --duration 10 \
    --iterations 3 \
    --verbose \
    --output results.txt
```

## Issue Fixed: Navigation Not Starting

### Symptom
Navigation wasn't happening. From the logs:
```
move_right_all() - LoadPos operation in queue, skipping move_right_all()
```

The `being_loaded_queue` had a `LoadPos` operation that never completed.

### Root Cause
Two issues:

1. **Discarded Task**: `initialize_dir_path()` returns a `Task<Message>` that performs async image loading. We were discarding it with `let _ = self.initialize_dir_path(...)` and returning a dummy task instead. The real loading task never ran.

2. **Premature `on_ready_to_navigate()`**: We were calling `on_ready_to_navigate()` immediately after `on_directory_loaded()`, before images had loaded. This caused replay to try navigating while `being_loaded_queue` still had pending work.

### Fix
1. Return the Task from `initialize_dir_path()` instead of discarding it:
   ```rust
   let load_task = self.initialize_dir_path(&path, 0);
   // ... notify replay controller ...
   Some(load_task)  // Return the real task
   ```

2. Move `on_ready_to_navigate()` call to `ImagesLoaded` handler after `LoadPos` completes:
   ```rust
   // In handle_image_loading_messages, after LoadPos handling:
   if let Some(ref mut replay_controller) = app.replay_controller {
       if matches!(replay_controller.state, ReplayState::WaitingForReady { .. }) {
           replay_controller.on_ready_to_navigate();
       }
   }
   ```

### Verification

After the fix, replay mode works correctly:
```
$ RUST_LOG=viewskater=debug cargo run --profile opt-dev -- --replay --test-dir /path/to/images --duration 10 --iterations 1

...
INFO viewskater::replay:397 Completed iteration 1/1
INFO viewskater::replay:410 Replay mode completed! All 1 iterations finished.
INFO viewskater::replay:418 === FINAL REPLAY SUMMARY ===
INFO viewskater::replay:419 Total directories tested: 2
INFO viewskater::replay:426 Overall Average UI FPS: 835.9
INFO viewskater::replay:427 Overall Average Image FPS: 152.0
INFO viewskater::replay:432 UI FPS Range: 0.0 - 1131.0
```

## Issue Fixed: Stuck in NavigatingLeft Phase

### Symptom
Replay mode got stuck during left navigation. Logs showed `move_left_all` being called repeatedly at index 1, but the replay controller timing never advanced:
```
DEBUG viewskater::app:772 move_left_all from self.skate_left block
DEBUG viewskater::navigation:196 move_left_all called
DEBUG viewskater::navigation:200 move_left_all called 2
DEBUG viewskater::navigation:207 move_left_all: current_index: 1
```

The replay controller's timing logic wasn't being called, so it never detected when the navigation phase should end.

### Root Cause
When `skate_left` is true, the keep-alive task was being `.take()`n but never returned:
```rust
if let Some(keep_alive_task) = self.replay_keep_alive_task.take() {
    if !self.skate_right && !self.skate_left {
        return keep_alive_task;  // Never reached during skate mode!
    }
}
// Task discarded, replay timing stalls
```

The keep-alive task is what triggers `ReplayKeepAlive` messages, which in turn call `update_replay_mode()` to check timing. Without it, the replay controller never got a chance to advance state.

### Fix
Batch the keep-alive task with the navigation task when skate mode is active:
```rust
let keep_alive_task = self.replay_keep_alive_task.take();

if self.skate_left {
    let nav_task = move_left_all(...);
    // Batch with keep-alive task if present (for replay mode timing)
    if let Some(keep_alive) = keep_alive_task {
        Task::batch([nav_task, keep_alive])
    } else {
        nav_task
    }
}
```

This ensures the keep-alive task is always returned, even during continuous navigation, allowing the replay controller to check timing and advance through phases.

## Issue Fixed: Message Queue Flooding

### Symptom
After fixing the stall issue, replay mode caused message queue overload warnings and lag spikes:
```
WARN viewskater:257 MESSAGE QUEUE OVERLOAD: 62 messages pending - clearing queue
WARN viewskater:257 MESSAGE QUEUE OVERLOAD: 77 messages pending - clearing queue
```

### Root Cause
The previous fix batched the keep-alive task with every navigation frame. But `update_replay_mode()` creates a new keep-alive task **every time it's called** (which is every frame). Each keep-alive schedules a 50ms delayed `ReplayKeepAlive` message.

During fast navigation (frames faster than 50ms), delayed messages accumulated faster than they could be processed, flooding the queue.

### Fix
Track whether a keep-alive is "in flight" to prevent scheduling multiple:
```rust
// New field
pub replay_keep_alive_pending: bool,

// Only create if none pending
if replay_controller.is_active() && !self.replay_keep_alive_pending {
    self.replay_keep_alive_task = Some(Task::perform(...));
}

// Set pending when returned
if let Some(keep_alive) = keep_alive_task {
    self.replay_keep_alive_pending = true;
    Task::batch([nav_task, keep_alive])
}

// Reset when message received
Message::ReplayKeepAlive => {
    app.replay_keep_alive_pending = false;
    Task::none()
}
```

This ensures only one keep-alive message is in flight at a time, preventing queue flooding while still maintaining replay timing.

## Potential Improvement: FPS Accuracy for Cached Images

### Problem
The sliding window cache pre-loads the first N images (e.g., N=5 means ~11 images total). These cached images display instantly, causing inflated FPS readings at the start of navigation.

### Current Solution
Added `--skip-initial` CLI option to skip metric collection for the first N navigations:
```rust
// In update_metrics()
if self.navigation_count < self.config.skip_initial_images {
    return;  // Don't collect metrics yet
}
```

### Limitation
Even after the skip period, iced's `upload_timestamps` VecDeque still contains timestamps from the cached-image frames. These old timestamps affect the FPS calculation until they naturally expire from the 2-second sliding window.

### Potential Fixes (No iced changes needed)
Both approaches use existing iced APIs:

1. **Reset approach**: Call `reset_image_fps()` when `navigation_count == skip_initial_images` to clear the VecDeque before collecting metrics.

2. **Filter approach**: Use `get_image_upload_timestamps()` to get raw timestamps, skip the first N entries, and calculate FPS manually in ViewSkater.

The reset approach is simpler. The filter approach gives more control but requires custom FPS calculation logic.

## Commits

```
97aa797 Port replay mode for performance benchmarking
286cdc9 Fix replay mode stalling during left navigation
4cd37c9 Prevent replay keep-alive message flooding
```

## References

- Roadmap: `docs/plans/roadmap-0.3-to-0.4.md` (Phase 2.1: Port Replay Mode)
- Legacy branch: `feat/replay-mode-legacy`
