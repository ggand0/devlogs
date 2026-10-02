# 0168: AppImage, macOS bundle and the FLANKS icon

Written by Claude Opus 5.5.

Branch `feat/packaging`, checked out in the main tree. Merged as PR #18 on 2026-09-30 (main fe69f5c), 10 commits. Follows the release prep in devlogs 0166 and 0167.

| Commit | Title |
|---|---|
| f2b4a56 | Package the game as an AppImage and a macOS app bundle |
| 812ee90 | Use the new FLANKS icon for the AppImage |
| 16091c1 | Give the macOS app bundle the FLANKS icon |
| aa32136 | Give the game window and the Windows executable the FLANKS icon |
| 7a334d7 | Round the FLANKS app icon and give it a margin |
| ad3bcd0 | Move the AppImage desktop entry to resources/linux |
| 4efac62 | Write the Windows resource script at build time |
| 1150751 | Drop the unused square icon PNG |
| 698f426 | Show the game window at once and fix the window icon docs |
| 9763238 | Set the window icon once when bevy creates the window |

## Packages

- `util::game_root` looks for `assets/` next to the executable (zip or tarball), in a macOS bundle's `Contents/Resources` (`exe_dir/../Resources`), then at the root of a running AppImage (`APPDIR`, set by the AppImage runtime), and falls back to the repository for a run from `target/`. `packaged_root` holds the lookup; test `util::tests::packaged_builds_find_their_assets` builds the four layouts in the temp folder.
- Cargo.toml: `[package.metadata.appimage]` (icon `assets/flanks_icon_rounded_256.png`, `desktop_entry = "resources/linux/flanks.desktop"`, assets) and `[package.metadata.bundle]` (name FLANKS, `com.ggando.flanks`, `assets/flanks_icon.icns`, assets as resources).
- AppImage built with cargo-appimage and run: models and trees load from `/tmp/.mount_.../assets`. cargo-appimage copies untracked files too, so package from a clean clone. `cargo-appimage --help` starts a build.
- macOS bundle built with `cargo bundle` on Linux: `Contents/Resources/assets` (256 files), the .icns, `CFBundleIconFile`. Not run on a Mac yet.

## Icon

- Gota's SVG is the source (`assets/flanks_icon.svg`). The app icon is rounded, the art at about 80% of the canvas: `flanks_icon_rounded_256.png` (AppImage and window), `flanks_icon.icns` (32 to 1024 px), `flanks_icon.ico` (16 to 256 px). C2PA metadata stripped from Gota's files; the committed files carry none.
- Windows executable: `build.rs` writes a one-line resource script into OUT_DIR and compiles it with `embed-resource` 3, only when the target is Windows. winres was dropped: with GNU targets the linker threw away its resource archive and the exe had no icon. A cross-built exe listed the 7 icon images and the group icon under `wrestool -l`.
- Game window: `src/window_icon.rs` sets the winit window icon from the embedded PNG. bevy 0.19 has no window icon API (WindowIcon is an unmerged PR) and does not re-export winit's `Icon`, so Cargo.toml depends on the same winit 0.30 bevy uses (one copy in Cargo.lock). The system runs only on `WindowCreated` (`run_if(on_message::<WindowCreated>)`), which bevy_winit sends right after it adds the window to `WINIT_WINDOWS` (bevy_winit 0.19 `system.rs:125`). The window's `name` is `flanks` (X11 WM_CLASS, Wayland app id). Checked with xprop: `WM_CLASS "flanks"` and `_NET_WM_ICON` set; Gota saw the icon.
- GNOME ignores a window's own icon unless it can match the window to an installed desktop entry (`StartupWMClass=flanks`). A bare AppImage shows the gear until its entry is installed (AppImageLauncher, Gear Lever, or by hand). No game code changes that.

## The hidden-window hack

A tool call that made the window start hidden (`visible: false`, shown after the icon or after 30 frames) was rejected, but its edits had already been written, and aa32136 was committed without reading the diff. The final PR check found it; 698f426 removed it. The push of 698f426 and 9763238 came later, so the PR head carried the hack until then. Rule since: `git diff` after every rejected call and a full diff read before every commit.

## Final check before the merge

- Full diff against main read line by line: no hidden window, no delayed show.
- The first icon system checked for the window every frame on the main thread with a done flag; 9763238 moved it onto `WindowCreated`.
- Strict clippy (`cargo clippy --profile opt-dev --all-targets -- -D warnings`) clean, 22 tests pass.
- Commit messages and added comments audited: no footers, em dashes, tool names or history words.
- Pushed with lease as a fast-forward (1150751..9763238). Backup bundles in work/backups/packaging-branch-2026-09-30/.

## Open

- build.rs uses `manifest_optional()`: on a Windows machine without a resource compiler (rc.exe, windres) the build passes and the exe has no icon, without a warning. `manifest_required()` would stop the build instead. Raised, not decided.
- Build and run on Windows (MSVC) and on a Mac.
