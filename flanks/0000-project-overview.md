# 0000: Project overview, current state (2026-09-22)

A technical overview of FLANKS as it stands today, for anyone (owner or agent) who needs the whole picture before reading the numbered devlogs. The README stays user-facing. This file is the map. Update it when a milestone lands.

## What it is

FLANKS is a real-time medieval battle game in Rust and Bevy 0.19. Every soldier is simulated individually. Two armies of up to 100 regiments of 1,000 men fight on a 1024 x 768 m field. The combat, morale and fatigue models are built from mechanics measured in Medieval II: Total War, with the aim of going beyond the classics rather than recreating them.

- Public repository ggand0/flanks since 2026-07-26, MIT or Apache-2.0. 209 commits. About 19k lines of Rust and WGSL in 30 modules.
- Version 0.1.0 tagged on 2026-09-20 at 87b7da7. `main` is 1c4dcc3, the merge of PR #6.
- Dev box: Ryzen 9 3900X (12 cores), RTX 3090, Ubuntu, 1600x900 window.
- Owner: Gota. He play-tests live, pushes and opens PRs himself, and decides every feel question.

## The game as played

Flow: Menu, Select Units, Battle, Results. The battle opens frozen in a deployment phase (gold zone lines, Begin Battle or Enter releases the sim). Escape pauses with Resume, Settings and Quit to Menu.

- Army size in the menu: 20k, 50k, 100k or 200k soldiers in total.
- Select Units: an M2TW-style picker. Drag cards from a roster into the army list, budget in regiment slots. The enemy is Random, one of five styles (Balanced Host, Iron Wall, Spear Hedge, Arrow Storm, Skirmish Horde) or Manual.
- Four soldier kinds: knight (heavy), man-at-arms (light), spearman, archer. The kind field is full at four.
- Orders: lasso selection, right click to move or attack, drag a battle line, control groups, halt. Formations: shield wall or spear wall (F), loose (L), blob (B), hold (H). Archers: fire at will (T), skirmish (K).
- HUD: unit cards with strength, morale, fatigue and ammo, regiment banners, a morale inspect panel, a settings modal with YAML persistence, a debug overlay (G).
- Audio: layered battle beds, steel, melee voices, war cries, celebrate and rout soundscapes, arrows. Mixed on the doctrine "rate from the sim, distance as volume".
- Skirmish AI opponent, victory and defeat outcomes, results screen.

## Architecture

### Data

- `units.rs`: `Units`, a structure of arrays. Soldiers are never entities. Columns: pos, pos_prev, vel, speed, team, kind, yaw, yaw_prev, group, hp, color, target, swing, swing_t, flash, death_t, home, ammo, plus a `generation` counter per battle. Dead soldiers are swap-removed, so indices shuffle every tick. Anything keyed by index must tolerate that.
- `orders.rs` `Groups`: one `GroupData` per regiment. Anchor, centroid, facing, order, state (formed, broken), engaged flags, morale, fatigue, wall spacing, charging, firing.
- `unit_types.rs`: per-kind stats (EDU-scaled), missile constants (range 120 m, BASE_DMG and FACTOR_MULT are the archery calibration knobs).
- `terrain.rs`: heightfield with 2 m cells, blocked mask, wade multiplier. `FL_MAP=river` opts into the river and vegetation map. Craters are disabled.

### Simulation, 30 ticks per second

Since PR #6 (item 0 of the scale plan) the heavy part of the tick runs on a dedicated worker thread.

1. `kick_tick` (end of the previous tick) fills a `TickJob` with copies of the soldier columns, the regiment snapshot and the archers' fire solutions, and sends it to the worker.
2. `run_tick_job` on the worker: `spatial.rs` grid rebuild (1.5 m cells, counting sort), then the parallel integrate over 2048-soldier chunks. About 800 lines per soldier: separation and crowd yield, steering to the slot or the order, hold deadzone, rout flight, terrain slope and wading, the swing state machine, the bow draw and loose, the spear-line hazard, stagger, wall slide on blocked ground. It emits damage events and arrow spawns per chunk.
3. `step_sim` on the main thread: install the finished job by buffer swaps, then the serial apply. Damage with the M2TW directional model (front, side, rear, shield side, armour piercing, charge bonus, spear impale, knockback and stagger), kills and morale hooks, the neighbor audit every 60 ticks, the FL_HASH fingerprint.
4. After it, in order: `update_arrows` (flight, body and ground hits, litter), `process_deaths` (swap-remove sweep, corpses), `update_fatigue`, `update_morale`, `clear_arrived_orders`, then the next kick.
5. Before it: `apply_reforms` (slot assignment, close ranks), `update_field` and `update_groups` (`frontline.rs`: density splat, blur, contour, engagement and enemy-near flags), `skirmish_and_ammo`.

Rules that hold the whole thing together:

- Every task scope reachable from the job goes through `util::sim_scope`. A plain Bevy scope on the worker steals `step_sim` and deadlocks (devlog 0079).
- Orders quantize to one tick (33 ms). `FL_PIPELINE=0` runs the tick inline, bit-identical to the pre-PR-#6 sim.
- The sim is bit-deterministic run to run, archery and AI included. `FL_HASH=n` proves it (devlog 0079).
- Melee: every regiment's slot grid is driven into the enemy (an attack order centers it on the target's live center of mass) and bodies stop the men; whoever has an enemy in weapon reach fights. The M2TW model from devlogs 0035 to 0042 sits behind `FL_RECTFIGHT`, which is OFF by default and which Gota prefers off: with it on, blocks flock oddly some time after engaging and soldiers die too fast (devlog 0119). "Simulate, never fake" is the design principle: physical events, no stance flags.
- Morale: a per-tick recomputed level with signed factors, values read from live M2TW memory (devlogs 0055 to 0057). Fatigue: six M2TW states. Breaks land at 81 to 98 percent losses.
- Archery: regiment fire solutions, per-soldier aim, flat or lofted arc that clears friendly blocks, range-independent scatter, its own damage curve (devlogs 0060 to 0065).

### Rendering

- `render_units.rs`: `sync_instance_data` runs every frame on all cores. It interpolates position and facing by the fixed clock's overstep, culls against a fresh frustum, picks a detail level, smooths four pose signals, and writes a 64-byte `InstanceData` into one of 16 buckets (4 kinds x 4 levels). Corpses go through the same pass from a 25k ring per kind. Arrows ride the same pipeline in their own buckets.
- One entity per bucket with `Mesh3d`, `InstanceMaterialData`, `NoFrustumCulling` and `NoAutomaticBatching` (required, devlog 0014). A custom `DrawMeshInstanced` command in the `Transparent3d` phase, instance attributes at locations 8 to 11.
- `unit_instancing.wgsl`: all animation is procedural in the vertex shader. Mesh UV x is the body part id, UV y the pivot height. The shader rotates rigid parts (legs, sword arm, spear arm, shield arm, bow arm) from anim channels: yaw, move amount, swing style and progress, stance band, hit flash, death topple, march in step, wall pose, stagger, celebrate hop.
- `unit_meshes.rs`: code-built cuboid soldiers, four levels per kind (knight 204 / 120 / 72 / 24 triangles). Levels are picked from on-screen height (28, 12, 3 px) with jitter and hysteresis (devlog 0077, PR #5).
- Terrain chunks, vegetation, water, banners (four Bevy entities per regiment) and the UI use Bevy's standard paths.

### Performance today (200k soldiers, widest view, clean desktop)

| | Value |
|---|---|
| Average | 66 fps, frames an even 15 and 16 ms |
| Fixed tick holds the frame | 6.3 ms (was 15 before PR #6) |
| Instance sync | 4.5 ms on all cores |
| Copies of the instance data | extract 2.0 ms main thread, write_buffer 1.5 ms render thread |
| GPU unit pass | 3.4 ms at full clock, GPU idle about 11 ms per frame |
| Sim job on the worker | grid 4.7 + integrate 7 ms, overlapping the frame |

The game is CPU-bound. Bevy runs every system on its compute pool, so the sim job's chunks delay all of Update, not only the parallel sync (devlog 0081). Frame anatomy and what each plan item changes: devlog 0081.

### The 1M plan (docs/plans/scale-to-1m.md)

Owner target: the renderer must be capable of 1M soldiers. The mainstream battle is 200k with textured 3D models, full environment and later pathfinding at 60 fps or more.

| Item | Status |
|---|---|
| 1 Distance LOD | DONE, PR #5 (devlog 0077) |
| 3a Vertex pulling experiment | POSITIVE (devlog 0078). One pulled draw path beats instancing at every level. Code on `exp/vertex-pull`, unmerged reference |
| 0 Tick off the frame path, FL_HASH, catch-up clamp | DONE, PR #6 (devlogs 0079, 0081) |
| 2 Build render data on the GPU, with item 3 folded in | NEXT. Handoff tmp/handoffs/HANDOFF-gpu-render-data-2026-09-22.md. Design doc first. Removes about 8 ms of CPU per frame at 200k |
| 9 Bigger battlefield | after item 2. The map caps near 360k soldiers |
| 6 Less fixed work per soldier | after measuring the tick above 200k |
| 7 Sleeping soldiers | DEFERRED by the owner, changes outcomes |
| 8 Sim on the GPU | last resort |

Measurement rules learned the hard way: GPU pass times follow the 3090's clock (550 to 1935 MHz), log or pin the clock (devlog 0078). A 10 s periodic hitch is gnome-shell, not the game (devlogs 0051, 0080). Keep benchmark load at normal game load, the owner works on the box. Do not propose a second task pool (devlog 0079).

## Assets track

- Today every mesh is code-built. Art direction decided 2026-09-20: Medieval II: Total War, balanced between performance and realism, fallback slightly casual (Kingdoms and Castles).
- The brief for generated or authored models is docs/plans/unit-asset-spec.md. Stage 1: rigid parts, vertex colors with alpha as team amount, four levels per kind, one GLB per kind, knight first. Budgets 2,000 to 3,000 / 600 to 800 / 150 to 250 / 24 to 60 triangles.
- Tooling set up 2026-09-22 (docs/internal/astra-blender-setup.md): Blender 5.2.2 LTS, Codex CLI 0.155 with GPT-6 Astra through an API key (the Free plan has no Astra), the ahujasid Blender MCP registered, a headless bpy pipeline proven end to end in assets_dev/_setup_smoke/.
- Export findings: vertex groups do not reach glTF, bake the part id into a second UV layer. Vertex color alpha needs `export_vertex_color="ACTIVE"`. "Facing +Z" in the spec means facing -Y in Blender.
- Engine gap: no glTF unit loader exists yet. Assets can only be judged in Blender until one is written.
- Licensing: commit only generated or CC0 assets, never raw Mixamo files. Sound effects are AI-generated plus one Pixabay loop.

## Workflow and conventions

- devlogs/: one entry per work chunk, four-digit numbers, README.md is the index. Next number 0082. Never committed.
- docs/plans/: design docs and plans, never committed. docs/internal/: local setup notes, never committed. tmp/: drafts (PR text), handoffs (thread entry points), scripts (measurement and hash tools), backups (refs and bundles before ref surgery), hash-baselines.
- Branch per feature, commit per milestone, build and clippy clean at each. Commit messages are public prose in the present tense, no prefixes, no footers. The owner pushes and opens the PR from tmp/drafts/pr-*.md.
- Verification: `FL_HASH` fingerprints (tmp/scripts/hashrun.sh, hashcmp.sh) for any sim refactor. Scripted batteries `FL_TEST_DIR`, `FL_TEST_CHARGE`, `FL_TEST_ARCHERY`, `FL_TEST_ROUT`, `FL_TEST_SURROUND`, `FL_TEST_FORM`, 110 s each, bands recorded in devlogs 0075 and 0079. `FL_TEST_FRONT=1` starts a battle without menus.
- Common knobs: `FL_UNITS` (per team), `FL_AI=0`, `FL_ENEMY_STATIC=1`, `FL_VOLUME=0`, `FL_CAM_LOCK=1` with `FL_CAM_DIST`, `FL_NO_CULL=1`, `FL_LOD_PX`, `FL_LOD_DEBUG=1`, `FL_PIPELINE=0`, `FL_CATCHUP`, `FL_SEED`, `FL_MAP=river`.
- Agent rules from the owner: no subagents, never send input to his screen, propose before touching feel-critical code, visual changes wait for his feel pass, no em dashes, no semicolons, short sentences, no hard wraps in prose docs.

## Where to read next

- History: devlogs/README.md, then the entry you need.
- The plan: docs/plans/scale-to-1m.md.
- The frame: devlogs/0081-cpu-frame-anatomy-200k.md.
- The sim tick: devlogs/0079-pipelined-tick-port.md.
- Rendering and LOD: docs/plans/unit-lod.md, devlogs 0076 to 0078.
- Melee model and evidence: devlogs 0035, 0036. Morale: 0055 to 0057. Archery: 0060 to 0065. Audio: 0069 to 0074.
- Assets: docs/plans/unit-asset-spec.md, docs/internal/astra-blender-setup.md.
