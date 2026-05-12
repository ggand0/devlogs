# 029 Window State Persistence

## PR #20: Egui Persistence (by hml-pip)

PR enables eframe's built-in `persistence` feature to save/restore window state across sessions. State is written to `app.ron` in eframe's `storage_dir` (e.g. `~/.local/share/viewskater-egui/app.ron`).

### Changes in the PR

- `Cargo.toml`: add `"persistence"` feature to eframe
- `src/app.rs`: remove `is_fullscreen` field from `App` struct, replace all usage with live viewport query `ctx.input(|i| i.viewport().fullscreen.unwrap_or(false))`
- `src/app/handlers.rs`: same fullscreen query replacement in `toggle_fullscreen` and escape-key handling
- `src/menu.rs`: same in menu bar fullscreen label toggle

The `is_fullscreen` removal is a good cleanup: instead of a manually-tracked bool that could drift out of sync, it reads the actual viewport state from egui.

### Window size not restoring — root cause

After enabling persistence, window **position** was restored but **size** was not. Two things were overriding the persisted size:

1. **`main.rs` — hardcoded `with_inner_size`**: `ViewportBuilder::default().with_inner_size([1280.0, 720.0])` runs unconditionally, overriding eframe's persisted size before the window even opens.

   **Fix**: check if `app.ron` exists (indicating a previous session); only set `with_inner_size` as a first-launch default.

2. **`app.rs` — first-frame DPI resize**: the `initial_size_set` block on the first frame of `update()` sends `ViewportCommand::InnerSize(1280/ppp, 720/ppp)` to correct for DPI scaling. This fires after eframe has already restored the persisted size, stomping on it.

   **Fix**: compare the current window size against the default (1280x720). If they differ, eframe already restored a persisted size, so skip the resize command.

### Persisted state format

The `app.ron` file stores egui memory and a `"window"` key:
```
"window": "(inner_position_pixels:Some(...), outer_position_pixels:Some(...),
            fullscreen:false, maximized:false, inner_size_points:Some((x:1024.0,y:576.0)))"
```

eframe reads this on startup and applies it to the viewport — but only if nothing else overwrites the size afterward.

### Known limitations (noted by contributor)

- Does not retain window size when exiting in fullscreen mode
- On Windows multi-monitor, exiting fullscreen on monitor 1 may restore to monitor 2
