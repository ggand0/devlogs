# 001: Phase 1 — Minimal egui Image Viewer

**Date:** 2026-03-07

## What was built

Phase 1 of the viewskater-egui prototype: a minimal image viewer that exercises the core rendering and navigation patterns needed for the LeRobot curation tool.

## Architecture

Three files, ~310 lines total:

- `src/main.rs` (~260 lines) — `App` struct, eframe integration, all UI logic
- `src/file_io.rs` (~50 lines) — directory enumeration, format filtering, path resolution
- `Cargo.toml` — 7 dependencies (eframe, egui, image, clap, log, env_logger, natord)

### App state

```rust
struct App {
    image_paths: Vec<PathBuf>,
    current_index: usize,
    current_texture: Option<TextureHandle>,
    zoom: f32,
    pan: Vec2,
}
```

Single `TextureHandle` — image is decoded synchronously on navigate via `image::open()` → `to_rgba8()` → `ctx.load_texture()`. No background thread yet (that's Phase 2's sliding window cache).

### Update loop structure

```
update()
  ├─ handle_dropped_files()   — drag-and-drop directory/file opening
  ├─ handle_keyboard()        — arrow keys, Home/End
  ├─ update_title()           — "filename.png (3/40) - viewskater-egui"
  ├─ show_bottom_panel()      — navigation slider + position label
  └─ show_central_panel()
       └─ show_image()        — contain-fit display, zoom/pan interaction
```

### Zoom/pan implementation

Interaction is processed *before* painting (zero-frame-delay):

1. `ui.allocate_rect(available, Sense::click_and_drag())` — captures all interaction
2. Scroll wheel → exponential zoom `(scroll * 0.003).exp()`, cursor-centered
3. Pinch-to-zoom via `ui.input(|i| i.zoom_delta())`
4. Drag → pan offset
5. Double-click → reset zoom=1.0, pan=ZERO
6. Paint image at computed `display_rect` using updated values

Cursor-centered zoom math:
```
delta_pan = cursor_rel * (1.0 - new_zoom / old_zoom)
```
where `cursor_rel = hover_pos - (available.center() + old_pan)`.

### File enumeration

- Supported formats: jpg, jpeg, png, bmp, webp, gif, tiff, tif, qoi, tga
- Hidden files (dotfiles) filtered out
- Natural sort via `natord` crate (handles `img2.png` before `img10.png`)
- Path resolution: accepts both files (uses parent dir, jumps to file index) and directories

## egui immediate mode vs iced's Elm architecture

In iced, the framework orchestrates separate phases:
```
Event → Message → update(msg) → view() → diff → render
```
You return a widget tree from `view()`, iced diffs it against the previous one, and renders the changes. You never touch rendering directly.

In egui, **there is only `update()`**. It's called once per frame, and you do everything in it — read input, mutate state, and describe what to draw — all inline. There's no separate `view()` return value, no widget tree diffing, no message enum. The UI is a side effect of running code, not a data structure.

### `ctx: &egui::Context`

`ctx` is the framework's entire API surface. Everything goes through it:

- **Read input**: `ctx.input(|i| i.key_pressed(...))` — closure gives a snapshot of all input for this frame (keys, mouse, scroll, dropped files). This replaces iced's `Event`/`Message` system.
- **Create panels/layout**: `egui::CentralPanel::default().show(ctx, |ui| { ... })` — `ctx` is the entry point. The closure receives `ui: &mut Ui`, which is the drawing cursor scoped to that panel.
- **Manage textures**: `ctx.load_texture(...)` — uploads pixel data to GPU, returns a `TextureHandle`.
- **Control window**: `ctx.send_viewport_cmd(ViewportCommand::Title(...))`.

### `ui: &mut Ui` inside panels

Once inside a panel closure, `ui` handles layout and widgets:
```rust
egui::TopBottomPanel::bottom("nav").show(ctx, |ui| {
    // ui is scoped to this panel's rect
    let response = ui.add(egui::Slider::new(...));  // returns Response
    if response.changed() { /* act immediately */ }
    ui.label("text");
});
```
`ui.add(widget)` returns a `Response` — did the user click, drag, hover? You check and act *immediately*. No message enum, no separate handler function.

### Comparison to a game loop

Very similar. A game loop: `poll_input() → update_state() → render()`. egui's `update()` is the body of that loop. eframe handles the outer loop (winit events, wgpu present, frame pacing). The difference: egui provides layout widgets (panels, sliders, labels) so you're not doing all the rect math yourself.

### Biggest contrast with iced

In iced, responding to an arrow key press:
1. `Event::KeyPressed` → `Message::NavigateRight`
2. `update()` handles the message, mutates state
3. `view()` rebuilds the entire widget tree
4. iced diffs and re-renders

In egui:
```rust
if ctx.input(|i| i.key_pressed(Key::ArrowRight)) {
    self.navigate(1, ctx);  // mutate state right here
}
// later in same function, slider and image read updated state
```
One function, one pass, everything in order. This is why egui fits continuous rendering (video playback, animation) — you're already in the render loop.

## Key egui API observations

- `ctx.load_texture()` returns a `TextureHandle` that manages GPU lifetime automatically. Old handles are freed on drop.
- `ui.painter().image()` paints at an arbitrary rect — no widget layout constraints. Good for zoom/pan.
- `raw_scroll_delta` (not `scroll_delta`) is the correct field in egui 0.31 for raw wheel input.
- `egui::ViewportCommand::Title()` updates the window title each frame — simple and works.
- `RawInput::dropped_files` provides drag-and-drop with `path: Option<PathBuf>` on desktop.

## What's deferred to Phase 2

- Sliding window cache (`Vec<Option<TextureHandle>>` + background decode thread)
- Async image loading via `mpsc` channels
- Smooth slider drag (currently decodes synchronously on every slider change)
- Continuous key-hold navigation at frame rate (currently uses OS key repeat)
