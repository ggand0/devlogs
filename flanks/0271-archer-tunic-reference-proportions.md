# Archer tunic reference proportions

Written by GPT-6.

The new `refs/archer/levy_archer_reference_tunic.png` and `archer_review3.png` expose a high, broad shoulder shelf above a narrow chest in arms_v4. The linen forearm also looks like a long smooth taper. This revision is `assets_dev/archer/arms_v5`; all prior iterations remain on disk unchanged. The coif, hands, lower body and equipment remain the preceding assets. The shoulder cape stays off for this shape review.

## Shape and deformation

Lower the upper shoulder crest from 1.470 m to 1.455 m and move it inward from x = ±0.205 m to ±0.177 m. The change is 15 mm vertically, not the 25 mm stated in the progress message. Lower the outside shoulder boundary from 1.428 m to 1.411 m and move it inward from ±0.254 m to ±0.237 m. The upper chest half-width grows from 0.158 m to 0.180 m, with depth from 0.122 m to 0.130 m. The cloth slopes from the neck into the sleeve and has more volume above the belt.

Bring the upper sleeve centres inward, flare the wool sleeve toward its hem and redistribute its first two rows beneath the lower armpit. The first row is at 43 percent of the shoulder-to-elbow distance; its inner contour is narrower to keep it outside the torso. The right linen sleeve has a fold above the wrist; the left remains fitted to its leather bracer. Map linen length by position instead of row number so the longer forearm section does not stretch the texture vertically.

The first draft crossed itself at the armpit in neutral and early draw. Moving the sleeve rows and narrowing the first inner contour resolves that overlap. A separate fold at the back of the shoulder during draw came from the changed geometry with the preceding influence value; reducing the rear crest's upper-arm influence from 0.25 to 0.15 resolves it. These changes are in the source geometry and weights, not hidden with the cape.

## Export checks

L0/L1/L2/L3: 2,994 / 680 / 250 / 56 triangles. Height 1.80 m. All 22 joint nodes and pivots are unchanged. Shoulder pivots remain (±0.205, 1.435, 0), elbows (±0.290, 1.155, -0.010), wrists (±0.350, 0.925, 0.130), and head (0, 1.485, 0), in metres. Detailed part coverage and all pivot positions are in `validation.json` and `build_report.json`.

The generated 820-triangle surface forms one closed component, with zero open edges, nonmanifold edges, reversed shared edges or degenerate triangles. Neutral edge/triangle checks detect zero self-crossings. Stand, ready, draw, raised aim and both release clips total 174 sampled poses, with zero detected self-crossings or seam openings in that surface. This is sampled testing, not continuous collision proof. Retained attributes and retained atlas pixels match the canonical source. The raw GLB inspector and surface inspector both ran; per-part open boundaries in the latter are expected at joined skin surfaces, and pre-existing defects elsewhere remain outside this revision.

Model SHA256: `4c15c7b96766040912fc8250c23e096223dc27f96576c88a25021efd1d10985a`.

The public game asset remains `d2ec53996146b9c7e0649b74a8a6a97dfd6e6aa570d04702c1bb6f188ec823c6`. No public code, animation JSON or installed asset changed. L1-L3 still retain the source geometry pending shape review; this is not a final asset delivery.

## Review

Front, rear and both side stills use the same scale. `front_comparison.png` shows v4, v5 and the new tunic reference, whose pose and perspective differ. `review.html` opens with this comparison. Fixed-view motion uses textures and the game skinning calculation, with all four views visible together. Neutral, level-shot and raised-shot orbit videos provide the additional textured coverage. The shot previews restart after each release hold without a reload transition. Reference crops stay local and ignored.

Inspected full-size cardinal views and consecutive-frame sheets for the draw and aim transitions. Video validation and final review records follow below. The remaining coif/neckline and cloth detailing still differ from the reference. No commit or in-game approval is claimed.

Final media checks: the four-view video has 87 frames at 24 fps (3.625 s, 1024 by 1096). The neutral, level-shot and raised-shot orbits have 192, 288 and 432 frames respectively, all at 24 fps. All four videos decoded without errors. Orbit manifests and validation records match the final GLB hash. All 34 review links resolve. Inspected all twelve fixed-view consecutive-frame sheets and all three orbit contact sheets, plus full-size cardinal and posed images. No real-time playback or in-game review is claimed.
