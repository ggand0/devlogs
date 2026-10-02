# 0168: Windows fps for the demo recording: the window size, not Windows

Written by Claude Opus 5.5 on Windows 10, in the NTFS tree at `D:\ggando\gamedev\flanks`, for copying into the Linux devlogs. The Windows clone had no CLAUDE.md at the time, so its rules were missed (see the failures); it was copied over from the NTFS tree afterwards.

2026-09-30, on the Windows side of the box: RTX 3090, Ryzen 9 3900X, primary monitor 3840x2160 at 150% scale plus a second monitor in portrait. Windows clone at `C:\Users\gotag\projects\flanks`, branch `feat/hover-rings-toggle`. One commit, 19c9b6b "Add exclusive fullscreen to the Window setting", not pushed.

## The question

0.2.0 is out. The Demo (`FL_DEMO=1`) recorded with OBS on Windows read 80 to 85 fps on the stats line, where Linux reads 110 to 115 with shadows in this kind of view. The question was whether to trim units in the Demo to reach 100 to 120 for a video on X and the Bevy Discord.

## Answer

Keep the 200k. Windows is not slower. The game sets no window size, so Bevy opens 1280x720 logical times the OS display scale, and the two OSes were set up differently:

| | Display | Scale | Game window, physical |
|---|---|---|---|
| Linux | 2560x1440 | 125% | 1600x900 |
| Windows, as found | 3840x2160 | 150% | 1920x1080 |
| Windows, changed | 2560x1440 | 125% | 1600x900 |

At 1920x1080 the game fills 44% more pixels. The soldier LOD bands also scale with the physical viewport height (`LodBands::new` with `px_per_unit` from `physical_viewport_size()`, in `render_units_gpu.rs` and `render_units_shadow.rs`), so at 1080 rows every switch to a coarser level sits 20% farther out and more soldiers draw at the finer levels. With Windows at 2560x1440 and 125% the window opens at 1600x900, and Gota's counter in a 200k battle read 125 to 130 fps, above Linux's 110 to 115.

## Readings on Windows, in order

Stats line fps, Gota's counter. Screenshots in `demo_recs/` of the NTFS tree.

| Setup | Window | fps | Notes |
|---|---|---|---|
| Demo, OBS recording (x264, Window Capture) | 1920x1080 | 80 to 85 | first report |
| Demo, OBS Game Capture, not recording, paused (debug0.png) | 1920x1080 | 82 | 179,051 soldiers, camera close and low looking across both armies |
| Demo, OBS closed | 1920x1080 | dips to 80 | same kind of view |
| Normal 200k, no `FL_DEMO`, Gota's usual check spot, paused (debug1.png) | 1920x1080 | 93 | 189,391 soldiers, windowed with a title bar |
| Borderless fullscreen | 3840x2160 | 45 | taken while my release build ran in the background (one busy core, fat LTO of the game crate); the 4K frame is GPU-bound, so roughly right |
| Normal 200k, display at 2560x1440 and 125% | 1600x900 | 125 to 130 | |

## Ruled out on the way

- OBS. The profile (`%APPDATA%\obs-studio\basic\profiles\Untitled\basic.ini`) recorded with x264 on the CPU (Simple mode, veryfast, 1920x1080 at 60), took the game through Window Capture, had preview on, and the scene also holds a visible Elgato capture source named "switch" under the game. Game Capture replaced Window Capture; the encoder is still x264. debug0 shows 82 fps with OBS not recording, and with OBS closed the same view dips to 80, so OBS was not the cost.
- Graphics backend. Bevy 0.19 enables Vulkan, Metal and DX12 by default. wgpu-core 29 adds the Vulkan instance before DX12 and `request_adapter` only sorts by device type (a stable sort), so the 3090 comes up on Vulkan on Windows, the same API as on Linux. `WGPU_BACKEND` overrides it. Note for a forced DX12 run: `main.rs` removes `VALIDATION_INDIRECT_CALL` on every backend, while Bevy keeps it whenever DX12 is enabled because wgpu's DX12 backend needs that pass for indirect draws, so DX12 may draw wrong.
- The "crash" during an OBS run. The log ends with `No windows are open, exiting`, `Closing window 65v0`, then `WARN bevy_render::gpu_readback: Failed to send readback result: sending into a closed channel`. That is a close request and a normal shutdown; the warning is a readback landing after its channel closed. No panic. Whether the window was closed by hand was not established.

## Commit 19c9b6b: exclusive fullscreen

The Settings "Window" row (General tab, Video) was already a Windowed and Borderless toggle over `video.fullscreen: bool`. It now cycles Windowed, Borderless, Fullscreen.

- `VideoSettings::window: WindowKind { Windowed, Borderless, Fullscreen }`, read with `#[serde(alias = "fullscreen")]` through `window_kind_or_bool`, an untagged helper that takes the enum or the old bool (true reads as Borderless). An old file keeps its other settings.
- `Toggle::Fullscreen` is now `Toggle::Window`, labelled by `WindowKind::label`, the same shape as the F3 overlay row.
- `window_mode`: Fullscreen is `WindowMode::Fullscreen(MonitorSelection::Primary, VideoModeSelection::Current)`. Primary, not Current: while it creates the window bevy_winit has no current monitor, and exclusive mode then hits `.expect("Unable to get monitor.")`. The current video mode is the monitor's own resolution, so on this 4K monitor Fullscreen renders 4K like Borderless, about 45 fps: it does not help the recording as it stands.
- Wayland: winit ignores `Fullscreen::Exclusive` there (warns and stays windowed), so Fullscreen maps to borderless on Linux when `WAYLAND_DISPLAY` is set.
- Tests: `settings::tests::window_reads_the_old_fullscreen_bool`, `window_round_trips`, `old_file_keeps_its_other_settings`. Strict clippy (`--all-targets -D warnings`) clean, `cargo test --profile opt-dev` 25 passed.
- Not run in game. The release build with it was cut off by a session restart; `target\release\flanks.exe` is still the 20:58 build without it.

## Benchmark script (not run live)

`tmp/flank-perf.ps1` in the Windows clone (`tmp/` is ignored). It runs the flank recipe of devlog 0147 (`FL_TEST_FRONT=1 FL_UNITS=100000 FL_CAM_LOCK=1 FL_CAM_X=-470 FL_CAM_Z=0 FL_CAM_YAW=-1.5708 FL_CAM_PITCH=0.35 FL_CAM_DIST=15`, `FL_SHADOWS` from `-Shadows 0|1`), clears every other `FL_` variable for the child and restores the shell's afterwards, writes stderr to `tmp\runs\flank-win\`, waits for the first battle fps line, measures `-Seconds` (default 120), closes the game, and prints the mean of the last 30 fps lines, the backend from the AdapterInfo line, soldiers drawn per LOD, the GPU pass timers and the frame pacing and thread lines, next to the Linux numbers of 0147 and 0163. `-Summarize <log>` re-reads a log.

```
powershell -ExecutionPolicy Bypass -File tmp\flank-perf.ps1
```

Tested on a synthetic log only. It does not set the window size: run it with the display at 2560x1440 and 125%, or it measures 1920x1080 against Linux numbers taken at 1600x900.

## For the recording

- Display at 2560x1440 and 125% for the 1600x900 frame; in OBS, fit the Game Capture source to the 1920x1080 canvas (Transform, Fit to screen).
- Switch the recording encoder from x264 to NVENC. x264 veryfast at 1080p60 takes cores from the sim pool, which is full width unless `FL_THREADS` is set. Not measured yet: the fps while recording at 1600x900.
- Hide the Elgato source in the recording scene.
- On Linux, OBS offers NVENC with the proprietary driver, and obs-vkcapture is the Game Capture equivalent for a Vulkan game (`obs-gamecapture <binary>`).

## The failures, plainly

- The window's physical size was the first thing to check and the last thing checked. OBS encoders and capture sources, the wgpu backend, the driver's present method and a benchmark script all came before asking how big the game window was on each OS, although debug1 already showed a titled window and reading the display resolution and scale took one command. Saved as memory `fps-compare-same-window-size` on the Windows side.
- Toggling shadows off was offered as a diagnostic without saying so plainly; it read as advice to record without shadows, which is not an option for the video.
- Fullscreen was suggested to skip the compositor before the monitor was known to be 4K; the reading it produced (45) measured the bigger frame, not the compositor.
- A `sed` rename of `Toggle::Fullscreen` also renamed `WindowKind::Fullscreen`; caught on the next read, before any build.
- The first version of the commit carried a Co-Authored-By footer, against CLAUDE.md. Reworded before any push: 6ba118e became 19c9b6b.
- Clippy (twice) and a fat-LTO release build ran while Gota was taking fps readings on the same machine, without the process check CLAUDE.md asks for (`pgrep -a -x flanks` on Linux; `Get-Process flanks` on Windows). The 4K reading overlapped the release build.

## Open

- Keep or revert 19c9b6b: asked, not answered. If kept, build release and try each value in game. A resolution choice for Fullscreen (1920x1080 on the 4K monitor) would make it useful for 1080p capture.
- The fps while recording with NVENC at 1600x900.
- Optional: an env var or setting that opens the window at a fixed physical size, so cross-OS runs compare the same frame without changing display settings.
