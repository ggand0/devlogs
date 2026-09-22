# 0084: Knight L0 model and export review (2026-09-22)

The first knight asset review is built in `assets_dev/knight/`. Read `0000-project-overview.md`, the full unit asset spec and the Blender setup notes before building. The setup notes explicitly make L0 the first review gate, so L1 to L3 and the full acceptance run are deferred until the owner reviews the model.

## Scope and environment

Worked on the existing `perf/gpu-render-data` checkout without switching branches, editing engine code, staging, committing or running Cargo. Claude's existing and concurrent source changes were left alone. No subagents, downloads, package installs, destructive commands, desktop input, GUI manipulation or live Blender execution were used. Blender ran in a separate factory-startup background process, with four CPU threads. The Snap launcher required sandbox escalation, which the owner approved for the version check and the specific knight script.

## Asset

Original generated geometry for a dismounted high-medieval knight. Great helm with oculars, brass nasal reinforcement and rivets. Fitted mail hauberk and chausses, split surcoat with continuous shoulder seams, belt and buckle, curved heater shield with a pale cross, leather gloves and boots, arming sword pointing forward. Stage 1 vertex colors only. Small geometric mail accents are a close-view suggestion, not a texture atlas.

Build source is `assets_dev/knight/build_knight.py`. It exports with the exact call from the spec. It saves the authoring blend before re-importing the exported GLB for every review render. `compose_review.py` arranges the actual renders without retouching. All WIP artifacts stay in the ignored knight directory.

## Measurements

- L0: 2,704 triangles. Budget: 2,000 to 3,000. No zero-area triangles.
- Height: 1.799999952 m. Ground: 0 m. Export is Y up, facing Z forward.
- 2,272 authoring vertices. 100 percent have one full-weight part group. Body 656, weapon arm 343, left leg 378, right leg 378, shield arm 517.
- Export: 4,182 vertices after normal and color splits. One mesh under `L0`, one opaque material, four pivot nodes.
- Hip pivots in exported XYZ: (+0.115, 0.910, 0) and (-0.115, 0.910, 0). Shoulder pivots: (-0.224, 1.435, 0) weapon and (+0.224, 1.435, 0) shield.
- `COLOR_0` is VEC4, alpha ranges from 0 to 1. `TEXCOORD_1` contains exactly (0, 0), (1, 1.435), (2, 0.910), (3, 0.910), (5, 1.435).
- GLB attributes checked by the build assertions and `python3 assets_dev/_setup_smoke/glb_inspect.py assets_dev/knight/knight.glb`. Output saved in `glb_inspection.txt`. Re-import was used for visual review, not as data proof.
- Four L0 screen targets from 50 degrees above the horizon: 60, 20, 8 and 3 px. Actual raster coverage at alpha >= 0.5: 60, 20, 8 and 2 rows. Antialiasing reduces the strongly covered rows at the 3 px target.

## Export contract contradiction

The spec's Parts section agrees with the current engine: body 0, sword arm 1, left leg 2, right leg 3, spear arm 4, shield arm 5, bow arm 6. Its later Export channels section lists a different order and says pivot v is zero. Asked the owner asynchronously and continued the independent geometry work. No answer had arrived when building the export. Used the recommended Parts convention and stated that assumption in progress updates. The conflicting spec was left untouched and is documented in the knight README for review before loader integration.

An additional measured exporter behavior matters here. Blender flips UV v to `1-v`. The build stores `1-pivot_height` in the authoring `part` UV map so the actual GLB contains the intended height. Byte assertions check the exact exported pairs and pivot translations.

## Review package and next gate

`knight_review.png` shows front, back and a blue-team recolor driven by the exported alpha. `knight_size_checks.png` shows native pixel sizes and clearly labeled enlargements. The full source scene is `knight.blend`, asset is `knight.glb`, and numbers are in `validation.json`.

Stop for the owner's L0 art review. This is not a final accepted asset. L1 to L3, the L0/L2 silhouette overlay and the 60-degree joint rotation checks remain. Nothing has been promoted to `assets/units/` or `tools/blender/`.
