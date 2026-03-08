# 006: Dual Pane Mode

**Date:** 2026-03-08

## Goal

Add side-by-side dual pane viewing with synced navigation — a core feature of viewskater for comparing image datasets.

## Architecture

### PaneState extraction

Extracted per-pane fields from `App` into a `PaneState` struct:

```rust
struct PaneState {
    image_paths: Vec<PathBuf>,
    current_index: usize,
    current_texture: Option<egui::TextureHandle>,
    zoom: f32,
    pan: egui::Vec2,
    cache: Option<cache::SlidingWindowCache>,
}
```

`App` holds `panes: Vec<PaneState>`. Single pane = 1 element, dual = 2.

Methods moved to `PaneState`: `open_path`, `load_sync`, `navigate`, `jump_to`, `show_content`, `show_image`, `can_navigate_forward/backward`, `poll_cache`.

`App` retains coordination logic: `handle_keyboard`, `handle_dropped_files`, `show_bottom_panel`, `show_central_panel`, plus `perf::ImagePerfTracker` (shared across panes).

### Dual pane layout

Uses `allocate_new_ui` (egui 0.31 replacement for deprecated `allocate_ui_at_rect`) to create sub-regions:

```rust
let available = ui.available_rect_before_wrap();
let divider_w = 2.0;
let half_w = (available.width() - divider_w) / 2.0;

let left_rect = Rect::from_min_size(available.min, vec2(half_w, available.height()));
let right_rect = Rect::from_min_size(
    pos2(available.min.x + half_w + divider_w, available.min.y),
    vec2(half_w, available.height()),
);
```

A vertical line is drawn between panes via `ui.painter().vline()`.

### Borrow checker: `split_at_mut`

Rendering two panes requires mutable access to both simultaneously (each pane's `show_content` needs `&mut self` for zoom/pan state). Solved with `split_at_mut(1)`:

```rust
let (first, rest) = self.panes.split_at_mut(1);
ui.allocate_new_ui(UiBuilder::new().max_rect(left_rect), |ui| first[0].show_content(ui));
ui.allocate_new_ui(UiBuilder::new().max_rect(right_rect), |ui| rest[0].show_content(ui));
```

### Synced navigation

Keyboard nav iterates all panes. The gating mechanism ensures sync:

```rust
// Only advance if ALL panes have the next image cached
let all_ready = self.panes.iter().all(|p| p.is_next_cached(1));
if all_ready {
    // fold instead of any() to avoid short-circuit
    let any_advanced = self.panes.iter_mut().fold(false, |acc, p| p.navigate(1) || acc);
}
```

Two bugs fixed during implementation:
1. **`iter_mut().any()` short-circuits** — once pane 0 returned true, pane 1's `navigate()` was never called. Fixed with `fold()`.
2. **Independent cache gating** — each pane independently decided whether to advance, causing desync. Fixed by pre-checking `all()` panes are cache-ready before advancing any.

### Slider

Single shared slider drives both panes. During drag, sync-decodes both panes. On release, both caches rebuild around the new position.

### Tab toggle

`Tab` toggles between single and dual pane:
- **Single → dual**: creates a second pane with the same directory and position as pane 0
- **Dual → single**: `truncate(1)` removes pane 1

### Entry points

- **CLI**: `viewskater-egui /dir1 /dir2` opens dual pane immediately
- **Drag-and-drop**: first drop opens pane 0, second creates pane 1, subsequent drops replace pane 0
- **Tab key**: toggles at runtime

### Draggable divider

The divider is implemented without any external widget — just `allocate_rect` for a hit area and `drag_delta()` for interaction.

**State**: `App` stores `divider_fraction: f32` (0.0–1.0, default 0.5). Left pane width = `(available_width - divider_w) * fraction`.

**Hit area**: A 12px-wide invisible rect centered on the 4px visual line, allocated via `allocate_rect(grab_rect, Sense::click_and_drag())`. This gives a comfortable grab zone without making the visual divider thick.

**Drag handling**:
```rust
if divider_response.dragged() {
    let usable = available.width() - divider_w;
    let delta = divider_response.drag_delta().x;
    self.divider_fraction = (self.divider_fraction + delta / usable).clamp(0.1, 0.9);
}
```

Fraction is clamped to 10%-90% so neither pane can be collapsed to zero. Double-click resets to 50/50.

**Visual feedback**: The divider line color changes on hover (gray 60 → 100) and drag (→ 140). `CursorIcon::ResizeHorizontal` is set when hovering or dragging.

**Total code for the draggable divider**: ~30 lines. The iced version required a forked `iced_aw::Split` widget (~400 lines) with custom `Renderer` trait implementations, `overlay()` for drag handles, and message-based state updates.

## Comparison with iced viewskater

| Aspect | iced viewskater | egui viewskater |
|--------|----------------|-----------------|
| Split widget | Fork of `iced_aw::Split` (~400 lines) with custom divider, resize handles, drag logic | ~30 lines: `allocate_new_ui` + `split_at_mut` + `vline` |
| Pane coordination | `LoadingStatus` queue with `panes_to_load` filtering, `LoadOperation` per-pane routing | `Vec<PaneState>` with `all()` gate + `fold()` advance |
| Synced nav | `are_panes_cached_next()` checks all panes before `render_next_image_all()` | `is_next_cached()` on each pane, `all()` gate |
| Slider sync | Per-pane `ImageIndex` in `SyncedImageSplit`, custom message routing | Single slider value applied to all panes |

The egui version is dramatically simpler — the split widget alone is ~13x less code.

## Fixing DnD pane targeting on Linux (X11)

### The problem

In dual pane mode, dropping a file onto the right pane almost always registered as the left pane. Debug logging revealed the pointer position was `(0.0, 0.0)` on most drops — egui had no cursor position during external DnD operations.

### Root cause: winit + XDnD protocol

On Linux X11, external drag-and-drop uses the XDnD protocol, which is separate from normal cursor movement. Winit 0.30.x only emitted `HoveredFile` / `DroppedFile` / `HoveredFileCancelled` events — none of which carry a cursor position. The `CursorMoved` event is not emitted during external DnD because the X11 pointer events are routed through the XDnD protocol instead.

The XDnD protocol works as follows:

1. **`XdndEnter`** — drag enters the window. Contains the list of supported data types but no position.
2. **`XdndPosition`** — sent every time the cursor moves over the window. Contains the cursor position packed into `data.l[2]` as `(x << 16) | y` in **root window coordinates** (not window-local).
3. **`XdndDrop`** — file is released. Contains **no position** — the application is expected to have stored it from the most recent `XdndPosition`.
4. **`XdndLeave`** — drag exits without dropping.

### Fix: custom winit fork + egui-winit patch

**winit fork** (`ggand0/winit`, branch `custom-dnd-0.30.13`):

Cherry-picked 6 commits from the existing `custom-dnd-0.30.0` branch (which added `DragEnter`, `DragOver`, `DragDrop`, `DragLeave` events with position fields to replace the old `HoveredFile`/`DroppedFile`/`HoveredFileCancelled` events) onto winit v0.30.13. Then fixed the X11 implementation which had placeholder `(0.0, 0.0)` positions:

1. Extract packed coordinates from `XdndPosition`'s `data.l[2]` via bit shifting
2. Convert from root window coordinates to window-local coordinates using `translate_coordinates(root, window, x, y)`
3. Store the position in `dnd.position` on the `Dnd` struct
4. Use the stored position when emitting `DragOver` (from `XdndPosition`) and `DragDrop` (from `XdndDrop`)
5. Emit `DragDrop` once with all paths instead of per-path

```rust
// XdndPosition handler — extract and convert coordinates
let packed_coordinates = xev.data.get_long(2);
let (root_x, root_y) = Dnd::unpack_position(packed_coordinates);
if let Ok(cookie) = wt.xconn.xcb_connection()
    .translate_coordinates(wt.root, window, root_x, root_y)
{
    if let Ok(reply) = cookie.reply() {
        self.dnd.position = Some((reply.dst_x, reply.dst_y));
    }
}

// XdndDrop handler — use stored position
let position = self.dnd.position
    .map(|(x, y)| PhysicalPosition::new(x as f64, y as f64))
    .unwrap_or_else(|| PhysicalPosition::new(0.0, 0.0));
```

**egui-winit patch** (`../egui-winit-custom`, local path dependency):

Replaced the `HoveredFile`/`DroppedFile`/`HoveredFileCancelled` handlers with `DragEnter`/`DragOver`/`DragDrop`/`DragLeave` handlers. Critically, each position-carrying event calls `self.on_cursor_moved(window, *position)` which:
- Sets `pointer_pos_in_points` on the egui-winit state
- Pushes `egui::Event::PointerMoved` into the raw input

This means egui's `pointer.hover_pos()` and `pointer.latest_pos()` are updated during DnD — the same mechanism used for normal cursor tracking.

**Cargo.toml patch**:
```toml
[patch.crates-io]
winit = { git = "https://github.com/ggand0/winit.git", branch = "custom-dnd-0.30.13" }
egui-winit = { path = "../egui-winit-custom" }
```

### Note on upstream status

Winit PR #4079 ("Rework DnD redux") was merged Jan 2025 into winit 0.31, which properly implements DnD with cursor positions. However, eframe 0.31.1 pins winit 0.30.x. When eframe upgrades to winit 0.31+, the custom fork and egui-winit patch can be dropped.

The iced version of viewskater required the same workaround — a forked winit with custom DnD events (`ggand0/winit`, branch `custom-dnd-0.30.1`), and a forked iced repo (`ggand0/iced`, branch `custom-0.13`) to handle them.
