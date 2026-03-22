# 016 Planned Features for 0.1.1

## Context

0.1.0 shipped with feature parity to the iced version's baseline: image rendering with sliding window cache, dual-pane modes, fullscreen, menu/footer/settings UI, cross-platform app icon, and support for images >8192px.

The features below are carried over from the iced version's later releases and the initial feature brainstorm for the egui rewrite.

## Planned for 0.1.1

### 1. Window state preservation
Save window size and position on quit, restore on next launch. This was complex in the iced version (PR ggand0/viewskater#61) due to platform differences in coordinate systems and multi-monitor handling. egui/eframe may simplify this since `ViewportBuilder` accepts position and size directly.

### 2. File association / double-click open (all platforms)
Open images from file managers via double-click or "Open With". CLI open (`viewskater-egui /path/to/image.jpg`) already works.

- **Linux**: `.desktop` file already has `Exec=viewskater-egui %f` and `MimeType` entries. Needs binary in `$PATH` and desktop file installed.
- **macOS**: Needs `CFBundleDocumentTypes` in Info.plist and handling of `open-file` events from the OS. The iced version implemented this.
- **Windows**: Needs registry entries for file type associations, or an installer that sets them up.

Goal is to support all three platforms in a single PR.

### 3. Crash logging
Save crash logs to disk for debugging. The iced version added this in 0.1.2.

### 4. Memory usage in FPS overlay
Add process memory usage to the existing FPS display in the menu bar / fullscreen overlay.

## Backlog (post-0.1.1)

### 5. Log export
Export debug logs and stdout output to file.

### 6. Compressed file format support
Open images from ZIP, RAR, and 7z archives without manual extraction.

### 7. Horizontal split mode
Add horizontal (top/bottom) layout option for dual-pane mode, alongside the existing vertical (left/right) split.
