# 0085: the generated knight goes in the game (2026-09-22)

Working from `main` a197872, uncommitted at the time of writing. Inputs: the asset track's handoff (work/handoffs/handoff-knight-l0-gpt6-astra.md), devlog 0084 (how the model was built), docs/plans/009-unit-asset-spec.md (the contract), `assets_dev/knight/knight.glb`.

Asked for: use the model for the knight kind, and a read on whether it works. Performance, animation, the look.

## What was built

`src/unit_glb.rs` (new, about 480 lines) reads a stage 1 glTF soldier and hands `setup_unit_mesh` the same four level mesh set `unit_meshes` builds in code. `render_units.rs` now calls `unit_glb::kind_lods(kind)` instead of `unit_meshes::build_kind_lods(kind)`. Nothing else in the renderer changed.

A kind uses its model when the file is there: `assets/units/<kind>.glb` first, then `assets_dev/<kind>/<kind>.glb`. The other three kinds have no file yet and keep their code-built meshes, so a mixed army already runs one imported kind next to three built ones.

Reading: `gltf` 1.4 with `names` and `utils`, the crate and feature set bevy's own loader already pulls in, so the dependency costs no extra build. `Gltf::from_slice` on the bytes, the scene walked with the parent chain applied, nodes named `L0` to `L3` taken as levels, `pivot_<part>` nodes kept for the pivot check.

Three transforms on the way in.

- SCALE. The spec authors at 1.80 m. A knight in the game is `2 * half_height` = 1.10 units tall and stands on the terrain at `half_height`, so the model is scaled by 0.611 and dropped until the feet sit at -0.55. Pivot heights ride the same transform: hip 0.910 m becomes 0.006, shoulder 1.435 m becomes 0.327.
- MIRROR. Every code-built mesh carries the shield on -X and the weapon on +X. The model is authored anatomically, shield in the left hand, which is +X for a figure facing +Z. Imported as authored its shield swings BACKWARDS on the shieldwall signal, so the import mirrors X, flips the normals and reverses the winding. `FL_GLB_MIRROR=0` imports it as authored.
- MISSING LEVELS. The file is L0 only. Far levels are drawn by the hundred thousand, so repeating L0 is not an option. Each missing level is derived from the finest one in the file.

## Deriving the far levels

Every part is cut into slabs along its longest axis and each slab becomes one box the size its vertices span. Along the split axis a box takes the slab's own bounds, not its vertices', so neighbours meet instead of leaving a ring of gaps up the figure. Boxes keep the part id and pivot they came from, so a pose is the same on both sides of a level switch. Slab counts are 8 per body and 4 per limb at L1, 3 and 2 at L2. The knight comes out at 276, 132 and 36 triangles against the spec's budgets of 800, 250 and 60.

The last level is one block stack, per the spec's "L3 is all body". Its mass comes from the body and the legs only. The first version bounded every part and produced a box as wide as the sword reach, a fat slab twice the figure's width. The colour still comes from every part, because at two pixels tall only the hue survives and it has to be the hue of the whole soldier.

Colours blend by what a player actually sees, measured. A soldier is built in layers, and this one has a surcoat over a mail hauberk with about the same surface area, of which the mail shows nothing. An area average therefore turned a blue regiment grey from L1 out, which breaks the spec's "a regiment must not change hue at a level boundary". `visible_weights` now renders the model into a 96 by 96 depth buffer from eight facings at the battle camera's 50 degrees and weights a surface by the pixels it wins. Hidden layers win nothing. The far levels came back blue.

`FL_GLB_FAR=code` fills the missing levels from the code-built set instead, for an A/B.

## What the loader checks

Every check below is a trap the spec calls out and every one of them is silent inside Blender.

- `COLOR_0` present and VEC4. A VEC3 means the exporter dropped the team amount in alpha and the error says which export flag brings it back.
- `TEXCOORD_1` present. Without it there are no part ids and no pivots.
- Part ids are whole numbers the engine has a part for, one pivot height per part.
- The pivot heights in `TEXCOORD_1` agree with the `pivot_<part>` empties within a centimetre. This is the check for the UV v flip: Blender writes `1 - v` on export, a build script that fails to compensate ships the wrong heights, and the empties are the only copy the exporter leaves alone.
- No triangle spans two parts, because it would tear when the parts rotate.
- Height near 1.80 m, feet near y = 0, triangle counts inside the budgets.
- Indices whole triangles and inside the vertex list. A malformed file must fail with a sentence, not with an out of bounds panic deep in the renderer.

The knight passes all of them silently. A file that fails logs one error line naming the file and the reason, and the code-built mesh stands in, so a battle still runs on a half exported model. Verified by feeding the loader a truncated GLB.

One line per fault, never one per vertex.

## Measurements

Recipe: `work/scripts/gpu-pass-measure.sh`, 200k soldiers (`FL_UNITS=100000`), all knights (`FL_HEAVY_FRAC=1 FL_SPEAR_FRAC=0 FL_ARCHER_FRAC=0`), the locked views of devlog 0077, muted, 26 s each. Loadavg 3.4 before the sweep. Core clocks logged per sample, and they differ between runs, so the pass times below are quoted with them.

| View | Drawn | Levels | Imported pass | Code pass |
|---|---|---|---|---|
| 900 m, culling off | 198k | all L3 | 1.06 to 1.43 ms at ~1560 MHz | 0.85 to 1.07 ms at ~1350 MHz |
| 120 m | 15 to 20k | L1 and L2 | 0.39 to 0.64 ms at ~930 MHz | 0.31 to 0.57 ms at ~870 MHz |
| 40 m | 3.2 to 4.5k | L0 and L1 | 0.93 to 0.99 ms | 0.25 to 0.55 ms |

Frame rate over the same runs: 80 to 156 fps imported against 88 to 150 fps code-built. The spread inside one run is wider than the gap between them, and the sim legs in the two logs are identical. The GPU still idles most of the frame (devlog 0082), so an extra half millisecond in the unit pass does not reach the frame.

At 40 m about 1,300 soldiers draw L0, which is 3.5M triangles in under a millisecond. The 2,704 triangle near mesh is affordable exactly as the spec assumed.

The one number worth watching for the 1M goal is L3. The derived level is 36 triangles against the code-built 24, and at 900 m that is the whole army. Clock-normalised it is about 1.5x the pass time, which matches the triangle ratio. At 1M that is a millisecond and a half, not nothing. Two slabs instead of three would match the code-built cost and lose the head as its own colour block.

Gota's own pass, 2026-09-23: a 100k all-knight battle ran 150 to 180 fps with dips to about 120, and he called the animation minimal but working. 200k untested.

Gates: `hashrun.sh` DIR and ARCHERY both 19 of 19 fingerprints equal to the baselines, so the sim is untouched. `FL_GPU_CHECK=1` on DIR compared 8,400 frames with 0 differences. Build and clippy clean.

## What it looks like

The knight reads as a knight at every zoom that matters. Great helm, mail hauberk, team surcoat with a cross, heater shield carrying the team colour, arming sword, brown boots. At 40 m the two armies read as blue and orange masses with no trouble. The rigid part animation needed no engine work: legs swing, the sword arm chops, the shieldwall swings the shield to the front, bodies topple and stay.

Two art findings worth a look before the next three kinds are built.

- TEAM COLOUR AREA. The model is 19% team colour by surface and 26% by the area a 50 degree camera sees. The code-built knight is 62%. The spec asks for team colour to dominate the cloth and for armies to read as two colours at every zoom. Against brown dirt the orange army now sits closer to the ground colour than the blue one does. Widening the surcoat and the shield face is a build script change, not an engine one.
- FOOTPRINT. At the same height the model covers 1.83 m2 of surface against the code-built knight's 3.49. Realistic proportions are slimmer than the chunky ones, so the same formation shows more ground between men and reads thinner at play zoom. The A/B screenshot pair at 45 m shows it plainly. If Gota wants the denser read, the levers are a wider model or a deliberate fattening of the far levels, the way `FL_LOD_WEAPON` already fattens far blades.

A third thing is engine side and pre-existing: the stab style translates the whole weapon arm forward by 1.05 local units on the strike, which is most of a body height. On the chunky mesh that reads as a lunge. On a detailed arm it reads as the arm leaving the shoulder. Worth a look with the model in place.

## The shield side, and a mismatch it uncovered

The engine draws every kind's shield on -X, which for a figure facing +Z is the anatomical RIGHT hand. movement.rs resolves shield cover on the other side: `left_side` is the +X side of the victim, where the sword is drawn. So the shield a player sees and the shield the sim credits have been on opposite sides for every kind, since before this branch.

This branch does not touch it. Mirroring the import was the small, contained choice, and it keeps the imported knight consistent with the three code-built kinds. Putting the convention right means mirroring four code-built meshes, flipping the sign of the shieldwall rotation in the shader and accepting that the behaviour baselines move, because flank damage would change sides. That is its own branch and its own feel pass.

## Knobs added

- `FL_UNIT_MESH=code`: code-built meshes for every kind, the A/B against an import.
- `FL_GLB_FAR=code`: import L0, fill the missing far levels from the code-built set.
- `FL_GLB_MIRROR=0`: import the model as authored, shield on the anatomical left.

## Open

- L1 to L3 from the asset track replace the derived ones the moment the file carries them. The loader already prefers file levels and says which levels it imported.
- The spec's part id table under "Export channels" contradicts its own Parts section. The asset follows Parts, which is what the shader reads. The table is corrected in this pass so the next kind is not built against the wrong one.
- Feel pass on the model in motion, and the art call on team colour area and footprint.
- Promotion to `assets/units/knight.glb` is still the asset track's gate. The loader already prefers that path.
