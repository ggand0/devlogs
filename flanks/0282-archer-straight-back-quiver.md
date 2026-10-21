# Archer straight back quiver
Written by GPT-6.

## Review and correction

Gota found the v2 placement decent but rejected the unnatural bend in the case. The instruction is to use a straight quiver and place it nicely. `quiver_v2` remains untouched on disk and uncommitted. The iteration table records the rejection.

`assets_dev/archer/quiver_v3` removes the 30 mm curved centreline offset. Every case ring now shares one straight axis, and the outer radii follow a linear taper so the side walls are straight as well. The opening retains its rolled lip. The five arrows, closed lining and chest attachment remain. There is no visible harness.

The complete rigid quiver is repositioned: base `(0.140, 0.980, -0.170)` m, mouth `(-0.205, 1.480, -0.250)` m. Endpoint distance is 612.7 mm. No geometry or animation outside the quiver changes. Body, hood, scarf, dagger and poses retain their approved values.

## Checks

- 2,932 / 680 / 250 / 56 triangles, 1.800 m rest height, 22 joints. The new quiver remains 228 triangles.
- Exported outer-wall deviation from the straight linear taper is 0.000033 mm, within float precision. All quiver vertices have full weight on the chest, so animation cannot bend it.
- All retained attributes, rig nodes and pivots match the approved dagger model. The source distance meshes are unchanged.
- Closed quiver surfaces: zero open edges, nonmanifold edges, inconsistent edge winding and degenerate triangles. Raw GLB channels inspected.
- 177 sampled poses pass the head, arm, bow, held-arrow and exposed-case-back crossing checks. Concealed front-side clothing contact remains, up to 20 edge crossings. Local case-to-clothing distances: 9.09 mm at rest, 2.18 mm standing, 1.59 mm melee ready. These are local contact samples, not a whole-case attachment measurement.
- Front, rear, both side and elevated stills and close-ups inspected. The straight silhouette is visible from both sides and behind.

## Scope and review

Review page: `assets_dev/archer/quiver_v3/review.html`. Full-body standing, melee-ready and shooting orbits use the same poses as v2. This corrects placement and case shape only. Arrow retrieval still targets the old hip position; L1–L3 still retain the previous quiver. Those remain separate stages. No final game GLB or animation JSON is changed. Pending review; no new commit.
