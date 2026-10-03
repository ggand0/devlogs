# 0159: main into feat/vegetation, README for 0.2.0

Written by Claude Opus 5.5.

2026-09-28, chores from work/handoffs/067-chores-2026-09-28.md, tasks 1 and 2.

## main into feat/vegetation

`git merge main` in the graphics tree (Astra's tree was clean, and no game or Blender was running) merged main 0a41859 into feat/vegetation at 694eade as b168a8d. Not pushed.

- One conflict, src/game_state.rs. PR #13 replaced the per-scenario env lists in `from_env` and `sync_scenario_env` with one table (`env_key`, `SCENARIO_ENVS`). Resolved by taking main's bodies and adding `Scene => Some("FL_SCENE")` to `env_key` and `"FL_SCENE"` to `SCENARIO_ENVS` (now 10). The `Scene` variant, its label and its `ALL` entry auto-merged. Nothing else reads the table, so `FL_SCENE` in it only feeds `from_env` and the menu's env sync.
- regiments.rs and terrain.rs auto-merged.
- opt-dev build and strict clippy (`--all-targets -D warnings`) pass. A 15 s `FL_SCENE=1 FL_MAP=sandbox FL_VOLUME=0` run showed no panic and no shader error, 0 units, 320 to 338 fps, with `render/sun_shadows/elapsed_gpu` at 0.08 to 0.11 ms. Log in tmp/runs/merge-main-vegetation/scene.log.
- The oaks need no code to join the shadows: sun shadows are Bevy's cascaded shadow maps (units add their own items to that phase), and the oaks are `StandardMaterial` meshes, foliage `AlphaMode::Mask(0.5)`. Cascades stop at 110 m; `FL_SHADOW_CASCADES=3 FL_SHADOW_DIST=280` reaches farther trees.
- Note for Astra: work/notes/027-shadows-on-vegetation-for-astra-2026-09-28.md (shadows on against `FL_SHADOWS=0` at the debug10 and debug12 views, `flanks::vegetation=debug` logs the oak level in each shot). Board updated.

## README for 0.2.0

New worktree `/home/gota/ggando/gamedev/flanks-main` on branch `docs/readme-0.2.0` off main 0a41859 (the main tree stays on fix/melee-orders). Commit 2a1f37b, not pushed. PR draft work/drafts/026-pr-readme-0.2.0.md.

Checked against the code, not memory:

- Keys from every `KeyCode` in src/ and the Settings Controls list (settings.rs ~544): Hold moved to B and Blob is gone (PR #13); added Ctrl+A/I/M, card clicks (Ctrl toggles, Shift ranges), Enter clears the selection, Enter begins the battle in deployment, F1/F2/F3, and G as the game's own "Banners and map lines". X (crater, F3 on) left out as a debug tool.
- Army sizes 20k/50k/100k/200k (`ARMY_SIZES`), 1,000-man regiments (500 at 20k), so 100 regiments per army at 200k. Five enemy army styles in `ARCHETYPES`.
- No `rust-version` in Cargo.toml. "Rust 1.95 or newer" stays, from Bevy 0.19's own `rust-version = "1.95.0"`. Adding the field to Cargo.toml would make cargo say it; not done, out of scope.
- New build line: models (`assets/units/*.glb`) and terrain textures (`*.ktx2`, `*.png`) are LFS per .gitattributes, so a clone without git-lfs gets pointer files.
- Audio sentence unchanged: every clip since is ElevenLabs (devlogs 0069, 0071, 0074; the massed footsteps are layered ElevenLabs clips).
- Screenshot not replaced yet: Gota takes it in game.

Left: tasks 3 (main7 baselines, can use the flanks-main tree) and 4 (stale worktree records).
