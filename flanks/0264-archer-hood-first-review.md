# Archer hood, first review
Written by GPT-6.

2026-10-18. Gota authorized the archer fixes with a review after the hood and another after the sleeves. This iteration stops at the hood review. No sleeve, equipment, shipped model or engine changes have been made.

## Candidate

`assets_dev/archer/hood_v1/` replaces the L0 linen coif, binding, ties, lowered wool cape and folded hood with a raised wool hood, pale inner lining and a shoulder cape. The cloth takes the team colour. The reference concept informs the garment; handoff 138 calls for a raised hood and later a back quiver, unlike the concept's lowered hood and hip quiver.

Review: `assets_dev/archer/hood_v1/review.html`. The comparison has front, three-quarter, side and back views, full-figure and shooting poses, and a 10-second video with head turns followed by the existing draw, elevated aim and release. No new game clip was authored.

The original 22-joint skeleton, hierarchy, bind matrices, rest positions and pivots are preserved exactly. Every retained L0 vertex attribute is unchanged. L1-L3 are also unchanged and still carry the preceding hood, pending the shape review. This is not a shipping candidate yet.

## Build and materials

`prepare_source.py` identifies and removes 545 L0 triangles belonging to the old head covering and cape. It reserves an unused 160 by 160 pixel rectangle in the existing 2048-square atlas, with an eight-pixel separation from all retained UV triangles across all four levels. Existing atlas pixels outside that rectangle and all existing UVs are preserved. The new patch uses the project's wool texture, normalized to a neutral team surface, plus a plain linen lining with zero team amount.

`build_hood.py` builds the hood and cape as two closed cloth shells, exports the new geometry with the exact export call in the asset spec, then combines it with the retained raw GLB attributes and original skeleton. The head covering rides the existing head joint. The cape follows the chest, with the outermost shoulder edges blended toward the existing upper-arm joints. Whole-shell topology is verified across attribute seams, rather than inferred from Blender's re-import.

Early checks exposed a rectangular face opening, shoulder protrusions and head/collar weighting that crossed the face during aiming. The opening now follows the brow and jaw, the crown has an additional rounding ring, the head covering follows the head throughout, and the cape is fitted over the existing shoulder covers. Those changes are all within the unreviewed hood_v1 iteration.

## Measurements

- L0: 2,982 triangles, compared with 2,987 before; new hood 348 and new cape 192 triangles.
- L1/L2/L3: 680 / 250 / 56 triangles, unchanged.
- Rest height: 1.80 m; 22 joints, original pivots and part IDs 18 and 19.
- New hood and cape: zero open edges, nonmanifold edges, inconsistent winding or degenerate triangles; positive signed volume.
- COLOR_0 VEC4, both UV sets, JOINTS_0 and WEIGHTS_0 retained. One opaque material. The new cloth uses the atlas alpha as team amount.
- 255 sampled poses: 81 head-turn poses at plus/minus 55 degrees, standing, ready, 61 draw samples, 61 elevated-aim samples, and 25 samples for each release.
- Zero measured crossings in all those samples for hood/face, hood/draw arm, hood/bow arm, hood/bow-and-string, cape/old shoulder surfaces and cape/forearms.
- Minimum new triangle area relative to rest is approximately 0.109 during elevated aim. The cape's narrow edge triangles compress as the arms rise; this is recorded rather than treated as a cloth simulation.

The crossing checks are sampled edge-triangle tests, not continuous collision or containment proofs. Hood/cape collar overlap and covered sleeve attachment overlap remain intentional. Existing topology outside the replacement is recorded in surface_validation.json and is not claimed repaired.

## Review media checks

The 1120 by 640 video contains 240 frames at 24 fps (10 seconds). All frames decode. Headless Firefox played beyond a complete loop, with advancing frame counters and no paused samples. Browser captures exposed an initial horizontal framing mistake; the camera now accounts for the wide aspect ratio, and every frame checks the projected head and upper torso bounds. Maximum normalized extents are 0.933 horizontally and 0.648 vertically, within the 0.98 limit. The close head-turn view and wider shooting view are intentional; bow tips and lower body are outside some close views. The static comparison and full-figure images provide the other views.

## Next

Wait for Gota's hood review. After it looks right, proceed to the sleeve rebuild and stop for the second requested review. Update the distance meshes to match the reviewed shapes before final model delivery. The quiver, sidearm and bow stow follow the requested review sequence.

No commits or pushes. The private assets repository stays on main. Shared _skel tools and other iterations are untouched.

Candidate SHA-256: 53ab92b18b1fe92f5ace99f97b4f4edaf2503ec8c9044e21f9b203c12dd9dacf
