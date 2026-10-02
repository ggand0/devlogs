# 0169: One Windows executable with the assets inside

Written by Claude Opus 5.5 on Windows 10, in the NTFS tree at `D:\ggando\gamedev\flanks`, for copying into the Linux devlogs. Code work in the Windows clone `C:\Users\gotag\projects\flanks`.

2026-09-30. Branch `fix/embed-assets` off main 53f0610 (0.2.0 plus fat LTO), one commit, 2475c5a "Embed the assets in the executable with the embed_assets feature", pushed to origin. Not merged.

## The problem

The 0.2.0 Windows build could not be uploaded as it was. `cargo build --release` gave a 72.2 MB `flanks.exe` (75,736,064 bytes, plus an 8.1 MB `flanks.pdb`), against 135 MB for the Linux AppImage and 141 MB for the macOS build. The difference was the assets: the exe held none of them.

- The AppImage is a squashfs image holding the binary and `assets` (cargo-appimage's `assets = ["assets"]`), and the macOS app bundle keeps them in `Contents/Resources/assets`. Both are one file to download.
- Windows has no such container. The packaging setup (f2b4a56, docs/internal/005) shipped Windows as a zip of the exe beside an `assets` folder. The expectation was a single file, as on the other two platforms.
- The exe ran from `target\release` only because `util::game_root` falls back to the compile-time repository path (`CARGO_MANIFEST_DIR`) when no `assets` folder sits next to it. On another machine the exe alone would find no models, trees, terrain textures or sounds. The unit shaders, `audio_levels.json` and the window icon were already compiled in (`embedded_asset!`, `include_str!`, `include_bytes!`); nothing else was.

Options: the zip (not one file once unpacked), an installer (Inno Setup or NSIS, extra tooling for nothing here), or the assets inside the exe. The last is the Windows counterpart of the AppImage.

## The fix (2475c5a)

A cargo feature, `embed_assets`, off by default. The release command for Windows is

```
cargo build --release --features embed_assets
```

It is a feature and not tied to the release profile because cargo cannot enable a feature per profile, `opt-dev` inherits from `release` (so `PROFILE` and `debug_assertions` read the same in both), and embedding in daily builds would link 115 MB into every build and recompile the crate on every asset edit.

- `build.rs`: with `CARGO_FEATURE_EMBED_ASSETS` set it walks `assets/` (dotfiles skipped) and writes `OUT_DIR/embedded_assets.rs`, a table `FILES: &[(&str, &[u8])]` of every file as its path relative to `assets` with `/` separators and an `include_bytes!` of its absolute path, sorted by path. `rerun-if-changed` on the `assets` directory makes cargo rescan the tree, so an added or removed file regenerates the table; `include_bytes!` tracks each file's contents. A file under 1 KB that starts with `version https://git-lfs` stops the build: a checkout without `git lfs pull` fails instead of shipping pointer files. 257 files, 115 MB.
- `src/game_files.rs`, new:
  - `GameFilesPlugin` registers Bevy's default asset source as a `MemoryAssetReader` over a `Dir` filled with the static slices (no copy; the same reader Bevy's own `embedded://` source uses). It is added before `DefaultPlugins`, because `AssetSourceBuilders::init_default_source` fills the default slot only when it is empty. Every `asset_server.load` (terrain textures, audio) then reads the embedded copy.
  - `read`, `read_to_string` and `is_file` for the loaders that open files themselves. A path inside `game_root()/assets` is answered from the embedded copy only, never from disk, so a missing file fails the same on the build machine as anywhere else. Any other path (an `FL_GLB_*` override, the `assets_dev` working copies) is read from disk. Paths are keyed with `relative_to`: a lexical `strip_prefix`, `.` and `..` resolved, `/` separators, as build.rs keys them. Without the feature all three are plain `std::fs` calls.
- `src/unit_glb.rs`: the model lookup (`is_file` for the override, the shipped file and the working copy), the GLB reads of the models and the arrow, an atlas stored as a file next to the model, and the shot and attack JSON next to it all go through `game_files`.
- `src/vegetation.rs`: the shipped-tree check and the read (`Gltf::from_slice` on the bytes instead of `Gltf::open`).
- `src/main.rs`: `.add_plugins(game_files::GameFilesPlugin)` ahead of `DefaultPlugins`.
- Still on disk, as they should be: `settings.yaml`, `last_setup.yaml`, `FL_SETUP` files, the `FL_LOG_AUDIO` dev log.

Linux and macOS packaging is unchanged: the AppImage and the app bundle already carry `assets`. If they ever build with the feature, their `assets` copies (`[package.metadata.appimage] assets`, the bundle resources) should go, or the assets ship twice.

docs/internal/005-release-packaging.md (NTFS tree) now describes the Windows build as one exe with this command, and how to test it: an `embed_assets` build never reads `assets` from disk, so starting it from any folder tests it, with `Start-Process flanks.exe -RedirectStandardError flanks.log` for a log.

## Verification

- Strict clippy (`--all-targets -D warnings`, opt-dev) clean without the feature and with it.
- `cargo test --profile opt-dev`: 26 passed without the feature, 27 with it. New: `game_files::tests::paths_key_like_build_rs` (keys, `.` and `..`, paths outside `assets`), and with the feature `every_asset_is_embedded`: every file on disk under `assets` is in the table, `read` serves it borrowed from the executable (not from disk), byte for byte equal to the file, and a missing model is not a file.
- The generated table in `target/opt-dev/build/flanks-*/out/embedded_assets.rs`: 257 `include_bytes!` entries, the count of files under `assets`.
- Release build on `fix/embed-assets`: `cargo build --release --features embed_assets`, 14 min 57 s, nearly all of it the game crate under fat LTO with one codegen unit (one core busy, 6.1 GB of memory at the ten-minute mark). `flanks.exe` 193,617,920 bytes (184.6 MiB), SHA-256 `68E3149130B9238D820C72A306EA7DEBAEE1C55DCEF4A3C5B7829D579043DB7C`. Copied to `tmp\release-0.2.0\flanks.exe` in the Windows clone before the tree went back to `feat/hover-rings-toggle`, and renamed `flanks_0.2.0.exe` after the tests.
- Run from `tmp\release-0.2.0`, no `assets` next to it: `FL_AUTOSTART=1 FL_DEPLOY=0 FL_UNITS=10000 FL_VOLUME=0`, closed after 45 s. The log shows the knight, man-at-arms, spearman and archer imported with all four levels, their atlases, the spearman's attack file and the archer's shot file, the arrow, all nine trees, the Grassland terrain, a 20k battle at about 270 fps, and no load error. The one warning is the shutdown readback (`Failed to send readback result: sending into a closed channel`).
- The same run with the repository's `assets` folder renamed away for its duration (the test docs/internal/005 prescribes; the script renamed it back in a `finally`): the same 14 load lines, no load error from the asset server (terrain textures, audio), so Bevy's default source is the embedded one and not the disk fallback. `git status` clean afterwards; `settings.yaml` and `last_setup.yaml` byte-identical to a backup taken before the runs.

## 0.1.0 rebuilt with the fix

For the record, a single Windows exe of 0.1.0 as well. Local branch `build/0.1.0-embed-assets` on the tag 87b7da7, not pushed: 2980088 "Embed the assets in the executable with the embed_assets feature" and 92811e9, a cherry-pick of e1eff45 "Open no console window with the game on Windows".

- 0.1.0 needs less of the fix. It has no build script and no LFS, and every asset goes through Bevy's asset server with `file_path` at `CARGO_MANIFEST_DIR/assets` (no `util::game_root`, no GLB models, no tree files). Its assets are 211 sound files and `shaders/water.wgsl`, 14 MB. So 2980088 is the embedding part of build.rs, a `game_files.rs` with only `GameFilesPlugin`, the feature, and the plugin in `main.rs`.
- The first build left out the console fix, so the exe opened a console window next to the game; 92811e9 added it. PE subsystem field: 3 (console) before, 2 (GUI) after, the same as the 0.2.0 exe.
- `cargo build --release --features embed_assets`: 5 min 40 s the first time (0.1.0's lockfile rebuilt 325 dependency crates; its release profile has no fat LTO), 31 s after the cherry-pick. `tmp\release-0.1.0\flanks_0.1.0.exe`, 103,157,760 bytes (98.4 MiB), SHA-256 `8DA248CD0EDEE549870D528243162122C440B824C2A135F3AE458CD610386FCA`.
- Run from its folder with the repository's `assets` renamed away, `FL_TEST_FRONT=1 FL_UNITS=10000 FL_VOLUME=0`, 40 s, before and after the console fix: no load error or warning (0.1.0 requests every sound at startup whatever the volume), a 20k battle at about 280 fps, no console window after the fix. `assets` restored and the settings files unchanged each time.

## How the branch was made

The work was done in the working tree of `feat/hover-rings-toggle`. The commit was built on main without a checkout: the diff against HEAD of the five changed files applied to a separate index read from main (`GIT_INDEX_FILE`, `git read-tree main`, `git apply --cached`), `src/game_files.rs` added by `hash-object`, then `write-tree`, `commit-tree -p main` and `git branch fix/embed-assets`. The five files matched the working tree exactly and `game_files.rs` had the same blob hash. After the push the six files were staged and the tree went to `fix/embed-assets` and back with `git switch`, which left `feat/hover-rings-toggle` clean at 19c9b6b with the changes only on the fix branch. For the release build the tree went to `fix/embed-assets` again, and back once the exe was copied out.

## The failures, plainly

- The first release build with the feature was started from `feat/hover-rings-toggle`, whose commits (the F4 hover-ring toggle, the Shift and left Alt pan speeds, the exclusive fullscreen option) are not in 0.2.0. Stopped when the branch question came up, before it finished.
- A `cargo run --profile opt-dev` from a Git Bash outside the app was reported as idle and possibly stuck, with an offer to end it. It was Gota's image viewer (viewskater-egui), running its exe; the process search only looked for `flanks`.
- "ETA for the build" was a request to build. It got an estimate and a wait for a go instead of the build.
- The first commit attempt used a scratchpad folder that did not exist yet; the chain stopped at its first step and created nothing.
- The build ETA was given as 5 to 10 minutes without a measured fat-LTO build on this machine; it took 15.
- The first 0.1.0 exe carried the asset fix but not the console fix, which a shippable Windows exe of 0.1.0 needed as much.

## Open

- Upload `tmp\release-0.2.0\flanks_0.2.0.exe` to the 0.2.0 release.
- PR and merge of `fix/embed-assets` into main.
