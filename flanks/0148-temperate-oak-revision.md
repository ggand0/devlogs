# 0148: Temperate oak revision
Written by GPT-6 Astra.

2026-09-27. Graphics tree, feat/vegetation.

## Direction

Gota asked to preserve oak v1 and revise it toward the temperate trees in grassland_map_ref.png. The current dry grassland texture is not the palette contract for reusable vegetation. V2 keeps one oak and the same triangle budgets.

## Asset

New folder: assets_dev/vegetation/oak_v2/. V1's 24 files were checked against SHA-256 hashes and remain unchanged. The builder replaces the horizontal bough fan with eight ascending boughs beginning lower on the trunk. Leafy forks grow at different positions along them. Thirty crown groups use varied ellipsoid shapes and heights rather than similar terminal balls. The resulting tree is taller and narrower, with foliage farther down the branch structure.

Foliage normals combine the local lobe and overall crown volumes, with an upward bias. This avoids the whole dark crown patch seen in v1 while retaining local shape. The branch geometry, volume normals, atlas, and baked far views are revised together. No material emission or lighting override is used in the game.

An original atlas was made with the built-in image generator. Two edits of the v1 atlas returned RGB images with a painted checkerboard, so they were rejected. A fresh generation returned actual RGBA cutouts and was copied unchanged. The exact successful prompt and provenance are beside the builder. Three oak sprays have neutral green leaves; the fourth quadrant contains grey-brown bark. This remains subject to visual review: in-game sunlight makes the leaves lighter than the Blender preview.

| Level | Triangles | Height | Width × depth |
|---|---:|---:|---:|
| L0 | 2672 | 13.164443 m | 10.250157 × 9.609492 m |
| L1 | 636 | 13.276800 m | 10.452212 × 9.996130 m |
| card | 4 | 16 m padded planes | 16 × 16 m padded planes |

The topology counts are unchanged from v1. Ground origin, identity node transforms, +Y-up export, opaque vertex alpha, wind UV weights, alpha-mask materials, and embedded RGBA images pass the raw GLB checks. L0/L1 height differs by 0.112 m. Far colour and tangent normal atlases were rebaked with a 16 m frame to contain the taller crown. Unit part IDs do not apply to this vegetation asset.

## Integration and validation

The only public source edit relative to the previous oak review is the fallback path in src/vegetation.rs, selecting oak_v2/oak.glb. The specimen remains at (-495,-60) in the grassland margin. No simulation inputs, terrain heights, water, shared randomness, or FL switches change; no hash run is required.

The opt-dev build and strict all-target clippy pass. glb_inspect.py and verify_oak.py pass. Logs are in tmp/runs/0015_vegetation-oak-v2/. The previous mip tests cover unchanged code and were not repeated for this path-only revision.

Matched captures at 40, 250 and 900 m, the opposite side and an overhead view are in tmp/shots/0015_vegetation-oak-v2/. The review page switches between preserved v1 and v2 captures at those positions. Current graphics-branch lighting has no cast tree shadows. The camera clip crosses the detail thresholds in both directions; level changes remain discrete, with hysteresis.

No forest population, wind, other species, final asset promotion, or performance claim is included in this revision. The private iteration is committed with its README row; the public source diff remains uncommitted for review. No push or PR.

Motion inspection used 88 sequential samples at 4 Hz across the whole 22-second clip. This tool interface does not play video directly; the full recording is available in the review page. No disappearance was seen in those samples. The log records L1→card at 16 px, card→L1 at 20 px, L1→L0 at 132.2 px and L0→L1 at 107.9 px. The overhead capture retains L1. Discrete changes and the small far silhouette remain review points, not a claim of seamless transitions.

The process guard was run on the host for every build, Blender run and game launch. Early Blender attempts refused while flanks-new ran. Gota then confirmed the GPU was free. Each capture stopped only the game it started. No synthetic desktop input was sent.

Private iteration commit: `4b7c9aa`, Add the temperate oak revision.
