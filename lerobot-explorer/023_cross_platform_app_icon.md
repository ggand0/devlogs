# Devlog 023 -- Cross-platform app icon

## Goal

Add a proper app icon for the 0.1.0 release: visible in Windows taskbar,
macOS Dock, and Linux GNOME taskbar for both `cargo run` and AppImage builds.
Mirror the approach used in viewskater-egui (see devlogs 014 and 015 there).

## Setup that "just works" everywhere

### Rust side

`src/main.rs`:

```rust
fn load_icon() -> Option<egui::IconData> {
    static ICON: &[u8] = include_bytes!("../assets/icon_256.png");
    let img = image::load_from_memory(ICON).ok()?.into_rgba8();
    let (width, height) = img.dimensions();
    Some(egui::IconData { rgba: img.into_raw(), width, height })
}

let mut viewport = egui::ViewportBuilder::default()
    .with_inner_size([1280.0, 720.0])
    .with_drag_and_drop(true)
    .with_app_id("tracelr");

if let Some(icon) = load_icon() {
    viewport = viewport.with_icon(std::sync::Arc::new(icon));
}
```

- `with_icon()` covers Windows (`WM_SETICON` via eframe) and macOS
  (`NSApplication.setApplicationIconImage`, which overrides the bundle icon
  at runtime -- see vs-egui 015).
- `with_app_id("tracelr")` normalizes X11 `WM_CLASS` and Wayland `app_id` so
  GNOME's window matcher can tie the running window to the `.desktop` file.

### build.rs

```rust
if env::var_os("CARGO_CFG_WINDOWS").is_some() {
    WindowsResource::new()
        .set_icon("./assets/icon.ico")
        .compile()?;
}
```

Embeds the multi-res ICO into the Windows PE via `winres`.

### Cargo.toml

```toml
image = { version = "0.25", default-features = false, features = ["png"] }
# build-dependencies
winres = "0.1"

[package.metadata.appimage]
icon = "./assets/icon_256.png"

[package.metadata.bundle]
name = "tracelr"
identifier = "com.ggando.tracelr"
icon = ["assets/tracelr.icns"]
short_description = "A fast desktop tool for exploring and tracing LeRobot datasets"
```

### Assets

All derived from a single `assets/tracelr_icon_v1.svg`:

- `icon_{16,32,48,64,128,256,512,1024}.png` via `inkscape --export-type=png`
- `icon.ico` via `convert icon_{16..256}.png icon.ico`
- `tracelr.icns` via `png2icns tracelr.icns icon_{16,32,48,128,256,512,1024}.png`

### Desktop files

Two `.desktop` files serve different purposes:

- `cargo-appimage.desktop` at the repo root -- bundled into AppImage builds.
  cargo-appimage v2.4+ requires this exact path.
- `resources/tracelr.desktop` -- install template for end users.

Both reference `Icon=tracelr` and `StartupWMClass=tracelr`.

### Linux install steps (README)

```bash
mkdir -p ~/.local/share/icons/hicolor/256x256/apps
cp assets/icon_256.png ~/.local/share/icons/hicolor/256x256/apps/tracelr.png
gtk-update-icon-cache -f ~/.local/share/icons/hicolor/
cp resources/tracelr.desktop ~/.local/share/applications/
```

## Two non-obvious bugs hit during this work

### Bug 1: icon invisible on `cargo run`

After installing `tracelr.desktop` and the hicolor PNG, the dock still showed
the default generic icon. WM_CLASS matched `StartupWMClass`, the PNG was in
place, `desktop-file-validate` reported valid, `gtk-update-icon-cache`
succeeded. All the usual checks passed.

**Diagnostic:**

```python
Gio.DesktopAppInfo.new('tracelr.desktop')  # -> returns NULL
Gio.DesktopAppInfo.new('viewskater-egui.desktop')  # -> OK
```

GIO silently refused to load `tracelr.desktop` even though it was textually
valid. gnome-shell's window matcher uses GIO's `DesktopAppInfo`, so if GIO
rejects the file, the window matcher never sees it, so no icon.

**Root cause:** GIO validates that `Exec=<command>` resolves to an existing
binary. Bare commands must be findable in `$PATH`. Absolute paths must point
to an existing file. A bisection confirmed:

| `Exec=` value | GIO parse |
|---------------|-----------|
| `Exec=tracelr %f` (not in `$PATH`) | NULL |
| `Exec=ls %f` (in `$PATH`) | OK |
| `Exec=/bin/ls %f` | OK |
| `Exec=/home/.../target/opt-dev/tracelr %f` | OK |
| `Exec=/nonexistent/foo` | NULL |

The viewskater-egui install worked by accident: its installed
`~/.local/share/applications/viewskater-egui.desktop` uses the absolute path
to its `target/opt-dev/` binary. The *committed* template uses the bare
command name, which would also fail for a developer checking out fresh.

**Fix:** the installed copy uses the absolute path to the local build; the
committed template at `resources/tracelr.desktop` keeps the bare `tracelr`
because AppImage / deb / brew installs put the binary in `$PATH`, where
GIO will accept it.

Wrong theories ruled out along the way:

- gnome-shell stale cache from before install -- inotify on
  `~/.local/share/applications/` hot-reloads new files; restart not needed.
- `WM_CLASS` mismatch -- verified `WM_CLASS(STRING) = "", "tracelr"` and
  `StartupWMClass=tracelr`.
- Icon cache stale -- rebuilt multiple times, unchanged behavior.
- Overly broad `Categories=Development;Utility;` -- only a hint, not an
  error.

### Bug 2: `cargo appimage` stopped at AppDir stage

`cargo appimage` produced `target/tracelr.AppDir/` but no `.AppImage` file.
Error:

```
icon{.png,.svg,.xpm} defined in desktop file but not found
For example, you could put a 256x256 pixel png into
  target/tracelr.AppDir/icon.png
```

`cargo-appimage.desktop` had `Icon=icon` (copied from viewskater-egui's
committed file), but cargo-appimage actually writes the icon to
`<AppDir>/<package-name>.png` (so, `tracelr.png` in our case). The filename
in `Icon=` must match.

**Why viewskater-egui builds without this fix:** a stale `icon.png` from a
pre-naming-convention build is still sitting in its
`target/viewskater-egui.AppDir/`. appimagetool finds it and succeeds by
coincidence. `rm -rf target/viewskater-egui.AppDir/ && cargo appimage`
would reproduce the same failure there.

**Fix:** `Icon=tracelr` in `cargo-appimage.desktop`, matching what
cargo-appimage produces.

## Per-platform mechanism summary

| Platform | Mechanism | Our setup |
|----------|-----------|-----------|
| Windows | `WM_SETICON` via eframe `app_icon.rs` | `with_icon()` + `icon.ico` embedded via `winres` |
| macOS Dock / app switcher | `NSApplication.setApplicationIconImage` via eframe each frame | `with_icon()` (overrides bundle icon) |
| macOS Finder / Get Info | `CFBundleIconFile` in `Info.plist` | `cargo bundle` + `tracelr.icns` in `[package.metadata.bundle]` |
| Linux X11 | `_NET_WM_ICON` is broken in winit/x11rb for 256x256 -> fall back to `.desktop` match via `StartupWMClass` | `with_app_id()` + `.desktop` in XDG + hicolor PNG |
| Linux Wayland | No programmatic icon API -- `.desktop` match via `app_id` | same as X11 |

## Key takeaways

- `Exec=` must resolve. Bare command names in committed templates are fine
  for installed binaries; for local `cargo run` testing, either symlink the
  build into `~/.local/bin` or install a desktop file with the absolute path.
- When debugging "icon not showing," check `Gio.DesktopAppInfo.new(...)`
  first -- if that returns NULL, nothing else matters.
- cargo-appimage names the icon after the package; the desktop file's
  `Icon=` field must match.
- `cargo run` not showing an icon on Linux is *not* expected behavior after
  the `.desktop` install -- vs-egui devlog 013's pre-014 caveat about this
  is outdated once the `with_app_id` + install fix lands.
