# Archer shoulder plane and sleeve root

Written by GPT-6.

`refs/archer/archer_review4.png` rejects the rounded shoulders and narrow-root, flared-hem sleeve in arms_v5. It marks the arms_v4 upper sleeve as good. The tunic reference has a shallow shoulder line, a visible turn into the upper arm and a straighter sleeve. The previous change treated shoulder slope as rounding and introduced a bell-shaped sleeve.

## Shape correction

Created `assets_dev/archer/arms_v6` and left every previous folder intact. The shoulder crest is at x = ±0.218 m, y = 1.451 m. It joins the 0.060 m neck radius at y = 1.500 m with a shallow line, then turns down toward the outside sleeve root at x = ±0.254 m, y = 1.415 m. The front and rear support vertices shape the chest and scapular transitions separately. Compared with v4, the crest is 19 mm lower and the outer root 13 mm lower; this does not restore the former high shoulder shelf.

The first four sleeve rows use the v4 fractions and widths again: shoulder-to-elbow fractions 0.29, 0.52, 0.73 and 0.83; radial widths 0.064, 0.066, 0.061 and 0.060 m. The centreline follows the existing shoulder and elbow. Removed v5's inward centre shift and reduced inner first-row radius, which narrowed the root while its hem widened. The inner shoulder boundary returns to the v4 contour. Chest, forearm linen, bracer and hands keep the v5 geometry.

The first export passed the neutral shape check but folded behind the drawing shoulder between 1.05 and 1.125 seconds. Changing the rear shoulder support's upper-arm influence from 0.25 to 0.18 resolves the detected fold without changing the neutral outline. Final checks use the exported GLB, not just the authored polygon surface.

## Validation

L0/L1/L2/L3: 2,994 / 680 / 250 / 56 triangles. Height 1.80 m. Twenty-two joints; existing joint nodes and pivot positions are unchanged. Shoulders remain (±0.205, 1.435, 0), elbows (±0.290, 1.155, -0.010), wrists (±0.350, 0.925, 0.130), in metres. Full joint positions and part coverage are in `build_report.json` and `validation.json`.

The 820-triangle rebuilt surface is one closed component with zero open edges, nonmanifold edges, inconsistent shared-edge winding or degenerate triangles. Neutral and 174 sampled stand, ready, draw, aim-up and release poses have no detected self-crossings or seam openings in that surface. This samples edge-triangle intersections; it is not continuous collision proof. Retained L0 attributes and retained atlas pixels are unchanged, as are L1-L3. Raw GLB and surface inspectors ran. Existing defects in retained source parts are not newly resolved by this revision.

Final model SHA256: `91789e1658bab86c92656b778d05fa83a0ce9017f8748768355fa06b3a38aa3b`.

## Review scope

The review opens with v5/v6/reference front views and a separate v4/v5/v6/reference sleeve-root detail. Full front, rear and both side views use the same camera scale. Textured fixed-view motion and orbit videos show the existing shot clips through the game skinning calculation. Reference crops remain local and ignored. The original coif remains and the shoulder cape stays off for the arm review. Lower-detail meshes retain the source geometry. No public code, installed model or animation JSON changed. No iteration is committed or shipped.

Final media checks: the four-view video contains 87 frames at 24 fps, 1024 by 1096. Neutral, level-shot and raised-shot orbits contain 192, 288 and 432 frames respectively, all at 24 fps. All four videos decode without errors. Render manifests and validation records match the final GLB hash; all 36 local review links resolve. Inspected front and sleeve comparisons, full-size cardinal and posed images, all twelve consecutive-frame sheets and all three orbit contact sheets. No real-time playback or in-game review is claimed. Prior v4 and v5 model hashes remain unchanged.
