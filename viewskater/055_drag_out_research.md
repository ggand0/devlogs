# Drag-Out (Drag Source) Feature Research

**Date:** 2026-05-04
**Issue:** #86 — Dragging the image outside the app into other applications

## Feature Description
Allow users to drag the currently displayed image from ViewSkater into external apps (GIMP, file managers, messaging apps, etc.) as a file drag-and-drop operation.

## Best Available Crate: `drag` v2.1 (CrabNebula)

- **Repo:** https://github.com/crabnebula-dev/drag-rs
- **Crate:** https://crates.io/crates/drag
- **Stars:** 117 | **License:** Apache-2.0 / MIT
- Built by the Tauri team specifically for dragging files OUT of a window

### API
```rust
drag::start_drag(
    &window,                        // &T: HasWindowHandle
    DragItem::Files(vec![path]),    // Files to drag
    Image::Raw(bytes),              // Preview icon
    |result, cursor_pos| { },       // Callback on drop/cancel
    Options::default(),
)
```

### Platform Implementations
| Platform | Method | Window Requirement |
|----------|--------|-------------------|
| Windows | OLE: `DoDragDrop`, `IDropSource`, `IDataObject`, `CF_HDROP` | `HasWindowHandle` |
| macOS | `NSDraggingSource`, `NSDraggingItem`, `NSPasteboardItem` via objc2 | `HasWindowHandle` |
| Linux (GTK) | GTK3 `drag_source_set`, `drag_begin_with_coordinates` | `gtk::ApplicationWindow` |

### Critical Linux Limitation
The `drag` crate's Linux implementation requires a `gtk::ApplicationWindow` reference, NOT a raw window handle. Standard winit does not expose a GTK window. Tauri's `tao` fork exposes `.gtk_window()` which is why it works there.

**Solution:** Add GTK window handle exposure to our custom winit fork (`ggand0/winit`, branch `custom-0.30`). This would allow passing the GTK window to the `drag` crate on Linux.

## Framework Support Status

### winit
- NO drag source support, not coming soon
- Relevant issues: #1499 (2020), #3229 (2023, filed by egui's emilk), #720
- Blocked on clipboard API redesign (#3367)

### iced
- No drag-out support. Only receives drops and internal widget DnD.
- ViewSkater's custom event loop accesses winit window directly, so integration bypasses iced entirely.

### egui
- No drag-out support. Same gap as iced, waiting on winit.

## Native Platform APIs (Reference)

**Windows:** `OleInitialize()` -> implement `IDropSource` + `IDataObject` -> `DoDragDrop()`. Use `CF_HDROP` format with `DROPFILES` struct. The `windows` crate provides all COM types.

**macOS:** Implement `NSDraggingSource` protocol, create `NSDraggingItem` with `NSPasteboardItem` containing file URLs, call `beginDraggingSession:withItems:event:source:` on the `NSView`.

**Linux/X11:** XDnD protocol v5 — take ownership of `XdndSelection`, send `XdndEnter`, `XdndPosition`, `XdndDrop` client messages. Use `text/uri-list` MIME type.

**Linux/Wayland:** Create `wl_data_source` via `wl_data_device_manager`, call `wl_data_device.start_drag()`, offer `text/uri-list` MIME type.

## Related Projects

**ripdrag** (https://github.com/nik012003/ripdrag)
- 689 stars, Rust + GTK4
- Standalone tool that creates a GTK4 window for dragging files out
- Uses GTK4's `DragSource` widget with `ContentProvider`
- Not integrable into winit apps directly, but good reference for the GTK approach

## Implementation Plan

### Phase 1: macOS + Windows (straightforward)
1. Add `drag = "2.1"` dependency
2. Detect mouse drag gesture on the image (mouse-down + movement beyond threshold)
3. Call `drag::start_drag(&window, DragItem::Files(vec![current_image_path]), ...)` 
4. ViewSkater already has `Arc<winit::window::Window>` which implements `HasWindowHandle`

### Phase 2: Linux (via winit fork)
1. Expose GTK window handle from our custom winit fork
2. Pass it to the `drag` crate's Linux backend
3. Alternatively: implement XDnD/Wayland drag source directly in the fork

### Integration Point in ViewSkater
- Trigger on mouse drag from the image area (not the slider/UI)
- Need to distinguish between pan gestures and drag-out intention (e.g., require drag to window edge, or use a modifier key, or a dedicated drag handle)
