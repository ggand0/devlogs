# 012 Dual Pane Improvements

Branch: `feat/dual-pane`

## Summary

Bugfixes and feature additions for dual-pane mode, including a new independent navigation mode with per-pane sliders, pane selection, synced zoom/pan, responsive UI, and zoom persistence.

## Commits

| Commit | Description |
|--------|-------------|
| `95de271` | Fix keyboard nav and slider when only second pane has images |
| `839547b` | Clip zoomed images to pane boundary in dual-pane mode |
| `5e713fc` | Add independent dual-pane mode with per-pane sliders |
| `a8f9689` | Split footer in dual-pane mode with divider and padding |
| `088b78f` | Add pane selection with accent bar indicator in independent mode |
| `d72460b` | Add clickable pane selection strips in independent mode |
| `1a6b4a3` | Make footer responsive to window width |
| `fba2dc6` | Make menu bar responsive to window width |
| `a4c456b` | Make empty pane message responsive and shorten text |
| `ae0c008` | Add synced zoom/pan for dual-pane mode |
| `e551de6` | Preserve zoom/pan state across navigation |
| `c7d2443` | Default sync zoom/pan to enabled |
| `02a20cd` | Add Reset Zoom menu item in View menu |
| `d6ad479` | Rename Reset Zoom to Reset Zoom/Pan in View menu |
| `5d3bc6a` | Extract handler methods into app/handlers.rs and consolidate slider logic |
| `b4c82ea` | Fix Tab keybind conflict, remove dead decode display, tighten visibility |

## Bugfixes

### Right-pane-only navigation broken
When only the second pane had images loaded (e.g. via drag-and-drop), keyboard navigation was completely blocked and the slider handle reset to zero on every interaction.

Root cause: two issues compounding.
- `is_next_cached()` returns `false` for empty panes, and `handle_keyboard` required `all()` panes to be cached before advancing. An empty pane blocked navigation for all panes.
- `show_slider_panel` used `panes.first()` for the slider position, which was pane 0 (empty), so the slider always showed index 0.

Fix: skip empty panes in the `all_ready` check (`p.image_paths.is_empty() || p.is_next_cached()`), and use `find(|p| !p.image_paths.is_empty())` for the slider's current index.

### Tab key bound to both toggle_dual and toggle_footer
Both `toggle_dual` and `toggle_footer` were bound to `Tab`. Because `toggle_footer` was checked first with an early return, `toggle_dual` via Tab was dead code. Fixed by removing the `toggle_dual` Tab binding — Tab now only toggles the footer.

### Zoomed images bleeding across panes
When zooming in on an image in dual-pane mode, the zoomed image would render past its pane boundary into the adjacent pane.

Fix: use `ui.painter_at(available)` instead of `ui.painter()` in `show_image()`. This creates a painter with a clip rect matching the pane's allocated area.

## Features

### Independent dual-pane mode (Ctrl+3)
A new layout mode where each pane has its own navigation slider, allowing independent browsing of different positions in each pane's image set.

- View menu now shows three radio options: Single Pane (Ctrl+1), Dual Pane Synced (Ctrl+2), Dual Pane Independent (Ctrl+3)
- Tab toggle preserves the last-used dual-pane mode
- Extracted `paint_nav_slider()` as a reusable slider rendering function shared by both the full-width synced slider and per-pane independent sliders
- Slider results are collected during rendering and processed after the panel closure returns, avoiding borrow conflicts

### Split footer
In dual-pane mode, the footer splits into two halves aligned with the pane divider. Each half shows metadata (filename, resolution, file size, index) for its respective pane. A divider line in the footer matches the central panel's divider. 4px padding separates footer text from the divider.

### Pane selection
In independent mode, panes can be selected/deselected to control which panes receive keyboard navigation input.

- 18px clickable strip at the top of each pane showing "1" / "2"
- Accent-colored when selected, muted gray when deselected
- Click to toggle, or use bare `1`/`2` keys
- Keyboard nav (arrows, Home/End, Shift+skate) only affects selected panes
- Both panes start selected by default
- Selection UI is scoped to independent mode only; synced mode always navigates all panes

### Responsive UI
All text-bearing UI elements progressively hide content as window width shrinks.

**Footer**: file size drops first, then resolution, then filename, then index shortens to just the number ("1" instead of "1 / 100"), then hides entirely. Text widths are measured with `ui.fonts()` before rendering to decide visibility.

**Menu bar**: menus always take priority over FPS display. FPS drops decode time first, then hides entirely. Menus hide right-to-left (Help, View, Edit). At extreme widths "File" shortens to "F", then the bar empties.

**Empty pane message**: "Drop an image or folder here" shortens to "Drop image", then hides.

### Synced zoom/pan
A "Sync Zoom/Pan" toggle in settings (Display section, default on, persisted). When enabled, zooming or panning in one pane propagates the values to the other pane. `show_image()` returns a bool indicating whether interaction occurred; after both panes render, the interacted pane's zoom/pan is copied to the other.

### Persistent zoom/pan
Zoom and pan state now persists across keyboard navigation and slider scrubbing. Previously, every `navigate()`, `jump_to()`, and slider drag reset zoom to 1.0 and pan to zero. Now the only resets are:
- Double-click on the image (explicit user action)
- `open_path()` (new content loaded)
- `close()` (clearing the pane)
- View > Reset Zoom menu item

### Reset Zoom menu item
Added "Reset Zoom" to the View menu, between the pane layout options and the overlay toggles. Resets zoom and pan for all panes.

## Architecture notes

### DualPaneMode enum
```rust
enum DualPaneMode {
    Synced,      // shared slider, all panes navigate together
    Independent, // per-pane sliders, selection controls keyboard nav
}
```

### Pane selection model
Selection state lives on `Pane` itself (`pub selected: bool`) rather than on `App`, so it scales naturally to N panes. Navigation uses an `is_active` closure to decide per-pane participation:
```rust
let use_selection = self.dual_pane_mode == DualPaneMode::Independent;
let is_active = |p: &Pane| !use_selection || p.selected;
```

### Slider result pattern
Per-pane slider actions are collected as `Vec<(usize, SliderResult)>` during the `CentralPanel::show()` closure, then processed after the closure returns via `apply_slider_result_one()`. This avoids borrow conflicts between the UI closure (which borrows panes for rendering) and the action handler (which mutates panes).

### Responsive text pattern
Footer and menu bar use a measure-before-render approach: all text widths are computed upfront via `ui.fonts(|f| f.layout_no_wrap(...))`, then visibility flags are set based on available width. This avoids the borrow conflict of holding a `measure` closure (immutable borrow on `ui`) while rendering (mutable borrow).

## Refactoring

### app.rs → app.rs + app/handlers.rs
Split the monolithic `app.rs` into two files following the same pattern as the iced version's `src/app/message_handlers.rs`:

- **`src/app.rs`** — `App` struct, `new()`, `DualPaneMode`, `SliderResult`, `paint_nav_slider()`, UI rendering methods (`update_title`, `show_slider_panel`, `show_central_panel`), and the `eframe::App` trait impl.
- **`src/app/handlers.rs`** — All handler/action methods: `handle_keyboard`, `handle_dropped_files`, `handle_menu_action`, `apply_slider_result_all/one`, `apply_settings_to_caches`, `set_single_pane`, `set_dual_pane`, `open_folder_dialog`, `open_file_dialog`, `close_images`.

Rust supports splitting `impl App` blocks across files in the same crate. The handler methods use `pub(super)` visibility so they're accessible from the parent module but not leaked to the rest of the crate.

### Slider result deduplication
`apply_slider_result_all` and `apply_slider_result_one` shared ~15 lines of identical per-pane logic (cache lookup → sync load → throttle). This was extracted into two methods on `Pane`:

```rust
// Returns true if image was loaded
pub fn apply_slider_target(&mut self, idx: usize, ctx: &egui::Context) -> bool;
// Cancel throttle, re-center sliding window cache
pub fn apply_slider_release(&mut self);
```

The app-level methods are now thin wrappers that loop/index into panes and call these `Pane` methods.

### Code cleanup
- Removed dead `_id_salt` parameter from `hover_row` and all 17 call sites
- Removed always-zero `decode_ms` parameter from `record_image_load()` and the misleading "Decode: 0.0ms" secondary FPS display; `fps_primary()`/`fps_secondary()` consolidated into single `fps_text()`
- Fixed per-frame `Pane::new(0, 0)` allocation in `update_title` (created a full Pane with Vec/LRU cache every frame just for `unwrap_or`)
- Tightened `pub` → `pub(crate)` across `Pane` (struct + all fields + methods), `DualPaneMode`, `MenuAction`, `ImagePerfTracker`, `show_menu_bar`, `show_footer`; made `Pane::load_sync` and `slider_loader` fully private
