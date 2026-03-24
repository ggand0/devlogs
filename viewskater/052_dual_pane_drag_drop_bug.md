# Dual-Pane Drag-and-Drop Bug Investigation

**Date:** 2026-03-24

## Bug

Drag-and-dropping files onto specific panes in dual-pane mode doesn't work on Linux. All dropped files load into the right (second) pane regardless of where the user drops them.

## Root Cause (Two Layers)

### Layer 1: winit fork had placeholder (0,0) positions on X11

The iced-compatible winit fork (rev `47974007`, branch `custom-dnd-0.30.0`) emitted `WindowEvent::DragDrop` with hardcoded `PhysicalPosition::new(0.0, 0.0)` on X11. The XdndPosition handler had the position extraction code commented out:

```rust
// In event_processor.rs XdndDrop handler (old code):
event: WindowEvent::DragDrop {
    paths: vec![path.clone()],
    position: PhysicalPosition::new(0.0, 0.0), // Placeholder
},
```

The egui-compatible winit fork (branch `custom-dnd-0.30.13`) had this fixed in commit `f94ea9ea`, which extracts the cursor position from `XdndPosition`'s packed coordinates and converts root window coordinates to window-local coordinates via `translate_coordinates`.

### Layer 2: Split widgets discarded the event position on Linux

Both `split.rs` and `synced_image_split.rs` had Linux-specific `FileDropped` handlers that discarded the event position with `_` and fell back to `cursor.position().unwrap_or_default()`:

```rust
Event::Window(iced::window::Event::FileDropped(path, _)) => {
    let cursor_pos = cursor.position().unwrap_or_default();
```

This was a workaround for Layer 1 — since the event position was always (0,0), `cursor.position()` was the only option. But `cursor.position()` is unreliable during external drag-and-drop on X11 because the window doesn't receive normal cursor movement events during a drag from another application. It returns a stale position, which happened to fall in the right pane.

## Affected Files

- `src/widgets/split.rs` — Linux FileDropped handler
- `src/widgets/synced_image_split.rs` — Linux FileDropped handler
- `../winit/src/platform_impl/linux/x11/event_processor.rs` — XdndPosition/XdndDrop handlers
- `../winit/src/platform_impl/linux/x11/dnd.rs` — Dnd struct (added position field)

## Commits That Introduced the Bug

### 1. `split.rs` — Commit `8243cd6`
- **Message:** "Fix split pane layout: center divider with even spacing and correct drop targets"
- **Date:** 2025-04-19
- **Branch:** `fix/split-layout`, merged via PR #33 (merge commit `61392e8`, 2025-04-20)

### 2. `synced_image_split.rs` — Commit `8d99560`
- **Message:** "Add synchronized zooming for dual-pane view"
- **Date:** 2025-05-20
- **Branch:** `feat/zoom-sync`, merged via PR #44 (merge commit `e85e151`, 2025-05-22)

### 3. Partial fix that missed Linux — Commit `86edb32`
- **Message:** "Fix file drop detection on macOS and Windows in SyncedImageSplit widget"
- **Date:** 2025-05-22
- **Branch:** `feat/zoom-sync` (same PR #44)
- Fixed only the macOS/Windows handler. Linux was left using `cursor.position()`.

## Fix

### winit fork
Cherry-picked commit `f94ea9ea` from `custom-dnd-0.30.13` onto `47974007` as branch `fix/x11-dnd-position`. The fix:
- Extracts cursor position from XdndPosition packed coordinates (`xev.data.get_long(2)`)
- Converts root window coordinates to window-local via `xcb translate_coordinates`
- Stores position in `Dnd.position` field, used by both DragDrop and DragOver events

### Split widgets
Changed both Linux handlers to use the event position instead of `cursor.position()`:

```rust
Event::Window(iced::window::Event::FileDropped(path, position)) => {
    let drop_position = Point::new(position.x as f32, position.y as f32);
    if first_layout.bounds().contains(drop_position) { ... }
    else if second_layout.bounds().contains(drop_position) { ... }
}
```

## Why the egui version worked

The egui version uses winit branch `custom-dnd-0.30.13` (commit `ea442bd8`) which already had the `f94ea9ea` fix. The iced version used `custom-dnd-0.30.0` (rev `47974007`) which did not.

## Event Pipeline

```
X11 XdndPosition → extract packed coords → translate_coordinates → dnd.position
X11 XdndDrop → WindowEvent::DragDrop { paths, position: dnd.position }
  → iced conversion.rs: position.to_logical(scale_factor), f64 → u64
  → iced_core window::Event::FileDropped(Vec<PathBuf>, PhysicalPosition<u64>)
  → SyncedImageSplit on_event() → check pane bounds → Message::FileDropped(pane_index, path)
```

## Note: Wayland

The winit fork has no drag-and-drop support on Wayland — only X11. The `DragDrop`/`DragOver`/`DragLeave` events are not emitted on Wayland at all.
