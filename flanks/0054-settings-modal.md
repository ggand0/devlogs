# 0054: Settings modal with YAML persistence

## What

Settings modal reachable from the main menu (Settings button) and in-battle via ESC -> pause overlay -> Settings. Values persist across sessions as YAML at the platform config dir (Linux: `~/.config/frontline/settings.yaml`) via the `dirs` crate + `serde_yaml_ng` (maintained fork of the abandoned `serde_yaml`, same API).

Contents:

- Audio: Master, Battle, Interface volume sliders (0-100%). Battle covers beds, combat one-shots, vox, horns, stings; Interface covers the selection/order/attack clicks.
- Camera: Pan speed slider (0.30x-2.00x, snapped to 0.05 steps), Edge pan toggle.
- Video: Window mode (Windowed / Borderless), VSync toggle.

## How

- `src/settings.rs`: `Settings` resource (serde, `#[serde(default)]` everywhere so the file survives schema growth in both directions; loaded values are clamped). Loaded in `main()` BEFORE the App is built so the window opens with the saved present mode / window mode instead of switching one frame in. Saves are debounced 0.8 s off resource change detection (a drag saves once, on release) plus an immediate save on modal close; write-then-rename so a crash can't truncate the file.
- Audio rework: the old `master()` OnceLock is gone. `FL_VOLUME` remains as `env_master()`, a pure multiplier on top of the settings volumes, so `FL_VOLUME=0` test runs stay muted regardless of the saved file. `one_shot` now takes the final volume; call sites scale by `battle_vol`/`ui_vol`. Live: beds re-target every frame, one-shots read at spawn.
- Sliders are custom (no bevy_ui_widgets): a track Button starts a drag captured in a `SliderDrag` resource; the fraction comes from the physical cursor x against the track's `ComputedNode` + `UiGlobalTransform`, so the drag survives the cursor leaving the node. Volume sliders play the UI click at the new volume on release.
- Modal layering: backdrop `FocusPolicy::Block` swallows picking; menu/pause input systems (including the Enter/Space start shortcut and ESC pause toggle) are gated on `settings::settings_closed`, so ESC closes the modal first, then the pause overlay, then resumes. The gate works same-frame because the despawn command lands after the run-condition checks.
- `OpenSettingsButton` is a public marker: any UI (deployment screen later) can spawn a button with it and get the modal for free.

## Verified (Xephyr protocol, devlog 0053)

llvmpipe + `FL_VOLUME=0 FL_UNITS=2000`, `XDG_CONFIG_HOME` pointed at a scratch dir so the real config stays untouched. Screenshots confirmed: menu button opens modal; Master drag 100% -> 60% updates fill + label and autosaves exactly `master: 0.6`; Edge pan toggle persists; ESC layering in battle (modal -> pause -> resume); relaunch loads 60% / Off back into the UI. Note: ESC needs `xdotool windowfocus` first in WM-less Xephyr, clicks do not.

## Follow-up: state gating (owner feedback)

Quit to Menu mid-battle left the audio running and the camera controllable behind the menu: `control_camera` and the three audio systems ran unconditionally in Update, so they kept working against the stale battle world (looping beds held their last volume; stale `SimStats.events` could keep spawning clangs).

- Audio systems now `run_if(in_state(Battle))`. `OnExit(Battle)` zeroes the bed sinks (one-shots deliberately survive into Results so the victory/defeat sting can finish); `OnEnter(Menu)` despawns every lingering one-shot.
- `control_camera` now runs only in Battle (still active while paused, M2TW style) and never while the settings modal is open, so edge pan cannot drift the world behind a modal. `apply_camera_transform` keeps running so the menu backdrop stays valid.
- Verified in Xephyr: quit-to-menu screenshot diff with the cursor parked at the screen edge for 3 s matches stationary-cursor noise (~15k px of water/banner animation); active edge pan would have shifted the entire frame.

The menu stays an overlay over the frozen battle world for now; a dedicated background scene (or despawning the world on menu entry) is a possible later polish item.

## Candidates for later

Music volume (no music yet), rotate/zoom sensitivity + invert, UI scale, resolution picker for exclusive fullscreen, colorblind team colors, FPS overlay toggle, persisted menu defaults (army size / AI), keybind remapping once input is centralized.
