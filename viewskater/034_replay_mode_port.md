# Devlog 034: Replay Mode Port from Legacy Branch

**Date:** 2025-12-26 to 2025-12-28
**Branch:** `feat/replay-mode` (ported from `feat/replay-mode-legacy`, merged to main)

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

## Final Implementation

### Files Added/Modified

| File | Change |
|------|--------|
| `Cargo.toml` | Added `clap = { version = "4.0", features = ["derive"] }` |
| `src/replay.rs` | New file (588 lines) - ReplayController, ReplayState, ReplayMetrics, ReplayAction |
| `src/main.rs` | Refactored CLI with clap `Args` struct, replay config creation |
| `src/app.rs` | Added `replay_controller` + keep-alive fields (+46 lines) |
| `src/app/replay_handlers.rs` | New file (230 lines) - extracted from app.rs |
| `src/app/message.rs` | Added `ReplayKeepAlive` variant |
| `src/app/message_handlers.rs` | Added handler for `ReplayKeepAlive`, ready signal after LoadPos |
| `.gitignore` | Added `benchmarks/` |

### Key Components

#### 1. ReplayController (`src/replay.rs`)
- Manages replay state machine: `Inactive` → `LoadingDirectory` → `WaitingForReady` → `NavigatingRight` → `NavigatingLeft` → `Finished`
- Tracks metrics: UI FPS, Image FPS (min/max/avg/last), memory usage
- Handles multiple iterations and multiple test directories
- Helper methods: `finalize_current_metrics()`, `should_transition()`

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
    #[arg(long, default_value = "text")] output_format: String,  // json/markdown/text
    #[arg(long, default_value = "1")] iterations: u32,
    #[arg(long, default_value = "0")] skip_initial: usize,  // skip cached images
    #[arg(long)] verbose: bool,
    #[arg(long)] auto_exit: bool,    // exit after replay completes
}
```

#### 3. Replay Handlers (`src/app/replay_handlers.rs`)
Extracted from app.rs to reduce file size (970 → 748 lines):
- `update_replay_mode()`: Called each frame to update replay state and metrics
- `process_replay_action()`: Handles actions like LoadDirectory, NavigateRight, NavigateLeft

### Usage

```bash
# Basic replay mode
cargo run --profile opt-dev -- --replay --test-dir /path/to/images

# Full options for CI
cargo run --profile opt-dev -- --replay \
    --test-dir /path/to/images \
    --duration 10 \
    --iterations 2 \
    --directions right \
    --skip-initial 30 \
    --output results.json \
    --output-format json \
    --auto-exit
```

## Issues Fixed

### 1. Navigation Not Starting

**Symptom:** Navigation wasn't happening. `being_loaded_queue` had a `LoadPos` operation that never completed.

**Root Cause:**
1. `initialize_dir_path()` returns a `Task<Message>` that was being discarded
2. `on_ready_to_navigate()` was called before images had loaded

**Fix:** Return the Task from `initialize_dir_path()` and move `on_ready_to_navigate()` to `ImagesLoaded` handler after `LoadPos` completes.

### 2. Stuck in NavigatingLeft Phase

**Symptom:** Replay mode got stuck during left navigation - timing never advanced.

**Root Cause:** Keep-alive task was being `.take()`n but never returned during skate mode.

**Fix:** Batch the keep-alive task with the navigation task when skate mode is active using `Task::batch([nav_task, keep_alive])`.

### 3. Message Queue Flooding

**Symptom:** `MESSAGE QUEUE OVERLOAD: 62 messages pending`

**Root Cause:** `update_replay_mode()` created a new keep-alive task every frame, faster than 50ms processing.

**Fix:** Track `replay_keep_alive_pending` flag to ensure only one keep-alive is in flight at a time.

### 4. Metrics Skewed by Idle Time

**Symptom:** FPS metrics included idle time at directory boundaries when navigation was stopped.

**Fix:** Added `at_boundary` flag to ReplayController. When at boundary, metrics collection is paused to avoid skewing results with idle frames.

### 5. RestartIteration Not Resetting Position

**Symptom:** New iterations started from where previous iteration ended instead of beginning.

**Fix:** `RestartIteration` action fully reloads the directory to reset pane position to index 0.

## Refactoring

### Removed Dead Code from replay.rs
- Removed unused `Pausing` state from `ReplayState` enum
- Removed unused `should_navigate_right()` and `should_navigate_left()` methods
- Combined redundant `Right` and `Both` match arms in `on_ready_to_navigate()`

### Extracted Helper Methods
```rust
/// Finalize current metrics and prepare for next phase
fn finalize_current_metrics(&mut self) {
    if let Some(mut metrics) = self.current_metrics.take() {
        metrics.finalize();
        if self.config.verbose { metrics.print_summary(); }
        self.completed_metrics.push(metrics);
    }
    self.at_boundary = false;
}

/// Check if navigation phase should transition
fn should_transition(&self, elapsed: Duration) -> bool {
    const MIN_METRICS_DURATION: Duration = Duration::from_secs(1);
    elapsed >= self.config.duration_per_directory ||
        (self.at_boundary && elapsed >= MIN_METRICS_DURATION)
}
```

### Extracted replay_handlers.rs
Moved `update_replay_mode()` and `process_replay_action()` from app.rs to dedicated module:
- app.rs: 970 → 748 lines (-222 lines)
- New replay_handlers.rs: 230 lines

## Metrics Output

### JSON Format
```json
{
  "image_fps": {
    "min": 45.2,
    "max": 62.1,
    "avg": 55.3,
    "last": 58.7
  }
}
```

### Markdown Format
```
| Directory | Direction | Duration | Frames | UI FPS (avg) | Image FPS (avg) | Image FPS (last) | Memory (avg) |
```

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

## Commits (13 total)

```
97aa797 Port replay mode for performance benchmarking
286cdc9 Fix replay mode stalling during left navigation
4cd37c9 Prevent replay keep-alive message flooding
7372918 Add --auto-exit flag for replay mode automation
fdc3d02 Add --output-format flag for replay results (json, markdown, text)
4c73ac0 Fix replay metrics skewed by idle time at directory boundaries
20b881b Fix RestartIteration not resetting pane position for new iteration
e68fa92 Add --skip-initial option to exclude cached images from FPS metrics
94e380c Switch to remote iced dependencies
edf9ac2 Fix clippy warnings in replay code
bf64ccd Refactor replay.rs: remove dead code and extract helpers
a40712a Extract replay handlers to separate module
2f2bcaa Add last_image_fps to replay metrics output
```

## Next Steps

- Add slider navigation mode (`--nav-mode slider`) for comparison benchmarking
- See `tmp_slider_replay_handoff.md` for implementation plan

## References

- Roadmap: `docs/plans/roadmap-0.3-to-0.4.md` (Phase 2.1: Port Replay Mode)
- Legacy branch: `feat/replay-mode-legacy`
