# 0077: Unit level of detail, build and measurement (2026-09-20)

Branch `feat/lod`. Plan: docs/plans/008-unit-lod.md. Background numbers: devlog 0076. Owner could not decide the four open design points on paper, so the defaults were built with every point switchable from the command line.

## State

All three milestones COMMITTED on `feat/lod`, build and clippy clean at each. Owner ran it and saw no difference from before, which was the goal.

- f487d5a: buckets keyed by kind and detail level, selection from on-screen soldier height, per-soldier jitter and hysteresis. Also the three flags, per-level counts in the overlay and the log, and empty buckets skip their draw. The five per-soldier smoothing arrays moved into one `SmoothState` struct, because the system hit Bevy's 16-parameter limit.
- 72561eb: the three coarser meshes for all four kinds.
- 0cfc1cd: the fallen go through the same per-frame cull and level pick as the living. The static corpse buckets and `sync_corpses` are gone. Sixteen draws in total.

## What was built

Selection: a soldier of height H at distance d covers `H * viewport_height / (2 * d * tan(fov / 2))` pixels. Thresholds are 28 / 12 / 3 px of soldier height. The window here is 1600x900, so the switch distances are about 39 m, 91 m and 362 m. Distance is Euclidean to the camera position, so turning in place never changes a level.

Meshes, triangles per level as logged at startup:

| Kind | L0 | L1 | L2 | L3 |
|---|---|---|---|---|
| Knight | 204 | 120 | 72 | 24 |
| Man-at-arms | 180 | 120 | 72 | 24 |
| Spearman | 204 | 132 | 72 | 24 |
| Archer | 480 | 168 | 72 | 24 |

The old header comments overstated two L0 counts (man-at-arms 192, archer about 500). The logged numbers are the real ones.

- L1 merges and drops sub-pixel detail. Helm as one block, sleeve and hand as one arm, blade without grip and crossguard, shield without stub and boss. Face and kettle hat stay, they are the kind's tell from above.
- L2 is six blocks: torso, head, two legs, weapon, shield. Parts keep their id and pivot, so legs still walk, spears still level, shields still front.
- L3 is a body block and a head block.
- Merged blocks take a blended material from `blend()`. The shader shows `mix(rgb, team, a)`, so the team amount averages directly and rgb averages by its visible share. Weights are areas as the battle camera sees them, looking down at about 50 degrees. The first pass used front-facing areas only and the far heads read too grey.

Flags: `FL_LOD=0` (old rendering), `FL_LOD_PX=a,b,c`, `FL_LOD_DEBUG=1` (tint by level), `FL_LOD_WEAPON=f` (thicker L2 weapon block, feel probe). A zero in `FL_LOD_PX` disables that switch, so `28,12,0` never uses the two-block level. `FL_MESH_TESS` now tessellates L0 only, which is the shape of an authored mesh set.

## Verification

Screenshots through `import -window`, locked camera, small muted battles. Each level forced on in a close-up: all four render whole, no holes, no inside-out faces. A low gameplay view with selection off, on and tinted: off and on are very close. The tint shows L0 nearest the camera, a scattered changeover to L1 (the jitter works, no hard line), L2 on the far regiments. Visible difference: slightly less white sparkle in far ranks, because L2 drops bright spear tips and faces.

## Measurement (RTX 3090, 197k alive, same locked views as 0076)

| View | Drawn | Levels L0/L1/L2/L3 | Off | On |
|---|---|---|---|---|
| 900 m, culling off | 197k | 0 / 0 / 0 / 197k | 5.57 ms | 3.7 ms |
| 280 m default | 81k | 0 / 0 / 62k / 19k | 3.2 ms | 2.3 to 2.5 ms |
| 120 m typical | 27k | 0 / 0.7k / 26k / 0 | 1.4 ms (0076) | 0.9 to 1.0 ms |
| 40 m close-up | 5.4k | 1.4k / 4.1k / 0 / 0 | 0.3 to 0.6 ms | 0.46 to 0.54 ms |
| 40 m, dense L0 (TESS 4, 3.3k tris) | 5.4k | 1.4k / 4.1k / 0 / 0 | about 2.5 ms (scaled from 0076) | 1.5 to 1.8 ms |

CPU sync did not move outside its usual noise (2.4 to 4.2 ms across these runs).

## Reading

- The goal of the branch holds. A dense near mesh now costs under 2 ms in the close-up and NOTHING at any other zoom. No soldier is at L0 beyond about 39 m. Authored models fit.
- The gain on today's art is about 25 to 35 percent of the unit pass. That is less than the 45 to 55 percent the plan predicted.
- The reason is the open question from 0076, now settled. With all 197k soldiers on 24 triangles the pass still costs 3.7 ms. Triangles dropped 8.6 times, vertices 8.5 times, cost 1.5 times. The floor is real: about 17 to 19 ns per drawn soldier on the 3090, independent of mesh size. Instanced draws of tiny meshes pay per instance.
- Consequence for the 1M ladder in 0076: the far-soldier item is confirmed, and it is not about impostor art. It is about not using one instance per far soldier. One non-instanced draw that derives the soldier from the vertex index and reads instance data from a storage buffer removes the per-instance cost. Out of scope here. It matters above roughly 500k, or on a GPU where this floor is higher.
- Unknown until measured on one: integrated GPUs. They are weaker at triangles than at instance setup, so the relative gain there may be larger than on the 3090.

## Owner feel pass (done: "looks same as before")

Commands he can still use for the two untried points:

```
cargo run --profile opt-dev                        # LOD on
FL_LOD=0 cargo run --profile opt-dev               # old rendering
FL_LOD_DEBUG=1 cargo run --profile opt-dev         # tint: L1 green, L2 yellow, L3 red
FL_LOD_PX=28,12,0 cargo run --profile opt-dev      # never use the two-block level
FL_LOD_WEAPON=2 cargo run --profile opt-dev        # thicker far weapons
```

Look for: pops while zooming over a march and a melee, a hue shift between bands, lost life at the default zoom, the far ranks' sparkle.

## Corpse path canary

`FL_TEST_DIR=1 FL_VOLUME=0`, 110 s. Tallies at t=109 s: front 586 / side 297 / rear 616, dmg per hit 22.1 / 28.5 / 49.6. IDENTICAL to the devlog 0075 baseline, so the sim is untouched. Fallen drawn rose from 0 to 557 in view while about 1,500 lay dead, so culling of bodies works.

## Left before a PR

- Owner has not yet tried `FL_LOD_PX=28,12,0` (no two-block level) or `FL_LOD_WEAPON=2` (thicker far weapons). Defaults stand unless he says otherwise.
- devlogs/README.md index is behind (0068 onward).
- Not done and not planned for this branch: arrows and the 50k stuck-arrow litter have no level or distance cull.

## PR #5 and the final diff review

Owner pushed `feat/lod` and opened PR #5 (ggand0/flanks). Final check of `main...feat/lod`, done by reading the whole diff: 3 files, 703 added, 118 removed, merge state clean, clippy clean with warnings as errors.

Checked and sound:
- The hysteresis `clamp` cannot panic. Every threshold passed under the wider bound is also passed under the narrower one, so the fine level never exceeds the coarse level. Holds for zero, negative, infinite and NaN thresholds, which all fall back to "switch disabled".
- The tessellated `cuboid` reproduces the old vertices and winding exactly at grid size 1, flipped and unflipped faces both.
- `blade()` spans the same reach as `sword()`, so a far-level swing lands where the full one does.
- Stale corpse scratch after a battle reset is never read. The final zip stops at the job count, and each job clears its own entry first.
- The empty-draw skip also applies to the arrow buckets, which share the draw command. Harmless.
- No leftover references to the removed corpse buckets or the old builder signatures. README needs no change.

Known cost, not a blocker: the per-chunk scratch vectors and the 16 bucket vectors keep their high-water capacity. Over a session that zooms in and out, CPU memory for instance data can grow to roughly twice what it was before (L0 and L1 never hold many, L2 and L3 can each hold everyone). The GPU-driven instance build in docs/plans/010-scale-to-1m.md removes these buffers entirely.

Style nit left as is: `blade()` sits in its own small `impl MeshBuf` block instead of the main one.
