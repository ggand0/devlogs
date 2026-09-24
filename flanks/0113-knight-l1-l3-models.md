# 0113: Knight L1 and L3 models
Written by GPT-6 Astra.

2026-09-24. Built authored L1 and L3 for the knight after Gota requested replacements for the automatic slab levels. Work stops at this kind for review before building the remaining units.

Candidate: `assets_dev/knight/lod_l1_l3_v1/knight.glb`. Combined review: `assets_dev/knight/lod_l1_l3_v1/review.html`. Build script: `tools/blender/lods/build_knight_l1_l3.py`.

## Geometry

| Level | Triangles | Parts |
|---|---:|---|
| L0 | 2768 | Preserved |
| L1 | 678 | body 236, arm_weapon 80, leg_l 76, leg_r 76, arm_shield 168, weapon 42 |
| L2 | 250 | Preserved |
| L3 | 52 | body 52 |

L1 keeps the flat-crowned helm and dark visor, mail at the collar and sides of the surcoat, belt, shaped boots, elbow vertices, gloves, sword blade and hilt. The shield has a narrow leather rim, a wood back and two closed straps, with 23 mm board thickness. The L1 sleeve contains enough rings around the elbow for the existing shader's bend.

L3 uses a tapered waist, a five-sided flat-crowned helmet, two leg masses and a closed triangular shield with a wood back. All vertices use part 0 and height 0, so it remains a static far silhouette. Arms, sword and straps are omitted at this level. It is designed for under 3 pixels, with larger views included to expose the geometry.

All new levels are 1.80 m high with their lowest point at zero. L1 retains the original part IDs and pivot heights. The installed L0 and L2 binary data, accessors, mesh metadata, atlas, material and existing nodes remain unchanged. New accessors and nodes are appended to the original GLB. No installed asset, src/ or renderer file was edited.

## Colour and review

Each new triangle samples the original atlas. L1 fits the visible means by animated part; L3 fits against the complete L0 rather than just L0's body part. Visibility is measured from eight facings at 50 degrees in linear light.

Visible team share is 33.41% for L0, 32.82% for L1, 34.32% for the unchanged L2, and 33.63% for L3. L1 mean RGB changes relative to L0 are approximately -2.34%, -2.40% and -2.49%. L3 changes are +1.50%, +0.28% and -2.76%.

The review contains red and blue teams at 20, 8 and 3 pixels with identical framing, plus 60-pixel and 256-pixel renders. Front, back and side overlays show L1 silhouette IoU of 91.1%, 91.7% and 89.3%; L3 is 73.3%, 73.4% and 75.9%, including the omitted arms and sword. Enlarged L3 views show simplifications that are intended for its sub-3-pixel use. The original source's complete silhouette sets the pixel framing.

## Checks

Ran `glb_inspect.py`, the setup-smoke inspector, `inspect_surfaces.py` and the new four-level verifier. L1 and L3 have zero open edges, nonmanifold edges, inconsistent winding and degenerate triangles. Both shields are closed with positive volume. COLOR_0 is VEC4 with white RGB, TEXCOORD_1 holds the intended IDs and original pivot heights, and all levels use the unchanged atlas material.

All L1 pivots pass the 0 to 60 degree sweep at 10 degree increments. Sampled the existing stab and overhead attacks over 90 frames; the checked shoulders and sword grip remain connected in every frame. Temporal frame sequences were inspected, and the paired H.264 video is 640x360, 90 frames over 6 seconds. These are asset-side comparisons, not an in-game runtime capture.

A clean build from the formatted repository scripts produced a byte-identical GLB: SHA-256 `6952f4f102419aea131a49b3466f0526f3646b4a4f91909578024d7a8eeea6a7`. Build inputs are the installed knight GLB and SHA-bound component labels under `tools/blender/lods/source_labels/`; no WIP authoring scene is required. Python compilation passes and review-page links resolve.

Existing L0 defects remain unchanged: 490 open edges and 12 same-winding edges. L2 remains closed and unchanged. The new levels pass their topology checks; this work does not repair the preserved L0.

Shared LOD tools now select source and destination levels by node name, support appending a named level and fit L3 against all visible source parts. Joint and motion tools accept `--level L1`, and the new render/composition scripts display all four levels. Other agents' untracked sword-motion work was left untouched.

Stopped for Gota's review, with no staging or commits. The note for Claude follows the review, as requested. Other kinds have not been started.
