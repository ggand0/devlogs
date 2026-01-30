# macOS Window State Testing Handoff

## Context
PR #61 implements automatic window state save/restore. We fixed issues on Linux X11 but introduced a regression on macOS.

## Current Branch
`feat/window-state` with 2 commits on top of the contributor's PR:
1. `2dbf65a` - Initialize window_position from CONFIG instead of (0,0)
2. `9fdcf72` - Add with_position and with_maximized for X11 compatibility

## The Problem
On macOS, window position is no longer retained after commit `9fdcf72`.

The commit removed `set_outer_position()` after window creation and added `.with_position()` during creation:
```rust
// Removed:
window.set_outer_position(PhysicalPosition::new(CONFIG.window_position_x, CONFIG.window_position_y));

// Added in WindowAttributes:
.with_position(PhysicalPosition::new(CONFIG.window_position_x, CONFIG.window_position_y))
```

## Platform Differences
- **X11**: `.with_position()` during creation works, `set_outer_position()` after creation doesn't
- **macOS**: `set_outer_position()` after creation works, `.with_position()` during creation apparently doesn't

## What Needs Testing
1. Confirm that window position is NOT retained on macOS with current code
2. Test if adding back `set_outer_position()` after line 1133 fixes macOS:
   ```rust
   window.set_outer_position(PhysicalPosition::new(CONFIG.window_position_x, CONFIG.window_position_y));
   ```
3. Verify this doesn't break X11 (needs both methods for cross-platform)

## Expected Fix
Keep both `.with_position()` (for X11) and `set_outer_position()` (for macOS/Windows):
```rust
let window = Arc::new(
    event_loop
    .create_window(
        winit::window::WindowAttributes::default()
            .with_inner_size(...)
            .with_position(PhysicalPosition::new(CONFIG.window_position_x, CONFIG.window_position_y))  // For X11
            .with_maximized(should_maximize)
            .with_title("ViewSkater")
            .with_resizable(true)
    )
    .expect("Create window"),
);
window.set_outer_position(PhysicalPosition::new(CONFIG.window_position_x, CONFIG.window_position_y));  // For macOS/Windows
```

## Test Steps
1. Build and run: `RUST_LOG=viewskater=debug cargo run --profile opt-dev`
2. Move window to a specific position
3. Close the app
4. Relaunch and verify window opens at saved position

## Files
- `src/main.rs:1117-1133` - Window creation code
- `src/app.rs:231` - window_position initialization
- `devlogs/041_window_state_pr_review.md` - Full context
