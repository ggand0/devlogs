# 009 — NuRec Volume Depth Compositing Investigation

2026-05-05

## Problem

Two NuRec Gaussian splat volumes in Isaac Sim 5.1 (desk 218k splats + bowl 566k splats) don't depth-test against each other. The bowl always renders in front of the desk from all camera angles, including angles where desk structures should occlude it. Each NuRec volume is composited independently over the framebuffer in scene-graph traversal order via `rtx:post:registeredCompositing` — no inter-volume depth testing exists.

The proxy relationship on a Volume prim only provides depth info for compositing against the rasterized depth buffer (meshes, robot arm), not against other volumes. This was confirmed by testing invisible proxies on both volumes — no change in volume-vs-volume depth.

## Approaches Attempted

### 1. Poisson Mesh Reconstruction from Splat Positions — FAILED

Attempted to reconstruct the bowl as a triangle mesh (which would participate in the standard depth buffer) using Poisson surface reconstruction on the Gaussian splat center positions.

**Pipeline:**
- Load 566k splats from `green_bowl.ply`
- Filter by opacity > 0.5 → 445k points
- Statistical outlier removal → 440k points
- Voxel downsample (0.003) → 291k points
- Estimate normals (PLY normals were all zero)
- Orient normals outward from centroid
- Poisson reconstruction depth=9 → 1.1M verts, 2.2M faces
- Density crop + bbox crop → 1.1M verts
- Quadric decimation → 51k verts, 100k faces
- Vertex color transfer from SH DC coefficients (C0 * f_dc + 0.5)
- Taubin smoothing

**Technical issues encountered:**
- Open3D's `orient_normals_consistent_tangent_plane` failed on 440k points → used `orient_normals_towards_camera_location` + flip instead
- Open3D's `compute_convex_hull` segfaulted on the 100k-face mesh
- Importing `pxr` alongside `open3d` in the sam-3d-objects venv segfaulted → split into separate steps
- Taubin smoothing introduced NaN vertices → 2980/51432 vertices had NaN colors
- USD export needed numpy float → Python float cast for `Gf.Vec3f`

**Fundamental failure:** Gaussian splat center positions are distributed throughout the object volume, NOT concentrated on the surface. SAM 3D generates splats from a single image — without multi-view supervision, the model fills the interior with splats to approximate the appearance from all angles. Poisson reconstruction assumes surface-sampled input, so the result was a ragged, holey, artifact-ridden mesh that looked nothing like the bowl.

**Render test:** Isaac Sim rendered the mesh but it was visually terrible — grey (UsdPreviewSurface doesn't pipe vertex colors through to the RTX renderer), full of holes, floating fragments, and ragged edges.

**Material issue:** Isaac Sim's RTX renderer requires OmniPBR materials (MDL) for proper rendering. `UsdPreviewSurface` with a single `diffuseColor` was set to the average vertex color but did not use per-vertex interpolation. Vertex colors via `primvars:displayColor` were not displayed by the RTX renderer without proper material binding.

### 2. Previous Attempts (from prior sessions, documented for completeness)

| Approach | Result |
|---|---|
| Invisible proxy on bowl only | Rasterized geometry occludes bowl. Desk volume does not. |
| Invisible proxies on both | No change for volume-vs-volume depth |
| Visible proxies on both | Scene breaks — grey overlays, splats disappear |
| Matte object on desk proxy | Desk proxy blocks bowl but also blocks desk volume |
| Merge into single volume (784k splats) | Correct depth. Reverted — bakes bowl into desk, not independently graspable. |
| Lightfield format (ParticleField3DGaussianSplat) | Isaac Sim 5.1 doesn't have the schema registered. Splats invisible. |
| Low-res marching cubes mesh (res 64, no colors) | Correct depth but terrible visual quality — blocky, no color |

## Research: Proper Splat-to-Mesh Conversion

### Methods that DON'T work on existing PLY files (need retraining)

- **SuGaR** — requires original training images + COLMAP poses to apply surface-alignment regularization during training. Cannot extract from pre-trained PLY.
- **2DGS** — fundamentally different representation (2D surfels). Cannot convert standard 3DGS to 2DGS.
- **Gaussian surfels / flattened approaches** — require retraining with flattening constraints.

### Methods that CAN work on existing PLY files

#### A. TSDF Fusion from Rendered Depth Maps (recommended)

Render the splat from many virtual camera viewpoints using a Gaussian splatting rasterizer, extract depth maps, fuse into a TSDF volume, then extract mesh via marching cubes.

- **Why this works:** Rendering from outside naturally produces clean surface depth — the rasterizer accumulates opacity front-to-back and the resulting depth represents the visible surface, not the internal splat distribution.
- **Dependencies available:** gsplat (in sam-3d-objects venv), Open3D 0.18.0 with `ScalableTSDFVolume`, numpy, torch
- **Effort:** ~100 line script. Define hemisphere camera orbit (e.g., 100 views), render depth+color per view with gsplat, TSDF integrate with Open3D, extract mesh, transfer vertex colors.
- **Quality expectation:** High for a convex bowl shape. Clean surface, proper topology.

#### B. Re-run SAM 3D with `decode_formats=['mesh']`

SAM 3D has a dedicated FlexiCubes mesh decoder (`slat_decoder_mesh`, 347MB checkpoint at `/data/sam-3d-objects/checkpoints/hf/slat_decoder_mesh.ckpt`). It extracts mesh from the structured latent (SLAT) — not from the Gaussian splat.

- **Previous session used `decode_formats=['gaussian']`** to avoid OOM on 24GB RTX 3090
- **Mitigation:** Use `decode_formats=['mesh']` only (skip GS decoder, save ~164MB VRAM). Still tight at ~13.7GB model load + activations.
- **Quality:** FlexiCubes at res 64 produces clean meshes with vertex colors. Ideal for simple shapes.
- **Risk:** May still OOM. Would need to re-run the full inference pipeline (not just the decoder — SLATs aren't cached).

#### C. GOF-style Opacity Field Marching Cubes

Evaluate accumulated opacity at grid points by ray-marching through the Gaussians, then extract the level-set surface. Custom implementation, uncertain quality for single-image volumetric splats.

### 3DGRUT Capabilities

3DGRUT has NO mesh extraction from splats. The transcode script (`threedgrut/export/scripts/transcode.py`) supports only: PLY ↔ lightfield ↔ NuRec. The `add_mesh_to_usdz.py` script adds external meshes to USDZ packages as proxies but doesn't generate them.

The playground renderer (`threedgrut_playground/engine.py`) handles depth internally for rendering effects but doesn't export depth maps. Would need modification for TSDF approach.

## Solution: SAM 3D Mesh Decoder

Re-ran SAM 3D inference with `decode_formats=['mesh']` instead of `['gaussian']`. The mesh decoder uses FlexiCubes at resolution 256 to extract a clean mesh directly from the structured latent (SLAT), NOT from the Gaussian splat positions.

**OOM workaround:** The pipeline loads all decoders unconditionally. After init, offloaded `slat_decoder_gs` and `slat_decoder_gs_4` to CPU and called `torch.cuda.empty_cache()` to free ~7GB VRAM. Also monkey-patched `postprocess_slat_output` to skip GLB creation (which requires Gaussian output).

**Postprocessing bug:** The `postprocess_slat_output` function unconditionally accesses `outputs["gaussian"][0]` when `"mesh"` is in outputs (for GLB export). Patched to skip when gaussian is missing.

**Pipeline output:** `MeshExtractResult` object with:
- `vertices`: [361586, 3] float32
- `faces`: [723012, 3] int64
- `vertex_attrs`: [361586, 6] float32 (first 3 channels are RGB vertex colors)
- `res`: 256
- `success`: True

Decimated to 50k verts / 100k faces via Open3D quadric decimation. Exported as USD with:
- Visible mesh at `/World/gauss/gauss` with `displayColor` primvar (per-vertex) and UsdPreviewSurface material
- Invisible concave collision proxy at `/World/gauss/mesh` (2578-vert / 5000-face decimated bowl mesh, `UsdPhysics.CollisionAPI` + `meshSimplification` approximation — objects can sit inside the bowl)
- Same hierarchy as `green_bowl_with_proxy.usdz` so stage reference works unchanged

## Files Produced

| File | Description | Status |
|---|---|---|
| `assets/table_objects/green_bowl_round/green_bowl_sam3d_mesh.obj` | 361k vert, 723k face FlexiCubes mesh with vertex colors (full res) | Good |
| `assets/table_objects/green_bowl_round/green_bowl_sam3d_decimated.obj` | 50k vert, 100k face decimated mesh | Good |
| `assets/table_objects/green_bowl_round/green_bowl_sam3d.usd` | USD with mesh + collision proxy + material | **Active** |
| `assets/table_objects/green_bowl_round/green_bowl_hq.obj` | 51k vert Poisson mesh | Failed quality |
| `assets/table_objects/green_bowl_round/green_bowl_hq_mesh.usd` | USD of Poisson mesh | Failed quality |
| `tmp/sam3d_bowl_mesh.py` | SAM 3D mesh inference script | Working |
| `tmp/reconstruct_bowl_mesh.py` | Poisson reconstruction (failed approach) | Archived |
| `tmp/render_bowl_comparison.py` | Isaac Sim headless render script | Working |

## Scene State

Stage `isaac_sim_stage1.usd` references `green_bowl_sam3d.usd` (mesh bowl with collision proxy). Mesh participates in standard rasterized depth buffer — depth compositing against the desk NuRec volume is correct. Bowl is an independent prim with physics collision.

Original convex hull collision was replaced with concave `meshSimplification` so objects can physically sit inside the bowl instead of resting on an imaginary lid.

To revert to original NuRec splat: change `/World/Workspace/green_bowl` reference back to `green_bowl_with_proxy.usdz`.
