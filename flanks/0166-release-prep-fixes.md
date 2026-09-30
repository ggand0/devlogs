# 0166: Release prep: death flash, packaged assets, menu rows, stats line, scaling tests

Written by Claude Opus 5.5.

Branch `fix/release-prep-0.2.0`, cut from main at 953629d (README and screenshot on top of PR #16), checked out in the main tree `/home/gota/ggando/gamedev/flanks`. The six commits below, not pushed. Gota plans to tag 0.2.0 after this branch merges.

| Commit | Title |
|---|---|
| e8e4dd3 | Stop dying soldiers from flashing white when hit flash is off |
| f548f77 | Find assets next to the executable in packaged builds |
| 7d13cf5 | Hide the front line by default |
| d1d104c | Pick the army size, map and AI from segmented rows |
| 817c9fe | Add a one-line stats readout to the F3 cycle |
| a32245b | Add tests for army sizes and morale across unit sizes |

Later commits on this branch (stats line fixes, battle setups, the Demo scenario) are in devlog 0167.

## 1. Death flash with hit flash off (e8e4dd3)

The unit shaders carry one float per soldier, `fx`: `[0,1]` is the hit flash, `(1,2]` is death progress. The instancing shader turns it into white with `clamp(fx, 0, 1) * step(fx, 1.0)`. A death starts at `fx = 2 - death_t / DEATH_TICKS` with `death_t = 18`, which is exactly 1.0, the value of a full hit flash.

The sim sets `death_t = 18` in `apply_damage`, after that tick's countdown in `soldier::tick_chunk`, so the renderer draws exactly one frame at 18: one full white frame per death, whatever the setting.

Fix: death progress is encoded only once it is past zero (`death_t < DEATH_TICKS`). The first tick falls through to the hit flash branch: the killing blow's flash when the option is on, nothing when off. Progress 0 is the living pose, so the pose is unchanged. Both render paths: `src/shaders/unit_build.wgsl` (GPU, default) and `src/render_units.rs` (the `FL_GPU_SYNC=0` fallback); the range comments in `unit_instancing.wgsl` and `render_units.rs` updated.

Verified by Gota in play. The first report of "still flashing" came from the main tree running the old binary while the fix sat in a side worktree (see the incident below).

## 2. Assets next to the executable (f548f77)

The game found `assets/` through `env!("CARGO_MANIFEST_DIR")`, which Cargo sets to the absolute path of the folder holding `Cargo.toml` and which is baked into the binary. Any build run on another machine looked for the build machine's folder and found no shaders, textures, sounds or models. This has been the case since 2026-07-10 (60bc2e8), so the 0.1.0 itch build may have the same problem.

`util::game_root()` returns the executable's own folder when it holds `assets/`, else the compile-time repository path (runs from `target/`). Cached in a `OnceLock`. Used by the Bevy `AssetPlugin` path (`main.rs`), the unit model loader (`unit_glb.rs`) and the vegetation loader (`vegetation.rs`). Left alone: `mixer.rs` (the `FL_LOG_AUDIO` dev log into the repo's `tmp/runs/audio`) and the vegetation test, which reads the repo.

Verified with a Windows build under Proton (below): with the source clone moved aside, the log shows every GLB loaded from `Z:\...\flanks-windows-x86_64\assets`.

## 3. Front line off by default (7d13cf5)

`InterfaceSettings::front_line` defaults to false. The Settings toggle stays. A saved settings file keeps its own value.

## 4. Segmented menu rows (d1d104c)

The Army, Map and AI rows each show every value as a button, the current one lit (`BTN_ACTIVE`, the colour the picker's enemy style chips already used, now shared). Order: Army, Map, AI.

- `Segment { Army(i), Map(i), Ai(i) }` component, `spawn_segment_row`, `menu_segments` (press) and `segment_style` (current value lit, the rest with the usual hover and press shades; `CustomStyled` keeps the shared hover system off them).
- `MapKind::ALL` replaces the cycling `MapKind::next()`: Grassland, Classic, River, Sandbox. A new map still sends `MapChanged`.
- `set_army_size`: 10k joins the sizes. Unit size is 200 at 5,000 a side (25 units), 500 at 10,000 (20 units), 1000 above. Picking the size already set returns early and keeps the unit picks and the enemy style; a new size resets both, as before.
- `OptionButton`, `menu_option_buttons` and `army_size_label` are gone. README range reads 10k to 200k.

A dropdown came first; its list was see-through and covered the AI and Map rows, so it was replaced before any push (backups below).

Morale needed no change for smaller units: casualties use the fraction lost and the exchange divides the death and kill rates by the unit's current strength (`morale.rs:396`). The leader bonus is army-wide. 25 units a side is what 50k already fields, so the card bar has handled that count all along.

## 5. Stats line in the F3 cycle (817c9fe)

`InterfaceSettings::overlay: Overlay { Off, Stats, Full }` replaces `debug_overlay: bool`, under a new key: an old settings file keeps its other settings and starts with the overlay off (the bool under the old key would have failed to parse as the enum and reset every setting). `InterfaceSettings::debug_overlay()` is true in Full only and still gates the debug tools (morale breakdown, X crater tool, front line debug draw). F3 and the Settings row "Overlay (F3)" cycle Off, Stats, Full.

The line: `176 fps | 36,840 soldiers | sim 4.5 ms` at the top left, on a black pill at 55% opacity with 6 px corners. Numbers in the menu's cream, units in a lighter grey (0.74) because the menu's dim grey was lost on the pill. Text spans with a `StatsValue` marker, rewritten only when the text changes.

- fps: the same two-second mean as the full overlay, now one `frame_rate()` used by both.
- soldiers: `CombatStats::alive` of both sides, with thousands separators (`thousands()`).
- sim: `grid_ms + step_ms + field_ms`, the per-tick phases; `audit_ms` runs every two seconds and is left out.
- Separator `|`: the built-in font has no `·` and drew a box.

## 6. Tests (a32245b)

- `morale::tests::losing_morale_scales_with_unit_size`, `winning_morale_scales_with_unit_size`: the real `update_morale` system on a bare `World` (Groups, an all-zero `InfluenceField`, `MoraleReadout`, `Time`), one 200-soldier and one 1000-soldier unit on opposite teams 1.5 km apart, each its army's captain. Every 20 ticks they lose 1 and 5 men (and in the winning test kill 2 and 10). After every tick the casualty term must be equal and the exchange term and morale level equal within 1e-4 and 1e-3. The losing run reaches 30% lost, past the 10% and 25% steps (-4); the winning run ends at 15% (-2). Both assert the exchange term stays off its limits, where absolute counting would show.
- `game_state::tests::army_sizes_split_into_whole_units`: every menu size divides into whole units, 10 to 100 a side, and the starting composition fills every slot.
- `unit_size_follows_army_size`: 200, 500, 1000, 1000, 1000.
- `picking_the_current_size_keeps_the_picks`: same size keeps picks and enemy style, a new size resets them.
- `InfluenceField::new` is `pub(crate)` for the test; no other non-test change.

## Verification

- Strict clippy (`--all-targets -D warnings`) clean after every commit. One slip: a command committed d1d104c's first version while clippy failed (overlapping match arms), because it tested a `grep` of the output instead of clippy's exit status. Fixed by amend; since then the exit status is checked.
- `cargo test --profile opt-dev`: 14 passed.
- Mutation checks on the morale tests: counting casualties as absolute men lost, and counting the exchange rates as absolute, each failed both tests. The file was restored byte for byte.
- Captures (`work/scripts/shots.sh`, no input sent): menu rows in `tmp/runs/menu-segments-2026-09-30/`, the stats line in `tmp/runs/stats-line-2026-09-30/` (a copy of Gota's settings with `overlay: Stats` via `XDG_CONFIG_HOME`), a 20k battle on fresh settings with no front line and no shader errors.
- Gota played the branch: death flash gone, dropdown worked (then replaced).

## Windows build without booting Windows

- Cross-compiled with what was installed: Rust's `x86_64-pc-windows-gnu` target and the system mingw. MSVC needs Microsoft's SDK downloaded; nothing new was installed.
- `opt-dev` build 3 min 15 s cold. `flanks.exe` 172 MB (the Linux opt-dev binary is 165 MB), 113 MB with symbols stripped (`strip = true` would do it). Imports only system DLLs; no mingw runtime DLLs to ship. Package with assets zips to 165 MB.
- Runs under Proton 11's wine on a Proton prefix: `tmp/run-windows-build.sh` (package in `tmp/runs/windows-gnu-2026-09-30/`). Vulkan: a 20k battle, no errors apart from gamepad support. DX12 cannot be tested under Wine: its stand-in for Microsoft's shader compiler lacks shader model 5.1 resource arrays, and vkd3d-proton rejects a texture format query that wgpu unwraps. Real Windows drivers and a clean machine remain unverified.
- The package built in that test predates commits 3 to 6; rebuild before a release.
- A GitHub Actions release workflow was tried on the private flanks-dev repo and cancelled at Gota's call. Left there: branch `ci/release-builds` (with the asset fix and a test-branch trigger), tag `0.0.0-ci.1`, 43 LFS objects. The tag filter `[0-9]+.[0-9]+.[0-9]+*` never triggered a run; `*.*.*` did.

## Incident: work in a side worktree

The branch was first made and built in a side worktree named `flanks-main`. Gota tested the main tree, found no branch, played the old binary and saw the death flash still there. The branch moved into the main tree; the side worktree became `flanks-agent` (detached, clean). Rule 11 in `docs/internal/003-parallel-worktrees.md` and memory `no-new-worktrees`: Claude never creates a worktree, all work goes in the main tree.

## Open

- Push and PR, when Gota says.
- Remove `flanks-agent` and its `target/`: asked, not answered.
- Stats pill for video: a short capture encoded at YouTube-like bitrates would settle the label contrast; a darker pill (about 0.8) and 18 px text are the likely changes.
- A spawn test (every unit at full strength inside its deployment zone) needs a generated terrain.
- Release chores: bump `Cargo.toml` (the menu shows v0.1.0), the Windows console window (`windows_subsystem`, not done), fresh packages.
- Backups: branches `backup/main-before-reword-2026-09-30`, `backup/release-prep-before-segments`, `-before-ai-segments`, `-before-10k-200`, `-before-clippy-fix`; bundles in `work/backups/main-reword-2026-09-30/`, `work/backups/release-prep-amend-2026-09-30/`, `work/backups/readme-0.2.0-2026-09-30/`.
