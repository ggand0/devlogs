# ViewSkater Replay Mode Implementation

**Date:** 2025-08-22

## Initial Request
User wanted to automate performance testing for their Rust image viewer (ViewSkater) by implementing a "replay" mode that:
- Launches GUI window (not CLI-only) to capture winit-related issues
- Automates continuous rendering by simulating Shift+A/D key presses
- Uses existing test directories of images
- Displays/records performance metrics (FPS values from ui.rs)
- Triggered via command-line option

## Implementation Overview
Created a comprehensive replay system with:

### Core Components
- **New `src/replay.rs` module** - Contains `ReplayController` state machine and `ReplayConfig`
- **CLI integration** - Added `clap` dependency for robust argument parsing
- **State machine** - `ReplayState` enum with states: `Inactive`, `LoadingDirectory`, `WaitingForReady`, `NavigatingRight`, `NavigatingLeft`, `Pausing`, `Finished`
- **Integration with existing navigation** - Uses existing `skate_right`/`skate_left` flags and `move_right_all`/`move_left_all` functions

### Command Line Interface
```bash
./viewskater --replay --test-dir <path> [OPTIONS]
```
Options include: `--duration`, `--nav-interval`, `--directions`, `--iterations`, `--verbose`

### Key Features
- **Multiple test directories** - Can test multiple image sets in sequence
- **Direction control** - Right, Left, or Both navigation patterns
- **Iteration control** - Specify number of replay cycles
- **Performance metrics** - Collects UI FPS, Image FPS, Memory usage
- **Comprehensive reporting** - Per-directory summaries and final aggregate report

## Major Issues Resolved

### 1. Compilation Errors
- **Scope issues** with `replay_config` in closures - Fixed by modifying `Runner` enum structure
- **Borrow checker conflicts** - Resolved by restructuring action computation vs application
- **Import path errors** - Corrected `IMAGE_RENDER_FPS` access path

### 2. Infinite Replay Loop
- **Problem**: Replay ran forever instead of stopping after specified iterations
- **Solution**: Added `--iterations` CLI argument and iteration tracking in `ReplayController`

### 3. Direction Switching Issues ("Both" mode)
- **Problem**: 500ms pause between right→left transition, 5-second delays
- **Solution**: Removed artificial pause, made transition immediate
- **Clarification**: Left navigation is time-based, doesn't need to return to index 0

### 4. Iteration Restart Delays
- **Problem**: 5-7 second delays between iterations due to full directory reloads
- **Solution**: Added `RestartIteration` action to avoid unnecessary reloading for same directory

### 5. Navigation Timing Issues
- **Problem**: Left/right navigation running longer than specified duration
- **Solution**: Separated duration timeout from navigation interval checks in state machine

### 6. Race Conditions (Major Issue)
- **Problem**: "Sometimes the first run won't happen" - iterations starting before app was ready
- **Root Cause**: Replay controller trying to navigate before app finished initialization
- **Solution**: Implemented readiness signaling system:
  - Added `WaitingForReady` state
  - Added `on_ready_to_navigate()` callback
  - Proper state cleanup for iteration restarts
  - Fixed both first iteration and subsequent iteration race conditions

## Final State
The replay mode now works reliably with:
- ✅ Consistent first iteration startup
- ✅ Proper iteration cycling without delays
- ✅ Accurate timing control
- ✅ Comprehensive performance metrics
- ✅ Clean termination after specified iterations
- ✅ Proper state management between iterations

## Files Modified
- `Cargo.toml` - Added clap dependency
- `src/main.rs` - CLI argument parsing and replay config setup
- `src/app.rs` - Integrated ReplayController, added action handling
- `src/replay.rs` - Complete replay system implementation (new file)

## Example Usage
```bash
# Basic replay with 2-second duration per direction
./viewskater --replay --test-dir "/path/to/images" --duration 2

# Multiple iterations with both directions
./viewskater --replay --test-dir "/path/to/images" --duration 2 --iterations 3 --directions both

# Custom navigation interval for slower systems
./viewskater --replay --test-dir "/path/to/images" --duration 5 --nav-interval 100 --verbose
```

## Technical Details

### State Machine Flow
1. `Inactive` → `LoadingDirectory` (when replay starts)
2. `LoadingDirectory` → `WaitingForReady` (when directory loads)
3. `WaitingForReady` → `NavigatingRight`/`NavigatingLeft` (when app signals ready)
4. Navigation states transition based on direction configuration and timing
5. `Finished` when all iterations complete

### Performance Metrics Collected
- **UI FPS**: Frame rate of the user interface
- **Image FPS**: Frame rate of image rendering pipeline  
- **Memory Usage**: Current memory consumption
- **Frame Counts**: Total frames processed per direction
- **Duration**: Actual time spent in each navigation phase

### Readiness Signaling
The key innovation that solved race conditions:
- Replay controller waits in `WaitingForReady` state
- App explicitly calls `on_ready_to_navigate()` when truly ready
- Ensures navigation only starts when app can properly respond
- Handles both initial directory loading and iteration restarts

## Implementation Summary
The implementation successfully automates the manual performance testing workflow while maintaining GUI rendering to capture the full spectrum of potential performance issues. The system is robust, configurable, and provides detailed performance insights for optimization work.
