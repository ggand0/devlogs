# PR #29: Custom Window Persistence (viewskater-egui)

Scope: this devlog covers PR #29 only (custom hand-written save/load to a YAML file, branch `feat/window-presistence`). PR #20 takes a different code path (eframe's built-in `persistence` feature) and its issues are tracked in [2026-05-31-pr20-egui-builtin-window-persistence.md](2026-05-31-pr20-egui-builtin-window-persistence.md).

**Date**: May 31, 2026
**Repo**: `ggand0/viewskater-egui`
**PR**: [#29 "Implement window persistence feature"](https://github.com/ggand0/viewskater-egui/pull/29)
**Branch**: `hml-pip:feat/window-presistence` (note: misspelled "presistence" in the branch name)
**State**: DRAFT
**Author intent (from PR body)**: *"Compared to the egui persistence, this PR is not really meaningful right now, I'll just use this for my personal experiments. If there's no further improvement or interest, I'll just close this later."*

## Overview

PR #29 replaces eframe's built-in `persistence` feature with hand-written save/load code. It is by the same author as the iced version's window-state PR ([viewskater #61](https://github.com/ggand0/viewskater/pull/61)) and follows the same architectural pattern, but stripped down to a minimal first version without the platform workarounds in #61.

## How it works

### Cargo
- Does **not** enable eframe's `persistence` feature.
- Adds only `egui = { ..., features = ["serde"] }` so `egui::Vec2` / `egui::Pos2` can be serialized.

### Persistence file
- Location: `dirs::config_dir() / viewskater-egui / states.yaml` (on Windows: `%APPDATA%\viewskater-egui\states.yaml`).
- Format: YAML via `serde_yaml`.
- Separate from eframe's `app.ron` (which is not used).

### What is persisted
A single struct in `settings.rs`:
```rust
pub enum WindowState { Window, Maxmized, FullScreen }  // sic: "Maxmized" typo

pub struct WindowStates {
    pub window_size: egui::Vec2,        // only meaningful when state == Window
    pub window_position: egui::Pos2,
    pub window_state: WindowState,
}
```

### Restore flow (`main.rs`)
1. `WindowStates::load()` — reads `states.yaml` or returns defaults.
2. If `window_position != Pos2::default()`, calls `viewport.with_position(position)`.
3. Matches on `window_state`:
   - `Window` → `with_inner_size(window_size)`
   - `Maxmized` → `with_maximized(true)` (note: does **not** apply `window_size`)
   - `FullScreen` → `with_fullscreen(true)` (note: does **not** apply `window_size`)

### Save flow (`handlers.rs::save_window_states`)
Reads from `ctx.input(|i| i.viewport())`:
- Position from `outer_rect.left_top()`
- Size from `outer_rect.size()`
- State from `viewport().maximized` / `viewport().fullscreen`

Then calls `WindowStates::save()` to write YAML.

Triggered from three sites:
- Quit menu action (`MenuAction::Quit`)
- Ctrl+Q keyboard shortcut
- `close_requested()` check in the update loop

## Comparison with iced PR #61 (by the same author)

Same family — both define a `WindowState` enum, both persist via YAML, both hand-roll save/load. But #61 is much more elaborate:

| Feature | #29 | iced #61 |
|---|---|---|
| `WindowState` enum (Window/Maximized/FullScreen) | ✅ | ✅ |
| Custom YAML persistence | ✅ | ✅ |
| Explicit save on quit | ✅ | ✅ |
| Multiple position fields (`last_windowed_position`, `position_before_transition`) | ❌ | ✅ |
| Monitor tracking (`last_monitor: Option<MonitorHandle>`) | ❌ | ✅ |
| Visibility check before save (`get_window_visible`) | ❌ | ✅ |
| Continuous state tracking via `WindowResized` / `PositionChanged` events | ❌ | ✅ |
| Platform workarounds (Windows / macOS / X11) | ❌ | ✅ |

The author has indicated they could port #61's logic if there is interest: *"at worst, I can port the code made for the Iced version, but I'm not sure the added code complexity is worth it."*

## Known issues (from #20 thread comments)

These are explicitly scoped to #29 (either reported by a tester who tried #29, or reported by the author against their own implementation):

- **YelovSK on #29**: *"Seems to work fine, except for the white flash and the unmaximize not remembering the size."* So #1 (maximize white flash) and #2 (unmaximize loses normal size) from the #20 issue list are still present in #29.
  - For #2 specifically, YelovSK noted: *"in your PR, unmaximize restores a smaller window, and in egui's persistence, it restores the maximized size."* The smaller-window outcome aligns with #29 not applying `window_size` when state is `Maxmized` — the unmaximize fall-back size is whatever winit defaults to, not the previously-saved windowed size.
- **hml-pip on #29** (DPI scaling): *"when you set the scaling to anything other than 100% for a high-resolution monitor on Windows, maximized or full-screen window does not recover properly on the next startup in my implementation."* Not yet diagnosed.

## Issues found locally

### Window size not restored (windowed mode) — fix ported from #20
On the unmodified `feat/window-presistence` branch, restarting in windowed mode came up at the default size, not the previously-saved `window_size` from `states.yaml`.

**Root cause**: same bug as the one fixed in PR #20 commit [`ad70399`](https://github.com/ggand0/viewskater-egui/pull/20/commits/ad70399). The first-frame DPI physical-pixel resize in `app.rs` (`if !self.initial_size_set { ... ctx.send_viewport_cmd(InnerSize(logical)) }`) was firing on every launch, overriding the restored size whenever scale ≠ 100%. #29 set up `with_inner_size(states.window_size)` correctly in `main.rs`, but the DPI resize stomped on it on the first frame.

**Fix applied locally** (same shape as #20):
- `main.rs`: compute `has_persisted_state = dirs::config_dir().map(|d| d.join("viewskater-egui").join("states.yaml").exists()).unwrap_or(false)`, pass `!has_persisted_state` into `App::new`.
- `app.rs`: rename field `initial_size_set` → `needs_dpi_resize`, take it as a constructor parameter, change the first-frame gate to `if self.needs_dpi_resize`.

**Status**: REPRODUCED ✅ and verified fixed locally. Not committed/pushed.

## Notes

- The enum variant is spelled `Maxmized` (typo). It gets serialized as `maxmized` into `states.yaml` due to `#[serde(rename_all = "snake_case")]`. Renaming the variant after merge would break existing users' state files unless handled with a serde alias.
- "Known issues" from the #20 PR thread above were not all tested locally; the local section captures only what was reproduced on this branch directly.
