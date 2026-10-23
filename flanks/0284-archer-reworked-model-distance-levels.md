# Archer L1 to L3 on the reworked model

Written by GPT-6. 2026-10-22. Graphics branch `feat/archer-hood-source`.

## Request and contract

Gota asked for the three distance variants requested by Claude in handoff 142. The integrated L0 already has the approved arms_v7 body, hood_v4 hood and scarf, melee_poses_v2 knife and quiver_v3 straight back quiver. This iteration changes the three distance meshes and writes the knife's part 20 during the build. It does not change the approved L0 geometry or animation clips.

Iteration: `assets_dev/archer/rework_lods_v1/`. Review: `review.html`. Candidate: `archer.glb`. Gota approved the distance models on 2026-10-23.

## Construction and corrections

- L1 reduces the connected upper tunic, shoulders, sleeves and hands together. Skin weights interpolate from the approved surface, constrained to the runtime's parent-child joint pairs. The face, hood, scarf, boots and equipment have separate triangle allocations.
- The first whole-mesh reduction accidentally omitted the large connected tunic component because it also included the skirt. The builder now handles that connection explicitly. Early convex hull construction also retained duplicate interior faces; hulls now start from points only.
- The first far torso and head reductions pulled their extremities away from the source and left a poor waist outline. L2 now uses measured closed body, head and leg profiles. L3 uses a tapered body, an eight-triangle hood, separate limbs and a back-quiver silhouette.
- Texture projection initially picked inside cloth surfaces and produced dark patches. Projection now requires source normals to face the target surface. Far cloth uses stable atlas samples to avoid stretching small source patches across large faces.
- A light arrow fan was too prominent at 8 px. L2 now models just the exposed tips above the quiver mouth.
- Convex bow hulls filled the curve and made thick triangular limbs. L2 now has two closed triangular-section curved staves. L3 has two narrow closed staves, still separate from the body. Parts 14 and 15 hide with the knife. Part 6 remains the arm and is not used to hide a weapon.
- The first L3 body was too broad from the elevated camera. Its upper ring now tapers into the neck. The farthest level stays within 60 triangles.
- The raw topology check found inherited open skirt hem and slit edges in L1. The final L1 skirt has closed outer and inner surfaces with connected hem, slit and waist edges. Both skirt halves retain the front and rear openings. Removing redundant detail elsewhere keeps the complete L1 at 800 triangles.
- The rectangular upper thighs in L2 protruded through the skirt sides. Their concealed cross section now fits inside the tunic. This changed no L0 shape.

## Final measurements

| Level | Triangles | Height | Use |
|---|---:|---:|---|
| L0 | 2932 | 1.800 m | Integrated close model, unchanged |
| L1 | 800 | 1.800 m | Skinned, 12 to 28 px |
| L2 | 240 | 1.800 m | Rigid parts, 3 to 12 px |
| L3 | 60 | 1.800 m | Rigid parts, below 3 px |

The L0 attribute arrays and indices match the integrated main-tree model exactly. The 22 joints, pivot translations, skeleton, embedded atlas and animation sidecars are retained. Knife vertices are part 20, rigid on hand_r. Each level retains separately hideable bow limbs and a knife. Quiver surfaces follow the torso.

All three new levels have zero open edges, nonmanifold edges, inconsistent winding and degenerate triangles when geometric seams are welded across parts. The per-part inspector still lists the intended shared boundaries in L1's continuous skinned arms; these close when the adjoining parts are considered together. L2 and L3 are closed even when inspected part by part. L0's existing topology is unchanged.

The common-frame silhouette comparison gives L2 overlap of 0.865 front, 0.865 rear and 0.884 side. At 8 px from the 50 degree review camera, covered image area is 0.958 of L0 for L1, 0.974 for L2, and 1.094 for L3. L3 is shown at 8 px for comparison but is intended below 3 px. These are image measurements, not a claim of identical silhouettes at close range.

## Checks and review media

- Raw GLB inspection: all four levels, VEC4 colour, integer part IDs, skin arrays, pivot heights and the 1.80 m height pass.
- `check_lods.py`: 273 samples across 19 archer and knight clips; all four knight shuffle clips included. Also 126 individual joint rotations at plus/minus 60 degrees on each axis. The numerical test checks valid deformation, not collision-free anatomy at arbitrary poses.
- `surfaces.json` records per-part checks; `validation.json` records whole-level closure and equipment visibility counts.
- Front, rear, left, right and 50 degree views, red/blue team colours, L0/L2 overlays, and 60/20/8/3 px comparisons were rendered and inspected.
- `near_comparison.mp4`: 12 seconds at 20 fps, L0/L1 skinned standing/ready knife poses and the existing shot sequence. The review blend between knife poses is not a delivered clip.
- `far_comparison.mp4`: 8 seconds at 20 fps, the rigid shooting transforms and a knife visibility switch. L0 is deliberately posed by parts in this comparison as a reference for L2/L3. Far levels do not play the near melee clips.
- Both videos played through a full loop in an isolated headless browser; 8 sequential screenshots per video were inspected. The browser reports 247 and 166 decoded frames including the next loop, with no media error.
- Runtime preview folder: `tmp/runs/archer-rework-lods/preview/`, containing the candidate and all 23 archer/knight JSON sidecars from the integrated tree. The existing integrated game binary loads all four authored levels and 22 joints. Short forced-level launch checks are recorded under `tmp/runs/archer-rework-lods/`; they are load/render smoke checks, not performance measurements.

Candidate SHA-256: `f344e31139e4648cecaea55a343427fd7e70c00b902a88d24ea7a52edb56eb80`.

## Remaining work

The distance models are approved. The model and portable build source are committed on the graphics branch for integration into `feat/archer-rework`. The back-quiver reload remains the next animation task. Sword draw stays skipped; knife attacks continue to use the knight clips, per handoff 142.

## Approval and promotion, 2026-10-23

Gota: “looks good, approved, feel free to document into the devlog”. Approval covers L1/L2/L3 at 800/240/60 triangles, with the previously approved L0 unchanged. The approved GLB checksum remains `f344e31139e4648cecaea55a343427fd7e70c00b902a88d24ea7a52edb56eb80`.

Promoted the build to `tools/blender/archer/lods/build_lods.py` with explicit source/output arguments and no private source-map dependency. It reads knife part 20 from the approved L0 and verifies all L0 channels with a fingerprint. A fresh background Blender run reproduced every attribute and index array of all four levels exactly; joints and pivots also match. Copied the approved bytes to `assets/units/archer.glb` in the graphics worktree. The main worktree remains for Claude/Gota to integrate.

Commits:

- Private assets repository `main`: `5f6397b`, Add the archer distance meshes (49 files, including the iteration row; images and videos use LFS).
- Graphics `feat/archer-hood-source`: `f34fb36`, Add the archer distance models (`assets/units/archer.glb` through LFS and the two portable build files).

Handoff 143 now records approval and the graphics commit to cherry-pick into `feat/archer-rework`. No pushes or changes to main-tree source were made. Development logs and handoffs remain uncommitted.
